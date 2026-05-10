# Zetter Chat Reference Resolution Layer 仕様書

関連:

- 現行 runtime: `src/services/deepseek-chat.ts`
- 現行 tool 群: `src/services/zetter-chat-tools.ts`
- 現行 system prompt: `src/services/zyn-system-prompt.ts`
- 既存包括仕様: `files/zetter-chat-harness-v2-spec.md`
- tool 経路分離仕様: `files/chat-tool-channel-spec.md`
- eval / deterministic 仕様: `files/zetter-chat-eval-determinism-spec.md`

---

## 目的

Zetter chat を、**「それっぽく答えるチャット」から「参照先を確定してから答えるチャット」へ上げる**。

本仕様の主目的は次の 3 点。

1. **自然文に含まれる参照先を回答前に解決する**
   - `はるのさん`
   - `この投稿`
   - `昨日`
   - `Zetter 全体`
   - `別垢`
   のような参照を、曖昧なまま回答しない

2. **routing を場当たり regex から 1 段抽象化する**
   - `resolve_user` の一点対策ではなく
   - **user / post / time / scope / relation をまとめて扱う参照解決層**を導入する

3. **tool-first / grounded-first の一貫性を作る**
   - Zetter データに依存する質問は、原則として
     - request classify
     - reference resolve
     - evidence plan
     - tool execute
     - grounded answer
     の順に処理する

---

## 背景

現行の Zetter chat は、tool 群・streaming・progress 表示・deterministic path・eval harness が徐々に整ってきている。一方で、**自然文の参照解決**はまだ弱い。

### 現状の主要課題

1. **Zetter データ質問の検出が lexical 依存**
   - `isZetterDataRequest()` は語彙ベースで、`ユーザー` は拾えても `ユーザ` や固有名だけの依頼に弱い
    - `はるのさんについて教えて` のような自然な依頼が `direct-answer` に落ちうる

2. **参照先を確定しないまま説明文を生成できる**
   - 実在確認前に人物説明を書ける
   - 存在しない別アカ、関係性、投稿傾向を推測で埋める事故が起きる

3. **参照の種類ごとに対処が分散している**
   - user だけでなく、post / time / scope / relation にも同種の問題がある
   - 個別 if 文の増築では、応用が利かず、将来の挙動も読みにくい

4. **guardrail が「参照未解決」を直接見ていない**
   - いまの safety / boundary / DSML guard はある
   - しかし「参照未解決なのに答えている」こと自体を止める層がない

---

## 2026-05 update: prompt-first reference contracts

この文書の初版は RRL をやや **code-first** に寄せて説明していたが、現行方針はより正確には **prompt-first / contract-validated / deterministic-binding** である。

現在の参照解決は次の役割分担を採る。

1. **prompt が意味を決める**
   - user / time / scope / relation / latest semantics を `IntentSpec` / `ReferenceSpec` 相当の JSON で決める
2. **code が contract を検証する**
   - enum drift
   - pseudo-user placeholder 混入
   - explicit date の未反映
   - latest scope の崩れ
   をチェックする
3. **壊れた参照は repair stage に戻す**
   - contract 違反の JSON をそのまま tool 実行へ進めず、bounded retry で reference repair を行う
4. **deterministic 層は semantic rewrite をしない**
   - `viewer-self -> viewer.userId`
   - resolved userId / date range の binding
   - safe clarification / limitation
   のみを担当し、意味の勝手な再解釈は行わない

特に現在は、次の contract を強制対象として扱う。

1. first-person request は `viewer-self` に落とす
2. `semantic.users` / `ReferenceSpec` に `me`, `self`, `viewer`, `私`, `自分` などを残さない
3. bare `最新N件` は `post-query + outputMode=latest + sortOrder=newest + latestScope=global`
4. user-scoped latest は `latestScope=user`
5. include latest (`Xも含めて最新N件`) は `latestScope=global-with-includes`
6. bare latest で勝手に `toDate=current-date` のような implicit range を足さない

以降の章は RRL の全体像として引き続き有効だが、**「コードで意味を解く層」というより、「prompt が出した参照を code が検証・repair・binding する層」**として読むのが現行状態に近い。

---

## 結論

導入すべきなのは、`resolve_user` 単体ではなく、**Reference Resolution Layer (RRL)** である。

Zetter chat の質問処理を、以下の 6 段に整理する。

1. **Request classify**
   - 雑談 / 生成依頼 / Zetter データ質問 / boundary request を分類する

2. **Reference extract**
   - user / post / time / scope / relation の候補を自然文から抽出する

3. **Reference resolve**
   - 候補を DB / 会話文脈 / deterministic parser で確定する

4. **Evidence plan**
   - 解決済み参照を前提に、必要な tool と不足情報を決める

5. **Tool execute**
   - 実データを取得する

6. **Grounded answer + guardrail**
   - 解決済み参照と tool result の範囲でだけ回答する

重要なのは、**Zetter 参照を含む依頼では「参照解決前の直答」を原則禁止にする**こと。

---

## 基本思想

### 1. 複雑化は「局所化」する

本仕様は、chat 全体を多エージェント化したり、常時 planner ループ化したりするものではない。  
複雑さは **RRL という 1 層に閉じ込める**。

### 2. 参照未解決のまま答えない

次のような依頼は、参照先が解決するまで回答本文を書いてはいけない。

- `はるのさんについて教えて`
- `この投稿どう思う？`
- `昨日の流れを教えて`
- `そらの別垢ある？`

### 3. contract を先に固め、binding はコードで行う

RRL はまず **prompt-first / contract-first** で動く。

- `@` / `＠` の正規化
- `さん` / `氏` / `ちゃん` の除去
- `今日` / `昨日` の JST 解釈
- `この投稿` の近傍参照
- `Zetter全体` / `この人` / `自分` の scope 解釈

ただし、これらは意味決定そのものではなく、**prompt が決めた参照仕様の binding / validation / repair 補助**として使う。  
contract に合わない場合は、そのまま実行せず confirmation / repair に戻す。

### 4. LLM は「参照決定」と「解決後の言語化」に分けて使う

LLM の役割は残るが、役割は狭める。

- `viewer-self`, `latestScope`, time scope などの semantic decision
- 解決済みデータの自然な説明
- 軽い要約
- 複数候補があるときの確認質問文
- 判断が微妙な routing 補助

---

## Non-goals

1. 最初から multi-agent 化すること
2. 長期記憶や人格化を強めること
3. 全質問を deterministic 化すること
4. 自然文の曖昧さをゼロにすること
5. relation claim を推測で埋めること

本仕様は **「すべてを hardcode する」ためのものではなく、「参照を確定しないまま hallucinate しない」ためのもの**である。

---

## スコープ

本仕様が対象にする参照は次の 5 種。

### 1. User reference

例:

- `@mizuki`
- `そら`
- `はるのさん`
- `ひよりちゃん`

### 2. Post reference

例:

- `この投稿`
- `さっきのポスト`
- `このスレ`
- `その返信`

### 3. Time reference

例:

- `今日`
- `昨日`
- `最近`
- `GW中`
- `先週`

### 4. Scope reference

例:

- `Zetter全体`
- `この人の投稿`
- `自分向け`
- `会話中の対象だけ`

### 5. Relation reference

例:

- `別垢`
- `よく絡む人`
- `相性が良い人`
- `よく返信している相手`

---

## アーキテクチャ

## Layer 0: Boundary / hard refusal

最初に従来どおり boundary request を弾く。

- server files
- source code
- env
- secrets
- shell
- arbitrary SQL

この層は RRL より先に評価する。

---

## Layer 1: Request classification

質問を次の大分類に分ける。

1. `boundary-refusal`
2. `casual-chat`
3. `direct-template`
4. `zettter-reference-request`
5. `image-analysis`

### 新方針

現行の `isZetterDataRequest()` は補助信号とし、**唯一の判定器にしない**。

Zetter 参照質問の判定条件は、少なくとも次のいずれかを満たすものとする。

1. 明示的な Zetter 語彙を含む
2. user/post/time/scope/relation の参照候補を 1 つ以上含む
3. `〜について教えて` `〜を探して` `説明書` `どんな人` `どう思う` のような対象依存の質問形を含む

これにより、`はるのさんについて教えて` は Zetter データ質問として扱う。

---

## Layer 2: Reference extraction

自然文から `ReferenceHint[]` を抽出する。

```ts
type ReferenceKind = "user" | "post" | "time" | "scope" | "relation";

type ReferenceHint = {
  kind: ReferenceKind;
  raw: string;
  normalized: string;
  source:
    | "explicit-handle"
    | "plain-name"
    | "deictic"
    | "date-word"
    | "scope-word"
    | "relation-word";
};
```

### 抽出ルール

#### user

- `@foo`
- `＠foo`
- `fooさん`
- `foo氏`
- 文頭 / 文中の単独固有名候補

#### post

- `この投稿`
- `その投稿`
- `さっきの投稿`
- `このスレ`

#### time

- `今日`
- `昨日`
- `最近`
- `先週`
- `GW中`

#### scope

- `Zetter全体`
- `全体`
- `この人の投稿`
- `自分の投稿`

#### relation

- `別垢`
- `相性`
- `よく絡む`
- `よく返信`

---

## Layer 3: Reference resolution

抽出した hint を `ResolvedReference[]` に変換する。

```ts
type ResolutionStatus = "resolved" | "ambiguous" | "unresolved" | "not-needed";

type ResolvedReference = {
  kind: ReferenceKind;
  raw: string;
  normalized: string;
  status: ResolutionStatus;
  confidence: "high" | "medium" | "low";
  resolvedValue?: Record<string, unknown>;
  candidates?: Array<Record<string, unknown>>;
  resolvedBy:
    | "exact-match"
    | "search-users"
    | "conversation-context"
    | "deterministic-date-parser"
    | "scope-parser"
    | "relation-parser";
};
```

---

## 3-1. User resolution

### 正規化

次を順に行う。

1. trim
2. lower-case
3. `＠` → `@`
4. 先頭 `@` 除去
5. 後尾 honorific 除去
   - `さん`
   - `氏`
   - `ちゃん`
   - `くん`
6. 句読点・末尾記号の軽量除去
   - ただし `.` `_` `-` は userId 候補として保持

### 解決手順

1. exact handle match
2. `search_users` による prefix / partial search
3. display name match
4. 会話文脈の active entity と照合

### 選定ルール

1. exact handle 1 件 → `resolved/high`
2. prefix match 1 件 → `resolved/high`
3. partial / displayName match が複数 → `ambiguous`
4. 0 件 → `unresolved`

### 重要制約

- **bio 一致だけで identity を確定しない**
- **存在しない別アカを推測しない**
- **候補 0 件で人物説明文を書かない**

---

## 3-2. Post resolution

`この投稿` `その投稿` `さっきの投稿` は、次の順で解決する。

1. 現在スレッド内の直近 assistant / user 文脈にある postId
2. 直近 tool result に含まれる post list の先頭対象
3. 明示 post URL / postId

0 件なら `unresolved` とし、**勝手に timeline の最新投稿へ寄せない**。

---

## 3-3. Time resolution

時間参照は JST 基準で deterministic に解決する。

### 例

- `今日` → 当日 00:00:00+09:00 〜 翌日 00:00:00+09:00
- `昨日` → 前日 00:00:00+09:00 〜 当日 00:00:00+09:00
- `最近` → デフォルト窓を定義
  - 例: 直近 7 日
- `GW中` → 年ごとの固定期間ではなく、まず明示的な range policy に従う

### 方針

- 実装が曖昧な期間語は、固定 policy を持つ
- policy が無い語は `ambiguous`
- date 解釈は tool metadata に `resolvedBy`, `startIso`, `endIso` を残す

---

## 3-4. Scope resolution

次の scope を内部表現として持つ。

```ts
type ResolvedScope =
  | "all-visible-users"
  | "single-user"
  | "viewer-self"
  | "current-thread"
  | "timeline-global";
```

### 例

- `Zetter全体` → `timeline-global`
- `この人の投稿` → `single-user`
- `自分の投稿` → `viewer-self`
- `このスレ` → `current-thread`

scope が未指定でも、request type に応じて安全な default を持つ。

---

## 3-5. Relation resolution

relation は **最も hallucinate しやすい** ため、特別扱いする。

### relation の種類

1. `interaction-based`
   - よく返信している
   - よく絡む
2. `ranking-based`
   - 相性が良い
3. `identity-claim`
   - 別垢
   - 同一人物

### 厳格ルール

- `interaction-based` と `ranking-based` は tool で裏付け可能なときのみ回答可
- `identity-claim` は **明示的根拠が無い限り断定禁止**
- `別垢っぽい` のような soft claim も既定では禁止

つまり、`そらの別垢ある？` に対しては、証拠がないなら  
**「見える範囲では確認できない」** が正答になる。

---

## Layer 4: Evidence planning

解決済み参照をもとに、必要な tool を選ぶ。

```ts
type EvidencePlan = {
  canAnswerDirectly: boolean;
  clarificationNeeded: boolean;
  clarificationQuestion?: string;
  tools: Array<{
    name: string;
    args: Record<string, unknown>;
  }>;
};
```

### 重要方針

RRL のあとに **tool 選択を再び LLM 任せにしない**。  
少なくとも代表ケースでは、**解決済み参照 → tool** の対応を deterministic に持つ。

### Deterministic tool mapping

| 条件 | 呼ぶ tool |
| --- | --- |
| resolved user 1人 + 人物説明/傾向依頼 | `analyze_user` |
| resolved user 1人 + 存在確認/検索依頼 | `search_users` |
| resolved user 2人 + 比較依頼 | `analyze_user_pair` |
| resolved user 1人 + 相性依頼 | `rank_compatible_users` |
| resolved time/scope + 流れ/件数/最近依頼 | `analyze_timeline` |
| resolved post 1件 + 投稿/スレ依頼 | `analyze_thread` |
| identity claim のみ | **tool を無理に呼ばず fallback** |

この mapping は Phase 1 では code で持ち、追加 tool が必要になったときだけ拡張する。

### 例

#### `はるのさんについて教えて`

- resolved user: `@haruno`
- plan:
  - `analyze_user(userId="haruno")`

#### `この投稿どう思う？`

- post unresolved
- plan:
  - clarification needed

#### `mizukiと相性の良い人`

- resolved user: `@mizuki`
- plan:
  - `rank_compatible_users(userId="mizuki", scope="all-visible-users")`

---

## Layer 5: Tool execution

tool 実行自体は現行の安全経路を使う。  
ただし、RRL 導入後は次を追加ルールにする。

### ルール

1. `analyze_user` / `rank_compatible_users` / `analyze_user_pair` は、原則 `resolved userId` 必須
2. `search_posts` の date range は、RRL で解決済みの `startIso/endIso` を優先
3. `identity-claim` に直接対応する tool が無い場合、回答は capability-bound fallback にする

---

## Layer 6: Grounded answer + guardrail

最終回答前に、参照整合性 guardrail を入れる。

## 6-1. Input-side guardrail

Zetter データ質問なのに、次の条件を満たす場合は `direct-answer` へ落としてはならない。

1. reference hint がある
2. resolved reference が 0 件
3. clarification もしていない
4. tool call もしていない

この場合は:

- clarification
- fallback
- もしくは resolver 再試行

へ送る。

## 6-2. Output-side guardrail

最終回答に出てくる以下を検査する。

1. userId / displayName
2. 投稿内容
3. 日付・件数
4. relation claim

### 原則

- tool result に無い user / post / count を足さない
- unresolved relation を断定しない
- ambiguous user を 1 人に決め打ちしない

違反時は:

1. final answer 再生成
2. それでも無理なら conservative fallback

---

## 既存 routing への組み込み方針

現行 `classifyHarnessGate()` を次のように整理する。

### 現状

- chitchat
- direct-template
- brief meta followup
- isZetterDataRequest

### 変更後

1. boundary
2. direct-template
3. reference-bearing request 判定
4. casual-chat
5. needs-router

### 重要変更

`reference-bearing request` を新設する。

これは次のいずれかを満たす依頼。

1. reference hint がある
2. Zetter 語彙がある
3. 対象依存の質問形がある

これに該当したら、**`direct-answer` へは落とさず RRL を通す**。

---

## 実装単位

## A. 新規 internal helper

`src/services/deepseek-chat.ts` もしくは近接 helper に、少なくとも次を追加する。

1. `extractReferenceHints(message)`
2. `resolveReferences(input)`
3. `buildEvidencePlan(input)`
4. `buildReferenceClarificationQuestion(input)`
5. `runReferenceOutputGuardrail(input)`

## B. 既存 helper の見直し

1. `isZetterDataRequest()`
2. `classifyHarnessGate()`
3. `buildDirectTemplateAnswer()`
4. compare / ranking / latest / timeline の direct path

## C. Tool layer

既存 tool を再利用するのを基本とする。  
ただし必要なら、将来的に次の composite tool を検討してよい。

1. `resolve_entities`
2. `resolve_post_reference`

ただし Phase 1 では **新 tool 追加は必須ではない**。

---

## 失敗時の挙動

### user unresolved

`はるのさんについて教えて`
→ 候補 0 件
→ 「見える範囲では該当ユーザーを見つけられませんでした」

### user ambiguous

複数候補
→ 候補を 2〜5 件まで短く提示し確認質問

### post unresolved

`この投稿どう思う？`
→ 文脈に post が無い
→ 「どの投稿か分かるように URL か内容を少し教えてください」

### relation unresolved

`そらの別垢ある？`
→ identity evidence 無し
→ 「見える範囲では確認できません」

---

## Eval 追加仕様

`fixtures/chat-evals/` に次を追加する。

```text
reference_resolution.json
reference_ambiguity.json
reference_guardrails.json
```

### 追加したいケース

1. `はるのさんについて教えて`
   - `search_users` または `analyze_user` が呼ばれる
   - direct-answer に落ちない

2. `そらの別垢ある？`
   - 断定しない
   - 「確認できない」系になる

3. `この投稿どう思う？`
   - 文脈なしなら clarification する

4. `昨日の流れを教えて`
   - deterministic time resolve が走る

5. `そらとmizukiを比較して`
   - compare path へ行く

### 採点観点

1. 必要な resolution が行われたか
2. 不要な direct-answer をしていないか
3. unresolved なのに断定していないか
4. ambiguity に対して確認質問したか

---

## 観測性

既存の request summary / progress events に、reference resolution の要約を残す。

最低限ほしい項目:

```json
{
  "referenceHints": ["user:haruno"],
  "resolvedReferences": ["user:@haruno"],
  "ambiguousReferences": [],
  "unresolvedReferences": [],
  "clarificationNeeded": false
}
```

これにより、今回のような問題で

- 検索に行ったのか
- direct-answer に落ちたのか
- 参照未解決のまま答えたのか

を追えるようにする。

---

## 段階導入

## Phase 1: User / Time / Scope

まず以下に絞る。

1. user reference
2. time reference
3. scope reference
4. input-side guardrail
5. eval fixtures

この段階で `haruno` 系の事故と、日時・対象の取り違えを大きく減らす。

## Phase 2: Post / Relation

次を追加する。

1. post deictic resolution
2. relation claim policy
3. output-side guardrail

この段階で `この投稿` や `別垢` 系の hallucination を抑える。

## Phase 3: Router 補助最適化

必要であれば:

1. 軽量 router-thinking
2. ambiguous resolution のみ LLM 補助
3. additional composite tool

ただし、Phase 1 / 2 で十分なら入れない。

---

## 採用理由

この構成を採る理由は 3 つある。

1. **場当たり対応を避けられる**
   - `resolve_user` 単体ではなく、参照問題全体を同じ抽象で扱える

2. **複雑さを局所化できる**
   - 全面 multi-agent 化より、デバッグしやすい

3. **精度向上が測定可能**
   - eval fixture と trace で、改善を確認できる

---

## 最終結論

Zetter chat の次の改善は、単なる `user 検索強化` ではなく、**Reference Resolution Layer の導入**である。

この層により、chat は

- 自然文の対象を先に確定し
- 必要な tool を選び
- 根拠のある範囲だけで答える

ようになる。

つまり今後の方針は、

- **直答を賢くする**のではなく
- **答える前の参照解決を正しくする**

である。
