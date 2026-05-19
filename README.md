# Zetter Chat Harness Design

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/anitigravitylab-oss/zetter-chat-harness-design)

このリポジトリには、Zetter chat harness の**機密情報を除いた設計文書**を収録しています。

This repository contains **sanitized architecture and design documents** for the Zetter chat harness.

この公開版には**設計資料のみ**を含め、次のものは含めません。

This public export intentionally includes **design-only materials** and excludes:

- application source code
- production data
- credentials, secrets, tokens, and connection details
- deployment runbooks and environment-specific operational notes

## 収録ドキュメント / Included documents

- `docs/zetter-chat-harness-spec.md`
  - 現行アーキテクチャ仕様。プランナー・DB エグゼキューター・直接回答パス・スレッドコンテキスト管理を含む
  - Current architecture: planner, DB executor, direct-answer path, thread context management

## 補足 / Notes

- Some documents refer to internal runtime component names to explain the architecture.
- This export is meant to share the **system design** rather than the full implementation.
- 一部の文書では、アーキテクチャ説明のために内部ランタイム上のコンポーネント名を参照しています。
- この公開版の目的は**実装全体ではなく、設計そのものを共有すること**です。
