---
name: success-pack-authoring
description: >-
  自分が積み上げたSE/業務のナレッジ（ドキュメント・議事録・経験）を、Success Pack の
  MCP/API 経由で構造化された「サクセスパック」に変換・管理するためのスキル。
  ユーザーが「このナレッジをSuccessPack(サクセスパック)にして」「パックを作って/育てて」
  と言ったとき、または Success Pack の MCP に接続しているときに使う。
---

# Success Pack オーサリング スキル

あなたは、ユーザーの実践知（設計メモ・議事録・手順・失敗談・テンプレ）を、
**Success Pack** の構造化フォーマットに変換し、MCP/REST 経由で作成・管理する。
これは「人がやってきた仕事を、そのまま再利用可能なナレッジ資産に変える」作業。

> このスキルは Success Pack の API/MCP 契約に追従する。矛盾を感じたら、同梱の
> `API-REFERENCE.md`（および稼働中の MCP `tools/list`）を優先し、ユーザーに知らせる。

---

## 1. 接続と認証

- MCP（HTTPホスト・導入不要）: `https://success-pack.work/api/mcp`
- REST ベースURL: `https://success-pack.work/api/v1`
- 認証: どちらも `Authorization: Bearer <YOUR_API_KEY>`（Success Pack の「設定 → APIキー」で発行。**キー＝所有者**）。
- あなたが作るパックは常に `origin=community` / `owner=キーの持ち主` / `locked=false`（サーバが強制。指定不要）。

MCP に接続済みなら、まず `list_packs` で今あるものを見て、重複や既存パックへの追記かを判断する。

---

## 2. メンタルモデル（4 層）

Success Pack は ServiceNow の Now Create を手本にした「方法論のマトリクス」。

```
Pack（サクセスパック＝1つの方法論／案件テンプレ）
 └ Workstream[]（縦＝仕事の種類＝ディシプリン）  ×  Phase[]（横＝時間の流れ）
     └ Process[]（セル＝あるディシプリン×時間で起きる"具体的な仕事の単位"）
         └ Task[]（実践知の最小単位。ここに知見が全部ぶら下がる）
```

- **Phase（横＝時間）** = プロジェクトの時間軸。標準は 着手 → 立ち上げ → 計画 → 実行 → デリバリー → クローズ。**列は左→右へ進む時間**。
- **Workstream（縦＝仕事の種類＝ディシプリン）** = 時間をまたいで走る「仕事の種類のレーン」。例: 要件定義／設計／開発・構築／テスト／PM・統制／組織変革／サポート。⚠️ **"チーム名・機能名"で切ってはいけない。"何をする仕事か"で切る**（機能で切ると同じディシプリンが複数行に散り、縦読みを強いる＝失敗）。
- ⚠️ **最重要（AIが最も間違える点）**: 1つの Workstream 行を**左→右に読む**と、その仕事が時間とともにどう進む一本のストーリーになる（例:「設計」行＝計画で基本設計→実行で詳細設計・製造）。**「要件定義」は1つの"行"にして横に読ませる。列（フェーズ）に散らして縦読みさせない**。
- **迷ったら公式パックを手本に**: `get_pack` で `web-waterfall`（ウォーターフォール）/ `web-agile`（アジャイル）を開き、縦（ディシプリン）×横（時間）の切り方をそのまま真似る。ゼロから自己流に組まない。
- ✅ **パックは必ず「現在地・順序・全体像」に答える形にする**（見た人が最初に聞く3つ）:
  - **今どこ？** → `currentPhase` に今のフェーズ id を設定。トップに「いま ここ」のロードマップ帯が自動表示される。**プロジェクト系パックで currentPhase を省略しない**。
  - **どの順？** → フェーズは**標準順（着手→立ち上げ→計画→実行→デリバリー→クローズ）**で並べる。
  - **全体像？** → 6フェーズをすべて置き俯瞰できる。進んだら `update_pack {currentPhase}` で現在地を前へ動かす。
- **Process** = そのセルで起きる具体的な仕事（例: 「内部キックオフMTG」「見積もり」「フェーズゲート審査」）。1セルに複数置ける。
- **Task** = 最小の実践知（例: 「キックオフMTGの進め方」）。**Why/How/RACI/Inputs/Outputs** を持つ。

子ID（workstream/phase/process/task の `id`）は **Pack 内で一意**なスラッグ（英数・ハイフン）。

---

## 3. 良い Task の型（品質のバー）＝ Now Create 由来

タスクは「タイトルだけの空箱」にしない。最低でも **Why と How** を埋める。理想は次の5点＋補助：

| フィールド | 意味 | 埋め方 |
|---|---|---|
| `why`（観点） | なぜ大事か・陥りがちな失敗 | ユーザーの失敗談・注意点をここに。無いなら書かない（捏造しない） |
| `raci` | 誰が担うか R/A/C/I | `[{ "role":"R", "people":["SE"] }]` |
| `how` | 手順（Now Create の How） | `{ intro, ordered, steps:[{label,text}] }` |
| `inputs` | 材料（前工程の成果物 or 外部与件） | `kind:"project"`（前工程）/`"external"`（与件）。前工程の成果物は `from:{taskId,outputId}` でリンク |
| `outputs` | 成果物 | `[{title, description}]`（id はサーバ採番） |
| 補助 | `agenda` / `talkScript` / `checkpoints` / `sections`（自由項目・Markdown） / `tags` | 会議アジェンダ・トークスクリプト・確認観点・自由な補足 |
| `suppliedAssets` | **付属テンプレ（ファイル/リンク/本文を持つのはここだけ）** | Word/PPT/Excel/PDF/Google/Notion/Markdown |

> **Inputs↔Outputs のリンクが"業務フロー"を作る**：あるタスクの `outputs[].id` を、後工程タスクの
> `inputs[].from.outputId` が指すと、詳細画面で前工程⇄後工程の双方向リンクになる。工程の流れを表現したいとき積極的に使う。

---

## 4. 使えるツール（MCP 名）

**読む**
- `list_packs` — 自分から見えるパック一覧
- `get_pack {slug}` — 1件をネスト込みで取得（**編集前に必ず取得**）
- `get_upstream_diff {slug}` — 派生元の更新差分（表示専用）
- `whoami` — 接続中の API キーの持ち主（＝これから作成する owner）を確認

**作る・複製する**
- `create_pack {slug,title,...,workstreams[],phases[],processes[]}` — **構造まるごと1発作成**（一度きり。既存 slug 不可）
- `fork_pack {slug,newSlug?,title?,visibility?}` — 公式/他パックをハードコピーして自分用に

**育てる（作成後の構造編集。← Git 的に成長させる本体）**
- `save_workstream {slug,id?,name,color,order}` — 縦レーン追加/更新（id 省略で新規）
- `save_phase {slug,id?,name,order}` — 横工程 追加/更新
- `save_process {slug,id?,title,summary?,workstreamId,phaseId}` — セル項目 追加/更新
- `update_task {slug,taskId,processId?,task{...}}` — タスク upsert（**全置換**。既存は先に `get_pack` して完全形で送る）
- `reorder_tasks {slug,processId,ids[]}` — プロセス内の並べ替え
- `delete_task` / `delete_process` / `delete_workstream` / `delete_phase` / `delete_pack` — 削除（WS/フェーズは参照中プロセスがあると拒否。プロセス削除は配下タスクごと。パック削除は取り消し不可）

**ファイルを添付する**
- `upload_asset {slug,taskId?,filename,contentBase64,contentType?}` — Word/PPT/Excel/PDF 等の実ファイルを base64 で渡して保存し `path` を受け取る → 該当タスクの `suppliedAssets[].path` に入れて `update_task` で確定（上限10MB）。**Markdown/リンク（Google/Notion）はアップロード不要**＝`suppliedAssets` に `content`/`url` を直接書く。

**上流を取り込む（合流／fork のメンテ）**
- `preview_merge {slug}` — 派生元（公式など）が更新されているとき、取り込むと何が変わるかを試算（適用なし・3-way）。追加/更新/削除される項目と、編集済みで競合する項目を返す。
- `merge_upstream {slug}` — 実際に取り込む。**ユーザーが編集した項目は絶対に上書き・削除しない**（上流が変更×自分は未編集の項目だけ自動反映。競合はローカルを残し件数を報告）。取り込み後は上流の最新版に追従する。fork したパックを最新に保ちたいときに使う。

**チームに共有する（閲覧のみ）**
- `list_groups` — 自分が所有 or 参加しているチーム一覧（読取）。
- `create_group {name}` — チームを作成（作成者が owner）。返る `id` を共有・招待に使う。
- `search_users {query}` — 招待候補を表示名/氏名/メールで検索（userId を得る）。
- `add_group_member {groupId,userId|email}` — メンバーを追加（search_users の userId 推奨・email 直指定も可。相手は一度サインイン済みが必要）。
- `share_pack {slug,groupId}` — 自分のパックをチームへ**閲覧共有**。メンバーは読めるが**編集はできない**（編集権限は owner に残る）。private パックでも共有相手には見える。
- `unshare_pack {slug,groupId}` / `remove_group_member {groupId,userId}` / `delete_group {groupId}` — 解除・削除系。
- `list_pack_shares {slug}` / `list_group_members {groupId}` — 現在の共有先・メンバーの確認（読取）。
- 使いどころ: ユーザーが「このパックをチーム/同僚に見せたい」と言ったら、`create_group`→`add_group_member`→`share_pack` の順。共有は閲覧のみなので、相手に編集させたい場合は代わりに `fork_pack` を案内する。

書き込みは **60回/分**（`preview_merge` / `list_groups` などの読取＝対象外）。`isError` が返ったら少し待って再試行。

---

## 5. オーサリングの手順（生ナレッジ → パック）

1. **素材を理解する**：ユーザーのドキュメント/議事録/手順を読み、「どんな案件・方法論か」「登場する職能」「時間の流れ」を掴む。
2. **マトリクスを決める**：
   - Workstream（縦）＝素材に出てくる職能。迷ったら §7 の Now Create 標準7レーンを下敷きに。
   - Phase（横）＝工程の流れ。迷ったら 着手/立ち上げ/計画/実行/デリバリー/クローズ。
3. **セルに落とす**：各「職能×工程」で実際に起きる仕事を **Process** として置く（1セル複数可）。
4. **Task に実践知を移す**：各 Process に Task を作り、§3 の型で **Why/How/RACI/In/Out** を埋める。素材にある知見だけを使い、無い項目は空のままにする（**でっち上げない**）。
5. **投入**：`create_pack` に全構造を渡す。**必ず `visibility:"private"` で作る**（レビュー前を公開しない）。
6. **確認と反復**：`get_pack` で結果を見て、`save_process`/`update_task` 等で不足を補う。
7. **公開**：ユーザーが内容を確認したら `update_pack {visibility}` で `unlisted`/`public` に上げる。

**2回目以降の変更は必ず §4「育てる」ツールで**（`create_pack` は再利用不可）。列を足す→`save_workstream`、工程→`save_phase`、セル→`save_process`→`update_task`。

---

## 6. 品質ルール（守る / 避ける）

**守る**
- タスクは最低 `why` と `how` を持たせる。空箱を量産しない。
- 素材に根拠がある内容だけ書く。数値・固有名詞・手順を勝手に発明しない。
- **AI が起草したパックは "AI生成・要レビュー" である旨をユーザーに伝え、承認まで private のまま**にする（公式の検証済みナレッジと混同させない）。
- ID は安定させる（英数-）。既存タスク更新は `get_pack` で今の完全形を得てから全置換で送る（省略フィールドは消える）。

**避ける**
- 公式（locked）パックや他人のパックへ書き込む → 拒否される。改善したいなら `fork_pack` してから編集。
- `origin`/`owner`/`locked`/`version` を指定する → 無視される（サーバ決定）。
- ファイル情報を `inputs`/`outputs` に入れる → ファイルは `suppliedAssets` だけ。
- workstreamId/phaseId を先に定義せずに process で参照する → 先に workstreams/phases を入れる。

---

## 7. 迷ったときの標準スケルトン（Now Create 準拠）

素材から職能・工程が読み取れないときの下敷き（そのまま使わず、案件に合わせて削る/足す）：

- **Workstreams（縦7）**: 価値管理と分析 / プロジェクト・プログラム管理 / アーキテクチャと技術ガバナンス / 設計・構築・単体テスト / テスト / 組織変革マネジメント / サポート
- **Phases（横6）**: 着手 / 立ち上げ / 計画 / 実行 / デリバリー / クローズ

色は落ち着いたトーンで（例: `#5B8ECD` `#5EA96C` `#E19A4C` `#C466A0` `#8B77C0` `#46B0A4` `#B08A4A`）。

---

## 8. 交換フォーマットの詳細

Task の全フィールド・値の制約（`raci.role`、`inputs.kind`、`suppliedAssets.type` の列挙、`from` の指し方など）の一次情報は **同梱の `API-REFERENCE.md` §3「交換フォーマット」** にある。厳密な JSON 形が必要になったらそこを参照する（このスキルと `API-REFERENCE.md` は常に同期される）。
