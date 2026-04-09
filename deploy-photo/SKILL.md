---
name: deploy-photo
description: stock写真をimg/にデプロイし、QMDパス更新・catalog更新・DVC管理を一括実行する。「写真を配置」「写真を差し替え」「画像をデプロイ」「deploy-photo」と言われた時に使用。
---

# 写真デプロイ

## 目的

`img/stock/` のストック写真を講義スライド用に `img/` へデプロイし、QMDパス更新・catalog.md更新・DVC管理・旧ファイル整理を一括で行う。手動ステップの漏れ（gitignore残骸、catalog未更新、JSON欠落、意味的不一致等）を防ぐ。

## 前提

- DVC初期化済み（`dvc remote` 設定済み）
- ImageMagick（`identify`, `magick`）がインストール済み
- stock写真は `img/stock/` 配下に既に配置されていること
- stock/にない写真を使いたい場合は、先に適切なサブディレクトリ（`pexels/`, `wikimedia/`, `original/`）に配置してからこのスキルを実行する

## 引数

ユーザーの指示から以下を読み取る（明示されない場合は確認する）:

| 引数 | 必須 | 説明 | 例 |
|:---|:---|:---|:---|
| source | 必須 | stock写真パス（`img/stock/` からの相対パスまたは絶対パス） | `pexels/aerial_greenhouse_complex_19851402.jpg` |
| target | 推奨 | デプロイ先ファイル名（`NN_NN_説明.jpg` 形式、小文字拡張子） | `01_22_大規模温室群.jpg` |
| qmd | 推奨 | 対象QMDファイル | `lectures/lec01.qmd` |
| slide | 任意 | 対象スライドの見出しまたは番号 | `スライド7左` |
| replace | 任意 | 差し替え元ファイル名（旧ファイル削除用。省略時は新規追加） | `01_18_施設栽培の実り.jpg` |
| class | 任意 | 画像CSSクラス（デフォルト: `.key-phrase-visual`） | `.main-visual` |

## 実行手順

### Phase 1: 事前検証

1. **ソースファイル存在確認**: `img/stock/{source}` が実在するか
2. **命名規則検証**: targetが `NN_NN_説明.{jpg,jpeg,png}` 形式か。拡張子は小文字に正規化する（`.JPG` → `.jpg`）。未指定時は講義番号＋次の連番＋説明で提案
3. **アスペクト比チェック**: `identify {source}` で寸法を取得
   - AR > 1.3（横長）: OK
   - 1.0 < AR <= 1.3: 注意（やや正方形寄り）
   - AR <= 1.0（縦長）: **警告** — coverクロップで上下が大きく切られる。`object-position` でのROI調整を推奨
4. **差し替え確認**（`replace` 指定時）: 旧ファイル `img/{replace}` と `img/{replace}.dvc` が存在するか
5. **意味的整合チェック**: ソース写真のメタデータJSON（`alt` / `description`）またはファイル名の説明部分と、デプロイ先スライドの見出し・キーフレーズ・スピーカーノートの内容を並べてユーザーに提示し、写真の内容がスライドの文脈に合っているか確認を求める。**写真の見た目と説明が一致しない場合（例: 「クリスマスケーキ」と書かれているが実際は誕生日ケーキ）を防ぐための重要なステップ。**

### Phase 2: メタデータJSON補完

ソース写真に対応するJSONメタデータが存在するか確認し、なければ作成する。
**JSON作成・更新した場合は Phase 6 で stock/ ディレクトリDVCの更新が必須になるためフラグを立てる。**

#### Pexels写真の場合（`pexels/` 配下）
必須14フィールド: `id`, `query`, `photographer`, `photographer_url`, `photographer_id`, `pexels_url`, `alt`, `avg_color`, `width`, `height`, `downloaded_size`, `download_date`, `credit_text`, `credit_html`

- **Pexels IDの抽出**: ファイル名の `_` 直前の連続数字列を photo_id とする。photographer_id は `pexels-{author}-{photographer_id}-{photo_id}_` パターンから抽出。パターンが一致しない場合はユーザーに確認
- `avg_color`: ImageMagickで算出。出力からHEX値（`#RRGGBB`）を抽出する:
  ```bash
  magick "img/stock/{source}" -resize 1x1 txt:- | grep -oP '#[0-9A-Fa-f]{6}'
  ```
- `width`, `height`: `identify` で取得
- `download_date`: 今日の日付
- その他: ユーザーに確認するか、Pexelsページから取得

#### Wikimedia写真の場合（`wikimedia/` 配下）
必須フィールド: `source`, `title`, `url`, `license`, `author`, `description`, `download_date`

#### ユーザー撮影の場合（`original/` 配下）
JSONメタデータは不要。catalog.mdのみで管理。

### Phase 3: ファイルデプロイ＋リサイズ

```bash
cp "img/stock/{source}" "img/{target}"

# 表示コンテキストに応じてリサイズ（アスペクト比維持、拡大禁止）
magick "img/{target}" -resize {MAX_WIDTH}x -quality 85 "img/{target}"

dvc add "img/{target}"
```

**リサイズ基準表**（QMDでの使用方法から最大幅を決定）:

| 表示コンテキスト | 最大幅 | 備考 |
|---|---|---|
| `data-background-image` | 1920px | 全画面背景 |
| `.main-visual`（100%列 or 単独） | 1920px | |
| `.main-visual`（60%列） | 1152px | 1920×0.6 |
| `.key-phrase-visual`（50%列） | 960px | 1920×0.5 |
| `.key-phrase-visual`（40%列） | 768px | 1920×0.4 |
| 縦長写真 | 高さ960px | `-resize x960` |

- JPEG品質: 85
- 元画像より小さい場合はリサイズしない（ImageMagick `-resize` は自動的に拡大しない）
- PNGは写真調ならJPEG変換を提案（コミットメッセージに元形式を記録）

**lec01教訓**: リサイズなしのデプロイで4500×2531pxの写真がそのまま埋め込まれ、PDF印刷が96MBに。リサイズにより19MB→Ghostscript圧縮で2.7MBに縮小。

**注意**:
- `dvc add` は自動的に `img/.gitignore` に新ファイルのエントリを追加する
- `dvc push` はPhase 6（最終確認後）にまとめて実行する

### Phase 4: 旧ファイル整理（`replace` 指定時のみ）

1. 旧DVCファイルを削除: `rm img/{replace}.dvc`
2. `img/.gitignore` から旧ファイルのエントリを削除（Editツールで。`dvc add` が旧エントリを自動削除しないため手動が必要）
3. 旧画像ファイル自体はgitignoreされているため残存する。気になる場合は `rm img/{replace}` で削除

### Phase 5: QMD・catalog更新

#### QMD更新
- `replace` 指定時: 旧パス `img/{replace}` を新パス `img/{target}` に置換
  - CSSクラスは `class` 引数に従う（デフォルト `.key-phrase-visual`）
  - 縦長写真の場合は `style="object-position: center NN%"` の付与を提案
- 新規追加時: 指定スライド位置に画像タグを挿入（ユーザーと配置を相談）

#### catalog.md更新
1. ソースの種別に応じた適切なセクションを特定（なければ新セクション作成を提案）
2. 新写真の行を追加:
   - 説明・ソース・ライセンス: メタデータJSONから自動抽出（JSONがない場合はユーザーに確認）
   - 使用状況: `lecNN スライドXX` 形式
3. `replace` 指定時: 旧写真の使用状況を `未使用（旧lecNN スライドXX）` に更新

### Phase 6: 検証・完了

1. `git diff --stat` で変更ファイル一覧を表示
2. 変更内容サマリーをレポート形式で表示（下記フォーマット）
3. Phase 2でJSON作成・更新した場合: `dvc add img/stock/{pexels,wikimedia}` でstock DVCを更新
4. `/photo-integrity --lec NN` を実行し、I1〜I6のクロスチェックでエラーがないことを確認
5. ユーザーに確認を求める
6. 確認後:
   - `dvc push` を実行（img/ の新ファイル + stock/ のDVC更新分を含む）
   - コミットメッセージ案を提示

### レポートフォーマット

```markdown
## 写真デプロイ完了レポート

| 項目 | 内容 |
|:---|:---|
| ソース | img/stock/pexels/xxx.jpg |
| デプロイ先 | img/01_22_大規模温室群.jpg |
| 差し替え元 | img/01_22_旧名.jpg（削除済み） |
| アスペクト比 | 1.78:1（横長、クロップ安全） |
| QMD更新 | lectures/lec01.qmd スライド17 |
| catalog更新 | 新行追加 + 旧行更新 |
| JSON | 既存 / 新規作成 |
| stock DVC | 更新済み / 変更なし |
| DVC | add済み、push待ち |

### 変更ファイル一覧
- img/01_22_*.jpg.dvc（新規）
- img/stock/pexels/xxx.json（新規、JSON補完した場合）
- lectures/lec01.qmd（パス更新）
- img/stock/catalog.md（行追加+更新）
- img/.gitignore（エントリ更新）
```

## 複数写真の一括デプロイ

複数の写真を同時にデプロイする場合、Phase 1〜5を各写真で順に実行し、Phase 6でまとめて検証・dvc push する。

## 同一写真の複数講義使用

同じstock写真を異なる講義で使う場合、講義ごとに別名でコピーする（`01_22_大規模温室群.jpg`, `02_05_大規模温室群.jpg` 等）。catalog.mdの使用状況列には全使用箇所を併記する（例: `lec01 スライド17, lec02 スライド5`）。

## エラー時の対応

- ソースファイルが存在しない → エラーメッセージを表示し中断。stock/に先に配置するよう案内
- target名が既存ファイルと重複 → ユーザーに確認（上書きか別名か）
- dvc add 失敗 → エラー原因を表示。DVC初期化済みか確認
- catalog.mdの該当セクションが見つからない → 新セクション作成を提案

## フルサイズ復元方法

Phase 3のリサイズにより元画像は上書きされるが、リサイズ前のバージョンはDVCキャッシュ/リモートに保持される。フルサイズが必要になった場合:

```bash
# リサイズ前のコミットを特定
git log --oneline img/{target}.dvc

# そのコミットの.dvcファイルを復元し、DVCで実ファイルを取得
git checkout <commit> -- img/{target}.dvc
dvc checkout img/{target}
```

復元後、再度リサイズするか、用途に応じてそのまま使用する。
