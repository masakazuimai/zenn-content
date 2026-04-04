---
title: "レスポンシブ対応のクリッカブルエリアジェネレーターをNext.jsで作った話"
emoji: "🖱️"
type: "tech"
topics: ["nextjs", "typescript", "react", "canvas", "css"]
published: true
---

## 作ったもの

画像にクリッカブルエリア（リンク領域）を視覚的に設定して、**レスポンシブ対応のHTMLコードを自動生成**する無料ツールを作りました。

https://codequest.work/generator/clickable-area/

## なぜ作ったか

HTMLの `<map>` + `<area>` タグでイメージマップを作る場面は今でもあります。バナー画像の一部だけリンクにしたい、写真の特定箇所にリンクを貼りたい、といったケースです。

ただし、従来の `<map>` タグには**致命的な問題**があります。

```html
<!-- 従来のイメージマップ -->
<img src="banner.jpg" width="1200" height="600" usemap="#map" />
<map name="map">
  <area shape="rect" coords="100,200,400,350" href="/link" />
</map>
```

`coords` がピクセル固定なので、**画像がレスポンシブで縮小されるとクリック領域がズレます**。

これを解決するために、CSS方式でパーセント指定のコードを出力するジェネレーターを作りました。

## レスポンシブ対応の仕組み

出力されるコードはこのようになります。

```html
<div style="position: relative; display: inline-block;">
  <img src="banner.jpg" alt="" style="width: 100%; height: auto; display: block;" />
  <a href="/link" aria-label="ボタン"
     style="position: absolute; left: 8.33%; top: 33.33%; width: 25.00%; height: 25.00%;">
  </a>
</div>
```

ポイントは3つです。

1. **`position: absolute` + パーセント指定** — 画像サイズに追従します
2. **JSが不要** — CSSだけで完結するため、どんな環境でも動きます
3. **`aria-label`** — アクセシビリティにも対応しています

## 技術スタック

- Next.js 15（App Router / 静的エクスポート）
- TypeScript
- Tailwind CSS v4
- Canvas API

## Canvas上での描画・操作

エリアの描画・選択・移動・リサイズはすべてCanvas APIで実装しています。

### インタラクションの設計

ユーザーの操作を3つのモードで管理する考え方です。

```typescript
// 操作モードの型定義
type Interaction =
  | null                                    // 待機中
  | { mode: "drawing"; startX: number; startY: number }   // 新規描画
  | { mode: "moving"; areaId: string; offsetX: number }    // 移動
  | { mode: "resizing"; areaId: string; handle: number }   // リサイズ
```

マウス操作の判定は、クリック位置によって振り分けます。

```typescript
function handleMouseDown(pos: { x: number; y: number }) {
  // 1. ハンドル上なら → リサイズモードへ
  const handle = findHandleAt(pos);
  if (handle !== null) {
    startResize(handle);
    return;
  }

  // 2. エリア上なら → 移動モードへ
  const area = findAreaAt(pos);
  if (area) {
    startMove(area, pos);
    return;
  }

  // 3. 空白なら → 新規描画モードへ
  startDraw(pos);
}
```

この設計により、1つのCanvasで「描画」「選択」「移動」「リサイズ」をすべてカバーできています。

### リサイズ時の丸め誤差問題

画像サイズを変更するとエリアの座標もスケールしますが、毎回 `Math.round()` すると**拡大縮小を繰り返すうちに位置がズレていきます**。

```typescript
// NG: 丸め誤差が蓄積する
const newX = Math.round(x * scale);

// OK: 浮動小数点のまま保持する
const newX = x * scale;
```

内部では浮動小数点のまま保持し、**HTML出力時にだけ丸める**ことで解決しました。

```typescript
// 座標出力時のみ整数に丸める
function formatCoord(value: number): string {
  return String(Math.round(value));
}
```

## CSS出力のコード生成

パーセント変換のロジックはシンプルです。

```typescript
function toPercent(value: number, base: number): string {
  return (value / base * 100).toFixed(2) + "%";
}

// 例: 画像幅1200pxに対して、x=100の位置
toPercent(100, 1200); // → "8.33%"
```

矩形エリアの場合、4つの値をパーセントに変換します。

```typescript
const style = {
  left: toPercent(x, imageWidth),
  top: toPercent(y, imageHeight),
  width: toPercent(w, imageWidth),
  height: toPercent(h, imageHeight),
};
```

円形エリアはバウンディングボックスで配置し、`border-radius: 50%` で円にします。

```typescript
// 円形: 中心座標と半径から矩形に変換
const style = {
  left: toPercent(cx - radius, imageWidth),
  top: toPercent(cy - radius, imageHeight),
  width: toPercent(radius * 2, imageWidth),
  height: toPercent(radius * 2, imageHeight),
  borderRadius: "50%",
};
```

## まとめ

| 項目 | 内容 |
|------|------|
| 課題 | `<map>` タグがレスポンシブ非対応 |
| 解決 | CSS方式（`position: absolute` + %指定）で出力 |
| 操作 | Canvas上でドラッグ描画・移動・リサイズ |
| 注意点 | 座標の丸めは出力時のみ（浮動小数点で保持） |

ツールはこちらから無料で使えます。

https://codequest.work/generator/clickable-area/

他にもCSS GridやFlexboxのジェネレーターなども公開しています。

https://codequest.work/tag/generator/

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=clickable-area-generator-nextjs) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=clickable-area-generator-nextjs) — ディレクターも使っているSEO診断ツールを公開中。
