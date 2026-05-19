# Zetter Chat Harness V3 設計仕様書

> **前バージョン**: `docs/zetter-chat-harness-v2-spec.md`  
> **実装ファイル**: `src/services/db-chat-harness-v2.ts`, `src/services/db-retrieval-agent.ts`

---

## 概要

V3 の最大の設計変更は**カスタムDB検索（ユーザー起動型）**の導入と、**プランナー・ファイナライザー双方へのコンテキスト増量**の2点である。

V2 設計では「自動 gate → flash router → executor → finalizer」という多段パイプラインを想定していたが、V3 では**ユーザーが明示的に投稿検索を ON にした場合のみ**プランナーを起動するシンプルなアーキテクチャに転換した。

この転換により:

- ルーティング誤りによる不要な DB 呼び出しがゼロになる
- 雑談・挨拶が tool planning に流れる問題が構造的に解消される
- ユーザーが「今から DB を調べてほしい」という意図を明示できる

---

## アーキテクチャ全体像

```
[ユーザー入力]
      │
      ├─ 画像添付あり ──────────────────────────────→ [Gemini 画像解析] → [ファイナライザー]
      │
      ├─ 投稿検索 ON ──→ [プランナー (DeepSeek Flash)]
      │                          │
      │                    kind=db │ kind=no_db / needs_clarification
      │                          ↓                    ↓
      │                  [DB エグゼキューター]   [ファイナライザー (直接回答)]
      │                          │
      │                          ↓
      │                  [ファイナライザー (DB 結果付き)]
      │
      └─ 投稿検索 OFF ─────────────────────────────→ [ファイナライザー (直接回答)]
```

---

## 1. カスタムDB検索（ユーザー起動型）

### 設計思想

V2 設計では、入力メッセージを自動でルーティングして「DB が必要かどうか」を判定していた。しかしこの自動判定は次の問題を引き起こしていた。

- 「フォローしているユーザーを調べましょうか？」のような存在しない機能への言及
- 挨拶が分類タイムアウトを引き起こす（プランナーを呼ぶ必要がない）
- ユーザーの意図とルーティング結果がずれる

V3 では**ユーザーが「投稿検索」チップを ON にしたときだけ**プランナーを起動する。OFF のままであれば、プランナーを一切呼ばず直接 LLM で回答する。

### UI との対応

| 状態 | フロー |
|---|---|
| 投稿検索 OFF | 直接回答（プランナー呼び出しなし） |
| 投稿検索 ON + DB 不要な質問 | プランナー → `no_db` → 直接回答 |
| 投稿検索 ON + DB が必要な質問 | プランナー → `db` → DB 実行 → ファイナライザー |
| 画像添付あり | Gemini 解析 → 直接回答（投稿検索は自動 OFF） |

### 重要な制約

- 画像添付時は投稿検索を無効化する（`imageDataUrls.length > 0` のとき `useDbSearch = false`）
- 投稿検索 ON でも `no_db` / `needs_clarification` 判定の場合、DB を呼ばず直接回答する

---

## 2. プランナー（Intent Classifier）

### 役割

プランナーはユーザーメッセージを受け取り、「どんな DB 操作をすべきか」を JSON で返す小さな LLM 呼び出しである。生データの取得・回答生成は行わない。

### モデルと設定

| 項目 | 値 |
|---|---|
| モデル | `deepseek-v4-flash` |
| タイムアウト | 30,000ms |
| `max_tokens` | 800 |
| `temperature` | 0 |
| 出力 | JSON のみ（説明文なし） |

### 入力コンテキスト（増量ポイント）

V3 では次の情報をプランナーに渡す。これが V2 との大きな違いである。

#### 1. 既知ユーザーリスト（動的生成）

投稿実績のある全ユーザーの `userId` / `displayName` をDBから取得し、プロンプトに含める。

```
@ymd / ymd
@shachi / しゃち
@aoao_art / あおあおアート
...（全投稿者）
```

この情報により:
- 「しゃちさんの投稿見せて」→ `userId: "shachi"` と解決できる
- 存在しないユーザー名でも「知らない名前でも DB を検索する」という判断ができる

#### 2. Few-shot Examples

分類精度を上げるためのサンプルをプロンプトに埋め込む。

```
「4月の投稿まとめて」    → kind=db, intent=timeline_summary
「先月どんな話題があった？」→ kind=db, intent=timeline_summary
「田中さんの投稿見せて」  → kind=db, intent=user_lookup（known users に存在しなくても）
「自分の投稿見せて」     → kind=db, intent=posts_list, isSelfReference=true
「俺の最近の投稿」      → kind=db, intent=posts_list, isSelfReference=true
```

#### 3. IMPORTANT ルール（明示的な優先ルール）

```
- 人物名が含まれる場合（known users 外でも）→ kind=db で検索する
- 月・期間が投稿や活動に関連して含まれる場合 → kind=db にする
- 投稿検索を明示的に ON にしているユーザーの依頼 → デフォルトで kind=db
```

#### 4. 自己参照の認識

`isSelfReference` フィールド：  
`私 / 自分 / 俺 / 僕 / うち / 自分自身 / me / myself` を含む場合に `true` とし、`viewer.userId` に紐づけたクエリを実行する。

### 出力 JSON スキーマ

```json
{
  "kind": "db" | "no_db" | "needs_clarification",
  "reason": "brief Japanese reason (no_db / needs_clarification 時のみ)",
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
  "assumptions": ["Japanese interpretation assumptions"]
}
```

### Intent 一覧

| intent | 用途 |
|---|---|
| `posts_list` | 特定ユーザー・日付・件数で投稿を取得。`isSelfReference=true` で自分の投稿も |
| `user_lookup` | ユーザーアカウントを名前から検索 |
| `topic_search` | キーワード・トピックで投稿を横断検索 |
| `timeline_summary` | 期間・月・週など時間スコープ付きの全体概況 |
| `user_analysis` | 特定ユーザーの投稿傾向・人物分析 |

---

## 3. DB エグゼキューター

プランナーが出力した JSON を受け取り、実際に DB クエリを実行してマテリアルを構築する。

### 実行上限

| 定数 | 値 | 意味 |
|---|---|---|
| `MAX_STEPS` | 3 | 1 リクエストあたりの DB 呼び出し上限 |
| `MAX_EVIDENCE` | 200 | evidence 配列の上限件数 |
| 取得上限 | 200 | 1 ステップあたりのレコード取得上限 |

### マテリアル（DbRetrievalMaterial）

DB 実行結果をまとめた構造体。ファイナライザーにそのまま渡される。

```typescript
{
  status: "ok" | "partial" | "needs_clarification" | "no_db" | "error";
  intent: string;
  brief: {
    headline: string;
    bullets: string[];
    suggestedTone?: "plain" | "careful" | "ranked";
  };
  assumptions: string[];
  coverage: {
    scope: string;
    viewerScoped: boolean;
    fromDate?: string;
    toDate?: string;
    limit?: number;
    complete: boolean;
  };
  claims: Array<{ claim: string; confidence: "low"|"medium"|"high"; evidenceIds: string[] }>;
  evidence: Array<{ id: string; kind: "post"|"user"|"aggregate"; excerpt?: string; ... }>;
  missing: string[];
  cautions: string[];
  nextSearchSuggestions: string[];
}
```

---

## 4. ファイナライザー（最終回答生成）

### 役割

DB マテリアルを受け取り、ユーザーへの自然言語回答を生成する。

### システムプロンプトのコンテキスト増量（V3 の核心）

V2 設計では「DB 結果をそのまま投げて回答させる」という構造だったが、V3 では**ファイナライザーのシステムプロンプトを大幅に充実させた**。

#### 1. Zetter プラットフォーム情報

ファイナライザーに Zetter の全機能情報を与えることで、存在しない機能（フォロー、DM、ハッシュタグ等）を誤って案内することを防ぐ。

**存在する機能**:
- 投稿（post / reply / repost）、引用、いいね、ブックマーク
- ミュート（ブロックなし）、通知、全文検索、画像添付（最大4枚）
- 音楽共有（iTunes API）、コメント
- AI アカウント: `@zetachan`, `@zett`

**存在しない機能（明示的に「ない」と回答する）**:
- フォロー / フォロワー
- ブロック（ミュートのみ）
- DM
- ハッシュタグ
- 非公開アカウント
- 認証バッジ
- リスト
- トレンドページ
- アルゴリズムフィード
- 投票

#### 2. 回答長・一覧化ルール

大量投稿を全件ベタ出しする問題を防ぐため、明示的なルールをシステムプロンプトに含める。

```
- 投稿を全件リストしない。件数・期間・テーマを要約し、代表例は3〜5件まで。
- 「全部見せて」と言われても一覧ではなく要約で返す。
- 個別の投稿テキストを引用するのは、ユーザーが特定の投稿を明示的に求めた場合のみ。
```

#### 3. スレッドコンテキスト

同一スレッド内の過去 DB 検索結果をシステムプロンプトに注入する（後述のスレッドコンテキスト管理を参照）。

---

## 5. スレッドコンテキスト管理

### 目的

同一スレッドで複数回 DB 検索を行った場合、過去の検索結果を記憶して回答に活用する。

### 保存先

`dev_chat_query_state` テーブル（PostgreSQL / D1）にスレッドごとに JSON で保存。

### データ構造

```json
{
  "entries": [
    {
      "seq": 1,
      "intent": "posts_list",
      "fetchedAt": "2026-05-19T10:00:00Z",
      "materialText": "（DB結果テキスト）",
      "active": true
    }
  ],
  "compressedSummary": {
    "createdAt": "2026-05-19T11:00:00Z",
    "coveredSeqs": [1, 2],
    "text": "（要約テキスト）"
  }
}
```

### 自動圧縮

アクティブな DB 検索結果の合計が **20,000文字** を超えると、`done` イベント送信後にバックグラウンドで自動圧縮が走る。

| 項目 | 値 |
|---|---|
| 圧縮モデル | `deepseek-v4-flash` |
| 圧縮後の最大トークン | 5,000 |
| 圧縮タイミング | `done` 送信後（ユーザーには見えない） |
| 閾値 | 20,000文字 |

圧縮プロンプトは「重要な人物情報・投稿傾向・特徴的な発言を保持し、冗長な一覧は省略してコンパクトにまとめる」という方針。

### システムプロンプトへの組み込み

```
=== 過去のDB検索サマリー（圧縮済み） ===
（compressedSummary.text）

=== 最新のDB検索結果 ===
[DB検索 1/2] 取得日時: ... / intent: posts_list
（materialText）
```

---

## 6. 画像解析パス

画像が添付された場合は投稿検索をバイパスし、Gemini で画像を解析してからファイナライザーに渡す。

| ステップ | 処理 |
|---|---|
| 1 | `buildGeminiImageContext()` で画像を解析 |
| 2 | 解析結果をテキスト化して `imageContext` に格納 |
| 3 | ファイナライザーの入力に `imageContext` を付与 |
| 4 | `useDbSearch = false`（投稿検索は自動無効化） |

Gemini API キーが未設定の場合、画像解析はスキップされるがエラーにはならない。

---

## 7. レート制限

ユーザーごとに費用ベースのレート制限を適用する。

- リクエスト開始時に枠を予約
- レスポンス完了後に実際のコストで精算
- `dev_chat_rate_limit_reservations` テーブルで管理

---

## 8. Progress フェーズ

ユーザー向けの進行表示は次のフェーズで構成される。

| フェーズ | 意味 |
|---|---|
| `thinking` | プランナーによる意図分類中 |
| `tool-call` | DB クエリ実行中 |
| `image-analysis` | Gemini 画像解析中 |
| `writing` | ファイナライザーによる回答生成中 |
| `saving` | 会話をDBに保存中 |

---

## 9. V2 設計との主な相違点

| 観点 | V2 設計 | V3 実装 |
|---|---|---|
| DB 検索起動 | 自動 gate → router で判定 | ユーザーが明示的に ON |
| プランナーコンテキスト | 最小限のシステムプロンプト | 既知ユーザーリスト + Few-shot + IMPORTANT ルール |
| ファイナライザーコンテキスト | DB 結果のみ | Zetter 機能情報 + 回答ルール + スレッド履歴 |
| スレッド記憶 | 未定義 | 自動保存・自動圧縮（閾値 20,000 文字 → 5,000 トークンに圧縮） |
| tool call | Native tool_calls | なし（DB は直接クエリ） |
| モデル | flash + tool augment loop | flash 固定（プランナー + ファイナライザーの 2 call） |
| 自己参照 | reference resolution 層で解決 | プランナープロンプトの `isSelfReference` フィールドで解決 |
| 全件リスト問題 | 未対処 | ファイナライザープロンプトで明示的に抑制 |

---

## 10. 実装ファイル

| ファイル | 役割 |
|---|---|
| `src/services/db-chat-harness-v2.ts` | ハーネス本体。ルーティング、Gemini 連携、ファイナライザー、スレッドコンテキスト管理、レート制限 |
| `src/services/db-retrieval-agent.ts` | プランナー（intent classifier）と DB エグゼキューター |
| `src/services/zetter-chat-tools.ts` | DB クエリ実行の低レベル実装 |
| `public/chat-widget.js` | 投稿検索チップの UI、toggle-tool アクション |

---

## 11. 既知の課題と今後の方向性

### 現在の課題

1. **プランナーの応答時間**  
   - プロンプトが大きくなったため（既知ユーザーリスト含む）、分類に 10〜30 秒かかる場合がある
   - 挨拶など DB 不要なメッセージでも投稿検索 ON の場合はプランナーを呼んでしまう

2. **no_db / needs_clarification 時の待ち時間**  
   - `no_db` と分かるまでプランナーを通すため、シンプルな質問でも 10〜20 秒かかる

### 検討中の改善

- **軽量プリフィルター**: プランナー呼び出し前に、DB が不要と判断できるメッセージをヒューリスティックで弾く高速パス
- **プランナーとファイナライザーの並列実行**: プランナーの判定を待たずにファイナライザーも並列起動し、`no_db` なら直接回答を返す
