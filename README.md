# Success Pack — AI オーサリング スキル

> あなたのAI（Claude など）に、積み上げたナレッジを [**Success Pack**](https://success-pack.work) の
> 「サクセスパック」に変換・管理させるための **スキル**と**接続ガイド**。

[Success Pack](https://success-pack.work) は、SE・業務の実践知（アジェンダ／トークスクリプト／
観点／テンプレ）を **方法論のマトリクス**として溜める、AIネイティブなナレッジ基盤です。
ServiceNow の Now Create を手本に、**Pack ＞ マトリクス(職能×工程) ＞ プロセス ＞ タスク** の
4層で知見を構造化します。

このリポジトリは、**利用者自身のAIに読ませる／組み込む**ための4点を提供します：

- **[`SKILL.md`](SKILL.md)** — 「生ナレッジ → 良いパック」への変換手順（Claude Code スキルとしてそのまま使える）
- **[`agents/success-pack-author.md`](agents/success-pack-author.md)** — オーサリング専任の**エージェント定義**（Claude Code のサブエージェントとしてインストール可）
- **[`examples/minimal-pack.json`](examples/minimal-pack.json)** — `create_pack` にそのまま渡せる**理想形のお手本**（縦=仕事の種類/横=時間/Why+How/業務フローリンクを1つずつ実演）
- **[`API-REFERENCE.md`](API-REFERENCE.md)** — MCP / REST の正確なツール契約と交換フォーマット

> スキル/エージェントには、実運用で踏んだ失敗から得たルールを焼き込んでいます：
> **縦=仕事の種類（チーム名・製品名で切らない）／横=標準6フェーズ／迷ったら公式パックを手本に／
> 捏造しない／private で作ってレビュー後に公開／納品前セルフチェック**。

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
3. **スキル／エージェントを入れる**（Claude Code の場合。どちらか片方でも動きます）：

   ```bash
   git clone https://github.com/taIshikawa/success-pack-skill
   # スキル（/success-pack-authoring で明示起動も可能）
   mkdir -p ~/.claude/skills/success-pack-authoring
   cp success-pack-skill/SKILL.md ~/.claude/skills/success-pack-authoring/SKILL.md
   # エージェント（「サクセスパックにして」で自動起用される）
   mkdir -p ~/.claude/agents
   cp success-pack-skill/agents/success-pack-author.md ~/.claude/agents/
   ```

   プロジェクト単位で入れるなら `~/.claude/` の代わりにリポジトリ直下の `.claude/` へ。
   Claude Desktop / claude.ai は [`SKILL.md`](SKILL.md) の本文をプロジェクト指示に貼るだけでもOK。
4. **頼む**：「この議事録をサクセスパックにして」「このパックに〇〇工程を足して」「チームに共有して」

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
| 上流の取り込み（合流） | `preview_merge`（試算） `merge_upstream`（あなたの編集は上書きしない3-way） |
| チーム共有 | `create_group` `search_users` `add_group_member` `share_pack`（view=閲覧/edit=編集可） `unshare_pack` `list_groups` `list_group_members` `list_pack_shares` `rename_group` `remove_group_member` `delete_group` |

全ツールの入出力・交換フォーマット・値の制約は [`API-REFERENCE.md`](API-REFERENCE.md) を参照。

---

## このリポジトリについて

- **スキル文書は自由に利用・改変可**（あなたのAIやワークフローに合わせて調整してください）。
- 内容は Success Pack の稼働中の API/MCP 契約に追従します。齟齬に気づいたら Issue へ。
- Success Pack 本体: **https://success-pack.work**
