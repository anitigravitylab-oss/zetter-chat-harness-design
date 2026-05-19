# Zetter Chat Harness — 設計仕様書 / Architecture Spec

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

**Zetter**（[z-etter.com](https://z-etter.com)）は日本語の小規模 SNS です。このリポジトリは、Zetter に組み込まれた AI チャット機能「**ZYN（ジン）**」のハーネス設計文書を収録しています。機密情報（認証情報・本番データ・接続情報）は除いた設計資料のみを公開しています。

**Zetter** ([z-etter.com](https://z-etter.com)) is a small Japanese social network. This repository documents the architecture of **ZYN (ジン)**, the AI chat assistant built into Zetter. It contains design materials only — credentials, production data, and connection details are excluded.

> **実装ファイル / Implementation files**
> - `src/services/db-chat-harness-v2.ts` — メインハーネス / Main harness (routing, finalizer, thread context, rate limiting)
> - `src/services/db-retrieval-agent.ts` — プランナー・DB エグゼキューター / Planner and DB executor

---

## アーキテクチャ全体像 / Architecture Overview

```
[ユーザー入力 / User input]
      │
      ├─ 画像添付あり / Image attached ───────────────→ [Gemini 画像解析 / Image analysis] ──→ [直接回答 / Direct answer]
      │                                                                                               ↑
      ├─ 投稿検索 ON / DB search ON ──→ [プランナー / Planner]                                      │
      │                                       │                                                      │
      │                                       ├─ kind=db ──→ [DB エグゼキューター / DB Executor]    │
      │                                       │                          ↓                           │
      │                                       ├─ kind=needs_clarification ──→ [DB 検索ファイナライザー / DB Finalizer]
      │                                       │                                                      │
      │                                       └─ kind=no_db ─────────────────────────────────────→ ┘
      │
      └─ 投稿検索 OFF / DB search OFF ──────────────────────────────────────────────────────→ [直接回答 / Direct answer]
```

ユーザーが「投稿検索」チップを ON にしたときだけプランナーを起動する。OFF のままであれば、プランナーを呼ばず直接 LLM で回答する。

The planner is invoked only when the user explicitly enables the "投稿検索 (post search)" chip. When it is off, the request goes directly to the LLM without any planner call.

---

## 1. プランナー（Intent Classifier） / Planner

`classifyIntentViaLlm` in `db-retrieval-agent.ts`

### 起動条件 / Activation condition

`tools` 配列に `"db-search"` が含まれ、かつ画像添付がない場合のみ呼ばれる。  
Called only when `tools` includes `"db-search"` and no images are attached.

### モデルと設定 / Model and settings

| 項目 / Item | 値 / Value |
|---|---|
| モデル / Model | `deepseek-v4-flash` |
| `max_tokens` | 800 |
| `temperature` | 0 |
| タイムアウト / Timeout | 30,000ms |
| 出力 / Output | JSON のみ / JSON only (no explanation) |

### コンテキスト / Context

- **既知ユーザーリスト / Known users list**: 投稿実績のある全ユーザーの `userId` / `displayName` を DB から取得してプロンプトに含める / All users with posts are fetched from the DB and included in the prompt
- **今日の日付（JST）/ Today's date (JST)**
- **ビューワーの userId / Viewer's userId**
- **IMPORTANT ルール・Few-shot examples** をプロンプトに埋め込み / Embedded in the prompt

### 出力 JSON スキーマ / Output JSON schema

```json
{
  "kind": "db" | "no_db" | "needs_clarification",
  "reason": "日本語の理由（no_db / needs_clarification 時のみ） / Japanese reason (no_db / needs_clarification only)",
  "intent": "posts_list | user_lookup | topic_search | timeline_summary | user_analysis",
  "userId": "target userId (without @) or null",
  "date": "YYYY-MM-DD or null",
  "dateLabel": "今日 | 昨日 | 指定日 | null",
  "limit": 10,
  "personQuery": "person name or null",
  "query": "keyword or null",
  "isSelfReference": false,
  "isRecentOrLatest": false,
  "isThisWeek": false,
  "assumptions": ["interpretation assumptions in Japanese"]
}
```

### Intent 一覧 / Intent types

| intent | 用途 / Purpose |
|---|---|
| `posts_list` | 特定ユーザー・日付・件数での投稿取得。`isSelfReference=true` で自分の投稿 / Posts by user, date, or count. `isSelfReference=true` for the viewer's own posts |
| `user_lookup` | ユーザーアカウントを名前から検索 / Search user accounts by name |
| `topic_search` | キーワードで投稿を横断検索 / Search posts by keyword |
| `timeline_summary` | 期間・月・週など時間スコープ付きの全体概況 / Overall activity summary for a time period |
| `user_analysis` | 特定ユーザーの投稿傾向・人物分析 / Posting tendencies and personality analysis for a specific user |

### ステップ構築 / Step building (`buildIntentRoute`)

LLM の JSON 出力を受けて、**コードで**実際の DB 実行ステップを組み立てる。LLM はステップを直接返さない。

The LLM returns a classification JSON; **code** then assembles the actual DB execution steps. The LLM does not return steps directly.

---

## 2. DB エグゼキューター / DB Executor

`runDbRetrievalHarness` / `executeRetrievalSteps` in `db-retrieval-agent.ts`

LLM 呼び出しなし。純粋なコード実行。  
No LLM calls. Pure code execution.

### 実行上限 / Limits

| 定数 / Constant | 値 / Value | 意味 / Meaning |
|---|---|---|
| `MAX_STEPS` | 3 | 1 リクエストあたりの DB 呼び出し上限 / Max DB calls per request |
| `MAX_EVIDENCE` | 200 | evidence 配列の上限件数 / Max evidence entries |
| 取得上限 / Fetch limit | 200 | 1 ステップあたりのレコード取得上限 / Max records per step |

### 人物参照の2段階解決 / Two-phase person resolution

`personQuery` がある場合（表示名での指定など）は2段階で処理する。  
When a `personQuery` is present (e.g., a display name), resolution is two-phase:

1. まずユーザー候補検索（`retrieve_db_records` + `users`）/ First, search for user candidates
2. 候補が一意に解決できたら → 投稿取得 or ユーザー分析のフォローアップステップを追加 / If uniquely resolved → add a follow-up step for posts or user analysis
3. 候補が複数 → `needs_clarification` として返す / Multiple matches → return `needs_clarification`
4. 候補なし → `partial` として返す / No match → return `partial`

### マテリアル（DbRetrievalMaterial）/ Material

DB 実行結果をまとめた構造体。ファイナライザーにそのまま渡される。  
A struct that aggregates DB results and is passed directly to the finalizer.

```typescript
{
  status: "ok" | "partial" | "needs_clarification" | "no_db" | "error";
  intent: string;
  brief: { headline: string; bullets: string[]; suggestedTone?: string };
  assumptions: string[];
  coverage: { scope: string; viewerScoped: boolean; fromDate?; toDate?; limit?; complete: boolean };
  claims: Array<{ claim: string; confidence: "low"|"medium"|"high"; evidenceIds: string[] }>;
  evidence: Array<{ id: string; kind: "post"|"user"|"aggregate"; excerpt?; postId?; userId?; createdAt?; ... }>;
  missing: string[];
  cautions: string[];
  nextSearchSuggestions: string[];
}
```

---

## 3. DB 検索ファイナライザー / DB Search Finalizer

`streamDevChatMessage` 内 / inside `db-chat-harness-v2.ts`

`material.status !== "no_db"` の場合の最終回答生成。`ok` / `partial` / `error` / `needs_clarification` がここに入る。  
`no_db`、および DB 取得自体が失敗した場合（`material === null`）は直接回答パスへ落ちる。

Generates the final answer when `material.status !== "no_db"`. Handles `ok`, `partial`, `error`, and `needs_clarification`.  
`no_db` and retrieval failures (`material === null`) fall through to the direct-answer path.

### モデルと設定 / Model and settings

| 項目 / Item | 値 / Value |
|---|---|
| モデル / Model | `deepseek-v4-flash` |
| `reasoningEffort` | `"medium"` |
| `max_tokens` | 10,000 |

### システムプロンプト / System prompt (minimal)

```
You are ZYN (ジン), the chat assistant for Zetter SNS.
You have been given database records from Zetter.
Answer the user's question directly and naturally in Japanese based on the records.

## Rules
- Answer exactly what the user asked.
- If the target data exists but is empty or missing, say so clearly.
- Do not mention 'DB検索', 'evidence', 'harness', or any internal system terms.
- No preambles like「データを確認しました」. Start with the answer.
```

Zetter プラットフォーム情報・スレッドコンテキストはここには含まれない。  
Zetter platform information and thread context are not included here.

### ユーザーメッセージの構成 / User message structure

```
Recent conversation history (for context only):
(last 6 messages)

User's request: 「(user message)」

=== DATABASE RECORDS (read all, do not list to user) ===
(buildDbMaterialText output)
=== END OF RECORDS ===

Now write your interpretation in natural Japanese.
```

---

## 4. 直接回答 / Direct Answer (`buildSystemPrompt` path)

次の場合に使われる / Used when:
- 投稿検索 OFF / DB search is off
- 投稿検索 ON だが `no_db` 判定（または DB 取得エラー）/ DB search is on but result is `no_db` (or retrieval error)
- 画像添付あり（`imageContext` もユーザーメッセージ先頭に付与）/ Image attached (`imageContext` prepended to user message)

`needs_clarification` はここではなく DB 検索ファイナライザーを通る（`status !== "no_db"` のため）。  
`needs_clarification` goes to the DB finalizer, not here (`status !== "no_db"`).

### モデルと設定 / Model and settings

| 項目 / Item | 値 / Value |
|---|---|
| モデル / Model | `deepseek-v4-flash` |
| `reasoningEffort` | `"low"` |
| `max_tokens` | 指定なし / Not specified |

### システムプロンプトの構成 / System prompt structure (`buildSystemPrompt`)

1. **ZYN ペルソナ / ZYN persona**: Zetter 専属 AI、日本語優先、意見を持ってよい / Zetter's built-in AI, Japanese-first, allowed to have opinions

2. **Zetter プラットフォーム情報 / Zetter platform info**

   存在する機能 / Available features: 投稿（post/reply/repost）、引用、いいね、ブックマーク、ミュート、通知、全文検索、画像添付（最大4枚）、音楽共有（iTunes API）、コメント、バグ報告  
   posts (post/reply/repost), quote posts, likes, bookmarks, mutes, notifications, full-text search, image attachments (up to 4), music sharing (iTunes API), comments, bug reports
   - AI アカウント / AI accounts: `@zetachan`, `@zett`

   存在しない機能 / Features that do NOT exist: フォロー/フォロワー、ブロック（ミュートのみ）、DM、ハッシュタグ、非公開アカウント、認証バッジ、リスト、トレンドページ、アルゴリズムフィード、投票  
   follow/followers, block (mute only), DMs, hashtags, private accounts, verified badges, lists, trending page, algorithmic feed, polls

3. **回答長・一覧化ルール / Response length and listing rules**
   - 「全部見せて」と言われても全件ベタ出しせず、件数・期間・テーマを要約し代表例は3〜5件まで / When asked to "show all", summarize volume/date range/themes and highlight 3–5 examples rather than listing everything verbatim
   - 特定投稿を明示的に求めた場合のみ個別テキストを引用 / Only quote individual post text when the user explicitly asks for a specific post

4. **スレッドコンテキスト / Thread context** (`hasContext=true` の場合のみ末尾に追加 / appended only when `hasContext=true`)

---

## 5. スレッドコンテキスト管理 / Thread Context Management

### 目的 / Purpose

同一スレッドで複数回 DB 検索を行った場合、過去の検索結果を記憶して直接回答パスに活用する。  
When the user performs multiple DB searches in the same thread, past results are retained and surfaced in subsequent direct answers.

### 保存先 / Storage

`dev_chat_query_state` テーブル（PostgreSQL / D1）にスレッドごとに JSON で保存。DB 検索が成功するたびに `saveThreadContext` でエントリーを追記する。  
Stored as JSON per thread in the `dev_chat_query_state` table (PostgreSQL / D1). A new entry is appended by `saveThreadContext` after each successful DB search.

### データ構造 / Data structure

```json
{
  "entries": [
    {
      "seq": 1,
      "intent": "posts_list",
      "fetchedAt": "2026-05-19T10:00:00Z",
      "materialText": "(buildDbMaterialText output)",
      "active": true
    }
  ],
  "compressedSummary": {
    "createdAt": "2026-05-19T11:00:00Z",
    "coveredSeqs": [1, 2],
    "text": "(DeepSeek-compressed summary)"
  }
}
```

### 注入タイミング / Injection point

`buildSystemPrompt` の末尾に追加される。**DB 検索ファイナライザーには注入しない。**  
Appended to the end of `buildSystemPrompt`. **Not injected into the DB search finalizer.**

### 自動圧縮 / Auto-compression

アクティブエントリーの合計文字数が **20,000文字** を超えると、`done` イベント送信後にバックグラウンドで自動圧縮が走る。  
When the total character count of active entries exceeds **20,000**, compression runs in the background after the `done` event is sent (invisible to the user).

| 項目 / Item | 値 / Value |
|---|---|
| 圧縮モデル / Compression model | `deepseek-v4-flash` |
| `max_tokens` | 5,000 |
| 圧縮タイミング / Timing | `done` 送信後 / After `done` is sent |
| 閾値 / Threshold | 20,000文字 / characters |

---

## 6. 画像解析パス / Image Analysis Path

`buildGeminiImageContext` in `db-chat-harness-v2.ts`

画像が添付された場合は投稿検索を無効化し（`useDbSearch = false`）、Gemini で画像を解析する。  
When images are attached, DB search is disabled (`useDbSearch = false`) and images are analyzed by Gemini.

| ステップ / Step | 処理 / Process |
|---|---|
| 1 | `analyzeImagesWithGemini` で画像を解析 / Analyze images |
| 2 | 解析結果を `imageContext` テキストに変換 / Convert result to `imageContext` text |
| 3 | ユーザーメッセージの先頭に `imageContext` を付与して直接回答パスへ / Prepend `imageContext` to user message and route to direct answer |

Gemini API キーが未設定の場合は画像解析をスキップ（エラーにはならない）。  
If the Gemini API key is not configured, image analysis is skipped silently.

---

## 7. Progress フェーズ / Progress Phases

| フェーズ / Phase | 意味 / Meaning |
|---|---|
| `image-analysis` | Gemini 画像解析中 / Analyzing images with Gemini |
| `tool-call` | DB クエリ実行中 / Executing DB queries |
| `thinking` | LLM による回答生成待ち / Waiting for LLM response |
| `writing` | 回答テキストをストリーミング中 / Streaming response text |
| `saving` | 会話を DB に保存中 / Saving conversation to DB |

---

## 8. レート制限 / Rate Limiting

`dev_chat_rate_limit_reservations` テーブルで管理。  
Managed via the `dev_chat_rate_limit_reservations` table.

- リクエスト開始時に枠を予約 / Reserve budget at request start
- レスポンス完了後に実際のコスト（DeepSeek Flash の cache hit/miss/output トークン）で精算 / Settle with actual cost (DeepSeek Flash cache hit/miss/output tokens) after response completes

---

## 9. 実装ファイル / Implementation Files

| ファイル / File | 役割 / Role |
|---|---|
| `src/services/db-chat-harness-v2.ts` | メインハーネス。ルーティング、Gemini 連携、DB 検索ファイナライザー、直接回答、スレッドコンテキスト管理、レート制限 / Main harness: routing, Gemini integration, DB finalizer, direct answer, thread context, rate limiting |
| `src/services/db-retrieval-agent.ts` | プランナー（`classifyIntentViaLlm`）、ステップ構築（`buildIntentRoute`）、DB エグゼキューター / Planner (`classifyIntentViaLlm`), step builder (`buildIntentRoute`), DB executor |
| `src/services/zetter-chat-tools.ts` | DB クエリ実行の低レベル実装 / Low-level DB query execution |
| `public/chat-widget.js` | 投稿検索チップの UI、`toggle-tool` アクション / Post search chip UI, `toggle-tool` action |
