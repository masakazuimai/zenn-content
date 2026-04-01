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

ユーザーの操作を3つのモードで管理しています。

```typescript
type Interaction =
  | null                        // 何もしていない
  | { mode: "drawing"; ... }    // 新規エリア描画中
  | { mode: "moving"; ... }     // エリア移動中
  | { mode: "resizing"; ... }   // ハンドルでリサイズ中
```

`mouseDown` で何をクリックしたかによってモードを振り分けます。

```typescript
const handleMouseDown = (e) => {
  const pos = getMousePos(e, canvas, scale);

  // 1. 選択中エリアのハンドルか？→ リサイズ開始
  if (selectedId) {
    const handleIdx = findHandle(pos.x, pos.y, selectedArea, hitRadius);
    if (handleIdx !== null) {
      setInteraction({ mode: "resizing", ... });
      return;
    }
  }

  // 2. エリアの上か？→ 移動開始
  for (const area of areas) {
    if (isPointInArea(pos.x, pos.y, area)) {
      setInteraction({ mode: "moving", ... });
      return;
    }
  }

  // 3. 空白エリア→ 新規描画開始
  setInteraction({ mode: "drawing", ... });
};
```

この設計により、1つのCanvasで「描画」「選択」「移動」「リサイズ」をすべてカバーできています。

### リサイズ時の丸め誤差問題

画像サイズを変更するとエリアの座標もスケールしますが、毎回 `Math.round()` すると**拡大縮小を繰り返すうちに位置がズレていきます**。

```typescript
// NG: 丸め誤差が蓄積する
x: Math.round(area.x * scaleX)

// OK: 浮動小数点のまま保持
x: area.x * scaleX
```

内部では浮動小数点のまま保持し、**HTML出力時にだけ丸める**ことで解決しました。

```typescript
function generateCoords(area: Area): string {
  if (area.shape === "rect") {
    return `${Math.round(area.x)},${Math.round(area.y)},${Math.round(area.x + area.width)},${Math.round(area.y + area.height)}`;
  }
  return `${Math.round(area.cx)},${Math.round(area.cy)},${Math.round(area.radius)}`;
}
```

## CSS出力のコード生成

パーセント変換のロジックはシンプルです。

```typescript
function toPercent(value: number, base: number): string {
  return (value / base * 100).toFixed(2) + "%";
}

// 矩形エリアの場合
const left = toPercent(area.x, imageWidth);     // "8.33%"
const top = toPercent(area.y, imageHeight);      // "33.33%"
const width = toPercent(area.width, imageWidth); // "25.00%"
const height = toPercent(area.height, imageHeight);
```

円形エリアは `border-radius: 50%` で再現しています。

```typescript
// 円形: バウンディングボックスで配置
const left = toPercent(area.cx - area.radius, imageWidth);
const top = toPercent(area.cy - area.radius, imageHeight);
const width = toPercent(area.radius * 2, imageWidth);
const height = toPercent(area.radius * 2, imageHeight);
// + border-radius: 50%
```

## デプロイ

GitHub Actionsで `npm run build`（静的エクスポート）→ FTPでConoHaに自動デプロイしています。

```yaml
- name: Build
  run: npm run build

- name: Deploy via FTP
  uses: SamKirkland/FTP-Deploy-Action@v4.3.5
  with:
    server: ${{ secrets.FTP_SERVER }}
    username: ${{ secrets.FTP_USERNAME }}
    password: ${{ secrets.FTP_PASSWORD }}
    server-dir: ${{ secrets.FTP_SERVER_DIR }}
    local-dir: out/
```

Next.jsの設定で `basePath` を指定しておけば、サブディレクトリでも正しく動作します。

```typescript
const nextConfig: NextConfig = {
  output: "export",
  basePath: "/generator/clickable-area",
  trailingSlash: true,
};
```

## まとめ

| 項目 | 内容 |
|------|------|
| 課題 | `<map>` タグがレスポンシブ非対応 |
| 解決 | CSS方式（`position: absolute` + %指定）で出力 |
| 操作 | Canvas上でドラッグ描画・移動・リサイズ |
| 注意点 | 座標の丸めは出力時のみ（浮動小数点で保持） |
| デプロイ | GitHub Actions + FTP |

ツールはこちらから無料で使えます。

https://codequest.work/generator/clickable-area/

他にもCSS GridやFlexboxのジェネレーターなども公開しています。

https://codequest.work/tag/generator/

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
