# Success Pack — AI 向け操作ガイド（サクセスパックの投入）

最終更新: 2026-07-22
対象読者: **外部の AI エージェント／別プロジェクトの LLM**。Success Pack にサクセスパックを作成・更新するための契約仕様。

> ステータス: **REST API・MCP ともに実装済み**（APIキー認証）。
> 認証: `Authorization: Bearer <YOUR_API_KEY>`（キーは Success Pack の「設定 → APIキー」で発行）。
> - REST ベースURL: `https://success-pack.work/api/v1`
> - MCP（HTTPホスト・導入不要）: `https://success-pack.work/api/mcp`
>   - 接続方法: MCP over HTTP に対応したクライアントで上記URL＋ヘッダ `Authorization: Bearer <YOUR_API_KEY>` を設定。
>   - 公開ツール（読取・本人確認）: `list_packs` / `get_pack` / `get_upstream_diff` / `whoami`
>   - 公開ツール（作成・複製）: `create_pack` / `fork_pack`
>   - 公開ツール（作成後に"育てる"＝構造編集）: `update_pack` / `update_task` / `delete_task` / `save_workstream` / `delete_workstream` / `save_phase` / `delete_phase` / `save_process` / `delete_process` / `reorder_tasks` / `delete_pack`
>   - 公開ツール（ファイル添付）: `upload_asset`
>   - 公開ツール（上流の取り込み＝合流）: `preview_merge`（試算）/ `merge_upstream`（適用）
>   - 公開ツール（チーム共有＝閲覧のみ）: `list_groups` / `create_group` / `add_group_member` / `share_pack` / `unshare_pack` / `list_pack_shares` ほか

---

## 1. これは何か
Success Pack は「SEの実践知」を **Pack（サクセスパック）** 単位で集約するプラットフォーム。
外部 AI が担うのは主に **1つの Pack を組み立てて投入する**こと。人手の入稿UIと**同じ操作レイヤー**を API/MCP で叩く。

## 2. ドメインモデル（投入時に組み立てる構造）
```
Pack（サクセスパック）
 └ Workstream[]（マトリクスの行）× Phase[]（列）
     └ Process[]（セル＝行項目。workstreamId × phaseId で配置）
         └ Task[]（実践知の最小単位）
             └ why(観点) / raci / agenda / talkScript / checkpoints / inputs / outputs / assets / tags
```
- 子ID（workstream/phase/process/task の `id`）は **Pack 内で一意**（スラッグ）。Pack を跨いだ一意性は不要。
- `Process.workstreamId` / `Process.phaseId` は同じ Pack 内の workstream/phase の `id` を指す。

## 3. 交換フォーマット（Pack JSON）
外部 AI はこの形の JSON を1件生成して投入する。`?` は任意。

```jsonc
{
  "slug": "acme-onboarding",           // Pack内グローバル一意（英数-）。community は owner 側で衝突回避
  "title": "自社オンボーディング",
  "description": "…",                   // ?
  "accent": "#0f766e",                 // ? カバー色
  "visibility": "private",             // "private" | "unlisted" | "public"（既定 private）
  "workstreams": [
    { "id": "pm", "name": "案件マネジメント", "color": "#b45309", "order": 1 }
  ],
  "phases": [
    { "id": "start", "name": "開始", "order": 1 }
  ],
  "processes": [
    {
      "id": "kickoff",
      "title": "立ち上げ",
      "summary": "…",                   // ? ホバー時の一言
      "workstreamId": "pm",
      "phaseId": "start",           // 並びは processes/tasks とも配列順（order フィールドは無い）
      "tasks": [
        {
          "id": "kickoff-mtg",
          "title": "キックオフMTG",
          "status": "published",        // "draft" | "published"（既定 draft）
          "summary": "…",               // ?
          "why": "…",                   // ? なぜ大事か・陥りがちな失敗
          "raci": [ { "role": "R", "people": ["SE"] } ], // ?
          "how": {                       // ? 手順（Now Create の How）
            "intro": "テンプレを用いて以下を明確化する:",
            "ordered": false,            // true=番号付き / false=箇条書き
            "steps": [ { "label": "背景", "text": "…" } ] // label は太字ラベル（任意）
          },
          "inputs": [                    // ? 材料。kind で分類            { "title": "要件定義書", "kind": "project",
              "from": { "taskId": "req-def", "outputId": "o1a2b3c4d" } },  // ? 前工程の成果物にリンク（相手の output.id を指す）
            { "title": "契約書",     "kind": "external" }  // 外部/与件（前タスクに紐づかない）
          ],
          "suppliedAssets": [            // ? 付属テンプレ（ファイル/リンク/本文を持つのはここだけ）
            { "title": "議事録テンプレ", "type": "word", "sizeLabel": "48 KB", "downloadUrl": "https://…" }, // 直リンク
            { "title": "設計メモ", "type": "gdoc", "url": "https://docs.google.com/document/d/…" },          // Google/Notion はリンク
            { "title": "準備チェックリスト", "type": "md", "content": "# 見出し\n- 箇条書き" }                 // Markdown は本文を content に
          ],
          "outputs": [ { "title": "議事録", "description": "…" } ], // ? 成果物（id はサーバ採番。更新時は get で得た id を保って送る＝安全。省略時はタイトル一致で既存 id を引き継ぐ）
          "sections": [                 // ? 著者が組む自由項目（固定枠に無いもの）
            { "label": "設計判断ログ", "body": "## 主要な判断\n- **DB**: PostgreSQL" } // body は Markdown
          ],
          "tags": ["会議", "立ち上げ"], // ?
          "agenda": ["背景と目的"],     // ? Success Pack独自の補助（任意）
          "talkScript": "…",            // ? Success Pack独自
          "checkpoints": [ { "text": "目的が言語化されているか", "hint": "…" } ] // ? Success Pack独自
        }
      ]
    }
  ]
}
```

### 値の制約
- `raci[].role`: `"R" | "A" | "C" | "I"`（実行/説明/相談/報告）
- `inputs[].kind`: `"project"`（前工程の成果物・既定）/ `"external"`（外部・与件）- `inputs[].from`（任意・project のみ）: 同 Pack 内の前工程 Output を参照 `{ taskId, outputId }`。`outputId` は相手タスクの `outputs[].id`（サーバ採番）。詳細画面で双方向リンク（入力→前工程 / 成果物→下流タスク）を表示。
- `suppliedAssets[].type`: `"word" | "ppt" | "excel" | "pdf" | "gslides" | "gdoc" | "gsheet" | "md" | "notion" | "other"`
  - `md` は `content`（Markdown 本文）を持ち、詳細画面でインライン描画（DL不要・GFM/表/チェックリスト対応・HTMLは無効化しXSS防止）。
  - `notion` は `url`（Notion 公開ページ）。詳細画面は「Notionで開く」＋埋め込みプレビュー試行。
  - Google 3種・`notion` は `url` にリンク、ファイル系は `downloadUrl`（静的）か Storage の `path`（→ 署名URL）。
- **ファイルを持つのは `suppliedAssets` のみ**。`inputs`/`outputs` は名前＋説明（重複させない）。
- `suppliedAssets` の URL（`url`/`downloadUrl`/`viewUrl`）は `http(s)://` または `/` 始まりのみ（`javascript:` 等は拒否）。
- 子 `id`（workstream/phase/process/task）は **Pack 内で一意**（task id はプロセスを跨いでも重複不可）。
- `visibility`: `"private"`（自分だけ）/ `"unlisted"`（URL共有・一覧非掲載）/ `"public"`（全体公開）
- `origin` / `owner` / `locked` / `version` は **サーバが決定**（投入側は指定しない）。外部投入は必ず `origin="community"`、`owner=APIキーの持ち主`、`locked=false`。

## 4. 操作契約（REST・MCP 実装済み・同名ツール）
認証は **API キー**（→ キーの持ち主 = owner に解決）。全エンドポイントに `Authorization: Bearer <YOUR_API_KEY>` が必要。
MCP は同じ操作を JSON-RPC の `tools/call`（`{"name": "<ツール名>", "arguments": {…}}`）で公開する。

| 操作 | REST | MCP ツール | 説明 |
|---|---|---|---|
| 一覧取得 | `GET /api/v1/packs` | `list_packs` | 自分から見える範囲（公式∪public∪own） |
| 取得 | `GET /api/v1/packs/:slug` | `get_pack` | 1件（ネスト込み） |
| 本人確認 | `GET /api/v1/me` | `whoami` | キーの持ち主（= 作成する owner）を返す |
| **投入（新規）** | `POST /api/v1/packs` | `create_pack` | §3 の Pack JSON を丸ごと作成（201, `{slug}`） |
| メタ更新 | `PATCH /api/v1/packs/:slug` | `update_pack` | title/description/accent/visibility/**currentPhase**（現在地。空文字で未設定に戻す） |
| タスク更新 | `PATCH /api/v1/packs/:slug/tasks/:id` | `update_task` | **全置換**（省略フィールドは消える。先に取得して完全な task を送る）。新規は `processId` 必須。既存タスクの `processId` 変更（移動）は不可 |
| タスク削除 | `DELETE /api/v1/packs/:slug/tasks/:id` | `delete_task` | 1件削除（MCP は `processId` も渡す） |
| WS 追加/更新 | `POST /api/v1/packs/:slug/workstreams` | `save_workstream` | `id` 省略で新規・指定で更新（縦レーン） |
| WS 削除 | `DELETE /api/v1/packs/:slug/workstreams/:id` | `delete_workstream` | 参照中プロセスがあると拒否 |
| フェーズ 追加/更新 | `POST /api/v1/packs/:slug/phases` | `save_phase` | `id` 省略で新規・指定で更新（横工程） |
| フェーズ 削除 | `DELETE /api/v1/packs/:slug/phases/:id` | `delete_phase` | 参照中プロセスがあると拒否 |
| プロセス 追加/更新 | `POST /api/v1/packs/:slug/processes` | `save_process` | `id` 省略で新規（セル項目・ws/phase を指す） |
| プロセス 削除 | `DELETE /api/v1/packs/:slug/processes/:processId` | `delete_process` | 配下タスクも cascade で削除 |
| タスク並替 | `POST /api/v1/packs/:slug/processes/:processId/reorder` | `reorder_tasks` | body `{ ids }` の順に order 振り直し |
| 継承 | `POST /api/v1/packs/:slug/fork` | `fork_pack` | 公式/他パックをハードコピーして own 化（新slugは REST=`slug` / MCP=`newSlug`） |
| 上流差分 | `GET /api/v1/packs/:slug/upstream` | `get_upstream_diff` | 派生元の更新差分（表示専用・base↔latest の2-way） |
| 取り込み試算 | `GET /api/v1/packs/:slug/merge` | `preview_merge` | 上流を取り込むと何が変わるかの試算（適用しない・3-way） |
| 取り込み（合流） | `POST /api/v1/packs/:slug/merge` | `merge_upstream` | 上流更新を実際に取り込む。「上流が変更 × 自分は未編集」のみ自動反映し、競合はローカル維持＋件数報告。取り込み後は上流最新に追従 |
| **パック削除** | `DELETE /api/v1/packs/:slug` | `delete_pack` | 丸ごと削除（**取り消し不可**） |
| ファイル添付 | `POST /api/v1/packs/:slug/assets` | `upload_asset` | テンプレ実ファイルを Storage に保存し `path` を返す（base64・上限10MB）→ task の `suppliedAssets[].path` へ入れて確定 |

### チーム共有（グループ）
自分のパックを **チーム（グループ）に「閲覧のみ」で共有** する。共有してもメンバーは読むだけで、**編集権限は owner に残る**。共有先は「自分が所有 or 参加しているチーム」のみ。

| 操作 | REST | MCP ツール | 説明 |
|---|---|---|---|
| チーム一覧 | `GET /api/v1/groups` | `list_groups` | 自分が所有 or 参加しているチーム（id/名前/役割/人数） |
| チーム作成 | `POST /api/v1/groups` | `create_group` | body `{name}`。作成者が owner・自動でメンバー（201, `{id}`） |
| チーム改名 | `PATCH /api/v1/groups/:id` | `rename_group` | body `{name}`（owner のみ） |
| チーム削除 | `DELETE /api/v1/groups/:id` | `delete_group` | 共有も全解除（パック自体は消えない・owner のみ） |
| メンバー一覧 | `GET /api/v1/groups/:id/members` | `list_group_members` | owner は全員・メンバーは自分の行のみ |
| ユーザー検索 | `GET /api/v1/users?q=` | `search_users` | 表示名/氏名/メールで招待候補を検索。返る `userId` を追加に使う |
| メンバー追加 | `POST /api/v1/groups/:id/members` | `add_group_member` | body `{userId}`（search_users で取得・推奨）か `{email}`。相手は一度サインイン済みが必要（owner のみ） |
| メンバー削除 | `DELETE /api/v1/groups/:id/members` | `remove_group_member` | `?userId=` か body `{userId}`（owner or 本人の退出。owner は外せない） |
| 共有先一覧 | `GET /api/v1/packs/:slug/shares` | `list_pack_shares` | このパックの共有先チーム（groupId/名前） |
| **共有する** | `POST /api/v1/packs/:slug/shares` | `share_pack` | body `{groupId}`。パックをチームへ閲覧共有（201） |
| 共有解除 | `DELETE /api/v1/packs/:slug/shares` | `unshare_pack` | `?groupId=` か body `{groupId}`（パック owner のみ） |

典型フロー: `create_group`（→ `id`）→ `add_group_member`（email で招待）→ `share_pack`（slug × groupId）。共有されたパックはメンバーの一覧に「共有・閲覧のみ」として並ぶ。相手に編集させたい場合は共有ではなく `fork_pack` を案内する。

### レート制限
書き込み系（create_pack / update_pack / update_task / delete_task / save_workstream / delete_workstream / save_phase / delete_phase / save_process / delete_process / reorder_tasks / fork_pack / delete_pack / upload_asset / merge_upstream / create_group / rename_group / delete_group / add_group_member / remove_group_member / share_pack / unshare_pack）は **1キーあたり 60回/分**（`whoami` / `preview_merge` / `list_groups` などの読取は対象外）。
超過時は REST が `429`、MCP は `isError` の結果を返す。少し待って再試行すること。

### 不変条件（サーバが必ず強制する）
1. 書き込めるのは **自分が owner の community パックのみ**。
2. **公式(locked)・他人のパックは書込不可**。公式への改善は「提案→レビュー→新版発行」の別導線。
3. 公開範囲を尊重：private は所有者のみ。
4. 継承は**ハードコピー**。`fork` 後は元と独立（自動追従しない。差分は `get_upstream_diff` で確認）。
5. **チーム共有は閲覧のみ**。共有相手は読めるが書き込めない（編集は owner だけ）。private パックでも共有相手には見える。

## 5. 最小の投入手順（外部AI視点）
1. §3 の Pack JSON を1件生成する（workstreams/phases を先に定義し、processes がそれを参照）。
2. `create_pack`（or `POST /api/v1/packs`）に JSON を渡す。`visibility` は既定 `private` で作り、確認後に上げる。
3. レスポンスの `slug` で `get_pack` して投入結果を確認。
4. 修正は `update_task` / `update_pack` で部分更新。
5. **後から"育てる"（Git的に成長させる）**：列を足す=`save_workstream`、工程を足す=`save_phase`、セル項目を足す=`save_process`→そこへ `update_task`。不要になったら対応する `delete_*`（プロセス削除は配下タスクも消える）。並べ替えは `reorder_tasks`。`create_pack` は一度きり（既存 slug には使えない）＝2回目以降の構造変更はこれらの粒度細かいツールで行う。

## 6. よくある誤り
- workstreamId/phaseId が processes 側で未定義 → **先に workstreams/phases を入れる**。
- 子 `id` の重複（同一 Pack 内） → Pack 内で一意にする。
- `origin`/`owner`/`locked` を指定 → 無視される（サーバ決定）。
- 公式 slug に対して書込 → 拒否（fork してから編集する）。

## 7. 関連
- **オーサリング スキル**: [SKILL.md](SKILL.md)（このリポジトリ。「生ナレッジ→良いパック」の手引き）
- **Success Pack 本体**: https://success-pack.work
- 稼働中のツール一覧は MCP の `tools/list`（JSON-RPC）でいつでも確認できる（このリファレンスはそのスナップショット）。
