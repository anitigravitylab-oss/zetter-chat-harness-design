# Zetter Chat 安全拡張 仕様書

## 目的

Zetter 内の情報を理解するチャットを、**安全に**より便利にする。

前提:

- チャットは Cloud Run 上で動作する
- 追加課金は原則 **DeepSeek のみ**
- ユーザーがチャット経由で **サーバー内ファイル・コード・秘密情報** に触れられてはいけない
- できるだけ **Zetter の DB 上の公開/認可済み情報** を使って賢くしたい

## 結論

安全に許可するのは **read-only の Zetter 専用ツール** までとする。

チャットは以下を **禁止** する:

- 任意 SQL 実行
- ファイル読み取り
- ソースコード閲覧
- シェル実行
- 任意 URL fetch
- 環境変数 / secret / 設定値の読み取り

チャットに与える能力は、**固定クエリを持つアプリ専用関数群** だけに限定する。

---

## セキュリティ境界

### 許可するもの

1. Zetter の DB に入っている情報のうち、**ユーザーが本来 UI 上で見える情報**
2. その情報を使った **検索、要約、集計、比較、説明**
3. 認可済み範囲での **read-only 分析**

### 許可しないもの

1. 任意 SQL
2. 任意ファイル/コード読み取り
3. shell / bash / python / git 実行
4. 内部 URL や metadata endpoint への到達
5. Cloud Run 環境変数、secret 名、token 値、DB 接続情報の返却
6. 管理者限定投稿や非公開相当データの bypass 取得

### 守るべき原則

1. **Tool は固定 schema + 固定 query**
2. **viewer 権限で毎回フィルタ**
3. **返す件数を制限**
4. **raw dump より要約中心**
5. **内部 ID や秘匿メタ情報は返さない**
6. **Zetter の app domain 以外の情報源を参照しない**

---

## 非目標

以下はこの仕様の対象外:

- サーバー運用補助
- 開発支援エージェント化
- デプロイ、コード編集、DB migration 実行
- repo / filesystem / terminal を読む汎用 assistant 化
- 任意の管理者業務自動化

---

## 採用アーキテクチャ

### 基本方針

- DeepSeek 本体は「推論とツール選択」だけ担当
- Zetter 固有データ参照は `src/services/zetter-chat-tools.ts` に集約
- 各 tool は backend 内の専用 service を呼ぶ
- DB には app 側 service 関数経由でアクセスする
- 返却値は JSON で小さく構造化する

### 禁止アーキテクチャ

- `run_sql(query)`
- `read_file(path)`
- `run_bash(command)`
- `fetch_url(url)`
- `inspect_env(name)`

---

## 追加実装するツールの範囲

## A. 既存強化

### 1. `search_users`

目的:

- あいまいな日本語から対象ユーザーを見つけやすくする

強化内容:

- `display_name`
- `user_id`
- `bio`
- 前方一致/部分一致の優先順
- 完全一致時の優先表示
- 漢字/かな/英字表記の候補提示

返却例:

```json
{
  "users": [
    {
      "userId": "sora",
      "displayName": "そら",
      "bio": "",
      "avatarUrl": "/api/avatar/sora?v=..."
    }
  ]
}
```

### 2. `get_user_profile`

目的:

- 特定ユーザーの基本情報を確実に返す

返却項目:

- `userId`
- `displayName`
- `bio`
- `avatarUrl`
- `createdAt`

### 3. `list_user_posts`

目的:

- 「この人どんな人？」に答える材料を返す

返却項目:

- 最近の投稿一覧
- 投稿本文
- 投稿日時
- 種別 (`post` / `reply` / `repost`)

制限:

- 最大 10 件

### 4. `summarize_user_activity`

目的:

- 活動傾向を要約用に返す

返却項目:

- 投稿数
- 直近投稿サンプル
- 対象期間

---

## B. 新規追加ツール

### 5. `list_user_topics`

目的:

- ユーザーが最近よく話している話題を返す

入力:

- `userId`
- `limit` (max 10)

出力:

- 頻出語句
- 代表投稿
- 最近の話題要約

備考:

- 形態素解析の重い追加依存は避ける
- 初期版は簡易トークン抽出でよい

### 6. `compare_users`

目的:

- 2 ユーザーの雰囲気や活動差を見る

入力:

- `userIdA`
- `userIdB`

出力:

- 投稿量差
- 最近の話題差
- 文体傾向差

### 7. `get_user_interaction_summary`

目的:

- 特定ユーザーが誰とよく絡むかを返す

入力:

- `userId`

出力:

- よく返信する相手
- よく返信される相手
- よく言及する相手

### 8. `summarize_thread`

目的:

- ある投稿スレッド全体の流れを短く要約する

入力:

- `postId`

出力:

- 元投稿の要点
- 返信の論点
- スレッドの雰囲気

### 9. `detect_burst_topics`

目的:

- 今日/最近急に伸びている話題を返す

入力:

- `date` or `range`

出力:

- 目立つキーワード
- 代表投稿
- 投稿数概況

### 10. `explain_user_match`

目的:

- 「なぜこのユーザー候補が出たのか」を説明する

入力:

- `query`
- `userId`

出力:

- display name 一致
- userId 一致
- bio 一致
- 関連投稿文脈

---

## 将来追加してよい範囲

以下は安全寄りなので将来追加可能:

- `list_frequent_repliers`
- `list_frequent_mentions`
- `get_user_posting_hours`
- `get_user_posting_days`
- `summarize_timeline_trends`
- `list_similar_users`

---

## 追加してはいけないツール

以下は実装禁止:

- `run_sql`
- `read_codebase_file`
- `read_server_file`
- `execute_shell`
- `fetch_internal_url`
- `list_env_vars`
- `dump_schema`
- `read_logs`

理由:

- DB 理解を超えて **サーバー理解** に入るため
- prompt injection や model jailbreak で被害面積が大きい
- ユーザーが本来 UI で見えない情報へ到達できる

---

## Prompt / Tool 運用方針

### 1. Zetter ツールは常時有効

理由:

- 「そらというアカウントについて調べて」
- 「どんな人？」
- 「最近何してる？」

のような曖昧依頼でも毎回 DB 検索に進ませるため。

### 2. 回答ポリシー

チャットは以下を優先する:

1. まずユーザー候補を特定
2. 必要ならプロフィール取得
3. 必要なら最近の投稿取得
4. その根拠をもとに説明

### 3. 不明時の振る舞い

- 候補が複数いる場合は候補提示
- 候補が 0 件なら 0 件と明示
- 憶測だけで「こういう人だと思う」と断定しない

---

## 安全対策 詳細

### 認可

- 既存 app の viewer/session を必ず使う
- 投稿取得は既存の可視性判定を通す
- 管理者限定投稿は一般ユーザーには返さない

### 出力制限

- ユーザー一覧は max 10
- 投稿一覧は max 10〜20
- 要約は max 3〜5 bullet 相当
- 長文全文 dump は避ける

### データ最小化

返してよい:

- `userId`
- `displayName`
- `bio`
- `avatarUrl`
- `createdAt`
- 投稿本文
- 投稿公開メタ

返してはいけない:

- internal user UUID
- session 情報
- token 類
- DB 主キーの内部用途値
- secret 名や env 名

### モデル防御

- tool description に「Zetter 内の公開/認可済み情報のみ」と明記
- tool result は簡潔な JSON のみ
- assistant 側で「コード・ファイル・環境は見えない」と system prompt に明記

---

## 実装対象ファイル

- `src/services/zetter-chat-tools.ts`
- `src/services/user.ts`
- `src/services/post.ts`
- `src/services/deepseek-chat.ts`
- 必要なら `src/types.ts`

原則として:

- 新規機能は **service 層に追加**
- route 層へ汎用 DB 開放はしない

---

## 実装フェーズ

### Phase 1

- `search_users` 強化
- `get_user_profile` 強化
- `list_user_posts` 強化
- `summarize_user_activity` 強化
- Zetter tools 常時有効化

### Phase 2

- `list_user_topics`
- `compare_users`
- `get_user_interaction_summary`
- `summarize_thread`

### Phase 3

- `detect_burst_topics`
- `explain_user_match`
- `summarize_timeline_trends`

---

## 受け入れ条件

1. 「そらというアカウントについて調べて」で `sora / そら` を引ける
2. 「どんな人？」でプロフィールと投稿を根拠に答えられる
3. 一般ユーザーから管理者限定投稿は見えない
4. チャット経由でコード、ファイル、shell、env に触れられない
5. 任意 SQL を model に渡していない
6. 追加課金は DeepSeek 以外に発生しない

---

## 追加課金方針

許可:

- DeepSeek API
- 既存 Cloud Run 実行コスト

避ける:

- 新しい外部 SaaS
- ベクトル DB
- 外部検索 API
- 追加 LLM ベンダー依存

画像解析の Gemini は既存機能として残せるが、テキスト系の Zetter 理解強化は **DeepSeek + 自前 tool** で完結させる。

---

## 最終方針

Zetter chat は **汎用 agent** ではなく、
**「Zetter の公開/認可済み情報を読む専用 analyst」** として強くする。

強化ポイントは prompt の長文化ではなく、

- 安全な read-only tool を増やす
- tool を使う判断を安定化する
- 候補提示と根拠説明を改善する

の 3 点とする。
