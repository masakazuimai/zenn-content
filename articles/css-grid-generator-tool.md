---
title: "CSS Grid を視覚的に組めるツール作った話（コード自動生成付き）"
emoji: "🔲"
type: "tech"
topics: ["css", "cssgrid", "react", "nextjs", "typescript"]
published: true
---

CSS Grid でレイアウトを組んでいるとき、こんな経験はないでしょうか。

「頭の中では完璧なレイアウトが浮かんでいるのに、`grid-column: 2 / span 3` という数字の組み合わせを手で書き始めた瞬間に自分を見失う」

ビジュアルで考えてコードを書く、この変換コストを下げたくてツールを作りました。

**CSS Grid Generator**
https://codequest.work/generator/grid/?utm_source=zenn&utm_medium=article&utm_campaign=css-grid-generator-tool

## CSS Grid の課題：ビジュアルとコードの乖離

CSS Grid 自体は強力です。2次元のレイアウトを柔軟に制御でき、複雑なページ構造も簡潔なコードで表現できます。

しかし、複雑なレイアウトになるほど「どのセルがどこに対応するか」を頭の中で追いかけるコストが増します。特にアイテムが複数のセルにまたがる（`grid-column: 2 / 4`、`grid-row: 1 / 3` のような）パターンは、書いては確認し、確認しては書き直す、という往復作業になりがちです。

```css
/* これを暗算で組み立てるのは地味につらい */
.header  { grid-column: 1 / 4; grid-row: 1; }
.sidebar { grid-column: 1;     grid-row: 2 / 4; }
.main    { grid-column: 2 / 4; grid-row: 2; }
.footer  { grid-column: 1 / 4; grid-row: 4; }
```

ブラウザの DevTools で確認しながら調整するのも手ですが、試行錯誤のサイクルが長くなります。

## 作ったもの：CSS Grid Generator

グリッドを視覚的に操作して、HTML と CSS を自動生成するツールです。

**主な操作フロー**

1. 列数・行数・ギャップを数値で設定する
2. グリッドセル上でマウスをドラッグして範囲を選択する
3. 選択した範囲がそのままグリッドアイテムとして配置される
4. 画面下部に HTML と CSS がリアルタイムで表示される
5. Copy ボタン一発でクリップボードへコピーする

ダブルクリックで配置済みのアイテムを削除できるので、「置いては調整する」の繰り返しもストレスが少ないです。

## 機能の詳細

### グリッドの基本設定

| 設定項目 | 内容 |
|---|---|
| Columns / Rows | 列数・行数（1以上の整数） |
| Gap | セル間の余白（px固定） |
| Width / Height | コンテナのサイズ（px / % / vw / vh / em / rem） |

単位をドロップダウンで切り替えられるのがポイントです。レスポンシブ対応を考えてコンテナ幅を `100%` にしたい場合も、そのまま指定できます。

### アイテムのサイズ指定

アイテムごとの幅・高さも `auto` / `100%` / カスタム値から選べます。グリッドセルいっぱいに広げる場合は `100%`、内部コンテンツに合わせる場合は `auto` と使い分けできます。

### 生成されるコード

HTML 側はこのような形で出力されます。

```html
<div class="grid-container">
  <div class="div-1">1</div>
  <div class="div-2">2</div>
  <div class="div-3">3</div>
</div>
```

CSS 側は `.grid-container` のベース定義に加え、各アイテムの配置が `grid-column` / `grid-row` で出力されます。レスポンシブ対応の `@media` クエリも自動的に付与されます。

```css
.grid-container {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  grid-template-rows: repeat(3, 1fr);
  gap: 8px;
  width: 100%;
  height: 500px;
}

.div-1 {
  grid-column: 1 / span 4;
  grid-row: 1 / span 1;
  width: 100%;
  height: 100%;
}

@media (max-width: 768px) {
  .grid-container {
    grid-template-columns: 1fr;
    grid-template-rows: auto;
  }
  .grid-container > div {
    grid-column: auto !important;
    grid-row: auto !important;
  }
}
```

## 実装で工夫した点

### ドラッグ選択の状態管理

ドラッグ操作はシンプルに見えて、状態の持ち方が重要です。

マウスダウン時に開始セルを記録し、マウスムーブ中に通過したセルを `Set` で重複排除しながら蓄積します。マウスアップ時に蓄積されたセルIDから `startRow` / `endRow` / `startCol` / `endCol` を計算して、アイテムとして確定します。

```typescript
const handleMouseUp = () => {
  if (selectedCells.length > 0) {
    const selectedRows = selectedCells.map((id) =>
      parseInt(id.split("-")[0], 10)
    )
    const selectedCols = selectedCells.map((id) =>
      parseInt(id.split("-")[1], 10)
    )

    const startRow = Math.min(...selectedRows)
    const endRow   = Math.max(...selectedRows)
    const startCol = Math.min(...selectedCols)
    const endCol   = Math.max(...selectedCols)

    const hasOverlap = gridItems.some((item) => {
      const rowOverlap = startRow <= item.endRow && endRow >= item.startRow
      const colOverlap = startCol <= item.endCol && endCol >= item.startCol
      return rowOverlap && colOverlap
    })

    if (!hasOverlap) {
      setGridItems((prev) => [
        ...prev,
        { id: `div-${prev.length + 1}`, startRow, endRow, startCol, endCol, content: `${prev.length + 1}` },
      ])
    }
  }
  setSelectedCells([])
  setIsSelecting(false)
}
```

ポイントは「矩形選択」であること。ドラッグで斜めに動いた場合も、通過したセルの最小・最大座標から矩形範囲として確定します。CSS Grid の `grid-column` / `grid-row` は矩形範囲しか表現できないので、この設計が自然です。

### 重複チェックのロジック

既存アイテムとの重複は、行と列それぞれの区間オーバーラップで判定しています。2つの区間 `[a1, a2]` と `[b1, b2]` が重なる条件は `a1 <= b2 && a2 >= b1` です。行・列の両方が重なる場合にのみ重複と判定します。

### コード生成はテンプレートリテラル

コード生成のロジックはシンプルで、状態（`columns`, `rows`, `gap`, `gridItems` など）をテンプレートリテラルに埋め込むだけです。状態が変わるたびに React が再レンダリングし、生成コードも自動で更新されます。別途「生成ボタン」は不要で、常に最新の状態が反映されます。

```typescript
const cssOutput =
  `.grid-container {
  display: grid;
  grid-template-columns: repeat(${columns}, 1fr);
  grid-template-rows: repeat(${rows}, 1fr);
  gap: ${gap}px;
  width: ${containerWidth}${containerWidthUnit};
  height: ${containerHeight}${containerHeightUnit};
}
` +
  gridItems
    .map(
      (item) => `
.${item.id} {
  grid-column: ${item.startCol} / span ${item.endCol - item.startCol + 1};
  grid-row: ${item.startRow} / span ${item.endRow - item.startRow + 1};
}
`
    )
    .join("")
```

## まとめ

CSS Grid は覚えることが多く、慣れるまで「プロパティは分かるけどレイアウトが思った通りにならない」という状態が続きやすいです。このツールを使うと、ビジュアルで確認しながらコードを得られるので、学習コストを下げる効果もあると思っています。

実際に触ってみてください。

https://codequest.work/generator/grid/?utm_source=zenn&utm_medium=article&utm_campaign=css-grid-generator-tool

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=css-grid-generator-tool) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=css-grid-generator-tool) — ディレクターも使っているSEO診断ツールを公開中。
