# Zetter Chat Tool Channel Separation 追加仕様書

関連:

- `chat-safe-tools-spec.md` は「何のツールを安全に許可するか」の仕様
- 本書は「そのツール呼び出しをどの経路で安全に扱うか」の仕様

## 目的

Zetter chat において、**ユーザー向け回答本文** と **内部 tool 呼び出し制御** を明確に分離し、以下を防ぐ。

- DSML / XML / 擬似 tool syntax のユーザー露出
- 複数 tool 呼び出し時の取りこぼし
- tool 実行途中での不完全回答
- tool 制御記法がそのまま会話履歴に保存される事故
- 「まず〜します」「取得します」のような**進行宣言だけで止まる**不完全終了
- 実際には段階遷移していても、ユーザーからは **`thinking` しか見えない** 観測性不足
- stream 開始前の失敗が **generic な request error** としてしか見えない問題

本仕様は、「本文と tool use は別チャネル」という設計思想を、Zetter chat の既存構成へ最小限で導入するためのものである。

---

## 背景

現状の `src/services/deepseek-chat.ts` は、DeepSeek の native `tool_calls` を第一経路として使っているが、補助的に **擬似 tool call fallback** を持っている。

現行構成:

1. 非 streaming completion で tool planning を試す
2. native `tool_calls` が来れば実行する
3. native `tool_calls` が無ければ、assistant 本文から `parsePseudoToolCall()` で 1 件だけ救済する
4. その後、最終回答を streaming で流す

この構成には以下の穴がある。

### 既知の不具合

1. `parsePseudoToolCall()` は **最初の `<invoke>` 1件しか拾えない**
2. `buildToolAugmentedMessages()` は **3 round 固定** で、複数 pseudo call に弱い
3. 最終 streaming 中は **DSML / XML / 擬似 tool syntax を無検査でユーザーに流す**
4. prompt では「DSML を出すな」と言っていても、**runtime で強制していない**
5. tool planning completion が **計画文だけ** を返しても `directAssistantText` として扱われ、そこで終了しうる
6. frontend の Thinking は **単一 `waitState` の上書き** なので、途中 phase が履歴として見えない
7. route / proxy / pre-stream 失敗は、状況によって **`リクエストに失敗しました`** に潰れて原因が見えない

結果として、モデルが

- native `tool_calls` ではなく
- 本文中に `<invoke ...>` 群を出し
- しかも複数件まとめて出す

と、最終的に **そのままユーザーへ露出** する。

---

## 目標

### Must

1. ユーザーは raw DSML / XML / 擬似 tool syntax を見ない
2. tool 呼び出しは **native `tool_calls` が唯一の正規経路**
3. pseudo tool syntax は **内部 recovery 専用**
4. 複数 pseudo call が来ても安全に処理できる
5. tool planning が終わるまで、ユーザー向け streaming へ移らない
6. 不正な tool syntax は会話履歴へ保存しない
7. 「これから調べます」のような進行宣言は、**final answer として確定しない**
8. 1 回の user request の内部フェーズが、**複数段階の進行表示**として見える
9. pre-stream failure / route failure / upstream failure を、UI で**種類別に識別**できる

### Should

1. 構造変更は主に backend に閉じる
2. 追加課金を増やさない
3. 既存の Zetter read-only tool 群はそのまま活かす
4. 「途中経過を出しつつ自走する」体験を、既存 SSE 上で再現する

### Non-goals

1. 多エージェント化
2. 外部オーケストレータ導入
3. ツール種類の拡大
4. frontend で XML を解釈すること
5. model の自由文そのものを途中メッセージとして無条件に都度表示すること

---

## 結論

Zetter chat では、次の 3 層構造を採用する。

### Layer 1: Native tool path

最優先。DeepSeek の `tool_calls` だけを正規経路とする。

### Layer 2: Internal recovery path

モデルが本文に擬似 tool syntax を出した場合のみ backend で回収する。  
この出力は **ユーザーには一切見せない**。

### Layer 3: User-visible answer path

最終回答本文だけを streaming する。  
この経路では **tool syntax を流さない**。

---

## 採用方針

## 1. 「本文」と「tool 制御」を別物として扱う

以下を厳密に分離する。

- assistant final answer
- assistant tool planning message
- tool result message
- internal recovery state

ユーザーに見せてよいのは **final answer だけ**。

---

## 2. Pseudo tool syntax は「正規機能」ではなく「事故回収」

重要:

- prompt に「DSML を出すな」と書くのは補助策
- 本質は **出ても漏れない runtime** を持つこと

そのため pseudo tool syntax は:

- model 機能としては扱わない
- backend recovery のみで使う
- recovery 失敗時はユーザーへ露出せず、再試行か簡潔な失敗応答に落とす

---

## 3. 最終 streaming の前に tool planning を完了させる

今後の基本フロー:

1. user message を組み立てる
2. 非 streaming completion で tool planning を行う
3. native `tool_calls` があれば実行
4. 無ければ pseudo syntax recovery を試す
5. tool planning が完了したら、**tools 無し**で final answer streaming を開始
6. final stream で tool syntax が出たら表示せず recovery / retry する

つまり、tool planning と final answer streaming を **フェーズ分離** する。

---

## 4. 「進行宣言」と「最終回答」を別物として扱う

ここで重要なのは、**「何をするか言って止まる」のではなく、やるところまで自走する**ことである。  
Zetter chat でも、以下を runtime ルールとして導入する。

### 原則

- 「まず〜を確認します」
- 「最初のステップとして〜を取得します」
- 「その後、〜を分析します」

のような文は、**途中経過または計画文**であり、原則として final answer ではない。

### final answer にしてよいもの

1. 具体的な調査結果・比較結果・要約結果を含む
2. 必要な tool 実行が既に終わっている
3. できない依頼に対する明示的な capability explanation / fallback
4. 雑談や簡単な説明など、そもそも追加調査が不要なケース

### final answer にしてはいけないもの

1. 未来形の作業宣言が主内容
2. 「まず」「次に」「最初のステップ」などの段取り説明のみ
3. 「調べます」「取得します」「確認します」で終わる
4. Zetter 調査質問なのに、まだ tool result が会話文脈へ追加されていない

### 判定方針

`directAssistantText` として確定する前に、**planning-draft classifier** を通す。

- 入力:
  - 最新 user request
  - assistant draft
  - その時点までの tool result 有無
  - tool planning round 数
- 出力:
  - `final-answer`
  - `planning-draft`
  - `fallback-answer`

`planning-draft` 判定なら、その本文はユーザーへ出さず、内部継続フェーズへ戻す。

---

## 実装仕様

## A. Pseudo tool parser の複数件対応

### 現状

- `parsePseudoToolCall(content)` は 1 件しか返さない

### 変更後

- `parsePseudoToolCalls(content)` に変更
- 返り値は `PseudoToolCall[]`
- `<tool_calls>...</tool_calls>` ラッパの有無に依存しない
- `<invoke name="...">...</invoke>` を **全件抽出**
- 1件も正常抽出できなければ空配列

### 正規化

- `normalizePseudoToolCall()` は維持
- 呼び出し側で全件 map/filter する
- unknown tool は捨てる
- known tool のみ recovery 対象とする

---

## B. Recovery 実行の複数件対応

### 現状

- pseudo call は 1 件だけ `runToolCalls()` に流す

### 変更後

- recovery 対象が複数件なら、**native tool_calls 相当の配列**へ変換してまとめて `runToolCalls()` に渡す
- 1 turn で複数 tool を処理できるようにする

### 制限

- 1 recovery turn あたり max 8 calls
- それ以上はエラーにするか先頭 8 件まで
- 同一 tool 同一引数の重複は dedupe 可

---

## C. Tool augmentation loop の終了条件見直し

### 現状

- `for (round = 0; round < 3; round += 1)`

### 問題

- 複数 pseudo call + 最終回答で 3 round では足りない

### 変更後

- `MAX_TOOL_AUGMENT_ROUNDS = 6` 程度へ拡張
- ただし無限ループ防止のため、以下で打ち切る:
  1. 同一 pseudo call セットが連続した
  2. 実行 tool 数が上限を超えた
  3. final answer が得られた
  4. 明確なエラー応答が必要

### 追加 state

- `seenToolCallFingerprints: Set<string>`
- fingerprint は `tool name + normalized args`

同じ要求が再帰的に繰り返されたら、再実行せず fail-safe に落とす。

---

## D. Final streaming に tool-syntax guard を入れる

### 現状

- SSE delta をそのまま `assistantText += delta`
- `enqueue("delta", { text: delta })`

### 問題

- ここで DSML が来ると即露出する

### 変更後

- final streaming では **guarded buffer** を使う
- 小さいチャンクを即表示せず、一時バッファに貯める
- 以下の疑わしいパターンを検出したら **表示停止**

対象例:

- `<...invoke`
- `<...tool_calls`
- `<...parameter`
- `DSML`
- XML 風 closing tags

### 動作

1. 安全と判断できる文字列だけを flush
2. 危険パターンを検出したら stream を中断
3. 中断後は backend 内で recovery モードへ移行
4. recovery 成功なら改めて final answer を再取得
5. recovery 失敗ならユーザーには簡潔なエラーだけ返す

重要:

- **疑わしい文字列は 1 文字も表示しない**

---

## E. Final answer request では tools を渡さない

tool planning 完了後の final answer streaming request は:

- `messages = tool-augmented messages`
- `stream = true`
- **`tools` は渡さない**

これにより、final answer 段では native `tool_calls` を基本的に発生させない。

ただしモデルが本文で擬似 syntax を出す可能性は 0 ではないため、D の guard は必要。

---

## F. Recovery retry policy

### 方針

- final stream で tool syntax を検出したら 1 回だけ内部 retry

### retry prompt 追加メッセージ

system 追加:

- You already have the necessary tool results.
- Do not emit tool syntax, XML, DSML, invoke tags, or function-like markup.
- Respond only with the final user-facing answer in plain Japanese prose.

### 制限

- retry は max 1 回
- retry でも漏れるなら失敗応答へ

---

## G. 保存前ガード

assistant 応答を DB 保存する前に、最終テキストへ以下を適用する。

### ルール

- DSML/XML/pseudo tool syntax を含む場合は **保存しない**
- `insertLegacyDevChatExchange()` / `insertThreadedDevChatExchange()` は、保存前の検証を通過した応答だけ受け入れる

### 失敗時

- ユーザー向けには「内部処理に失敗しました。もう一度試してください。」系の短文
- raw syntax は保存しない

---

## H. 観測性

追加ログを入れる。

### ログ項目

1. native tool_calls count
2. pseudo recovery count
3. pseudo recovery parsed call count
4. final stream guard triggered
5. retry used
6. save blocked by syntax guard

### 目的

- 再発頻度の把握
- DeepSeek モデル/モード差の把握
- prompt 調整の効果測定

ログは内部用であり、ユーザーへ返さない。

---

## I. Prompt 側の扱い

prompt には引き続き以下を明記する。

1. native tool calls のみ使う
2. DSML/XML/手書き function syntax を出さない
3. final answer は plain prose only

ただし prompt は **補助策** であり、主対策ではない。

本質は runtime 側で

- 漏らさない
- 保存しない
- 再試行する

ことにある。

---

## J. Autonomous continuation loop

### 目的

1 回の user request を、**内部では複数ステップに分けて最後まで完了させる**。

### 新しい内部 state machine

1. `analyze-request`
2. `plan-tools`
3. `run-tools`
4. `integrate-results`
5. `classify-draft`
6. `continue-or-finalize`
7. `stream-final-answer`
8. `save-final-answer`

### 重要ルール

- `classify-draft = planning-draft` の場合、**request は終わっていない**
- backend は追加の planning / tool / final-answer request を継続する
- 途中でユーザーへ見せるのは **progress event** であり、モデルの計画文そのものではない

### 継続時の追加 system 指示

計画文で止まりかけた場合、次のような意図を追加で与える。

- 段取り説明だけで止まらないこと
- 必要なら今すぐ tool を呼ぶこと
- すでに十分な文脈があるなら最終回答を返すこと
- 「これから何をするか」ではなく「何を見つけたか」を述べること

### 上限

- continuation retry は 2 回程度まで
- それでも planning-draft のままなら、failure ではなく
  - 「今は最後まで完了できなかった」
  - 「できる範囲」
  を説明する fallback へ落とす

---

## K. User-visible progress channel を追加する

### 背景

現状の `status` は単一 `waitState` を上書きするだけなので、ユーザーからは  
**「質問と会話の流れを整理しています。」しか見えない** 状態になりやすい。

### 方針

ユーザー向け進行表示を 2 層に分ける。

1. **current status**
   - 今この瞬間の phase
   - 従来の `status` 相当
2. **progress timeline**
   - この request 内で何段階進んだかの履歴
   - 上書きせず積み上げる

### 新規 SSE event

`progress`

payload 例:

```json
{
  "phase": "tool-call",
  "title": "mizuki の交流相手を確認中",
  "detail": "返信先や最近のやり取りを集めています。",
  "tools": [{ "name": "get_user_interaction_summary", "label": "交流要約" }],
  "items": [
    "まず mizuki の交流頻度を確認",
    "候補ユーザーのプロフィールを比較"
  ],
  "kind": "checkpoint"
}
```

### 表示ルール

- `status` は現在動作中のラベルとして使う
- `progress` はチャット UI 内にタイムラインとして蓄積する
- 1 request につき最大 8 件程度まで
- 最終回答後は timeline を残してよい

### 生成方針

progress 文言は、**backend の phase / tool 実行内容から組み立てる**。  
model の自由文をそのまま途中表示には使わない。

---

## L. pre-stream failure を structured error にする

### 背景

現状は stream 開始前の失敗が non-2xx response になり、frontend では

- payload に `error` がなければ
- `リクエストに失敗しました (status)`

となりやすい。

### 方針

失敗を次の 4 類型に分ける。

1. `validation_error`
2. `availability_error`
3. `upstream_error`
4. `internal_error`

### route 仕様

- request validation / auth / feature availability のような明確な 4xx は、そのまま JSON で返してよい
- それ以外の **chat runtime 内部失敗** は、可能なら SSE を開始して fallback 文まで返す
- どうしても SSE を開始できない場合も、JSON payload に
  - `errorCode`
  - `error`
  - `requestId`
  を含める

### frontend 仕様

- `errorCode` があれば generic 文言で潰さず、その種別に応じた表示にする
- requestId があれば toast / debug 表示へ含める

---

## M. frontend Thinking UI を timeline 型へ拡張する

### 現状

- `waitState` は 1 個だけ
- 新しい phase が来るたびに上書き
- `writing` / `saving` は一瞬で消える

### 変更後

- widget state に `progressEntries` を追加
- `status` と `progress` を別管理
- streaming 中の assistant bubble に
  - current status
  - progress timeline
  - tool chips
  - step items
  を描画する

### UX 目標

- 「考え中」だけでなく、
  - 調査計画
  - 情報取得
  - 結果整理
  - 回答整形
  が順に見える
- 「段階的に進んでいる感覚」を出す

### 注意

- 過剰にうるさくしない
- 同じ phase の重複進捗は dedupe する
- timeline 文言は簡潔にする

---

## N. ログ / request tracing

追加する観測情報:

1. requestId
2. route classification (`validation_error` / `upstream_error` / `internal_error`)
3. planning-draft 判定回数
4. continuation retry 回数
5. progress event emitted count
6. final fallback reason

目的:

- 「どこで止まったか」を後追いできるようにする
- `リクエストが失敗しました` の原因を本番で追えるようにする

---

## 対象ファイル

主対象:

- `src/services/deepseek-chat.ts`
- `public/chat-widget.js`
- `public/styles.css`
- `src/routes/chat.ts`

補助:

- `src/services/zyn-system-prompt.ts`

変更不要見込み:

- `src/services/zetter-chat-tools.ts`
- DB schema

---

## 実装フェーズ

## Phase 1: Hotfix

最優先:

1. `parsePseudoToolCalls()` へ複数件対応
2. `buildToolAugmentedMessages()` の round 拡張
3. final stream guard 追加
4. 保存前ガード追加

効果:

- raw DSML 流出を即止める

## Phase 2: Structural cleanup

1. tool planning と final streaming の責務分離を明確化
2. recovery retry policy 導入
3. fingerprint ベースの loop 防止

効果:

- 再発しにくい構造にする

## Phase 3: Observability

1. ログ/カウンタ追加
2. モデル別の発生傾向を確認
3. 必要なら prompt を微調整

## Phase 4: Continuation runtime + timeline UX

1. planning-draft classifier 導入
2. autonomous continuation loop 導入
3. `progress` event 追加
4. frontend timeline 表示
5. structured error / requestId 表示

---

## 受け入れ条件

1. `mizukiと相性の良さそうなユーザートップ3` 系の質問で raw DSML が画面に出ない
2. 複数 pseudo tool call が出ても内部 recovery できる
3. final streaming 中に DSML パターンが出た場合、ユーザーには見えず retry または失敗応答になる
4. raw DSML を含む assistant message は DB に保存されない
5. 既存の native tool_calls 正常系は壊れない
6. 追加課金は実質 DeepSeek の retry 1 回以内に収まる
7. 「まず最初に〜を取得します」のような計画文だけでは request が完了しない
8. 調査系の質問では、UI に少なくとも 2 段階以上の progress が見える
9. `質問と会話の流れを整理しています。` 以外の phase も、timeline 上で確認できる
10. pre-stream failure 時に generic error だけで終わらず、原因分類または requestId が取得できる
11. follow-up で「もう一度おねがい」と送っても、途中計画から自然に継続できる

---

## 最終方針

Zetter chat は、

- **何を読めるか** は `chat-safe-tools-spec.md` で制限し
- **どう tool を呼ぶか** は本書で制御する

二層構造で守る。

Prompt で祈るのではなく、
**runtime で本文と tool 制御を分離し、漏れを防ぐ** のが最終方針である。
