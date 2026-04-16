---
name: visual-check
description: Playwrightを使ったスライドの自動視覚チェック。render後にHTMLを開き、全スライド（または指定スライド）をスクリーンショット・DOM測定して、はみ出し・空白過多・グラフサイズ不足を検出する。「見た目を確認」「スクリーンショット」「はみ出しチェック」「visual check」と言われた時に使用。
---

# スライド視覚チェック

## 目的

quarto render 後のHTMLをPlaywright MCPで開き、スライドを自動巡回してスクリーンショット取得・DOM測定を行い、視覚的な問題を検出する。render→目視→修正サイクルを短縮する。

## 前提

- Playwright MCPが設定済みであること（`mcp__playwright__*` ツールが利用可能）
- 対象HTMLが `quarto render` 済みであること
- ビューポートは 1920x1080 で設定済み（MCP設定の `--viewport-size`）

## verify-qmd / visual-designer との役割分担

| スキル | 対象 | タイミング | チェック内容 |
|:---|:---|:---|:---|
| verify-qmd | QMDソース | render前後 | 構造・コンテンツ・品質（テキストベース） |
| visual-designer | QMDソース | verify-qmd後 | ビジュアル提案（何を追加すべきか） |
| **visual-check** | **レンダリング済みHTML** | **render後** | **視覚的問題の検出（実際の見た目）** |

推奨実行順序: `verify-qmd` → `quarto render` → `visual-check`

## 引数

- ファイルパス: HTMLファイルの絶対パス（必須。なければ対応するQMDから `lectures/lecNN.html` を推定）
- `--slide N`: 特定スライド番号のみチェック（複数指定可: `--slide 6,17,18`）
- `--all`: 全スライド（デフォルト）
- `--screenshot-only`: 問題検出なし、スクリーンショット取得のみ

## 処理フロー

### Phase 1: HTMLを開く

1. `mcp__playwright__browser_navigate` で `file://` パスを開く（キャッシュ回避: `?v=timestamp` 付与）
2. ページ読み込み完了を待つ

### Phase 2: スライド巡回

`mcp__playwright__browser_evaluate` でスライド総数を取得し、各スライドに移動してチェック。

各スライドで実行:

1. **スライド移動**: `Reveal.slide(h, 0)` で水平インデックス指定
2. **CSSクラス取得**: `Reveal.getCurrentSlide().className` からスライドクラスを取得
3. **スクリーンショット**: `mcp__playwright__browser_take_screenshot` で取得
4. **DOM測定**: 以下のJavaScriptを `mcp__playwright__browser_evaluate` で実行

```javascript
(() => {
  const slide = Reveal.getCurrentSlide();
  const rect = slide.getBoundingClientRect();

  // 下辺の点線位置（.slides の ::before で配置。bottom値を動的取得）
  const slidesEl = document.querySelector('.reveal .slides');
  const slidesRect = slidesEl.getBoundingClientRect();
  const bottomPx = parseFloat(getComputedStyle(slidesEl, '::before').bottom) || 18;
  const dotLineY = slidesRect.top + slidesRect.height - bottomPx;

  // コンテンツの実際の最下端（notes, speaker-trigger を除外）
  // 注意: 直接子要素のbottomで計測。ネスト内のoverflow:visibleなはみ出しは検出漏れの可能性あり
  const children = slide.querySelectorAll(':scope > *:not(.notes):not(.speaker-trigger)');
  let maxBottom = 0;
  children.forEach(el => {
    const r = el.getBoundingClientRect();
    if (r.bottom > maxBottom) maxBottom = r.bottom;
  });

  // 画像のレンダリングサイズ
  const images = slide.querySelectorAll('img.main-visual, img.col-icon, img.key-phrase-visual, img.key-phrase-illustration');
  const imgData = Array.from(images).map(img => ({
    src: img.src.split('/').pop(),
    cls: img.className,
    rendered: { w: img.clientWidth, h: img.clientHeight },
    natural: { w: img.naturalWidth, h: img.naturalHeight },
    scaleFactor: img.naturalWidth > 0 ? img.clientWidth / img.naturalWidth : 0
  }));

  // key-phraseスライドの画像充填率
  const isKeyPhrase = slide.classList.contains('key-phrase');
  const imgFillRatio = (isKeyPhrase && imgData.length > 0)
    ? Math.max(...imgData.map(i => i.rendered.h)) / usableHeight
    : null;

  // 空白率
  const contentHeight = maxBottom - rect.top;
  const usableHeight = dotLineY - rect.top;
  const blankRatio = usableHeight > 0 ? Math.max(0, (usableHeight - contentHeight) / usableHeight) : 0;

  return {
    overflow: maxBottom > dotLineY,
    overflowPx: Math.max(0, Math.round(maxBottom - dotLineY)),
    blankRatio: Math.round(blankRatio * 100),
    contentHeight: Math.round(contentHeight),
    usableHeight: Math.round(usableHeight),
    images: imgData,
    isKeyPhrase,
    imgFillRatio: imgFillRatio !== null ? Math.round(imgFillRatio * 100) : null
  };
})()
```

### Phase 3: 問題検出

| # | チェック項目 | 閾値 | 重要度 |
|:-:|:---|:---|:---|
| VC1 | コンテンツはみ出し | overflowPx > 0 | 致命的 |
| VC2 | 空白過多 | blankRatio > 40%（汎用）、> 25%（key-phraseスライド） | 警告 |
| VC3 | 画像が極端に小さい | rendered.h < 50px **または** rendered.w < 50px（CSS伝播バグの可能性） | 致命的 |
| VC4 | 画像スケール過大 | scaleFactor > 1.5（拡大表示で画質劣化） | 情報 |
| VC5 | 画像スケール過小 | scaleFactor < 0.3（70%以上縮小、R出力が過大） | 情報 |
| VC6 | key-phrase画像充填不足 | isKeyPhrase かつ imgFillRatio < 35%（画像がスライドの35%未満） | 警告 |

### Phase 4: 出力

```markdown
## 視覚チェックレポート: [ファイル名]

### サマリー
| 指標 | 値 |
|:---|---:|
| チェックスライド数 | X |
| 致命的 | X |
| 警告 | X |
| 情報 | X |

### 問題一覧
| スライド# | CSSクラス | チェック | 詳細 | 修正案 |
|:-:|:---|:---|:---|:---|

### 画像サイズ詳細
| スライド# | ファイル名 | 元サイズ | 表示サイズ | 倍率 | 判定 |
|:-:|:---|:---|:---|:---|:---|
```

## 注意事項

- reveal.jsのスライド番号は0始まり。ユーザーの「スライド17」は `Reveal.slide(16, 0)` に対応
- `--slide` 指定時もPhase 1は必ず実行
- このスキルはQMDを編集しない。問題を報告するのみ。修正はユーザーまたは他のスキルが行う
- `.speaker-trigger` と `.notes` はコンテンツ計測から除外する（表示エリア外に配置されるため）
