# Zetter Chat — ZYN（ジン）設計仕様書 / Design Spec

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

**JA:** ZYN（ジン）は SNS アプリ **Zetter** に組み込まれた、会話型の AI アシスタントです。投稿・ユーザーに関する質問への回答、タイムラインの要約、ユーザー分析、添付画像の解析を行います。本ドキュメントは **現在の確定設計（現行設計）** を記述します。

**EN:** ZYN is the conversational AI assistant embedded in the **Zetter** social app. It answers questions about posts and users, summarizes the timeline, analyzes users, and interprets attached images. This document describes the **current design**.

> このドキュメントは現行設計のみを扱います。旧（5層パイプライン）設計は末尾の「旧設計サマリ」を参照。
> This document covers the current design only. The previous (5-layer pipeline) design is summarized at the end.

> **JA:** 本ドキュメントは設計思想の概要であり、実装の定数値・判定条件・プロンプト本文（逐語）は含みません。
> **EN:** This document describes design intent at a conceptual level; it does not include implementation constants, decision thresholds, or verbatim prompt text.

---

## 今回の大きな変更 / What changed in this update

**JA:** 本セクションは 2026-06-29（tool-loop v2 初期）以降の進化を反映した **2026-07-14 更新**です。基本アーキテクチャ（単一プロンプト + tool-calling ループ）は変わっていませんが、以下が積み上がりました:

- **ツールが 5→6 個**: 投稿の集計に特化した専用ツール `aggregate_posts` を追加。既存の検索ツールにも時間帯フィルタが追加
- **「正直な着地」**（#151）: ターン数・履歴長ベースの素朴な上限から、**ターン数と経過時間の二重予算**＋**最終ターン契約**（予算超過時はツール呼び出しを物理的に不能にし、必ず文章で着地させる）に置き換え
- **本文捏造対策 / content grounding**（#132）: システムプロンプトに「records へのグラウンディング」節を新設し、取得件数と全体件数の混同禁止・投稿時刻の verbatim 扱いなどを明文化
- **メモリ基盤 v2**: スレッド内 running summary（L1）＋ユーザー横断の日次ダイジェスト（L2）の 2 層を追加。直近の会話履歴ウィンドウ自体は維持
- **能力の正典化・出典リンク**（#259）: 「Capability Charter」節と出典明示ルールをプロンプトに追加
- **画像添付の修正**（#197 / #199）: 直接添付画像の解析不具合を修正し、投稿添付画像も同じ解析経路に統合

**EN:** This section is the **2026-07-14 update**, covering evolution since 2026-06-29 (early tool-loop v2). The core architecture (single prompt + tool-calling loop) is unchanged, but the following accumulated on top:

- **Tools 5→6**: added a dedicated post-aggregation tool, `aggregate_posts`; the existing search tool also gained time-of-day filtering
- **"Honest landing"** (#151): replaced the naive turn/history-based cap with a **dual budget over turn count and elapsed time**, plus a **final-turn contract** — once the budget is exceeded, further tool calls are physically disabled so the model always lands a prose answer
- **Content grounding / anti-fabrication** (#132): a new "grounding to records" prompt section bans conflating the sampled-result count with the total-match count, and mandates verbatim post timestamps, among other rules
- **Memory v2**: added a two-tier memory — an in-thread running summary (L1) and a cross-user daily digest (L2) — layered on top of the still-unchanged recent history window
- **Capability canon + citation links** (#259): added a "Capability Charter" section and source-citation rules to the prompt
- **Image attachment fixes** (#197 / #199): fixed an analysis issue affecting directly-attached images, and unified attached-post images onto the same analysis path

---

## アーキテクチャ全体像 / Architecture overview

```
                 user message (+ optional images / attached post)
                              │
        ┌─────────────────────▼─────────────────────┐
        │  Harness scaffolding                       │
        │  • JST date grounding (injected per req)   │
        │  • image context (direct + attached post)  │
        │  • thread context load                     │
        │  • memory: cross-user digest →             │
        │            thread running summary →        │
        │            recent raw history              │
        └─────────────────────┬─────────────────────┘
                              │  single comprehensive system prompt
                              │  + recent history window
        ┌─────────────────────▼─────────────────────┐
        │            TOOL-CALLING LOOP               │
        │   tool execution budget: turn count OR    │
        │   elapsed time, whichever comes first      │
        │                                            │
        │   ┌───────────────────────────────────┐   │
        │   │ LLM call                          │   │
        │   └───────────────┬───────────────────┘   │
        │       has tool_calls?                      │
        │        │ yes               │ no + stop     │
        │        ▼                   ▼               │
        │   execute tools       break → final answer │
        │   push results,                            │
        │   repeat                                   │
        │                                            │
        │   budget exceeded → final-turn contract:   │
        │   tool calls disabled for that turn        │
        │   + a wrap-up instruction forces a prose   │
        │   landing (no more tool calls)             │
        └─────────────────────┬─────────────────────┘
                              │  final answer
        ┌─────────────────────▼─────────────────────┐
        │  Safety nets                               │
        │  deterministic, non-LLM checks on the      │
        │  final answer, plus marker/format          │
        │  normalization                             │
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

**JA:** 単一モデルをループで回します。上限は固定ターン数ではなく、**ツール実行予算**（PR #151「正直な着地」）――ターン数と経過時間の二重予算のうち、いずれか早く達した方で管理されます。各ターンの流れ:

1. LLM を呼び出す
2. assistant の応答を履歴に積む
3. ツール呼び出しがあれば実行し、結果を履歴に積む
4. これを繰り返す

ツール呼び出しが無く自然に終了したとき、そのターンの本文が最終回答になります。

**最終ターン契約 / Force-stop:** 予算を超えたターンでは、ツール定義そのものを API リクエストから外して呼び出しを物理的に不能化し、着地を促す指示を注入して、必ず文章で着地させます。個別のモデル呼び出しには別途タイムアウトがあります。

**EN:** A single model runs in a loop. Instead of a fixed turn cap, the loop is governed by a **tool execution budget** (PR #151, "honest landing") — a dual budget over turn count and elapsed time, whichever is reached first. Each turn: call the LLM → push the assistant response to history → execute any tool calls and push their results to history → repeat. When the model stops without calling a tool, that turn's content is the final answer.

**Final-turn contract / force-stop:** once the budget is exceeded, that turn is sent with no tool definitions at all — physically disabling further tool calls — plus an instruction telling the model to wrap up in prose. Individual model calls separately carry their own timeout.

---

## 6 つの固定ツール / The six fixed tools

**JA:** PR #147 で集計専用の `aggregate_posts` が追加され 5→6 個になり、PR #151 で検索ツールに時間帯フィルタが加わりました。

**EN:** PR #147 added the aggregation-only `aggregate_posts` tool (5→6), and PR #151 added time-of-day filtering to the search tool.

| Tool | 用途 / Purpose |
|---|---|
| `query_posts_v2` | 投稿の検索・取得 / Post search & retrieval |
| `summarize_timeline` | 固定ウィンドウのタイムライン概観（流れ・話題） / Recent timeline overview for a fixed window |
| `analyze_user` | 単一ユーザーの深掘り分析（人物像・話題・トーン） / Single-user deep analysis |
| `lookup_user_profile` | 軽量プロフィール取得 / Lightweight profile lookup |
| `compare_users` | 2 ユーザーの比較・相性診断 / Two-user comparison & compatibility |
| `aggregate_posts` | 投稿の集計（件数系の質問に直答） / Post aggregation for count-style questions |

**JA:** ツールは **無条件で常時登録**されます。いつ・どれを使うかは LLM が判断します（旧設計のような「投稿検索チップを ON」というオプトインは不要）。引数はコード側で検証されます。

**EN:** All tools are **registered unconditionally**; the LLM decides when and which to use (no opt-in "post-search chip" as in the old design). Arguments are validated on the code side before each call executes.

---

## 単一システムプロンプト / Single system prompt

**JA:** 旧の最小（約 8 行）プロンプトに代えて、単一の包括的なシステムプロンプトを組み立てて使います。2026-06-29 時点の構成から、PR #132・#149・#259 でカバー領域が拡張されました。現在カバーしている領域:

- identity — ZYN としての人格・役割
- 言語 — 応答言語の既定
- 日付グラウンディング — 現在日付の注入
- 正直性プロトコル — 能力否定・推測の抑止
- ツールルーティング — ツール選択の方針
- 一人称・既定範囲の解釈
- 空振り時の再試行方針
- ツール呼び出しの作法
- 誤りの自己訂正
- 文体とキャラ — トーン・キャラクター表現
- 会話内記憶 — スレッド内の記憶の扱い
- 安全境界 — 内部実装の非開示
- 取得結果への本文グラウンディング — 捏造防止
- ZYN の立ち位置・AI キャストの扱い
- 能力の正典 — 機能範囲の正確な提示
- 出典明示 — 情報源の明示
- 製品知識 — Zetter プラットフォームの事実

**EN:** Instead of the old minimal (~8-line) prompt, a single comprehensive system prompt is assembled. Since the 2026-06-29 structure, coverage was expanded by PR #132, #149, and #259. Areas currently covered:

- Identity — ZYN's persona and role
- Language — default response language
- Date grounding — injecting the current date
- Honesty protocol — curbing ability-denial and speculation
- Tool routing — how tools are chosen
- First-person / default-scope interpretation
- Empty-result retry policy
- Tool-call hygiene
- Mistakes & self-correction
- Tone & character — voice and character expression
- In-conversation memory — handling thread-level memory
- Safety boundaries — non-disclosure of internals
- Grounding to retrieved results — anti-fabrication
- ZYN's stance and handling of AI cast accounts
- Capability canon — accurate scope of features
- Source citation — citing sources
- Product knowledge — facts about the Zetter platform

---

## セーフティネット / Safety nets

**JA:** LLM の出力とは独立に、決定的（非 LLM）な安全網が複数あり、最終回答を検査・補正します。能力詐称系の失敗を検知すると段階的に介入し、軽度なら訂正の注記を回答に付与し、重度ならツールの強制実行やそのターンのやり直しを行います。

**EN:** Independent of the LLM's own output, several deterministic (non-LLM) safety nets inspect and repair the final answer. They detect ability-denial-style failures and intervene in stages: minor cases get a corrective note appended to the answer, while more serious cases trigger a forced tool re-execution or a retry of that turn.

---

## grounding（日付・能力） / Grounding

**JA:** JST の現在日付をリクエスト時に算出し、毎回システムプロンプトへ注入します。ツール結果を真実の源として明示し、相対的な日付表現はツールに渡す前に絶対日付へ変換させます。

**EN:** The current JST date is computed per request and injected into the system prompt every time. Tool results are treated as the source of truth, and relative date expressions are converted to absolute dates before being passed to tools.

---

## 本文グラウンディング（records 捏造対策） / Content grounding (anti-fabrication)

**JA:** PR #132（2026-07-01 マージ）で新設された、ツール結果への忠実性を求める規範です。要点は 2 つ: (1) ツール結果に含まれていない投稿本文・時刻・著者を推測で書かない。(2) 実際に見たサンプルの件数と、条件に合致する全体件数を混同せず、区別して語る。

**EN:** Introduced by PR #132 (merged 2026-07-01), this is a norm requiring fidelity to tool results. Two key points: (1) never write post bodies, timestamps, or authors that aren't present in the tool results; (2) never conflate the number of sampled results actually seen with the total number of matches, and clearly distinguish between the two when reporting.

---

## メモリ基盤 v2（L1/L2） / Memory v2 (L1/L2)

**JA:** 直近の生履歴・スレッド内の running summary（L1）・ユーザー横断の日次ダイジェスト（L2）の 3 層でメモリを構成します。一定の閾値を超えると要約を再合成し、固定の順序でシステムプロンプトへ注入します。

**EN:** Memory is composed of three tiers: the raw recent-history window, an in-thread running summary (L1), and a cross-user daily digest (L2). When certain thresholds are exceeded, the summaries are recomputed and injected into the system prompt in a fixed order.

---

## normalization（DSML 解析・マーカー除去） / Normalization

**JA:** モデルがネイティブの tool call ではなく、テキスト（DSML と呼ぶ内部表記）でツール呼び出しを書いてしまった場合に救済し、ループを継続させる層です。あわせて、残った内部マーカーの除去や本文の整形も行います。

**EN:** A layer that rescues cases where the model writes a tool call as text (using an internal notation called DSML) instead of a native tool call, allowing the loop to continue. It also strips any remaining internal markers and normalizes the response text.

---

## observability（推論トレース） / Observability

**JA:** トレース用の環境変数フラグを有効にしたときのみ動作します。各ターンの推論トレースと、最後に安全網の発火有無を含む最終サマリを記録します。

**EN:** Gated by an environment-variable trace flag. Per-turn reasoning traces are recorded, along with a final summary that includes whether each safety net fired.

---

## 引き継ぎ（旧ハーネスから保持） / Retained from the old harness

| 機能 / Capability | 内容 / Detail |
|---|---|
| 画像解析 / Image analysis | 直接添付と投稿添付の 2 経路が同じ解析処理に合流 / Direct attachments and attached-post images converge on the same analysis path |
| スレッド文脈 / Thread context | 引き続きロード・圧縮保存を実行 / still loaded and compacted |
| レート制限 / Rate limiting | ループ後にレート制限の消費を確定 / enforced after the loop |
| 再処理 / Reprocess | 再処理経路を保持 / preserved |
| 永続化 / Persistence | 保存処理は不変 / unchanged |

---

## 旧設計との比較 / Comparison with the previous design

| 観点 / Aspect | 旧 / Old | 新 / New |
|---|---|---|
| **制御主体 / Control principal** | Code-driven: `classifyIntentViaLlm` の JSON → `buildIntentRoute` がコードで固定 DB ステップを組み立て | LLM-driven: モデルが毎ターン自律的にツールを選択。コード側の固定ステップ写像はない |
| **意図分類 / Intent classification** | 必須: DB アクセス前に planner LLM が意図分類 JSON を生成 | 廃止: LLM がユーザー文と文脈から直接、呼ぶか・どれを呼ぶかを判断 |
| **DB アクセス起動 / DB activation** | Opt-in: ユーザーが検索機能を明示的に有効化する必要 | 常時利用可: ツールを無条件登録し、取得タイミングは LLM が判断 |
| **1 リクエストの LLM 呼数 / LLM calls** | 最低 2 回以上（分類用 + 最終整形用、＋任意の追加呼び出し） | 単一モデルのループ。ツール実行予算に達するまで継続し、超過後は必ず文章で着地する契約がある |
| **根拠注入 / Evidence injection** | 取得素材をテキストブロックとしてユーザー文に埋込 | ツール結果を会話ターン履歴に差し込む |
| **システムプロンプト範囲 / Prompt scope** | 最小の identity + ルールのみ。プラットフォーム情報や DB 素材は別途ユーザー文へ注入 | 単一の包括的プロンプトに、日付・正直性・ルーティング・本文グラウンディング・能力正典・出典・製品知識などを一本化 |
| **能力詐称ガード / Ability-claiming guard** | 専用なし。planner が黙って取得をスキップし得た | 正直性プロトコル＋複数の決定的な安全網が段階的に介入。加えて能力正典（Capability Charter）をプロンプトに明文化 |
| **テキストでのツール呼び出し対応 / Text-form tool-call handling** | 該当なし（構造化データでその種のリスクはなかった） | テキストで書かれたツール呼び出しをネイティブ呼び出しへ救済しループを継続 |
| **ターン / ステップ上限 / Turn limit** | `executeRetrievalSteps` による固定回数のコード反復。完了後 LLM 再入なし | ターン数と経過時間の二重予算。超過後は必ず文章で着地する契約で強制的に終了 |
| **人物解決 / Person resolution** | 2 段の固定コード: `search_users` → 条件付き後続の投稿/分析ステップ | LLM が解決戦略を決定。専用ツールを独立して利用 |

### なぜ変えたか / Why we changed it

**JA:** 旧はコード主導の固定パイプラインで、LLM の役割は意図分類と最終文整形に限られていました。結果としてルーティングが硬直し、planner が黙って取得を飛ばす・能力を実際より低く見せてしまう、といった失敗が起きやすい構造でした。新設計では LLM がツールを主導し、ハーネスは grounding（日付・能力の事実）/ normalization（テキスト整形）/ observability（トレース）/ safety net（安全網）という **足場**に徹します。これにより柔軟なルーティングと、決定的な安全網による嘘の抑止を両立します。

**EN:** The old fixed, code-driven pipeline confined the LLM to intent classification and final-text formatting, producing rigid routing and failure modes such as the planner silently skipping retrieval or the model understating its own abilities. The new design lets the LLM drive tool use while the harness acts purely as **scaffolding** — grounding (date/ability facts), normalization (text formatting), observability (traces), and safety nets (deterministic checks) — combining flexible routing with deterministic suppression of hallucinated ability denials.

---

## 旧設計サマリ（参考・deprecated） / Previous design summary (deprecated)

**JA:** 旧設計は概ね次の 5 層パイプラインでした（詳細は git 履歴を参照）:

1. **Intent classification** — planner LLM が `{kind, intent, userId, date, ...}` JSON を生成
2. **Route building** — `buildIntentRoute` がコードで固定 DB ステップ列を組み立て
3. **Retrieval execution** — `executeRetrievalSteps`（`MAX_STEPS=3`）が DB を叩き `DbRetrievalMaterial` を生成
4. **Evidence injection** — 素材を `=== DATABASE RECORDS ===` ブロックとしてユーザー文に埋込
5. **Finalization** — finalizer LLM が最終文を生成

**EN:** The previous design was roughly a 5-layer pipeline (see git history for details): (1) intent classification via a planner LLM emitting a `{kind, intent, userId, date, ...}` JSON; (2) route building by `buildIntentRoute` assembling fixed DB steps in code; (3) retrieval execution via `executeRetrievalSteps` (`MAX_STEPS=3`) producing `DbRetrievalMaterial`; (4) evidence injection of that material as a `=== DATABASE RECORDS ===` block inside the user message; (5) finalization by a finalizer LLM. This path is **deprecated** and no longer used.
