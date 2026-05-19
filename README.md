# Zetter Chat Harness 設計仕様書

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

このリポジトリには、Zetter chat harness の**機密情報を除いた設計文書**を収録しています。  
This repository contains **sanitized architecture and design documents** for the Zetter chat harness.

> **実装ファイル**
> - `src/services/db-chat-harness-v2.ts` — メインハーネス（ルーティング・ファイナライザー・スレッドコンテキスト管理・レート制限）
> - `src/services/db-retrieval-agent.ts` — プランナー（intent 分類）と DB エグゼキューター

---

## アーキテクチャ全体像

```
[ユーザー入力]
      │
      ├─ 画像添付あり ──────────────────────────────────→ [Gemini 画像解析] ──→ [直接回答]
      │                                                                               ↑
      ├─ 投稿検索 ON ──→ [プランナー]                                                │
      │                       │                                                      │
      │                       ├─ kind=db ──→ [DB エグゼキューター]                  │
      │                       │                      ↓                               │
      │                       ├─ kind=needs_clarification ──→ [DB 検索ファイナライザー]
      │                       │                                                      │
      │                       └─ kind=no_db ──────────────────────────────────────→ ┘
      │
      └─ 投稿検索 OFF ────────────────────────────────────────────────────────────→ [直接回答]
```

ユーザーが「投稿検索」チップを ON にしたときだけプランナーを起動する。
OFF のままであれば、プランナーを呼ばず直接 LLM で回答する。

---

## 1. プランナー（Intent Classifier）

`classifyIntentViaLlm` in `db-retrieval-agent.ts`

### 起動条件

`tools` 配列に `"db-search"` が含まれ、かつ画像添付がない場合のみ呼ばれる。

### モデルと設定

| 項目 | 値 |
|---|---|
| モデル | `deepseek-v4-flash` |
| `max_tokens` | 800 |
| `temperature` | 0 |
| タイムアウト | 30,000ms |
| 出力 | JSON のみ（説明文なし） |

### コンテキスト

- **既知ユーザーリスト**: 投稿実績のある全ユーザーの `userId` / `displayName` を DB から取得してプロンプトに含める
- **今日の日付（JST）**
- **ビューワーの userId**
- **IMPORTANT ルール・Few-shot examples** をプロンプトに埋め込み

### 出力 JSON スキーマ

```json
{
  "kind": "db" | "no_db" | "needs_clarification",
  "reason": "日本語の理由（no_db / needs_clarification 時のみ）",
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
  "assumptions": ["日本語の解釈前提"]
}
```

### Intent 一覧

| intent | 用途 |
|---|---|
| `posts_list` | 特定ユーザー・日付・件数での投稿取得。`isSelfReference=true` で自分の投稿 |
| `user_lookup` | ユーザーアカウントを名前から検索 |
| `topic_search` | キーワードで投稿を横断検索 |
| `timeline_summary` | 期間・月・週など時間スコープ付きの全体概況 |
| `user_analysis` | 特定ユーザーの投稿傾向・人物分析 |

### ステップ構築（`buildIntentRoute`）

LLM の JSON 出力を受けて、**コードで**実際の DB 実行ステップを組み立てる。LLM はステップを直接返さない。

---

## 2. DB エグゼキューター

`runDbRetrievalHarness` / `executeRetrievalSteps` in `db-retrieval-agent.ts`

LLM 呼び出しなし。純粋なコード実行。

### 実行上限

| 定数 | 値 | 意味 |
|---|---|---|
| `MAX_STEPS` | 3 | 1 リクエストあたりの DB 呼び出し上限 |
| `MAX_EVIDENCE` | 200 | evidence 配列の上限件数 |
| 取得上限 | 200 | 1 ステップあたりのレコード取得上限 |

### 人物参照の2段階解決

`personQuery` がある場合（表示名での指定など）は2段階で処理する:

1. まずユーザー候補検索（`retrieve_db_records` + `users`）
2. 候補が一意に解決できたら → 投稿取得 or ユーザー分析のフォローアップステップを追加
3. 候補が複数 → `needs_clarification` として返す
4. 候補なし → `partial` として返す

### マテリアル（DbRetrievalMaterial）

DB 実行結果をまとめた構造体。ファイナライザーにそのまま渡される。

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

## 3. DB 検索ファイナライザー

`streamDevChatMessage` 内（`db-chat-harness-v2.ts`）

`material.status !== "no_db"` の場合の最終回答生成。`ok` / `partial` / `error` / `needs_clarification` がここに入る。  
`no_db`、および DB 取得自体が失敗した場合（`material === null`）は直接回答パスへ落ちる。

### モデルと設定

| 項目 | 値 |
|---|---|
| モデル | `deepseek-v4-flash` |
| `reasoningEffort` | `"medium"` |
| `max_tokens` | 10,000 |

### システムプロンプト（シンプル）

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

### ユーザーメッセージの構成

```
Recent conversation history (for context only):
（直近6件の会話履歴）

User's request: 「（ユーザーのメッセージ）」

=== DATABASE RECORDS (read all, do not list to user) ===
（buildDbMaterialText の出力）
=== END OF RECORDS ===

Now write your interpretation in natural Japanese.
```

---

## 4. 直接回答（`buildSystemPrompt` パス）

次の場合に使われる:
- 投稿検索 OFF
- 投稿検索 ON だが `no_db` 判定（または DB 取得エラー）
- 画像添付あり（`imageContext` もユーザーメッセージ先頭に付与）

`needs_clarification` はここではなく DB 検索ファイナライザーを通る（`status !== "no_db"` のため）。

### モデルと設定

| 項目 | 値 |
|---|---|
| モデル | `deepseek-v4-flash` |
| `reasoningEffort` | `"low"` |
| `max_tokens` | 指定なし |

### システムプロンプト（`buildSystemPrompt`）の構成

1. **ZYN ペルソナ**：Zetter 専属 AI、日本語優先、意見を持ってよい
2. **Zetter プラットフォーム情報**

   存在する機能: 投稿（post/reply/repost）、引用、いいね、ブックマーク、ミュート、通知、全文検索、画像添付（最大4枚）、音楽共有（iTunes API）、コメント、バグ報告
   - AI アカウント: `@zetachan`, `@zett`

   存在しない機能（「ない」と明示する）: フォロー/フォロワー、ブロック（ミュートのみ）、DM、ハッシュタグ、非公開アカウント、認証バッジ、リスト、トレンドページ、アルゴリズムフィード、投票

3. **回答長・一覧化ルール**
   - 「全部見せて」と言われても全件ベタ出しせず、件数・期間・テーマを要約し代表例は3〜5件まで
   - 特定投稿を明示的に求めた場合のみ個別テキストを引用

4. **スレッドコンテキスト**（後述。`hasContext=true` の場合のみ末尾に追加）

---

## 5. スレッドコンテキスト管理

### 目的

同一スレッドで複数回 DB 検索を行った場合、過去の検索結果を記憶して直接回答パスに活用する。

### 保存先

`dev_chat_query_state` テーブル（PostgreSQL / D1）にスレッドごとに JSON で保存。

DB 検索が成功するたびに `saveThreadContext` でエントリーを追記する。

### データ構造

```json
{
  "entries": [
    {
      "seq": 1,
      "intent": "posts_list",
      "fetchedAt": "2026-05-19T10:00:00Z",
      "materialText": "（buildDbMaterialText の出力）",
      "active": true
    }
  ],
  "compressedSummary": {
    "createdAt": "2026-05-19T11:00:00Z",
    "coveredSeqs": [1, 2],
    "text": "（DeepSeek による圧縮テキスト）"
  }
}
```

### 注入タイミング

`buildSystemPrompt` の末尾に追加される。**DB 検索ファイナライザーには注入しない。**

### 自動圧縮

アクティブエントリーの合計文字数が **20,000文字** を超えると、`done` イベント送信後にバックグラウンドで自動圧縮が走る。

| 項目 | 値 |
|---|---|
| 圧縮モデル | `deepseek-v4-flash` |
| `max_tokens` | 5,000 |
| 圧縮タイミング | `done` 送信後（ユーザーには見えない） |
| 閾値 | 20,000文字 |

---

## 6. 画像解析パス

`buildGeminiImageContext` in `db-chat-harness-v2.ts`

画像が添付された場合は投稿検索を無効化し（`useDbSearch = false`）、Gemini で画像を解析する。

| ステップ | 処理 |
|---|---|
| 1 | `analyzeImagesWithGemini` で画像を解析 |
| 2 | 解析結果を `imageContext` テキストに変換 |
| 3 | ユーザーメッセージの先頭に `imageContext` を付与して直接回答パスへ |

Gemini API キーが未設定の場合は画像解析をスキップ（エラーにはならない）。

---

## 7. Progress フェーズ

| フェーズ | 意味 |
|---|---|
| `image-analysis` | Gemini 画像解析中 |
| `tool-call` | DB クエリ実行中（投稿検索 ON で実行前） |
| `thinking` | LLM による回答生成待ち |
| `writing` | 回答テキストをストリーミング中 |
| `saving` | 会話を DB に保存中 |

---

## 8. レート制限

`dev_chat_rate_limit_reservations` テーブルで管理。

- リクエスト開始時に枠を予約
- レスポンス完了後に実際のコスト（DeepSeek Flash の cache hit/miss/output トークン）で精算

---

## 9. 実装ファイル

| ファイル | 役割 |
|---|---|
| `src/services/db-chat-harness-v2.ts` | メインハーネス。ルーティング、Gemini 連携、DB 検索ファイナライザー、直接回答、スレッドコンテキスト管理、レート制限 |
| `src/services/db-retrieval-agent.ts` | プランナー（`classifyIntentViaLlm`）、ステップ構築（`buildIntentRoute`）、DB エグゼキューター |
| `src/services/zetter-chat-tools.ts` | DB クエリ実行の低レベル実装 |
| `public/chat-widget.js` | 投稿検索チップの UI、`toggle-tool` アクション |
