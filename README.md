# Zetter Chat Harness Design

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

このリポジトリには、Zetter chat harness の**機密情報を除いた設計文書**を収録しています。

This repository contains **sanitized architecture and design documents** for the Zetter chat harness.

現在の公開ドキュメントは、初期の flash-only coordinator 案に加えて、**reasoning barrier / prompt-driven contract repair / deterministic binding** を中心にした現行方向も反映しています。

この公開版には**設計資料のみ**を含め、次のものは含めません。

This public export intentionally includes **design-only materials** and excludes:

- application source code
- production data
- credentials, secrets, tokens, and connection details
- deployment runbooks and environment-specific operational notes

- アプリケーションのソースコード
- 本番データ
- 認証情報、シークレット、トークン、接続情報
- デプロイ手順や環境依存の運用メモ

## Included documents

## 収録ドキュメント

- `docs/zetter-chat-harness-v2-spec.md`
  - end-to-end harness architecture, including reasoning barrier updates / ハーネス全体アーキテクチャと reasoning barrier 更新
- `docs/zetter-chat-reference-resolution-spec.md`
  - reference resolution layer design with prompt-first contracts / prompt-first 契約を含む参照解決レイヤ設計
- `docs/zetter-chat-eval-determinism-spec.md`
  - eval harness and deterministic/contract regression design / eval harness と deterministic / contract 回帰設計
- `docs/chat-tool-channel-spec.md`
  - separation of user-visible answer flow and tool-control flow / ユーザー向け回答と tool 制御経路の分離設計
- `docs/chat-safe-tools-spec.md`
  - safe tool boundary and read-only tool design / 安全な tool 境界と read-only tool 設計

## Notes

## 補足

- Some documents refer to internal runtime component names to explain the architecture.
- This export is meant to share the **system design** rather than the full implementation.
- 一部の文書では、アーキテクチャ説明のために内部ランタイム上のコンポーネント名を参照しています。
- この公開版の目的は**実装全体ではなく、設計そのものを共有すること**です。
