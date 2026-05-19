# Zetter Chat Harness Design

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

このリポジトリには、Zetter chat harness の**機密情報を除いた設計文書**を収録しています。

This repository contains **sanitized architecture and design documents** for the Zetter chat harness.

現在の公開ドキュメントは、**カスタムDB検索（ユーザー起動型）** と **プランナー・ファイナライザーへのコンテキスト増量** を中心とした V3 設計を反映しています。

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

### 現行設計（V3）

- `docs/zetter-chat-harness-v3-spec.md`
  - **現行**アーキテクチャ仕様。カスタムDB検索・プランナー・ファイナライザー・スレッドコンテキスト管理を含む / Current architecture: custom DB search, planner, finalizer, thread context management

### 過去の設計文書（参照用）

- `docs/zetter-chat-harness-v2-spec.md`
  - V2 ハーネス設計（flash-only coordinator, reasoning barrier 更新を含む） / V2 harness design with flash-only coordinator and reasoning barrier updates
- `docs/zetter-chat-reference-resolution-spec.md`
  - 参照解決レイヤ設計 / Reference resolution layer design
- `docs/zetter-chat-eval-determinism-spec.md`
  - Eval harness と deterministic 回帰設計 / Eval harness and deterministic regression design
- `docs/chat-tool-channel-spec.md`
  - ユーザー向け回答と tool 制御経路の分離設計 / Separation of answer flow and tool-control flow
- `docs/chat-safe-tools-spec.md`
  - 安全な tool 境界設計 / Safe tool boundary design

## Notes

## 補足

- Some documents refer to internal runtime component names to explain the architecture.
- This export is meant to share the **system design** rather than the full implementation.
- 一部の文書では、アーキテクチャ説明のために内部ランタイム上のコンポーネント名を参照しています。
- この公開版の目的は**実装全体ではなく、設計そのものを共有すること**です。
