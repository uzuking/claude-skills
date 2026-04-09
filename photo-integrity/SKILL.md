---
name: photo-integrity
description: catalog.md・QMD・DVC・実ファイル・メタデータJSONの整合性をクロスチェックする。「写真の整合性チェック」「catalogを確認」「photo integrity」と言われた時に使用。
---

# 写真整合性チェック

## 目的

`img/stock/catalog.md`（出典・使用状況管理）、`lectures/lec*.qmd`（スライド内画像参照）、DVC管理ファイル（`.dvc`, `.gitignore`）、メタデータJSON の間の不整合を検出・報告する。

## 引数

- `--lec NN`: 特定の講義番号に絞る（省略時は全講義）。I1/I2はフィルタ対象、I3〜I7は常に全体を検査
- `--fix`: 検出した問題の修正案を提示し、ユーザー確認後に実行する（デフォルトはレポートのみ）

## verify-qmd / visual-check との役割分担

| スキル | 対象 | チェック内容 |
|:---|:---|:---|
| verify-qmd | QMDソース | 構造・コンテンツ・品質（S7で画像参照の存在確認を含む） |
| visual-check | レンダリング済みHTML | 視覚的問題の検出（はみ出し、空白等） |
| **photo-integrity** | **写真管理全体** | **catalog↔QMD↔DVC↔JSON↔実ファイルのクロスチェック** |

I2はverify-qmd S7と重複するが、photo-integrityでは写真管理の文脈で統合レポートするため保持する。

## 前提

- 画像（スライド用）: `img/NN_NN_説明.{jpg,jpeg,png,tiff}` 形式で配置
- ストック写真: `img/stock/` 配下のサブディレクトリ（動的に列挙する）
  - `pexels/`, `wikimedia/`: 画像 + メタデータJSON
  - `original/`: ユーザー撮影画像（JSONメタデータは不要）
- DVC管理:
  - `img/*.dvc` — 個別ファイル単位（1ファイル = 1 .dvc + 1 .gitignoreエントリ）
  - `img/stock/*.dvc` — ディレクトリ単位（`pexels.dvc`, `wikimedia.dvc`, `original.dvc`）

## 実行手順

### Phase 1: データ収集

全てRead/Glob/Grepツールで実行し、Bashは使わない（ファイル一覧取得のlsを除く）。

1. `img/stock/catalog.md` を読み込み、全テーブル行から「ファイルパス」と「使用状況」を抽出
   - 「未使用（旧lecNN ...）」は「未使用」として扱う。「旧」プレフィックス付きは過去の使用履歴
2. `lectures/lec*.qmd` から全画像参照パスを抽出（`![](img/...)` パターン）
3. `img/*.dvc` ファイル一覧を取得（個別ファイルDVC）
4. `img/stock/*.dvc` ファイル一覧を取得（ディレクトリDVC）
5. `img/.gitignore` と `img/stock/.gitignore` のエントリ一覧を取得
6. `img/stock/` 直下のサブディレクトリを動的に列挙し（`ls img/stock/`）、各ディレクトリの実ファイル一覧を取得
7. `img/stock/` 配下の `*.json` メタデータファイルを一覧取得

### Phase 2: 8つのクロスチェック

| # | チェック | 方法 | 重要度 |
|:-:|:---|:---|:---|
| **I1** | catalog使用状況 vs QMD実参照 | catalogで「lecNN スライドXX」と記載されたstockファイルについて、そのスライド番号に対応するQMD内に**いずれかのimg/画像参照が存在するか**を確認する。stock→imgのファイル名マッピングは不要。逆方向（QMDが参照するimg/ファイルがcatalogに使用中と記載されているか）もチェック | 致命的 |
| **I2** | QMD参照画像 vs 実ファイル存在 | QMDが参照する `img/*.*` が実在するか | 致命的 |
| **I3a** | 個別DVC vs 実体ファイル | `img/*.dvc` の各エントリに対応する実体ファイル（`img/*.{jpg,jpeg,tiff}`）が存在するか | 警告 |
| **I3b** | ディレクトリDVC vs 実体 | `img/stock/*.dvc` の各エントリに対応するディレクトリが存在し、中にファイルがあるか（dvc pull済みか） | 警告 |
| **I4** | .gitignore vs DVCファイル | `img/.gitignore` のエントリと `img/*.dvc` が1:1対応するか。`img/stock/.gitignore` のエントリと `img/stock/*.dvc` が1:1対応するか。孤立エントリがないか | 警告 |
| **I5** | stock写真 vs catalog登録 | `img/stock/` 配下の画像ファイルが全て `catalog.md` に登録されているか | 情報 |
| **I6a** | Pexels JSON完全性 | `pexels/` 配下の各画像に対応するJSONが存在し、必須フィールドを持つか。必須14フィールド: `id`, `query`, `photographer`, `photographer_url`, `photographer_id`, `pexels_url`, `alt`, `avg_color`, `width`, `height`, `downloaded_size`, `download_date`, `credit_text`, `credit_html`（canonical: `reference/講義テンプレート設計.md`） | 警告 |
| **I6b** | Wikimedia JSON完全性 | `wikimedia/` 配下の各画像に対応するJSONが存在し、必須フィールドを持つか。必須7フィールド: `source`, `title`, `url`, `license`, `author`, `description`, `download_date`（canonical: `reference/講義テンプレート設計.md`） | 警告 |
| **I7** | JSONスキーマ統一性 | 同一ソース内のJSONのフィールドセットが統一されているか（フィールド数・名前の差異を検出） | 情報 |
| **I8** | 画像サイズ適正性 | `img/NN_NN_*.{jpg,png}` の幅が1920pxを超えていないか（`identify` で確認）。超過している場合、deploy-photoのリサイズ基準に従ってリサイズを推奨 | 警告 |

### Phase 3: レポート出力

```markdown
## 写真整合性レポート

### サマリー
| チェック | 問題数 | 対象数 | 結果 |
|:---|---:|---:|:---|
| I1 catalog↔QMD | | | |
| I2 QMD↔実ファイル | | | |
| I3a 個別DVC↔実体 | | | |
| I3b ディレクトリDVC↔実体 | | | |
| I4 gitignore↔DVC | | | |
| I5 stock↔catalog | | | |
| I6a Pexels JSON | | | |
| I6b Wikimedia JSON | | | |
| I7 JSONスキーマ | | | |
| I8 画像サイズ | | | |

### 検出された問題
| # | チェック | 重要度 | 内容 | 修正案 |
|:-:|:---|:---|:---|:---|
```

### Phase 4: 修正（`--fix` 指定時のみ）

致命的・警告の問題について具体的な修正コマンドまたは編集内容を提示し、ユーザー確認後に実行する。

## チェック対象外

- `img/icons/` 配下のSVGアイコン（別管理体系）
- `img/` 直下で `.dvc` ファイルが存在しないファイル（Rスクリプト生成のグラフ等、DVC管理対象外）
- `archive_2024/` 配下の旧年度ファイル
- `img/stock/original/` のファイル命名はNN_NN形式に従わない（ユーザー撮影のオリジナルファイル名を保持）

## 既知の制約

- I1は catalog.md の「使用状況」列の記述形式（`lecNN スライドXX左/右`）に依存する。自由記述には対応しない
- I5で `original/` 配下のファイルにはJSONメタデータは不要（ユーザー撮影のため）。I6a/I6bのチェック対象外
- ファイル拡張子は大文字小文字の両方を対象とする（`.jpg`, `.JPG` 等）
