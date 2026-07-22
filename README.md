# Success Pack — AI オーサリング スキル

> あなたのAI（Claude など）に、積み上げたナレッジを [**Success Pack**](https://success-pack.work) の
> 「サクセスパック」に変換・管理させるための **スキル**と**接続ガイド**。

[Success Pack](https://success-pack.work) は、SE・業務の実践知（アジェンダ／トークスクリプト／
観点／テンプレ）を **方法論のマトリクス**として溜める、AIネイティブなナレッジ基盤です。
ServiceNow の Now Create を手本に、**Pack ＞ マトリクス(職能×工程) ＞ プロセス ＞ タスク** の
4層で知見を構造化します。

このリポジトリは、**利用者自身のAIに読ませる**ための2点を提供します：

- **[`SKILL.md`](SKILL.md)** — 「生ナレッジ → 良いパック」への変換手順（AIが従うガイド）
- **[`API-REFERENCE.md`](API-REFERENCE.md)** — MCP / REST の正確なツール契約と交換フォーマット

---

## クイックスタート

1. **APIキーを発行**：Success Pack にログイン →「設定 → APIキー」で発行（発行時のみ平文表示）
2. **あなたのAIを接続**（Success Pack のキー発行画面に、キーを差し込んだコマンド/設定がそのまま表示されます）：

   - **Claude Code**（AI自身に接続させられます）:
     ```
     claude mcp add --transport http success-pack https://success-pack.work/api/mcp --header "Authorization: Bearer <YOUR_API_KEY>"
     ```
     「このコマンドを実行して」と頼めば AI が自分で接続します。クロスプロジェクトで使うなら末尾に `--scope user`。
   - **Claude Desktop**: `claude_desktop_config.json` の `mcpServers` に追記して再起動：
     ```json
     { "mcpServers": { "success-pack": { "url": "https://success-pack.work/api/mcp", "headers": { "Authorization": "Bearer <YOUR_API_KEY>" } } } }
     ```
   - **claude.ai（Web）**: 現状カスタムの Bearer トークンに未対応（OAuthコネクターのみ）→ 上記2つのいずれかをご利用ください。
3. **スキルを読ませる**：[`SKILL.md`](SKILL.md) をAIのコンテキストに入れる
   （Claude Desktop / Claude Code のプロジェクト指示、claude.ai のプロジェクト など）
4. **頼む**：「この議事録をサクセスパックにして」「このパックに〇〇工程を足して」

AI が `list_packs` / `create_pack` / `save_process` / `update_task` などのツールを使って、
あなたのナレッジを構造化・投入・更新します。**作成物は既定で private（あなただけ）**。

### REST でも同じことができます

MCP を使わないクライアントは、同じ操作を REST で叩けます（同名・`https://success-pack.work/api/v1`）。
詳細は [`API-REFERENCE.md`](API-REFERENCE.md)。

---

## 仕組み（要点）

- 認証はAPIキー＝**キーの持ち主が所有者**。作られるパックは `community`（あなたの所有）。
- 書き込めるのは **自分のパックだけ**。公式パック・他人のパックは読めても書けません。
- 公開範囲は `private` / `unlisted`（リンク共有）/ `public`（全体）から選べます。
- 書き込みは 1キーあたり 60回/分。

## MCP ツール一覧（抜粋）

| 目的 | ツール |
|---|---|
| 読む・本人確認 | `list_packs` `get_pack` `whoami` `get_upstream_diff` |
| 作る・複製 | `create_pack`（構造まるごと1発） `fork_pack` |
| 育てる（構造編集） | `save_workstream` `save_phase` `save_process` `update_task` `reorder_tasks` `delete_task` `delete_process` `delete_workstream` `delete_phase` `delete_pack` |
| ファイル添付 | `upload_asset` |

全ツールの入出力・交換フォーマット・値の制約は [`API-REFERENCE.md`](API-REFERENCE.md) を参照。

---

## このリポジトリについて

- **スキル文書は自由に利用・改変可**（あなたのAIやワークフローに合わせて調整してください）。
- 内容は Success Pack の稼働中の API/MCP 契約に追従します。齟齬に気づいたら Issue へ。
- Success Pack 本体: **https://success-pack.work**
