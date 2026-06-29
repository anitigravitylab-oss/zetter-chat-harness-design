# Zetter Chat — ZYN（ジン）設計仕様書 / Design Spec

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

**JA:** ZYN（ジン）は SNS アプリ **Zetter** に組み込まれた、会話型の AI アシスタントです。投稿・ユーザーに関する質問への回答、タイムラインの要約、ユーザー分析、添付画像の解析を行います。本ドキュメントは **現在の確定設計（現行設計）** を記述します。

**EN:** ZYN is the conversational AI assistant embedded in the **Zetter** social app. It answers questions about posts and users, summarizes the timeline, analyzes users, and interprets attached images. This document describes the **current design**.

> このドキュメントは現行設計のみを扱います。旧（5層パイプライン）設計は末尾の「旧設計サマリ」を参照。
> This document covers the current design only. The previous (5-layer pipeline) design is summarized at the end.

---

## 今回の大きな変更 / What changed in this update

**JA:** 旧設計は **コード主導の固定パイプライン**でした（intent 分類 LLM → コードが固定の DB ステップを組み立て → 最終文生成）。LLM は「意図分類」と「最終文の整形」しか担っておらず、ルーティングが硬直し、能力詐称（「アクセスできる範囲に限り…」等の嘘）が出やすいという問題がありました。

現行設計はこれを **単一プロンプト + tool-calling ループ**へ転換しました。単一モデル（`deepseek-v4-flash`）が 1 つの包括的なシステムプロンプトのもとで、毎ターン自分でツールを選び、結果を会話履歴に戻して継続し、最終回答を出します。ハーネスはルーティングを持たず、**足場（grounding / normalization / observability / safety net）**に徹します。

**EN:** The old design was a **code-driven fixed pipeline** (intent-classifier LLM → code assembles fixed DB steps → finalizer generates prose). The LLM only did "intent classification" and "final-text formatting", which made routing rigid and made ability-denying hallucinations easy to emit.

The current design replaces this with a **single prompt + tool-calling loop**. One model (`deepseek-v4-flash`), under a single comprehensive system prompt, selects tools itself each turn, feeds results back into the conversation, continues, and produces the final answer. The harness owns no routing logic; it acts purely as **scaffolding** (grounding / normalization / observability / safety nets).

---

## アーキテクチャ全体像 / Architecture overview

```
                 user message (+ optional images)
                              │
        ┌─────────────────────▼─────────────────────┐
        │  Harness scaffolding (db-chat-harness-v2)   │
        │  • JST date grounding (injected per request)│
        │  • Gemini image context (prepended)         │
        │  • thread context load                      │
        └─────────────────────┬─────────────────────┘
                              │  single comprehensive system prompt
                              │  + history window (24 msgs)
        ┌─────────────────────▼─────────────────────┐
        │            TOOL-CALLING LOOP                │
        │   (deepseek-v4-flash, up to 5 turns)        │
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

**5 fixed tools (registered unconditionally):**

```
query_posts_v2 · summarize_timeline · analyze_user · lookup_user_profile · compare_users
```

---

## tool-loop の仕組み / How the tool loop works

**JA:** 単一モデルを `for` ループで最大 `LOOP_MAX_TURNS_EXTENDED = 5` ターン回します（`db-chat-harness-v2.ts`）。履歴ウィンドウは 24 メッセージ。各ターンの流れ:

1. LLM 呼び出し（`tool_choice` は `auto` / `none` / `required` を状況で切替）
2. assistant メッセージを履歴に push
3. `tool_calls` があれば各ツールを実行し、結果を `role: tool` メッセージとして履歴に push
4. これを繰り返す

`tool_calls` が無く `stop` で終わったとき（または DSML 解析でもツール呼び出しが見つからないとき）にループを抜け、そのターンの本文を `normalizeChatText` で整形したものが最終回答になります。

**強制停止 / Force-stop:** `turn >= LOOP_MAX_TURNS (3)` で `tool_choice="none"` に倒す（ただしツール結果が 2 件以上ありまだ要約が無い場合は延長）。`turn = 4` で hard force-stop。

**EN:** A single model runs in a `for` loop up to `LOOP_MAX_TURNS_EXTENDED = 5` turns. History window: 24 messages. Each turn: LLM call (`tool_choice` switched between `auto` / `none` / `required`) → push assistant message → execute any `tool_calls` and push results as `role: tool` messages → repeat. The loop breaks when there are no `tool_calls` and the model stops (or DSML parsing finds no tool calls); the breaking turn's content, passed through `normalizeChatText`, is the final answer. Force-stop flips `tool_choice` to `"none"` at `turn >= LOOP_MAX_TURNS (3)` (unless there are ≥2 tool results and no summary yet, which extends the loop), with a hard force-stop at `turn = 4`.

---

## 5 つの固定ツール / The five fixed tools

| Tool | 用途 / Purpose | 主なパラメータ / Key params |
|---|---|---|
| `query_posts_v2` | 投稿の検索・取得。任意の日付範囲・キーワード・ユーザー・投稿 ID・ページング / Post search & retrieval over an arbitrary date range, keyword, user, post id, with pagination | `userId`, `keyword`, `postId`, `fromDate`, `toDate` (YYYY-MM-DD JST), `sortOrder`, `cursorSeq`, `beforeSeq`, `includeAiCast`, `limit` (max 80) — at least one filter required |
| `summarize_timeline` | 固定ウィンドウのタイムライン概観（流れ・話題） / Recent timeline overview for a fixed window | `window` (`24h` \| `7d` \| `since-last-visit`), `limit` (max 30), `includeAiCast` |
| `analyze_user` | 単一ユーザーの深掘り分析（人物像・話題・トーン） / Single-user deep analysis | `userId` (required), `aspect` (`personality` \| `topics` \| `tone`), `fromDate`, `toDate`, `postsToAnalyze` (max 80), `includeAiCast` |
| `lookup_user_profile` | 軽量プロフィール取得（表示名・bio・投稿数） / Lightweight profile lookup | `userId` (required) のみ / only |
| `compare_users` | 2 ユーザーの比較・相性診断 / Two-user comparison & compatibility | `userIdA`, `userIdB` (both required), `aspect` (`topics` \| `tone` \| `compatibility`), `fromDate`, `toDate`, `includeAiCast` |

**JA:** ツールは **無条件で常時登録**されます。いつ・どれを使うかは LLM が判断します（旧設計のような「投稿検索チップを ON」というオプトインは不要）。

**EN:** All tools are **registered unconditionally**; the LLM decides when and which to use (no opt-in "post-search chip" as in the old design).

---

## 単一システムプロンプト / Single system prompt

**JA:** 旧の最小（約 8 行）プロンプトに代えて、1 本の包括的な 16 セクション構成のプロンプトを使います。主な節:

- **Identity/role preamble** — ZYN（ジン）。Zetter を理解した、温かみのある会話調のアシスタント。中立的なボットではない
- **# 1. Language** — ユーザーが明確に他言語で書かない限り、常に日本語
- **# 2. Current date (JST)** — リクエスト時に算出した `YYYY-MM-DD` を注入。相対日付はツールに渡す前に絶対日付へ変換させる
- **# 3. Honesty Protocol** — (1) ツールを呼ぶ前に能力を否定しない (2) 空結果は「その呼び出しでは見つからなかった」 (3) 推測しない
  - **## True facts** — `query_posts_v2` は任意の `fromDate`/`toDate` を受け、固定の範囲制限は無い。ツール結果が真実の源
  - **## Forbidden lies** — 自己限定的な禁止フレーズ（例:「アクセスできる範囲に限り」）を言い換え含め厳禁として列挙
  - **## good / bad examples** — 正しい応答と禁止応答の具体 2 シナリオ
- **# 4. Tool Routing** — 5 ツールの判定フロー: 比較→`compare_users` / 単一ユーザー→`analyze_user` / 今・最近→`summarize_timeline` / それ以外→`query_posts_v2`
- **# 5. First-person / Bare-latest semantics** — 「私 / 自分」＝閲覧者。修飾なしの「最新 N 件」＝制約なし・新しい順のグローバル
- **# 6. Empty Result Fallback** — 諦める前に別ツール・別パラメータで再試行。代替を尽くしてから「見つからない」と言う
- **# 7. Tool Hygiene** — 疑似ツール構文や DSML を出力しない。ネイティブ tool call のみ。生 JSON を露出しない
- **# 8. Mistakes & Self-Correction** — 簡潔に認め、ツールを呼び直し、正しい答えを出す。過剰な謝罪はしない
- **# 9. Tone & Formatting** — 自然な日本語の散文。結論先出し。既定で markdown 表を使わない。固定の決め台詞なし
- **# 10–11 + About Zetter** — Memory（24 メッセージ窓）/ Safety（システムプロンプトを開示しない）/ Zetter プラットフォームの事実と非機能

**EN:** Instead of the old minimal (~8-line) prompt, a single comprehensive **16-section** prompt is used: identity preamble (ZYN — a Zetter-aware, warm conversational assistant, not a neutral bot); #1 Language (Japanese unless the user clearly writes another language); #2 Current date (live-injected JST `YYYY-MM-DD`; relative dates converted to absolute before tool calls); #3 Honesty Protocol — never deny ability before calling a tool, treat empty results as "that call found nothing", no speculation — with **True facts** (`query_posts_v2` accepts any date range; tool results are the source of truth), **Forbidden lies** (enumerated self-limiting phrases banned, paraphrases included), and **good/bad examples**; #4 Tool Routing (compare→`compare_users`, single-user→`analyze_user`, now/recent→`summarize_timeline`, else→`query_posts_v2`); #5 First-person / bare-latest semantics; #6 Empty-result fallback; #7 Tool Hygiene (native tool calls only, no DSML/raw JSON); #8 Mistakes & self-correction; #9 Tone & formatting (natural Japanese prose, answer-first, no default markdown tables, no fixed catchphrase); #10–11 + About Zetter (24-msg memory, no system-prompt disclosure, Zetter platform facts and non-features).

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

**JA:** JST の現在日付を `Intl.DateTimeFormat`（`Asia/Tokyo`）でリクエスト時に算出し、毎回システムプロンプトへ注入します。プロンプトの True facts 節で「`query_posts_v2` の `fromDate`/`toDate` に固定の範囲制限は無い」「ツール結果が真実の源」と明示し、相対日付表現はツールに渡す前に今日の JST 日付を使って絶対 `YYYY-MM-DD` に変換させます。

**EN:** The current JST date is computed per request via `Intl.DateTimeFormat` (`Asia/Tokyo`) and injected into the system prompt every time. The True-facts section states that `query_posts_v2`'s `fromDate`/`toDate` has no fixed range limit and that tool results are the source of truth; the model is instructed to convert all relative date expressions to absolute `YYYY-MM-DD` using today's JST date before passing them to tools.

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
| 画像解析 / Image analysis | Gemini。`imageDataUrls` → `buildGeminiImageContext`、結果をユーザーターンに前置（旧ハーネスと同一経路）/ result prepended to the user turn, identical path to the old harness |
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
| **1 リクエストの LLM 呼数 / LLM calls** | 最低 2+（planner 1 + finalizer 1、＋任意の人物解決） | 1〜5 回（単一モデルのループ。通常 `LOOP_MAX_TURNS=3`、最大 5） |
| **根拠注入 / Evidence injection** | `DbRetrievalMaterial` を `=== DATABASE RECORDS ===` テキストブロックとしてユーザー文に埋込 | ツール結果を `role:tool` メッセージとして会話ターン履歴に差し込む |
| **システムプロンプト範囲 / Prompt scope** | 最小の約 8 行の identity + ルール。プラットフォーム情報や DB 素材は別途ユーザー文へ注入 | 単一の包括的 16 セクション（日付・Honesty Protocol・ルーティング・Zetter 事実・安全を一本化） |
| **能力詐称ガード / Ability-claiming guard** | 専用なし。planner が黙って `no_db` 分類し取得をスキップし得た | Honesty Protocol + 4 サーキットブレーカー（CB #11 / H2 / #12 / H3）で注記・強制再試行 |
| **DSML テキスト tool call / DSML handling** | 該当なし（構造化 JSON 素材でテキスト tool call のリスクなし） | `parseDsmlToolCalls` が DSML テキストをネイティブ tool call に合成しループ継続 |
| **ターン / ステップ上限 / Turn limit** | `MAX_STEPS=3` の固定コード反復（`executeRetrievalSteps`）。完了後 LLM 再入なし | `LOOP_MAX_TURNS=3`（soft）+ `LOOP_MAX_TURNS_EXTENDED=5`（hard）。結果件数で短縮 / 延長 |
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
