# Zetter Chat Harness V2 仕様書

関連:

- 現行 runtime: `src/services/deepseek-chat.ts`
- 現行 tool 群: `src/services/zetter-chat-tools.ts`
- 現行 system prompt: `src/services/zyn-system-prompt.ts`
- 既存の tool channel 分離仕様: `files/chat-tool-channel-spec.md`

---

## 目的

 一般的な **coordinator / direct-answer-first / phased orchestration** の考え方を、**DeepSeek flash 固定**のまま Zetter chat に取り込み、以下を同時に満たす。

1. 質問ごとに最適な処理経路へ入る
2. 不要な tool planning / retry / 再生成を減らす
3. DB 取得件数を必要以上に絞らず、回答の質を落とさない
4. output token と API 呼び出し回数を抑える
5. progress 表示を backend の実際の段階に対応させる

本仕様は **包括版** とし、backend harness、tool 実行方針、prompt 分離、progress phase、計測、テストまで含む。

---

## 背景

現行の `deepseek-chat.ts` は、tool safety と継続性を高める修正を積み上げており、破綻しにくさは上がっている。一方で、次の点が token 消費と実行品質の両面でボトルネックになっている。

### 現状の主要課題

1. **tool 経路に入りすぎる**
   - `shouldUseZetterTools(messages)` が実質 `messages.length > 0` なので、ほぼ常に tool planning へ入る
   - 雑談や短い説明も tool 定義つき prompt になりやすい

2. **同じ request の中で役割が混ざっている**
   - 現行は 1 系列の flow の中で、分類、方針決定、tool 実行、最終回答生成、guard retry を順に兼務している
   - 「何をすべきか決める役」と「実際に答える役」が明確に分かれていない

3. **tool augmentation loop が高コスト**
   - `MAX_TOOL_AUGMENT_ROUNDS = 6`
   - `MAX_FINAL_DRAFT_RETRIES = 2`
   - planning-draft / pseudo tool recovery / stream guard retry が積み重なると、1 request あたりの内部 call 数が膨らむ

4. **履歴が毎回太い**
   - `DEV_CHAT_HISTORY_LIMIT = 40`
   - tool planning と final answer の複数 call で同じ履歴を繰り返し送る

5. **tool 出力が generic で重い**
   - `executeZetterChatTool()` は各 tool の返り値をそのまま `JSON.stringify()` して返す
   - 一部 tool は `activityToToolPost()` ベースの大きな payload を返す
   - planner / finalizer が本当に必要な形に整えられていない

6. **system prompt が tool 使用を広く促しすぎる**
   - `zyn-system-prompt.ts` は「Zetter 投稿やユーザー等なら tool を使う」を強めに教える
   - しかし runtime 側で「使わなくてよい質問」を先にふるい分けていない

7. **progress はあるが、phase は実装都合寄り**
   - 現在の UI は current-only で適切
   - ただし phase は「内部 retry を何度も回している」ことを自然には表現していない

---

## 設計方針

本仕様で採る考え方は次の通り。

### 1. Coordinator の責務を明確に分ける

coordinator は

- 直接答えられるなら答える
- 必要な時だけ worker を使う
- 調査結果を理解してから次の指示を書く

という役割になっている。

Zetter chat では multi-agent 化はしないが、**request の先頭で「何をすべきか決める小さな司令役」を置く**、という思想はそのまま使える。

### 2. 役割分離の質が、モデル切替より重要

ここでは、model routing そのものよりも

- research
- synthesis
- implementation
- verification

の責務分離が中心だった。  
本仕様でも、**まず orchestration を正す**。常時 pro に逃がす設計は採らない。

### 3. 直接答えられるものは直答する

設計原則として「答えられるなら委譲しない」を採る。  
Zetter chat ではこれを **direct-answer-first** として採用する。

### 4. 「based on your findings」を禁止する発想を runtime に移す

設計上、理解の丸投げを禁じる。  
Zetter chat ではこれを、

- planner は route だけ返す
- executor は DB と tool を実行する
- finalizer は実データだけで答える

という **中間表現の厳格化**で再現する。

---

## 結論

次期 harness は **flash-only coordinator pipeline** にする。

### 採用方針

1. **モデルは `deepseek-v4-flash` + `instant` 固定のまま**
2. request の先頭に **小さな router call** を追加する
3. router は **DB 生データを見ない**
4. tool 実行は router が返した route/budget に従う
5. final answer は原則 1 回で終える
6. deterministic に返せるものは model を通さず返す
7. tool 結果は「そのまま JSON を投げる」のではなく、**役割別に整形**してから後段へ渡す

---

## Non-goals

1. multi-agent 化
2. `deepseek-v4-pro` 常時利用
3. 外部 orchestrator 導入
4. frontend の大幅 redesign
5. 全 tool の完全作り直し

---

## 新アーキテクチャ

## 全体像

1. **Gate**
   - deterministic に危険/不要/簡易ケースを切る
2. **Router**
   - 小さい flash call で route を JSON 化する
3. **Executor**
   - route に応じて tool / DB / image analysis を実行する
4. **Synthesizer**
   - 必要なら final answer を生成する
5. **Guard**
   - 出力長、tool syntax 混入、保存を処理する

---

## Layer 0: Deterministic Gate

最初に model を呼ぶ前に、backend で次を判定する。

### Gate categories

1. `boundary-refusal`
   - サーバーファイル、ソース、env、秘密情報、SQL 直読など
2. `direct-chitchat`
   - あいさつ、短い雑談、軽い説明
3. `direct-template`
   - DB 集計だけで定型返答できる質問
4. `needs-router`
   - Zetter データ参照、比較、要約、画像、曖昧な質問

### Must

1. `shouldUseZetterTools()` を廃止し、Gate 結果で tool 使用可否を決める
2. system prompt の「Zetter 話題なら必ず tool」依存を減らす

---

## Layer 1: Flash Router

router は **小さい flash completion** でよい。  
ここでは user 向け文章を書かせず、JSON だけ返させる。

### Router input

1. 最新 user message
2. 直近の短い履歴要約
3. 画像有無
4. 利用可能な tool 名一覧
5. backend 制約

### Router input から外すもの

1. 生の tool 結果
2. 長い会話履歴 40 件
3. 大きな DB JSON
4. 返信文のスタイル細部

### Router output contract

```json
{
  "route": "direct-answer | direct-template | zetter-lookup | zetter-summary | zetter-compare | timeline-trend | image-analysis | boundary-refusal",
  "needsTools": true,
  "toolPlan": [
    { "name": "search_posts", "intent": "keyword search", "priority": "high" }
  ],
  "depth": "light | standard | deep",
  "answerProfile": "short | standard | evidence-first | ranked",
  "allowDirectFinalWithoutModel": false,
  "reason": "short internal reason"
}
```

### Router responsibilities

1. 何を調べるか決める
2. どの程度まで深掘るか決める
3. 最終回答を短く返すべきか決める
4. model を通さず template で返せるか判定する

### Router non-responsibilities

1. DB を読む
2. 最終回答本文を書く
3. tool result を要約する

---

## Layer 2: Execution Plan

router の JSON を受けて、backend が **Execution Plan** を作る。

### Execution Plan fields

1. `route`
2. `retrievalBudget`
3. `toolCalls`
4. `resultShape`
5. `finalizerMode`
6. `progressPhasePlan`

### Retrieval budget

ユーザーの意向に合わせ、**DB 側の取得上限は現行より大きくしてよい**。  
ただし「どれだけ取るか」と「何を後段へ渡すか」は分ける。

#### 例

| route | DB scan | model に渡す形 |
|---|---:|---|
| zetter-lookup | 20-80件 | 該当投稿 + 代表例 |
| zetter-summary | 50-200件 | summary + highlights |
| zetter-compare | 60-240件 | counts + topics + recent examples |
| timeline-trend | 100-300件 | counts + topics + highlights |
| image-analysis | 画像結果 + 必要最小限の Zetter 情報 | structured evidence |

ここで重要なのは、**DB 取得件数は広げても、後段の役割に合わせて形を変える**こと。

---

## Layer 3: Tool Policy

現行の tool 群は残しつつ、**返却形状と route との対応**を見直す。

### 基本方針

1. list 系 tool は「証拠提示向け」
2. summarize / stats / topics 系 tool は「要約向け」
3. finalizer が欲しい形に近い tool を優先する
4. route と不整合な tool は backend 側で抑制する

### 重要変更

1. **tool の limit は model 任せにしない**
   - `args.limit` は参考値
   - 最終的な取得量は `Execution Plan.retrievalBudget` が決める

2. **tool 結果は profile 別に整形する**
   - `raw-posts`
   - `summary-material`
   - `aggregate-only`
   - `evidence-pack`

3. **重い full post payload を減らす**
   - `activityToToolPost()` の完全体は必要時だけ
   - 通常は `summarizeToolPost()` 相当の軽量表現を基本にする

### 推奨方針

#### A. 既存 tool 名は維持

互換性を優先し、まずは tool 名を残す。

#### B. 内部結果だけ profile 化

`executeKnownZetterTool()` の最終返却前に、route/profile に応じて

- summary
- highlights
- sampled posts
- counts
- topics

を組み合わせる。

#### C. 生 JSON 全量返却をやめる

`executeZetterChatTool()` は `JSON.stringify(result)` の前に、**ToolEnvelope** を通す。

```json
{
  "tool": "search_posts",
  "profile": "summary-material",
  "query": "最近のゼッターの様子",
  "meta": {
    "scanned": 120,
    "returned": 12
  },
  "summary": {
    "topAuthors": [],
    "topics": [],
    "highlights": []
  },
  "records": []
}
```

### ルール

1. model に渡す raw records は必要最小限
2. `meta.scanned` は残す
3. `summary` は常に残す
4. `records` は route に応じて 0-12 件程度に抑える

---

## Layer 4: Direct Template Path

次のような質問は、tool 実行後も final model call を省略できる。

1. 件数確認
2. 単純ランキング
3. 期間集計
4. 「最近誰が多い？」のような短い定型集約

### 方針

1. backend で日本語テンプレートを持つ
2. route が `direct-template` のときは finalizer を呼ばない
3. これにより output token と 1 call を削減する

---

## Layer 5: Finalizer

finalizer は **最終回答専用** とする。

### Finalizer input

1. user message
2. 短い会話履歴要約
3. route
4. ToolEnvelope 群
5. answerProfile

### Finalizer rules

1. 原則 1 call
2. output は短く制約する
3. raw JSON の全文列挙を禁止
4. post 引用は上限を設ける

### Output policies

1. `short`
   - 結論 2-4 行
2. `standard`
   - 短い段落 + 箇条書き 3-5 件
3. `evidence-first`
   - 結論 + 根拠 2-3 件
4. `ranked`
   - 上位 N 件 + ひとこと理由

### Retry policy

1. `MAX_FINAL_DRAFT_RETRIES` は 2 から **1** へ下げる
2. planning-draft は finalizer では原則発生させない
3. それでも崩れたら template fallback へ落とす

---

## Prompt 分離方針

現行の 1 つの system prompt に全責務を載せるのではなく、role ごとに prompt を分ける。

### 1. Router prompt

内容:

1. route を決めるだけ
2. JSON 以外を返さない
3. 直接答えられる/定型返答できる時はそれを選ぶ
4. tool は必要最小限を選ぶ

### 2. Finalizer prompt

内容:

1. ToolEnvelope を信頼する
2. 必要以上に長く書かない
3. 全件列挙しない
4. 結論を先に出す

### 3. Boundary/template messages

これらは model 非依存の backend 定型文に寄せる。

---

## Loop / Retry / Guard 見直し

### 現行からの変更

1. `shouldUseZetterTools()` 廃止
2. `MAX_TOOL_AUGMENT_ROUNDS` を **6 -> 2** を基本にする
3. `deep route` のみ最大 3
4. pseudo tool recovery は維持するが、事故回収専用
5. finalizer retry は 1 回まで

### Guard 方針

1. DSML / XML / pseudo tool syntax 混入は引き続き禁止
2. 長すぎる output は backend で短縮再生成ではなく template fallback も検討する
3. 「まず〜します」で終わる文は final answer にしない

---

## 会話履歴の扱い

### 現行問題

40 件履歴が planner と finalizer に繰り返し入る。

### 変更後

1. **router 用履歴**
   - 直近 4-8 message 相当の短い要約
2. **finalizer 用履歴**
   - 直近の user intent と必要最小限の文脈
3. **tool 実行用**
   - 原則、履歴ではなく route / latest request / tool args を中心にする

### 実装方針

`fetchThreadedDevChatMessages()` の生配列をそのまま downstream 全部に渡さず、

1. `buildRouterContext()`
2. `buildFinalizerContext()`

を新設する。

---

## Progress Phase の再定義

現在の current-only UI は維持し、backend phase だけ整理する。

### 推奨 phase

1. `thinking`
   - 入力理解 / gate 判定
2. `tool-planning`
   - router / execution plan 作成
3. `tool-call`
   - DB / tool 実行
4. `tool-result`
   - 結果整形
5. `writing`
   - finalizer or template shaping
6. `saving`
   - 保存

### phase copy 方針

visible title は現行の短いひらがな系を維持してよい。

例:

- `thinking`: `りかいちゅう`
- `tool-planning`: `さくせんちゅう`
- `tool-call`: `しらべちゅう`
- `tool-result`: `せいりちゅう`
- `writing`: `へんじちゅう`
- `saving`: `ほぞんちゅう`

### 注意

router と finalizer retry をユーザーに細かく見せすぎない。  
phase は **ユーザーが暇にならない程度**に保ち、内部 loop の細部は露出しない。

---

## 計測 / 観測性

現行の `dev_chat_usage_records` を拡張活用する。

### 追加で記録すべきもの

1. request 単位 route
2. gate category
3. tool call 数
4. deep route かどうか
5. finalizer を呼んだか
6. template path を使ったか
7. retry 回数
8. total output chars

### 推奨テーブル

`dev_chat_request_summaries`

例:

- `request_id`
- `user_id`
- `route`
- `gate_result`
- `tool_call_count`
- `augment_rounds`
- `finalizer_calls`
- `template_used`
- `prompt_tokens`
- `completion_tokens`
- `total_tokens`
- `completed_at`

### 目的

1. 「無駄に tool に入った」割合を見る
2. 「direct-template で救えた」割合を見る
3. route ごとの平均 token を見る
4. regression を deploy 後に確認する

---

## 実装対象ファイル

### 必須

1. `src/services/deepseek-chat.ts`
   - gate / router / execution plan / finalizer 分離
   - history context builder
   - retry policy 見直し

2. `src/services/zetter-chat-tools.ts`
   - ToolEnvelope 導入
   - route/profile ベースの返却形状調整
   - retrieval budget 上書き対応

3. `src/services/zyn-system-prompt.ts`
   - monolithic prompt を縮小
   - router/finalizer 用 prompt 分割

4. `schema.sql`
5. `schema.postgres.sql`
   - request summary 系テーブル追加時に更新

### できれば

1. `public/chat-widget.js`
   - phase 名追加が必要なら最小限で追従
2. `/tmp/zetter-chat-ui-test/tests/*.spec.js`
   - phase / final behavior の回帰確認

---

## 段階的 rollout

### Phase 1

1. gate を導入
2. `shouldUseZetterTools()` を削除
3. direct-template path を追加

### Phase 2

1. flash router を導入
2. route を usage に記録
3. finalizer 1 回化

### Phase 3

1. ToolEnvelope 化
2. route/profile ごとの返却形状調整
3. retrieval budget を route 管理へ移行

### Phase 4

1. token 集計ダッシュボード的な SQL を整備
2. heavy route のみ微調整

---

## 受け入れ条件

次の条件を **すべて**満たした場合のみ受け入れとする。

1. 雑談・軽い説明で tool planning に入らない
2. direct-template 対象質問で finalizer を呼ばない
3. 長い Zetter 質問でも回答の具体性が落ちない
4. progress 表示が current-only のまま自然
5. DSML / pseudo tool syntax は引き続き露出しない
6. route / gate / call 数を request 単位で後追いできる
7. 本番 deploy 後の確認で、以下の実測条件をすべて満たす

### 実測条件

#### A. 雑談 request

対象例:

- `こんにちは`
- `おはよう`

満たすべき条件:

1. `dev_chat_request_summaries.route` が `direct-answer` または `direct-chitchat`
2. `tool_call_count = 0`
3. `finalizer_calls <= 1`
4. `augment_rounds = 0`
5. `dev_chat_usage_records.stage` に `tool-augment-round-*` が出ない

#### B. direct-template request

対象例:

- `最近よく投稿してる人は？`
- `今日の投稿数はどれくらい？`

満たすべき条件:

1. `dev_chat_request_summaries.route = 'direct-template'`
2. `template_used = true`
3. `finalizer_calls = 0`
4. 回答が短い定型文として完了する
5. `tool_call_count >= 0` は許容するが、tool 結果の全文列挙が出ない

#### C. Zetter summary request

対象例:

- `最近のゼッターの様子をおしえて`

満たすべき条件:

1. `route` が `zetter-summary` または同等の summary 系 route
2. `finalizer_calls = 1`
3. `augment_rounds <= 2`
4. `tool_call_count >= 1`
5. 回答は「結論 + 要点」の短い形に収まり、raw JSON 全文ではない

#### D. Progress UI

満たすべき条件:

1. progress は処理中に current-only で見える
2. 完了後 assistant bubble に progress timeline が残らない
3. 既存の `〜ちゅう` 表示が崩れない
4. UI に model/thinking 切替は出ない

#### E. Safety

満たすべき条件:

1. user-facing final text に DSML / XML / pseudo tool syntax が含まれない
2. tool failure 時も raw tool JSON をそのまま露出しない
3. boundary request で server file / env / secret / arbitrary SQL を読まない

---

## 推奨テスト

### Unit

1. gate classification
2. router JSON parse
3. route -> tool policy mapping
4. template formatter
5. planning-draft guard

### Integration

1. 雑談
2. 軽い説明
3. ユーザー検索
4. 投稿検索
5. 今日の様子要約
6. 比較質問
7. 画像 + Zetter 質問
8. tool failure fallback

### Browser

1. current-only progress 維持
2. final bubble に progress が残らない
3. 送信から完了まで phase が自然に変わる

---

## 実装完了の定義

この仕様における「実装完了」は、**コードを書き終えた時点ではなく、deploy と本番確認まで完了した時点**を指す。

### 実装完了とみなす条件

次の条件を **すべて**満たした場合のみ、本タスクは完了とする。

1. 本仕様で定義した code change が `src/services/deepseek-chat.ts`、`src/services/zetter-chat-tools.ts`、`src/services/zyn-system-prompt.ts`、必要なら schema / frontend test に反映されている
2. local validation が通る
3. 必要な deploy が完了している
4. 本番環境で実際に chat を動かし、受け入れ条件の実測条件 A-E を満たすことを確認できている
5. usage / summary 記録から route, gate, call 数, template 使用有無を追跡できる

### 完了ではない状態

次のいずれかに当てはまる場合、**実装完了ではない**。

1. local build/test は通るが、未 deploy
2. deploy は済んだが、本番 chat の実地確認をしていない
3. 本番で route / call 数 / template 使用有無を確認できない
4. 期待 route に入らず、雑談が tool planning に流れる
5. direct-template request で finalizer が呼ばれる
6. raw JSON / DSML / pseudo tool syntax が UI に出る
7. progress UI が壊れる
8. production browser test が落ちる

---

## 実行手順

この仕様だけで最後まで作業できるよう、実装から完了判定までの順序を固定する。

### Step 1: 実装

1. `deepseek-chat.ts` に gate / router / execution plan / finalizer 分離を入れる
2. `shouldUseZetterTools()` を削除するか、未使用状態にする
3. `zetter-chat-tools.ts` に ToolEnvelope と route/profile ベースの整形を導入する
4. `zyn-system-prompt.ts` を router/finalizer 向けに分離する
5. request summary 記録を追加する
6. schema 変更が必要なら `schema.sql` と `schema.postgres.sql` を更新する
7. 必要な unit / integration / browser test を追加または更新する

### Step 2: Local validation

最低限、次を通すこと。

1. `node --check public/chat-widget.js`  
   - frontend を触った場合は必須
2. `npm run typecheck`
3. `npm run build:cloudrun`
4. `/tmp/zetter-chat-ui-test` の relevant local tests
   - `tests/local-final-progress.spec.js`
   - `tests/local-live-status.spec.js`
5. 追加した unit/integration test

### Step 3: Deploy

#### 原則

1. `public/**` を触ったら **Workers を deploy**
2. `src/services/**` / `src/routes/**` / `src/schemas.ts` / `src/server.ts` を触ったら **Cloud Run を deploy**
3. schema を増やしたら、本番 Cloud SQL(Postgres) に必要な table/index があることを確認する

#### この仕様の default

本仕様の実装では backend 変更が必須で、progress / phase / browser verification にも影響しやすいため、**特別な理由がなければ Workers と Cloud Run の両方を deploy 対象とする**。

#### Deploy 後の必須確認

1. `https://z-etter.com/api/health` が 200
2. 公開 asset hash が最新変更を指す
3. Cloud Run latest ready revision が更新されている

---

## 本番確認手順

deploy 後は、次の順で **本番確認**を行う。

### 1. Production browser test

`/tmp/zetter-chat-ui-test/tests/prod-chat-progress.spec.js` を実行し、pass すること。

これが fail した場合、**実装完了ではない**。

### 2. 実チャット確認

本番 `/chat` で、少なくとも次の 3 種類の request を送る。

1. 雑談 request
   - 例: `こんにちは`
2. direct-template request
   - 例: `最近よく投稿してる人は？`
3. Zetter summary request
   - 例: `最近のゼッターの様子をおしえて`

### 3. 本番 DB / usage 確認

上記 request について、`request_id` を特定し、`dev_chat_usage_records` と `dev_chat_request_summaries` を確認する。

確認すべきこと:

1. 雑談 request:
   - `tool_call_count = 0`
   - `augment_rounds = 0`
   - `tool-augment-round-*` usage row が無い
2. direct-template request:
   - `route = direct-template`
   - `template_used = true`
   - `finalizer_calls = 0`
3. Zetter summary request:
   - summary 系 route
   - `finalizer_calls = 1`
   - `augment_rounds <= 2`

### 4. UI / output 確認

各 request について、次を確認する。

1. current-only progress が出る
2. 完了後に progress timeline が残らない
3. raw JSON / DSML / pseudo tool syntax が出ない
4. model/thinking 切替 UI が出ない

---

## 継続修正ルール

本仕様の重要条件:

**どれか 1 つでも未達なら、実装は未完了である。**

未達時は次のループを回すこと。

1. fail した条件を特定する
2. その条件を満たすためにコードを修正する
3. affected tests を再実行する
4. 再 deploy する
5. 本番確認を再実行する

### 明示ルール

#### 雑談が tool planning に入った場合

gate / router を修正し、**雑談 request が direct-answer / direct-chitchat で完了するまで修正を続ける**。

#### direct-template request で finalizer が呼ばれた場合

template path と route 判定を修正し、**`finalizer_calls = 0` になるまで修正を続ける**。

#### Zetter summary request で augment round が多すぎる場合

router の toolPlan、ToolEnvelope、retry policy を修正し、**`augment_rounds <= 2` になるまで修正を続ける**。

#### raw JSON / DSML / pseudo tool syntax が見えた場合

guard / finalizer / ToolEnvelope を修正し、**本番画面で見えなくなるまで修正を続ける**。

#### production browser test が落ちた場合

UI / progress / stream / phase 実装を修正し、**`tests/prod-chat-progress.spec.js` が pass するまで修正を続ける**。

#### route / call 数が DB で追えない場合

summary 記録実装と schema を修正し、**本番 request ごとに route, gate, tool_call_count, augment_rounds, finalizer_calls, template_used が取れるまで修正を続ける**。

---

## 最終完了判定

次の文を機械的な完了判定とする。

> Workers / Cloud Run / 必要な schema が deploy 済みであり、本番 `/chat` に対して雑談 request、direct-template request、Zetter summary request を実行した結果、`dev_chat_request_summaries` と `dev_chat_usage_records` で route / gate / tool_call_count / augment_rounds / finalizer_calls / template_used を確認でき、さらに production browser test が pass し、UI に raw JSON / DSML / pseudo tool syntax / 完了後 progress timeline / model-thinking switcher が表示されないことを確認できた場合、実装完了とする。

> 上記のいずれかを確認できない場合、または条件を満たさない場合、完了とはみなさず、条件を満たすまで実装の修正、deploy、再確認を続ける。

---

## 最終判断

現時点で最も費用対効果が高いのは **pro を足すことではなく、flash の前段に coordinator 的な routing を導入すること**である。

つまり、

1. **まず gate**
2. **次に小さな flash router**
3. **その後に tool 実行**
4. **最後に短い finalizer**

という順で、役割分離を flash-only で再構成する。  
これが、**質の高いタスク遂行** と **token 消費の節約** を最も両立しやすい。
