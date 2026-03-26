# ビジュアル型カタログ

visual-designer スキルが提案時に参照する型一覧。

## 型 x 適用場面 x 実装手段

| 型 | 別名 | 適用場面 | 実装手段 | revealjsレイアウト | 制作コスト |
|:---|:---|:---|:---|:---|:---|
| 因果フロー図 | フロー図 | 原因→結果の連鎖を説明 | SVG (`/svg-precision`) | `.main-visual` or `.columns` 内 | 中 |
| 比較図（サイドバイサイド） | — | A vs B、before/after の対比 | 数値データ→R / 概念的比較→SVG | `.columns width="50%" x 2` | 中 |
| プロセス図（フローチャート） | — | 手順・段階・ステップ | SVG (`/svg-precision`) | `.main-visual` | 中〜高 |
| 構造図（断面・配置） | — | 部品・部位・階層の説明 | SVG (`/svg-precision`) | `.main-visual` | 高 |
| 棒グラフ | バーチャート | 量の比較（国別、年次別等） | R (`ggplot2`, `geom_col + coord_flip`) | `.main-visual` or `.columns` | 中 |
| 折れ線グラフ | ラインチャート | 時間変化・推移・トレンド | R (`ggplot2`, `geom_line`) | `.main-visual` | 中 |
| 円グラフ / ツリーマップ | パイチャート | 構成比・割合 | R (`ggplot2` or `treemapify`) | `.main-visual` | 低〜中 |
| 散布図 | — | 相関関係 | R (`ggplot2`, `geom_point`) | `.main-visual` | 低〜中 |
| ジャーニーマップ / タイムライン | ロードマップ | 時系列・ロードマップ・講義構成 | SVG (`/svg-precision`) | `.main-visual` | 中〜高 |
| 教育イラスト | — | 概念図・情景・模式図 | nanobanana pro (lecture-framework のイラスト生成ツール) | `.main-visual` | 低 |
| 写真 | — | 実物・装置・現場・植物 | CC画像取得 or 自撮り | `.main-visual` or `.columns` 内 | 低 |

> **型名の別名**: INFOGRAPHICコメントでは「バーチャート」「パイチャート」等のカタカナ表記も使用可。

## revealjsレイアウトパターン

**`.main-visual` と `.r-stretch` の使い分け:**
- `.main-visual`: 標準。lec01.qmd では全画像がこのクラス。CSSテーマで統一スタイリングされる
- `.r-stretch`: revealjs組み込み。スライドの残り高さいっぱいに引き伸ばす。テキストが少ない全面表示向け

原則として `.main-visual` を使用する。

### 画像をスライド全面に表示

```qmd
## スライドタイトル {.detail}

![](img/01_07_植物工場内部.png){.main-visual}
```

### 左右2カラム（均等分割: 画像比較）

```qmd
## スライドタイトル {.detail}

::: {.columns}
::: {.column width="50%"}
![](img/01_01_画像A.png){.main-visual}
:::
::: {.column width="50%"}
![](img/01_02_画像B.png){.main-visual}
:::
:::
```

### 左右2カラム（非対称: テーブル＋注釈）

```qmd
## スライドタイトル {.table-slide}

::: {.columns}
::: {.column width="55%"}
| 項目 | 値 |
|:---|---:|
| ... | ... |
:::
::: {.column width="45%"}
::: {.key-message}
ポイントをここに
:::
:::
:::
```

### 画像＋キーメッセージ

```qmd
## スライドタイトル {.detail}

::: {.columns}
::: {.column width="60%"}
![](img/01_05_グラフ.png){.main-visual}
:::
::: {.column width="40%"}
::: {.key-message}
この数字が意味すること
:::
:::
:::
```

## INFOGRAPHICコメントのテンプレート

lec01.qmd から抽出した実例パターン:

### HTMLコメント（`##` 見出し直後に配置）

```qmd
## スライドタイトル {.detail}

<!-- INFOGRAPHIC: 型 — 内容の簡潔な説明 -->
```

実例:
- `<!-- INFOGRAPHIC: 因果フロー図 — ☀️太陽を蓄える → 🌧️雨を遮る → 🍅高品質な野菜 -->`
- `<!-- INFOGRAPHIC: バーチャート — 中国の棒が圧倒的に長い。日本は8番目 -->`
- `<!-- INFOGRAPHIC: サイドバイサイド比較 — トマト積み上げ or 人型スケール、日本1:オランダ6 -->`
- `<!-- INFOGRAPHIC: 円グラフ — 農業総産出額の部門別構成。園芸（野菜＋果実）≒4割 -->`
- `<!-- INFOGRAPHIC: ジャーニーマップ — 光(2-5)→温熱(6-9)→水(10-11)→統合(12-14) -->`

> **重要**: `<!-- INFOGRAPHIC: -->` コメントと `📊 インフォグラフィック依頼` は必ずセットで記述すること。片方のみ存在する場合は visual-designer が整合性警告を出す。

### スピーカーノート内の詳細仕様

```qmd
::: {.notes}
📊 **インフォグラフィック依頼**:

- 種類: [因果フロー図 / 棒グラフ / サイドバイサイド比較 / 等]
- 内容: [図に含める要素の説明]
- 強調: [最も伝えたいポイント]
- レイアウト: [.r-stretch / .columns 比率 / 等]
- 制作方法: [SVG(/svg-precision) / R(ggplot2) / nanobanana / CC画像]
- データソース: [reference/ファイル名 or 出典]

> 現在のテキスト/テーブルはフォールバック。図完成後に置換。
:::
```

## ファイル命名規則

- 画像: `img/NN_NN_説明.拡張子`（例: `01_03_ハウスの種類・形.png`）
  - NN = 講義番号（01〜14）
  - NN = 連番
  - 説明 = 日本語で内容を簡潔に
- Rスクリプト: `scripts/lecNN_説明.R`（例: `scripts/lec01_世界施設面積.R`）
