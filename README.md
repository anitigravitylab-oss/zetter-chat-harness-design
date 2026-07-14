# Zetter Chat — ZYN（ジン）設計仕様書 / Design Spec

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

**JA:** ZYN（ジン）は SNS アプリ **Zetter** に組み込まれた、会話型の AI アシスタントです。投稿・ユーザーに関する質問への回答、タイムラインの要約、ユーザー分析、添付画像の解析を行います。本ドキュメントは **現在の確定設計（現行設計）** を記述します。

**EN:** ZYN is the conversational AI assistant embedded in the **Zetter** social app. It answers questions about posts and users, summarizes the timeline, analyzes users, and interprets attached images. This document describes the **current design**.

> このドキュメントは現行設計のみを扱います。旧（5層パイプライン）設計は末尾の「旧設計サマリ」を参照。
> This document covers the current design only. The previous (5-layer pipeline) design is summarized at the end.

---

## 今回の大きな変更 / What changed in this update

**JA:** 本セクションは 2026-06-29（tool-loop v2 初期）以降の進化を反映した **2026-07-14 更新**です。基本アーキテクチャ（単一プロンプト + tool-calling ループ、`deepseek-v4-flash` 続投）は変わっていませんが、以下が積み上がりました:

- **ツールが 5→6 個**: `aggregate_posts`（groupBy: day/weekday/hour/month/author/kind の集計専用ツール）を追加。`query_posts_v2` にも `hourFrom`/`hourTo`（時間帯フィルタ）が追加
- **「正直な着地」**（#151）: ターン数・履歴長ベースの素朴な上限から、**ツール実行予算**（既定 10 ターン or 45 秒）＋**最終ターン契約**（予算超過時は `tools: []` で物理的に呼び出し不能にし、必ず文章で着地させる）に置き換え
- **本文捏造対策 / content grounding**（#132）: システムプロンプトに「records へのグラウンディング」節を新設し、`returned`/`fetched`/`totalMatched` の混同禁止・投稿時刻の verbatim 扱いなどを明文化
- **メモリ基盤 v2**: スレッド内 running summary（L1）＋ユーザー横断日次ダイジェスト（L2）の 2 層を追加。24 メッセージの直近履歴ウィンドウ自体は維持
- **能力の正典化・出典リンク**（#259）: 「Capability Charter」節と出典明示ルールをプロンプトに追加（2026-07-13 の本番デプロイに包含済み）
- **画像添付の修正**（#197 / #199）: 直接添付の Gemini 応答 JSON 構造ゆれ対応、投稿添付画像も同じ Gemini 解析経路に統合

**EN:** This section is the **2026-07-14 update**, covering evolution since 2026-06-29 (early tool-loop v2). The core architecture (single prompt + tool-calling loop, `deepseek-v4-flash` retained) is unchanged, but the following accumulated on top:

- **Tools 5→6**: added `aggregate_posts` (a dedicated aggregation tool with `groupBy`: day/weekday/hour/month/author/kind); `query_posts_v2` also gained `hourFrom`/`hourTo` (time-of-day filtering)
- **"Honest landing"** (#151): replaced the naive turn/history-based cap with a **tool execution budget** (default 10 turns or 45s) plus a **final-turn contract** — once the budget is exceeded, the next turn is sent with `tools: []`, physically preventing further tool calls so the model always lands a prose answer
- **Content grounding / anti-fabrication** (#132): a new "grounding to records" prompt section bans conflating `returned`/`fetched`/`totalMatched` and mandates verbatim post timestamps, among other rules
- **Memory v2**: added a two-tier memory — an in-thread running summary (L1) and a cross-user daily digest (L2) — layered on top of the still-unchanged 24-message recent history window
- **Capability canon + citation links** (#259): added a "Capability Charter" section and source-citation rules to the prompt (included in the 2026-07-13 production deployment)
- **Image attachment fixes** (#197 / #199): fixed Gemini response JSON-shape drift silently dropping directly-attached images, and unified attached-post images onto the same Gemini analysis path

---

## アーキテクチャ全体像 / Architecture overview

```
                 user message (+ optional images / attached post)
                              │
        ┌─────────────────────▼─────────────────────┐
        │  Harness scaffolding (db-chat-harness-v2)   │
        │  • JST date grounding (injected per request)│
        │  • Gemini image context (direct + attached  │
        │    post, both via buildGeminiImageContext)  │
        │  • thread context load                      │
        │  • memory: L2 (cross-user digest) →         │
        │            L1 (thread running summary) →    │
        │            recent raw history (24 msgs)      │
        └─────────────────────┬─────────────────────┘
                              │  single comprehensive system prompt
                              │  + history window (24 msgs)
        ┌─────────────────────▼─────────────────────┐
        │            TOOL-CALLING LOOP                │
        │   (deepseek-v4-flash)                       │
        │   tool budget: ≤10 turns OR ≤45s wall-clock │
        │   (ZYN_TOOL_BUDGET_TURNS / _WALL_MS env)    │
        │                                             │
        │   ┌───────────────────────────────────┐    │
        │   │ LLM call (tool_choice auto/none/   │    │
        │   │           required)                │    │
        │   └───────────────┬───────────────────┘    │
        │       has tool_calls?                       │
        │        │ yes               │ no + stop      │
        │        ▼                   ▼                │
        │   execute tools       break → final answer  │
        │   push role:tool                            │
        │   results, repeat                           │
        │                                             │
        │   budget exceeded → final-turn contract:    │
        │   tool defs omitted from the API request    │
        │   + wrap-up system message forces a prose   │
        │   landing (no more tool calls)              │
        └─────────────────────┬─────────────────────┘
                              │  final answer
        ┌─────────────────────▼─────────────────────┐
        │  Safety nets (zyn-circuit-breaker)          │
        │  CB#11 / CB H2 / CB#12 / H3                  │
        │  + normalizeChatText (strip DSML/markers)   │
        └─────────────────────┬─────────────────────┘
                              ▼
                  answer to user + persistence
```

**6 fixed tools (registered unconditionally):**

```
query_posts_v2 · summarize_timeline · analyze_user · lookup_user_profile · compare_users · aggregate_posts
```

---

## tool-loop の仕組み / How the tool loop works

**JA:** 単一モデルを `for` ループで回します（`db-chat-harness-v2.ts`）。上限は固定ターン数ではなく **ツール実行予算**（PR #151「正直な着地」）で管理されます: `TOOL_BUDGET_DEFAULT_MAX_TURNS = 10` ターン **または** `TOOL_BUDGET_DEFAULT_WALL_MS = 45_000`（45 秒）のいずれか早い方（`isToolBudgetExceeded` が OR 条件で判定）。それぞれ `ZYN_TOOL_BUDGET_TURNS` / `ZYN_TOOL_BUDGET_WALL_MS` 環境変数で上書き可能。履歴ウィンドウは変わらず 24 メッセージ（`LOOP_HISTORY_WINDOW = 24`）。各ターンの流れ:

1. LLM 呼び出し（`tool_choice` は `auto` / `none` / `required` を状況で切替）
2. assistant メッセージを履歴に push
3. `tool_calls` があれば各ツールを実行し、結果を `role: tool` メッセージとして履歴に push
4. これを繰り返す

`tool_calls` が無く `stop` で終わったとき（または DSML 解析でもツール呼び出しが見つからないとき）にループを抜け、そのターンの本文を `normalizeChatText` で整形したものが最終回答になります。

**最終ターン契約 / Force-stop:** ツール予算（10 ターン or 45 秒）を超えたターンでは `isToolBudgetExceeded` が真になり、そのターンに限りツール定義を API リクエストから外して（`tools`/`tool_choice` キーごと省略）ツール呼び出しを物理的に不能化し、`TOOL_BUDGET_FINAL_TURN_SYSTEM_MESSAGE`（「ツール呼び出しはここまで（予算上限）。…」）を system メッセージとして注入して、必ず文章で着地させます。個別の DeepSeek 呼び出しには別途 `DEEPSEEK_COMPLETION_TIMEOUT_MS = 90_000`（90 秒）のタイムアウトがあります。

**EN:** A single model runs in a `for` loop (`db-chat-harness-v2.ts`). Instead of a fixed turn cap, the loop is governed by a **tool execution budget** (PR #151, "honest landing"): `TOOL_BUDGET_DEFAULT_MAX_TURNS = 10` turns **or** `TOOL_BUDGET_DEFAULT_WALL_MS = 45_000` (45s), whichever comes first (`isToolBudgetExceeded` checks both with OR), each overridable via `ZYN_TOOL_BUDGET_TURNS` / `ZYN_TOOL_BUDGET_WALL_MS`. The history window remains 24 messages (`LOOP_HISTORY_WINDOW = 24`). Each turn: LLM call (`tool_choice` switched between `auto` / `none` / `required`) → push assistant message → execute any `tool_calls` and push results as `role: tool` messages → repeat. The loop breaks when there are no `tool_calls` and the model stops (or DSML parsing finds no tool calls); the breaking turn's content, passed through `normalizeChatText`, is the final answer. **Final-turn contract / force-stop:** once the tool budget (10 turns or 45s) is exceeded, `isToolBudgetExceeded` fires and that turn is sent with no tool definitions at all (the `tools`/`tool_choice` keys are omitted from the API request) — physically disabling further tool calls — plus a `TOOL_BUDGET_FINAL_TURN_SYSTEM_MESSAGE` system message telling the model to wrap up in prose. Individual DeepSeek calls separately carry a `DEEPSEEK_COMPLETION_TIMEOUT_MS = 90_000` (90s) timeout.

---

## 6 つの固定ツール / The six fixed tools

**JA:** `TOOL_CALL_LOOP_TOOL_DEFINITIONS`（`zetter-chat-tools.ts`）に定義。PR #147 で集計専用の `aggregate_posts` が追加され 5→6 個になり、PR #151 で `query_posts_v2` に時間帯フィルタ (`hourFrom`/`hourTo`) が加わりました。

**EN:** Defined in `TOOL_CALL_LOOP_TOOL_DEFINITIONS` (`zetter-chat-tools.ts`). PR #147 added the aggregation-only `aggregate_posts` tool (5→6), and PR #151 added time-of-day filtering (`hourFrom`/`hourTo`) to `query_posts_v2`.

| Tool | 用途 / Purpose | 主なパラメータ / Key params |
|---|---|---|
| `query_posts_v2` | 投稿の検索・取得。任意の日付範囲・時間帯・キーワード・ユーザー・投稿 ID・ページング / Post search & retrieval over an arbitrary date range, time-of-day range, keyword, user, post id, with pagination | `userId`, `keyword`, `postId`, `fromDate`, `toDate` (YYYY-MM-DD JST), `hourFrom`, `hourTo` (0–23, JST, inclusive, day-crossing allowed), `sortOrder` (`newest`/`oldest`), `cursorSeq`, `beforeSeq`, `includeAiCast`, `limit` (1–80, default 20) — at least one of `userId`/`keyword`/`postId`/`fromDate`+`toDate`/`hourFrom`+`hourTo` required |
| `summarize_timeline` | 固定ウィンドウのタイムライン概観（流れ・話題） / Recent timeline overview for a fixed window | `window` (`24h` \| `7d` \| `since-last-visit`), `fromDate`/`toDate` (指定時は window より優先 / take precedence over window if given), `limit` (1–30, default 20), `includeAiCast` |
| `analyze_user` | 単一ユーザーの深掘り分析（人物像・話題・トーン） / Single-user deep analysis | `userId` (required), `aspect` (`personality` \| `topics` \| `tone`), `fromDate`, `toDate`, `postsToAnalyze` (1–80, default 80), `includeAiCast` |
| `lookup_user_profile` | 軽量プロフィール取得（表示名・bio・投稿数） / Lightweight profile lookup | `userId` (required) のみ / only |
| `compare_users` | 2 ユーザーの比較・相性診断 / Two-user comparison & compatibility | `userIdA`, `userIdB` (both required), `aspect` (`topics` \| `tone` \| `compatibility`), `fromDate`, `toDate`, `includeAiCast` |
| `aggregate_posts` | 投稿の集計（曜日別・時間帯別など件数系の質問に直答） / Post aggregation for count-style questions (by weekday, by hour, etc.) | `groupBy` (required: `day`\|`weekday`\|`hour`\|`month`\|`author`\|`kind`), `fromDate`, `toDate`, `userId`, `keyword`, `includeAiCast` |

**JA:** ツールは **無条件で常時登録**されます。いつ・どれを使うかは LLM が判断します（旧設計のような「投稿検索チップを ON」というオプトインは不要）。パラメータの妥当性は `validateToolArgs` がツール呼び出し前に強制します。

**EN:** All tools are **registered unconditionally**; the LLM decides when and which to use (no opt-in "post-search chip" as in the old design). Parameter validity is enforced by `validateToolArgs` before each tool call executes.

---

## 単一システムプロンプト / Single system prompt

**JA:** 旧の最小（約 8 行）プロンプトに代えて、`buildLoopSystemPrompt`（`zyn-system-prompt.ts`）が組み立てる 1 本の包括的な **# 1〜# 15 + 番号なし 2 節**構成のプロンプトを使います。2026-06-29 時点の番号付き 11 節（# 1〜# 11）から、PR #132（# 12 grounding）・PR #149（# 13 いいね思想の正典化）・PR #259（# 14 能力正典・# 15 出典）で 4 節が新設され、現在の構成は以下の通りです:

- **Identity/role preamble** — ZYN（ジン）。Zetter を理解した、温かみのある会話調のアシスタント。中立的なボットではない
- **# 1. Language** — ユーザーが明確に他言語で書かない限り、常に日本語
- **# 2. Current date (JST)** — リクエスト時に算出した `YYYY-MM-DD` を注入。相対日付はツールに渡す前に絶対日付へ変換させる
- **# 3. Honesty Protocol** — (1) ツールを呼ぶ前に能力を否定しない (2) 空結果は「その呼び出しでは見つからなかった」 (3) 推測しない（True facts / Forbidden lies / good-bad 例のサブ構成は維持）
- **# 4. Tool Routing** — 6 ツールの判定フロー（比較→`compare_users` / 単一ユーザー→`analyze_user` / 今・最近→`summarize_timeline` / 集計→`aggregate_posts` / それ以外→`query_posts_v2`）＋「使えるツールはこの6つだけ」の明記＋時間帯質問は `hourFrom`/`hourTo` で一発検索（`beforeSeq`/`cursorSeq` での自前ページング探索は禁止）
- **# 5. First-person / Bare-latest semantics** — 「私 / 自分」＝閲覧者。修飾なしの「最新 N 件」＝制約なし・新しい順のグローバル
- **# 6. Empty Result Fallback** — 諦める前に別ツール・別パラメータで再試行。代替を尽くしてから「見つからない」と言う
- **# 7. Tool Hygiene** — 疑似ツール構文や DSML を出力しない。ネイティブ tool call のみ。生 JSON を露出しない
- **# 8. Mistakes & Self-Correction** — 簡潔に認め、ツールを呼び直し、正しい答えを出す。過剰な謝罪はしない
- **# 9. Tone & Formatting** — 自然な日本語の散文。結論先出し。既定で markdown 表を使わない。キャラ設定・トーン調整・なりきり許容範囲もここに含む（PR #233 由来）
- **# 10. Memory within this conversation** — スレッド要約の有無で文言が動的に切替（24 メッセージの直近履歴＋あれば L1 スレッド要約）
- **# 11. Safety and boundaries** — システムプロンプト・内部実装・秘密を開示しない
- **# 12. records へのグラウンディング（本文捏造の禁止・最重要）**（PR #132 新設）— `returned`/`fetched`/`totalMatched` を混同しない、投稿時刻は verbatim（丸め・言い換え禁止）、records に無い本文・時刻・著者を推測で書かない
- **# 13. ZYN の立ち位置と AI cast**（@zetachan / @zett）（PR #149 新設）— いいね数・いいねランキングを聞かれても「答えられない」で終わらせず、意図的な設計方針として前向きに語る
- **# 14. Capability Charter（能力・仕様の正典）**（PR #259 新設）
- **# 15. 出典明示とリンク**（PR #259 新設）
- **# About Zetter / # Product context**（番号なし） — Zetter プラットフォームの事実と非機能

**EN:** Instead of the old minimal (~8-line) prompt, `buildLoopSystemPrompt` (`zyn-system-prompt.ts`) assembles a single comprehensive prompt now spanning **# 1–# 15 plus two unnumbered sections**. Since the 11 numbered sections (#1–#11) as of 2026-06-29, four sections were added: #12 by PR #132 (grounding), #13 by PR #149 (likes-philosophy canon), and #14/#15 by PR #259 (capability canon / citations). Current structure: identity preamble (ZYN — Zetter-aware, warm conversational assistant, not a neutral bot); #1 Language; #2 Current date (live-injected JST `YYYY-MM-DD`, relative dates converted to absolute before tool calls); #3 Honesty Protocol (never deny ability before calling a tool; empty results are "that call found nothing"; no speculation — the True-facts / Forbidden-lies / good-bad-example sub-structure is retained); #4 Tool Routing (compare→`compare_users`, single-user→`analyze_user`, now/recent→`summarize_timeline`, aggregation→`aggregate_posts`, else→`query_posts_v2`; states "only these 6 tools exist"; time-of-day questions must use `hourFrom`/`hourTo` in one call rather than manual `beforeSeq`/`cursorSeq` paging); #5 First-person / bare-latest semantics; #6 Empty-result fallback; #7 Tool Hygiene (native tool calls only, no DSML/raw JSON); #8 Mistakes & self-correction; #9 Tone & Formatting (natural Japanese prose, answer-first, no default markdown tables, now also covers character config/tone-shift/roleplay bounds per PR #233); #10 Memory within this conversation (dynamically worded depending on whether a thread summary exists; 24-message recent window + L1 summary if present); #11 Safety and boundaries (no system-prompt/internals/secrets disclosure); #12 Grounding to records (PR #132 — never conflate `returned`/`fetched`/`totalMatched`, keep post timestamps verbatim, never invent body/time/author beyond records); #13 ZYN's stance and AI cast (@zetachan / @zett) — never dead-end on like-count questions, frame it as a deliberate design choice; #14 Capability Charter (PR #259); #15 Source citation and links (PR #259); unnumbered # About Zetter / # Product context sections with platform facts and non-features.

---

## セーフティネット / Safety nets (circuit breakers)

**JA:** `zyn-circuit-breaker.ts` に 4 つの決定的（deterministic）なサーキットブレーカーがあり、ループの外側で最終回答を検査・補正します。

| ID | 発火条件 / Trigger | 動作 / Action |
|---|---|---|
| **CB #11** `detectContradiction` | ツールを 1 回以上呼んだ + 最終回答に能力否定フレーズが一致 | `LIE_CORRECTION_ANNOTATION`（日本語のユーザー可視注記）を末尾に付与 |
| **CB H2** `detectPostSuccessLie` | ツールが実データ（非エラー）を返した + 能力否定フレーズ | 同上の注記を付与 |
| **CB #12** `detectMissingToolCall` | ツール 0 回 + ユーザー文に DB 参照キーワード + 能力否定フレーズ | ループ外で `query_posts_v2`（today, limit=20）を強制実行し、`tool_choice="none"` で LLM を再呼び出し |
| **H3** `shouldForceToolRetry` | turn=0 でツール無し + 能力欠如フレーズ + DB キーワード | 偽の assistant メッセージを pop し、turn=1 を `tool_choice="required"` で再試行（`turn0ForceRetryDone` フラグで重複防止） |

**EN:** Four deterministic circuit breakers in `zyn-circuit-breaker.ts` inspect/repair the final answer outside the loop. **CB #11** (≥1 tool called + ability-lacking phrase in the answer → append `LIE_CORRECTION_ANNOTATION`), **CB H2** (≥1 tool returned real non-error data + ability-lacking phrase → same annotation), **CB #12** (0 tools called + DB-reference keyword in the user message + ability-lacking phrase → force-call `query_posts_v2` (today, limit=20) out of loop and re-invoke the LLM with `tool_choice="none"`), and **H3** `shouldForceToolRetry` (turn=0 no-tool + ability-lacking phrase + DB keyword → pop the false assistant message and retry turn=1 with `tool_choice="required"`, guarded by `turn0ForceRetryDone`). `LIE_CORRECTION_ANNOTATION` is a user-visible Japanese note appended to the answer.

---

## grounding（日付・能力） / Grounding

**JA:** JST の現在日付を `Intl.DateTimeFormat`（`Asia/Tokyo`）でリクエスト時に算出し、毎回システムプロンプトへ注入します（# 2 節）。プロンプトの True facts 節で「`query_posts_v2` の `fromDate`/`toDate` に固定の範囲制限は無い」「ツール結果が真実の源」と明示し、相対日付表現はツールに渡す前に今日の JST 日付を使って絶対 `YYYY-MM-DD` に変換させます。時間帯を問う質問は `hourFrom`/`hourTo`（0–23, JST）で一度に絞り込ませ、`beforeSeq`/`cursorSeq` を使った自前ページング探索は # 4 節で明示的に禁止しています。

**EN:** The current JST date is computed per request via `Intl.DateTimeFormat` (`Asia/Tokyo`) and injected into the system prompt every time (§2). The True-facts section states that `query_posts_v2`'s `fromDate`/`toDate` has no fixed range limit and that tool results are the source of truth; the model is instructed to convert all relative date expressions to absolute `YYYY-MM-DD` using today's JST date before passing them to tools. Time-of-day questions must be resolved in one call via `hourFrom`/`hourTo` (0–23, JST); §4 explicitly forbids manual paging (`beforeSeq`/`cursorSeq`) as a substitute.

---

## 本文グラウンディング（records 捏造対策） / Content grounding (anti-fabrication)

**JA:** PR #132（2026-07-01 マージ）で新設された、システムプロンプト **# 12** 節。ツール結果に無い本文・時刻・著者・出来事を推測で書くのを防ぐための規範です。要点:

- `records[]` に入っている投稿だけが実際に読んだ本文。それ以外を推測で書かない
- `records` は代表サンプルであり全件ではない。`totalMatched`（総数）が `returned`（表示件数）より多いときは「○件中△件を見た範囲では」と前置きする
- 件数・種別の集計には `totalMatched` / `kindCountsAll`（全件・正確な値）を使い、個別エピソードは `records` 内のものだけを挙げる
- `returned`（読めた件数）/ `fetched`（DB 取得件数）/ `totalMatched`（DB 総数）の 3 値を混同しない
- 投稿時刻（`dateTimeJst` / `timeJst`）は verbatim（そのまま）で扱い、丸めたり言い換えたりしない

**EN:** Introduced as system-prompt **§12** by PR #132 (merged 2026-07-01). A set of norms preventing the model from inventing post bodies, timestamps, authors, or events not present in tool results. Key points: only posts inside `records[]` are content the model actually read; `records` is a representative sample, not the full result set, so when `totalMatched` (total count) exceeds `returned` (shown count) the model must preface with "of N matching, based on the M I saw..."; count/category aggregates must use `totalMatched` / `kindCountsAll` (exact, full-set figures), while individual anecdotes may only cite items inside `records`; the three meta values `returned` (readable count) / `fetched` (DB-retrieved count) / `totalMatched` (DB total count) must never be conflated; post timestamps (`dateTimeJst` / `timeJst`) must be handled verbatim, never rounded or paraphrased.

---

## メモリ基盤 v2（L1/L2） / Memory v2 (L1/L2)

**JA:** 2026-06-29 時点では 24 メッセージの直近履歴窓のみでしたが、その上に 2 層のメモリを追加しました（`db-chat-harness-v2.ts`）。直近履歴窓の 24 メッセージ自体は変わらず維持されます（`LOOP_HISTORY_WINDOW = 24`）。プロンプトへの注入順は **L2（ユーザー横断）→ L1（スレッド要約）→ 直近ツール結果の生データ** の固定順です。

- **L1（スレッド内 running summary）**: `L1_KEEP_WINDOW = 24`、`L1_OUTPUT_CHAR_LIMIT = 2,000`、`L1_INPUT_CHAR_LIMIT = 24,000` など。バックログ件数・文字数やツール結果のオーバーフロー（`L1_TOOL_OVERFLOW_TRIGGER_COUNT = 8` 件 / `L1_TOOL_OVERFLOW_TRIGGER_CHARS = 12,000` 文字、`L1_BACKLOG_TRIGGER_COUNT = 16` 件 / `L1_BACKLOG_TRIGGER_CHARS = 8,000` 文字）のいずれかが閾値を超えると `shouldRecomputeThreadMemory` が再合成をトリガーし、`buildL1MemoryBlock` が `<thread_memory>` フェンスで注入。システムプロンプト内では「# 10. Memory within this conversation」として、要約の有無で文言が動的に切替わります
- **L2（ユーザー横断・日次ダイジェスト）**: `L2_MAX_USERS_PER_RUN = 50`、`L2_INPUT_CHAR_LIMIT = 20,000`、`L2_OUTPUT_CHAR_LIMIT_PER_FIELD = 1,200`、`L2_THREAD_DIGEST_MAX_CHARS = 1,000`。`buildL2MemoryBlock` が `<user_memory>` フェンスで facts/prefs を注入
- チャット DB は本番 PostgreSQL（D1 ではない）。L1/L2 テーブルは初回チャット時に ensure 作成される

**EN:** As of 2026-06-29 there was only the 24-message recent-history window; a two-tier memory layer has since been added on top (`db-chat-harness-v2.ts`). The recent-history window itself is unchanged (`LOOP_HISTORY_WINDOW = 24`). Injection order into the prompt is fixed: **L2 (cross-user) → L1 (thread summary) → raw recent tool results**.

- **L1 (in-thread running summary)**: `L1_KEEP_WINDOW = 24`, `L1_OUTPUT_CHAR_LIMIT = 2,000`, `L1_INPUT_CHAR_LIMIT = 24,000`, among others. `shouldRecomputeThreadMemory` triggers recomputation when backlog count/chars or tool-result overflow (`L1_TOOL_OVERFLOW_TRIGGER_COUNT = 8` entries / `L1_TOOL_OVERFLOW_TRIGGER_CHARS = 12,000` chars, `L1_BACKLOG_TRIGGER_COUNT = 16` entries / `L1_BACKLOG_TRIGGER_CHARS = 8,000` chars) exceed thresholds; `buildL1MemoryBlock` injects it inside a `<thread_memory>` fence. In the system prompt this surfaces as "§10. Memory within this conversation", with wording that switches dynamically depending on whether a summary exists.
- **L2 (cross-user daily digest)**: `L2_MAX_USERS_PER_RUN = 50`, `L2_INPUT_CHAR_LIMIT = 20,000`, `L2_OUTPUT_CHAR_LIMIT_PER_FIELD = 1,200`, `L2_THREAD_DIGEST_MAX_CHARS = 1,000`. `buildL2MemoryBlock` injects facts/prefs inside a `<user_memory>` fence.
- The chat DB is production PostgreSQL (not D1). L1/L2 tables are created on first chat via an ensure step.

---

## normalization（DSML 解析・マーカー除去） / Normalization

**JA:** モデルがネイティブ tool call ではなくテキストでツール呼び出しを書いてしまう場合に備えた整形層です。

- `parseDsmlToolCalls` — 全角 `｜｜DSML｜｜` または半角 `||DSML||` の invoke/parameter マーカーを本文から検出し、ツール名と引数を抽出して `DeepSeekToolCall` を合成。DSML ブロック前の散文は保持。モデルがテキストでツールを書いてもループを継続させる
- `stripInternalMarkers` — (1) 最初の DSML 開始位置で本文を切り詰め (2) 残ったインラインマーカーを除去
- `normalizeChatText` — 上記 2 段に加え CRLF・過剰な空行を整形してラップ

**EN:** A formatting layer for when the model writes textual tool calls instead of native ones. `parseDsmlToolCalls` detects fullwidth `｜｜DSML｜｜` or ASCII `||DSML||` invoke/parameter markers in the content, extracts the tool name and args, synthesizes `DeepSeekToolCall` objects, and preserves prose before the DSML block — keeping the loop running. `stripInternalMarkers` truncates at the first DSML opener and removes any remaining inline markers. `normalizeChatText` wraps both steps plus CRLF and excess blank-line normalization.

---

## observability（推論トレース） / Observability

**JA:** `ZYN_TRACE_LOG="true"` のときのみ有効。`appendDevChatLlmOutput` を各ターンで呼び（`stage="turn-N"`: assistantText, reasoningContent, finishReason, toolCalls トレース）、最後に 1 回（`stage="final"`: turnCount, finalAnswer, 4 ブレーカーの真偽 — contradiction / postSuccessLie / missingTool / turn0RetryDone）記録します。保存先は `dev_chat_llm_outputs`。トークン使用量はターンごとに `appendDevChatUsageRecord` で別途記録します。

**EN:** Gated by `ZYN_TRACE_LOG="true"`. `appendDevChatLlmOutput` is called per turn (`stage="turn-N"`: assistantText, reasoningContent, finishReason, toolCalls trace) and once at the end (`stage="final"`: turnCount, finalAnswer, the four circuit-breaker booleans). Records go to `dev_chat_llm_outputs`. Per-turn token usage is recorded separately via `appendDevChatUsageRecord`.

---

## 引き継ぎ（旧ハーネスから保持） / Retained from the old harness

| 機能 / Capability | 内容 / Detail |
|---|---|
| 画像解析 / Image analysis | Gemini。直接添付（`imageDataUrls` → `buildGeminiImageContext`）と投稿添付（`attachedPostId` → `post_media_items` から取得 → 同じ `buildGeminiImageContext`、最大 4 枚、JPEG/PNG/WebP 限定）の 2 経路が同じ Gemini 解析関数に合流。注入位置は経路で異なり、直接添付は解析結果をユーザーターンに前置、投稿添付は直近ツール結果チャネル（`buildRecentToolContextBlock`）経由で注入。PR #197 で直接添付の Gemini 応答 JSON 構造ゆれによる無言失敗を修正、PR #199 で投稿添付画像も同経路に統合 / Two paths — direct attachment (`imageDataUrls` → `buildGeminiImageContext`) and attached-post images (`attachedPostId` → fetched from `post_media_items`, max 4, JPEG/PNG/WebP only) — converge on the same Gemini analysis function, but inject differently: direct-attachment results are prepended to the user turn, while attached-post results flow in via the recent-tool-results channel (`buildRecentToolContextBlock`). PR #197 fixed a silent failure from Gemini response JSON-shape drift on direct attachments; PR #199 unified attached-post images onto the same analysis path |
| スレッド文脈 / Thread context | `loadThreadContext` + `maybeCompressAndSaveThreadContext` を引き続き実行 / still run |
| レート制限 / Rate limiting | ループ後に `settleDevChatRateLimitReservation` を強制 / enforced after the loop |
| 再処理 / Reprocess | `reprocessMessageId` 経路を保持（`buildAssistantOnlyFinalState`）/ preserved |
| 永続化 / Persistence | `insertThreadedDevChatExchange` / `insertLegacyDevChatExchange` は不変 / unchanged |

---

## 旧設計との比較 / Comparison with the previous design

| 観点 / Aspect | 旧 / Old | 新 / New |
|---|---|---|
| **制御主体 / Control principal** | Code-driven: `classifyIntentViaLlm` の JSON → `buildIntentRoute` がコードで固定 DB ステップを組み立て | LLM-driven: モデルが毎ターン `tool_choice='auto'` で自律的にツール選択。コード側のステップ写像なし |
| **意図分類 / Intent classification** | 必須: DB アクセス前に planner LLM が `{kind, intent, userId, date, ...}` JSON を生成 | 廃止: LLM がユーザー文と文脈から、呼ぶか・どれを呼ぶかを直接決定 |
| **DB アクセス起動 / DB activation** | Opt-in: ユーザーが「投稿検索」チップを ON にする必要（`tools` に `db-search`） | 常時利用可: ツールを無条件登録。取得タイミングは LLM が判断 |
| **1 リクエストの LLM 呼数 / LLM calls** | 最低 2+（planner 1 + finalizer 1、＋任意の人物解決） | 単一モデルのループ。ツール実行予算（10 ターン or 45 秒、いずれか早い方）に達するまで継続し、超過ターンはツール定義を外した最終ターン契約で必ず着地 |
| **根拠注入 / Evidence injection** | `DbRetrievalMaterial` を `=== DATABASE RECORDS ===` テキストブロックとしてユーザー文に埋込 | ツール結果を `role:tool` メッセージとして会話ターン履歴に差し込む |
| **システムプロンプト範囲 / Prompt scope** | 最小の約 8 行の identity + ルール。プラットフォーム情報や DB 素材は別途ユーザー文へ注入 | 単一の包括的 # 1〜# 15 + 番号なし 2 節（日付・Honesty Protocol・ルーティング・本文グラウンディング・能力正典・出典・Zetter 事実・安全を一本化） |
| **能力詐称ガード / Ability-claiming guard** | 専用なし。planner が黙って `no_db` 分類し取得をスキップし得た | Honesty Protocol + 4 サーキットブレーカー（CB #11 / H2 / #12 / H3）で注記・強制再試行。加えて能力正典（Capability Charter, PR #259）をプロンプトに明文化 |
| **DSML テキスト tool call / DSML handling** | 該当なし（構造化 JSON 素材でテキスト tool call のリスクなし） | `parseDsmlToolCalls` が DSML テキストをネイティブ tool call に合成しループ継続 |
| **ターン / ステップ上限 / Turn limit** | `MAX_STEPS=3` の固定コード反復（`executeRetrievalSteps`）。完了後 LLM 再入なし | ツール実行予算: `TOOL_BUDGET_DEFAULT_MAX_TURNS=10` ターン **or** `TOOL_BUDGET_DEFAULT_WALL_MS=45,000`（45秒）、いずれか早い方（PR #151）。超過後はツール定義を外した最終ターン契約で強制着地 |
| **人物解決 / Person resolution** | 2 段の固定コード: `search_users` → 条件付き後続の投稿/分析ステップ | LLM が解決戦略を決定。`lookup_user_profile` / `analyze_user` を独立ツールとして利用 |

### なぜ変えたか / Why we changed it

**JA:** 旧はコード主導の固定パイプラインで、LLM の役割は intent 分類と最終文整形に限られていました。結果としてルーティングが硬直し、planner が黙って「DB 不要」と分類して取得を飛ばす・能力詐称の嘘（「アクセスできる範囲に限り…」等）が出る、といった失敗が起きやすい構造でした。新設計では LLM がツールを主導し、ハーネスは grounding（日付・能力の事実）/ normalization（DSML・マーカー）/ observability（トレース）/ safety net（ブレーカー）という **足場**に徹します。これにより柔軟なルーティングと、決定的な安全網による嘘の抑止を両立します。

**EN:** The old fixed, code-driven pipeline confined the LLM to intent classification and final-text formatting, producing rigid routing and failure modes such as the planner silently classifying "no DB" and skipping retrieval, or emitting self-limiting lies. The new design lets the LLM drive tool use while the harness acts purely as **scaffolding** — grounding (date/ability facts), normalization (DSML/markers), observability (traces), and safety nets (deterministic breakers) — combining flexible routing with deterministic suppression of hallucinated ability denials.

---

## 旧設計サマリ（参考・deprecated） / Previous design summary (deprecated)

**JA:** 旧設計は概ね次の 5 層パイプラインでした（詳細は git 履歴を参照）:

1. **Intent classification** — planner LLM が `{kind, intent, userId, date, ...}` JSON を生成
2. **Route building** — `buildIntentRoute` がコードで固定 DB ステップ列を組み立て
3. **Retrieval execution** — `executeRetrievalSteps`（`MAX_STEPS=3`）が DB を叩き `DbRetrievalMaterial` を生成
4. **Evidence injection** — 素材を `=== DATABASE RECORDS ===` ブロックとしてユーザー文に埋込
5. **Finalization** — finalizer LLM が最終文を生成

**EN:** The previous design was roughly a 5-layer pipeline (see git history for details): (1) intent classification via a planner LLM emitting a `{kind, intent, userId, date, ...}` JSON; (2) route building by `buildIntentRoute` assembling fixed DB steps in code; (3) retrieval execution via `executeRetrievalSteps` (`MAX_STEPS=3`) producing `DbRetrievalMaterial`; (4) evidence injection of that material as a `=== DATABASE RECORDS ===` block inside the user message; (5) finalization by a finalizer LLM. This path is **deprecated** and no longer used.
