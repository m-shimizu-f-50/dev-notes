# Engineering Knowledge Base

このリポジトリは、ソフトウェア開発に関する技術ドキュメントをまとめたナレッジベースです。  
AI（Cursor など）を活用してドキュメントを生成し、学習・設計・レビューの参考資料として整理しています。

---

## 📂 ディレクトリ構成

```text
docs
│
├─ database
│   ├─ schema-design.md
│   ├─ index-design.md
│   └─ normalization.md
│
├─ react
│   ├─ hooks.md
│   ├─ state-management.md
│   └─ performance.md
│
├─ api
│   ├─ rest-design.md
│   └─ authentication.md
│
├─ redis
│   └─ caching-strategy.md
│
├─ security
│   └─ password-hashing.md
│
└─ infrastructure
    └─ scalability.md



## 質問フォーマット（テーマと要望を指定）

以下の形式でCursorに依頼してください。

prompts/doc-generator.md のプロンプトを使用して  
docs 配下に新しいフォルダを作成し、技術ドキュメントを生成してください。

作成するファイル:
docs/<folder-name>/<file-name>.md

テーマ:
<作成したいテーマ>

例:

prompts/doc-generator.md のプロンプトを使用して  
docs/tools フォルダを作成し、以下のドキュメントを作成してください。

作成ファイル:
docs/その他/SuperWhisperの使い方.md

テーマ:
Superwhisperの使い方