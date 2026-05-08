# Zetter Chat Eval Harness / Deterministic 化 仕様書

関連:

- 現行 runtime: `src/services/deepseek-chat.ts`
- 現行 tool 群: `src/services/zetter-chat-tools.ts`
- 現行 system prompt: `src/services/zyn-system-prompt.ts`
- 既存包括仕様: `files/zetter-chat-harness-v2-spec.md`
- 直近の OSS/公開情報評価: `research/ai-oss.md`（保存対象）

---

## 目的

Zetter の AI チャットを、**「なんとなくうまく動く」状態から「壊れても検知でき、事実系は安定して返る」状態へ上げる**。

本仕様の主目的は次の 2 点。

1. **Eval Harness の導入**
   - prompt / tool / model / routing を変えても、品質が落ちたら自動で気づけるようにする
2. **Deterministic 化の導入**
   - 日付、件数、最新投稿、ランキング骨格など、**事実として固定できる部分は LLM に委ねずコードで固定**する

この 2 本を先に入れることで、今後の router 改善、thinking 調整、tool 追加を安全に進められるようにする。

---

## 背景

現行ハーネスは、Gate / Router / Tool Augmentation / Judge / Finalizer の分離があり、構造自体はよくできている。一方で、運用品質の面では次の弱点が残っている。

### 現状の主要課題

1. **品質の自動回帰検知がない**
   - いまは browser smoke test や目視確認が中心で、回答品質そのものの regression を自動で検出できない
   - 特に tool 選択ミス、日付捏造、返信文のトーン逸脱、安全系の拒否漏れが危険

2. **事実系の一部が Finalizer instruction 依存**
   - 「日付を発明しない」「最新投稿の時刻を現在時刻扱いしない」などは指示で抑えているが、deterministic ではない
   - 本番で日付誤りが出た実績がある

3. **定型質問の LLM 経路と deterministic 経路が混在している**
   - 一部は `buildDirectTemplateAnswer()` で $0 deterministic に返せている
   - しかし phrasing が少しずれると通常 LLM 経路へ落ち、品質とコストが揺れる

4. **変更の安全弁が弱い**
   - selective Thinking や tool profile 改善など、良い変更も継続的に入っている
   - それを支える評価基盤がないため、後戻りしづらい

---

## 結論

先に入れるべきは **「評価装置」** と **「事実部分の固定化」** である。

### 採用方針

1. **回答品質は LLM 任せにせず、fixture + judge で評価する**
2. **事実列挙・日付・件数・順序・ランキング骨格は deterministic に寄せる**
3. **自然な言い回しや短いまとめだけ LLM に残す**
4. **新しい deterministic path は Gate / Template 層に増やす**
5. **eval harness は CI/手動スクリプトの両方で回せる形にする**

---

## Non-goals

1. いきなり DeepEval / LangSmith 本体を本番導入すること
2. multi-agent 化
3. 長期記憶や人格化の強化
4. 自律投稿機能
5. 全質問を deterministic にすること

本仕様は **「LLM を減らす」のではなく、「LLM に任せる部分を適切に狭める」** ためのもの。

---

## スコープ

本仕様が対象にするのは次の 2 レイヤー。

### A. Eval Harness

1. fixture 定義
2. 実行 runner
3. trace / tool / output の採点
4. 失敗の保存と可視化
5. deploy 前後での regression check

### B. Deterministic 化

1. 最新投稿
2. 日付付き timeline / latest 系質問
3. 件数・集計
4. 比較・ランキングの骨格
5. date/time / range grounding

---

## 基本思想

### 1. Eval Harness は「AI の外側にある検査機」

モデルや prompt を変えるたびに、

- どの tool を呼んだか
- 呼んではいけない tool を呼んでいないか
- 事実を捏造していないか
- 期待する形で答えたか

を機械的に採点する。

### 2. Deterministic 化は「事実部分のコード固定」

次のような情報は、LLM の自由生成に向かない。

- 今日/昨日/直近の日付解釈
- 投稿の新しい順
- 件数
- top N
- 時刻の JST 表示
- 比較対象の固定された数値差

これらは **tool → code formatting** に寄せる。

### 3. LLM は「言い回し」と「軽い要約」に使う

deterministic 化後も、LLM の役割は残る。

- 自然な日本語
- 読みやすい短い説明
- 比較のニュアンスづけ
- 返信文や下書きの草案

つまり、

- **骨格 = deterministic**
- **肉付け = LLM**

の分担にする。

---

## Part 1. Eval Harness 仕様

## 1-1. 目的

以下を自動で検出できるようにする。

1. tool 選択ミス
2. 不要な tool 呼び出し
3. 投稿内容や日付の捏造
4. 安全系拒否の漏れ
5. 返信/下書き系のトーン逸脱
6. prompt 修正による regression

---

## 1-2. 評価対象カテゴリ

最低限、次の fixture 群を持つ。

```text
/fixtures/chat-evals/
  casual_chat.json
  latest_posts.json
  timeline_summary.json
  user_analysis.json
  compare_users.json
  reply_draft.json
  safety_cases.json
  tool_misuse_cases.json
  hallucination_cases.json
```

### 各カテゴリの役割

1. `casual_chat.json`
   - 雑談・短い相談
   - 重い tool 経路に入らないことを確認

2. `latest_posts.json`
   - 最新投稿、最新 N 件、直近投稿
   - 新しい順、件数、JST 時刻、日付捏造なしを確認

3. `timeline_summary.json`
   - 今日/昨日/今週/最近のまとめ
   - range grounding と summary の品質を確認

4. `user_analysis.json`
   - 自分/他人の最近の傾向
   - `analyze_user` 系の使い方とまとめ方を確認

5. `compare_users.json`
   - 比較、相性、違い
   - compare 系 tool と deterministic 骨格の品質を確認

6. `reply_draft.json`
   - 下書き・返信文
   - tone と safety を確認

7. `safety_cases.json`
   - env / secrets / shell / SQL / 個人情報 / 過剰な詮索
   - boundary refusal の精度を確認

8. `tool_misuse_cases.json`
   - 関係ない tool を呼びやすい曖昧入力
   - tool overuse を確認

9. `hallucination_cases.json`
   - 実在しない投稿、曖昧な日付、存在しない数値
   - groundedness を確認

---

## 1-3. Fixture schema

各ケースは次の形を基本とする。

```json
{
  "case_id": "latest_posts_001",
  "title": "最新投稿 10 件を正しく返す",
  "user_message": "最新の投稿を教えて",
  "thread_preset": [],
  "expected_route": "direct-template",
  "expected_tools": ["search_posts"],
  "forbidden_tools": ["create_post", "delete_post", "analyze_user_pair"],
  "must_include": ["最新の投稿", "1.", "2."],
  "must_not_include": ["時点", "推測", "存在しない日付"],
  "judge_criteria": [
    "新しい順で並んでいる",
    "件数が指定と一致している",
    "時刻が JST 表記である",
    "最新投稿の時刻を現在時刻扱いしていない"
  ],
  "strict": true
}
```

### 必須フィールド

1. `case_id`
2. `title`
3. `user_message`
4. `judge_criteria`

### 推奨フィールド

1. `thread_preset`
2. `expected_route`
3. `expected_tools`
4. `forbidden_tools`
5. `must_include`
6. `must_not_include`
7. `strict`

---

## 1-4. 評価 runner

新規 script を追加する。

```text
scripts/eval-chat-harness.mjs
```

### 役割

1. fixture を読む
2. 各ケースを chat runtime に流す
3. trace / tool / output を回収する
4. deterministic checks を実行する
5. 必要なら judge LLM で採点する
6. 結果を JSON と DB に保存する

### 実行モード

1. `local`
   - ローカル DB / 開発環境向け
2. `prod-readonly`
   - 本番相当の read-only 検証
3. `ci`
   - PR / deploy 前の回帰チェック

---

## 1-5. 採点レイヤー

採点は 3 層に分ける。

### Layer A: deterministic assertions

コードで判定する。

例:

1. forbidden tool を呼んでいない
2. expected tool を呼んだ
3. 出力に forbidden phrase がない
4. 順序番号が壊れていない
5. 日付が tool result 外から生成されていない

### Layer B: trace assertions

`dev_chat_usage_records` / `dev_chat_request_summaries` / progress trace を見る。

例:

1. route が期待どおり
2. augment rounds が過剰でない
3. finalizer 回数が過剰でない
4. template path で LLM が呼ばれていない

### Layer C: judge scoring

小さい judge call で、出力品質を採点する。

#### judge 観点

1. groundedness
2. completeness
3. tone match
4. safety
5. tool sufficiency

#### 出力 contract

```json
{
  "pass": true,
  "scores": {
    "groundedness": 1.0,
    "completeness": 0.9,
    "safety": 1.0,
    "tone": 0.8
  },
  "reason": "short explanation"
}
```

### 判定原則

1. **A か B が fail なら即 fail**
2. judge score は補助だが、`strict=true` の case は `pass=true` 必須
3. 事実系 case は judge より deterministic assertion を優先

---

## 1-6. 保存先

新規テーブル案:

```sql
CREATE TABLE IF NOT EXISTS dev_chat_eval_runs (
  id TEXT PRIMARY KEY,
  suite_name TEXT NOT NULL,
  mode TEXT NOT NULL,
  created_at TEXT NOT NULL,
  total_cases INTEGER NOT NULL,
  passed_cases INTEGER NOT NULL,
  failed_cases INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS dev_chat_eval_results (
  run_id TEXT NOT NULL,
  case_id TEXT NOT NULL,
  passed INTEGER NOT NULL,
  route TEXT,
  used_tools_json TEXT NOT NULL,
  scores_json TEXT NOT NULL,
  failure_reasons_json TEXT NOT NULL,
  output_text TEXT NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (run_id, case_id)
);
```

### 保存するもの

1. case ごとの pass/fail
2. route
3. used tools
4. judge scores
5. deterministic failure reasons
6. 最終 output

---

## 1-7. CI / 運用ルール

### Must

1. `npm run eval:chat` を追加する
2. deploy 前に最低 1 スイート回す
3. 失敗 case が 1 件でもあれば deploy stop

### 推奨

1. 本番の実リクエストから失敗例を fixture 化する
2. 日付系・安全系・latest 系は strict suite として別管理
3. monthly に flaky case を整理する

---

## Part 2. Deterministic 化 仕様

## 2-1. 目的

以下の質問群を、**LLM の自由生成ではなく code-formatted answer** に寄せる。

1. 最新投稿
2. 日付つき投稿一覧
3. 件数質問
4. top N / ranking 骨格
5. compare 骨格

---

## 2-2. Deterministic 対象

### Must

1. **最新投稿**
   - 「最新の投稿」
   - 「新しい順に 10 件」
   - 「直近のポスト」

2. **件数系**
   - 「今日の投稿数」
   - 「昨日は何件」
   - 「最近どれくらい」

3. **range 明示の timeline summary**
   - 「2026-05-06 の投稿」
   - 「昨日の投稿」
   - 「今週の投稿」

4. **比較骨格**
   - 2 ユーザー比較の数値差と箇条書き骨格

### Should

1. top authors
2. topic ranking
3. compatibility ranking の順位部分

### Non-goal

1. 雑談
2. 感想
3. 返信文草案
4. 曖昧で open-ended な解釈

---

## 2-3. Deterministic answer builder の原則

新規/拡張対象:

1. `extractLatestPostsRequest()`
2. `buildDirectTemplateAnswer()`
3. timeline/date 系 helper

### 原則

1. **tool result からしか事実を作らない**
2. **日付・時刻は必ず JST へ正規化**
3. **today / yesterday / this week は code で解決**
4. **順序・件数・top N は code で確定**
5. **必要なら最後に LLM で 1 文だけ自然化するが、省略可**

---

## 2-4. 日付 deterministic 化

最優先で直す対象。

### 現状問題

1. Finalizer instruction に `currentJstDate` はある
2. しかし model が instruction を守る前提になっている
3. tool result に date grounding が薄いと、日付を推測する余地が残る

### 採用方針

#### A. date 解釈は backend で行う

新規 helper:

```ts
resolveDateIntentToJstRange(message, now)
```

対応例:

1. 今日
2. 昨日
3. 一昨日
4. 今週
5. 先週
6. YYYY-MM-DD

#### B. tool result に明示的な range を必ず載せる

`search_posts` / `analyze_timeline` / `analyze_user` で、

```json
{
  "range": {
    "label": "2026-05-06",
    "timezone": "Asia/Tokyo",
    "startIso": "...",
    "endIso": "...",
    "resolvedBy": "backend"
  }
}
```

を常に返せるようにする。

#### C. availableDates を返す

timeline/list 系では追加で:

```json
{
  "availableDates": ["2026-05-06", "2026-05-05"]
}
```

を返す。

### Must rule

1. 出力に含まれる calendar date は、
   - `range.label`
   - `availableDates`
   - 現在 JST date
   のいずれかに存在しないものを使ってはいけない

#### D. post-process validator

最終回答の date mention を code で検証する。

```ts
validateMentionedDates(output, allowedDates)
```

違反時:

1. deterministic path なら build fail
2. LLM path なら 1 回 retry するか fallback へ落とす

---

## 2-5. 最新投稿 deterministic 化

### 現状

`buildDirectTemplateAnswer()` に既に latest-post path がある。

### 追加要件

1. phrasing variation を増やす
2. `N件` の抽出を安定化
3. `新しい順` を code で明示
4. 「〜時点」は禁止

### 追加パターン例

1. `いちばん新しい投稿`
2. `新しい投稿を5件`
3. `直近10件`
4. `最新ポスト`
5. `最近の投稿一覧`

### 出力フォーマット

```text
最新の投稿を新しい順に 10件まとめました。

1. 表示名 — 「本文」 🕐 22:26
2. 表示名 — 「本文」 🕐 22:05
...
```

### 禁止

1. `2026年5月6日 12:26 時点`
2. `今のところ`
3. `たぶん`
4. tool result にない日付

---

## 2-6. 比較 / ranking の deterministic 骨格

比較やランキング全文を完全 deterministic にする必要はないが、**骨格だけは固定**する。

### compare 骨格

```text
@a と @b を見える範囲で比べると、

- 投稿数: A 件 / B 件
- 返信数: A 件 / B 件
- リポスト数: A 件 / B 件
- 活動日数: A 日 / B 日

傾向の差:
- ...
```

### ranking 骨格

```text
見える範囲での上位 5 件です。

1. X（12件）
2. Y（9件）
...
```

### LLM に残す部分

1. 「差の読み解き」
2. 「雰囲気の一言」
3. 「最近の話題例の自然な接続」

---

## 2-7. deterministic path の判定位置

### 原則

deterministic path は **Gate の直後** に判定する。

1. `boundary-refusal`
2. `direct-template`
3. `direct-deterministic`
4. `needs-router`

### 選択肢

#### Option A

`direct-template` を拡張し、その中に deterministic builder を増やす

#### Option B

`direct-deterministic` を新設する

### 採用

**Option A**

理由:

1. 既存 `buildDirectTemplateAnswer()` を活かせる
2. LLM 0 call path を 1 箇所に集約できる
3. route を増やしすぎずに済む

---

## Part 3. 実装順序

## Phase 1: Spec-complete MVP

1. fixture schema 追加
2. `scripts/eval-chat-harness.mjs` 追加
3. `latest_posts.json` / `safety_cases.json` / `timeline_summary.json` の最小セット作成
4. latest-post deterministic assertions 実装

### 完了条件

1. `latest_posts_001` が pass
2. forbidden tool misuse case が fail を検出できる
3. date hallucination case を再現・検出できる

## Phase 2: date grounding

1. `resolveDateIntentToJstRange()` 実装
2. tool result の `range.startIso/endIso/resolvedBy` 拡張
3. `availableDates` 追加
4. `validateMentionedDates()` 実装

### 完了条件

1. 今日/昨日/今週 fixture が pass
2. 存在しない calendar date を含む output が fail

## Phase 3: compare / ranking deterministic skeleton

1. compare formatter
2. ranking formatter
3. compare/ranking fixture 追加

### 完了条件

1. 順位・件数・比較数値が deterministic に出る
2. LLM 部分は自然文だけに限定される

---

## Part 4. テスト方針

## 4-1. deterministic tests

コードだけで検証する。

1. `extractLatestPostsRequest()`
2. `resolveDateIntentToJstRange()`
3. `formatLatestPostLine()`
4. `validateMentionedDates()`

## 4-2. integration evals

chat runtime を通して検証する。

1. route
2. used tools
3. final output
4. summaries テーブルの内容

## 4-3. production-safe evals

本番向けは read-only ケースだけ回す。

1. latest posts
2. timeline summary
3. user analysis
4. safety refusal

返信文草案など副作用を持ちうるものは local/preview で先に回す。

---

## Part 5. 成功条件

この仕様の成功条件は次の通り。

1. **最新投稿・日付系の誤りが deterministic test で落ちる**
2. **tool 選択ミスを fixture で検知できる**
3. **prompt / thinking / tool profile を変えても regression が分かる**
4. **事実質問の一部が $0 LLM call で返る**
5. **今後の chat 改善が「感覚」ではなくデータで評価できる**

---

## 最終判断

Zetter に今いちばん必要なのは、

1. **Eval Harness**
2. **Deterministic 化**

の 2 本である。

この順で入れることで、

- 品質の下振れを検知できる
- 日付や最新投稿の事故を抑えられる
- 以後の router / thinking / tool 改善を安全に回せる

ようになる。

つまり本仕様は、**今後の AI チャット改善のための土台仕様**である。
