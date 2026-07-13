---
title: "スマホの横スクロール、あのConsoleスニペットは誤検知する — Playwrightで潰しきる"
emoji: "📐"
type: "tech"
topics: ["playwright", "css", "frontend", "githubactions", "テスト"]
published: true
---

## 結論から

- 「はみ出し要素を探す」でよく共有される Console スニペットは、**`overflow: hidden` の内側と `position: fixed` の要素まで拾う**
- 手元の検証では、真犯人1件に対して**誤検知が4件**出た
- 除外条件を足せば真犯人だけが残る。そこまで来たら Playwright に載せて CI で落とせる

`overflow-x: hidden` を body に付けて蓋をするのは、原因を隠しているだけです。原因要素を特定して、二度と混入させないところまで持っていきます。

## よく共有されるスニペットの問題

横スクロールが出たとき、DevTools の Console にこれを貼る、という手順はよく見かけます。

```js
document.querySelectorAll('*').forEach((el) => {
  if (el.getBoundingClientRect().right > document.documentElement.clientWidth) {
    console.log(el);
  }
});
```

要素の右端がビューポート幅を超えていたら出力する、という素直なロジックです。**手で1ページ見るぶんには十分**なのですが、これをそのまま自動テストに持っていくと破綻します。誤検知が多すぎて、CI が常に赤くなるからです。

## 何を誤検知するのか

検証用に、3種類のはみ出しを含む HTML を用意しました。

```html
<style>
  /* ① 本物：ページに横スクロールを発生させる */
  .real-overflow { width: 120%; }

  /* ② 偽陽性A：親が overflow:hidden。実際にはスクロールしない */
  .carousel { overflow: hidden; }
  .carousel__track { display: flex; width: 3000px; }
  .carousel__item { width: 300px; }

  /* ③ 偽陽性B：position:fixed の装飾。スクロール幅に影響しない */
  .decor { position: fixed; left: 100vw; width: 200px; }
</style>
```

ビューポート幅 375px で、上のスニペットを走らせた結果です。

```
ページの横スクロール: { scrollWidth: 428, clientWidth: 375, overflow: true }

[素朴版] 検出: [ 'real-overflow', 'carousel__track', 'carousel__item', 'carousel__item', 'decor' ]
```

**5件ヒットしましたが、実際に横スクロールを起こしているのは `real-overflow` だけです。** 残り4件は、レイアウト上まったく問題がありません。

- `carousel__track` と `carousel__item`：祖先の `.carousel` が `overflow: hidden` なので、はみ出した部分はクリップされる。これはカルーセルの正常な実装
- `decor`：`position: fixed` の要素はページのスクロール可能領域を広げない

つまりこのスニペットは、**「ビューポートより右にある」ことしか見ていない**わけです。実際に横スクロールを生むかどうかは、祖先のクリッピングと position を見ないと判定できません。

## 除外条件を足す

必要な除外は3つです。

1. 祖先に `overflow-x` が `hidden` / `clip` / `auto` / `scroll` の要素があれば、そこで切られるので除外
2. `position: fixed` は除外
3. 幅・高さが 0 の要素（非表示・計測用の空要素）は除外

```js
const findOverflowingElements = () => {
  const limit = document.documentElement.clientWidth;

  const isClipped = (el) => {
    for (let p = el.parentElement; p && p !== document.documentElement; p = p.parentElement) {
      const overflowX = getComputedStyle(p).overflowX;
      if (['hidden', 'clip', 'auto', 'scroll'].includes(overflowX)) return true;
    }
    return false;
  };

  return [...document.querySelectorAll('*')].filter((el) => {
    const rect = el.getBoundingClientRect();
    if (rect.width === 0 || rect.height === 0) return false;
    if (rect.right <= limit + 1) return false;          // 1px は丸め誤差の許容
    if (getComputedStyle(el).position === 'fixed') return false;
    if (isClipped(el)) return false;
    return true;
  });
};
```

同じ HTML に対して実行すると、こうなります。

```
[精密版] 検出: [ 'real-overflow' ]
```

真犯人だけが残りました。ここまで来て初めて、自動化する価値が出ます。

:::message
`rect.right <= limit + 1` の `+1` は、`transform: scale()` や小数を含む幅指定でサブピクセルの丸めが起きたときの誤検知を避けるためのマージンです。ここを厳密に `<=` にすると、実害のない 0.5px のはみ出しで CI が落ちます。
:::

## Playwright に載せる

判定ロジックが固まったら、あとは**複数のビューポート × 複数の URL**で回すだけです。横スクロールは特定の幅でだけ出るので、幅を振ることに意味があります。

```ts
// tests/overflow.spec.ts
import { test, expect, type Page } from '@playwright/test';

const VIEWPORTS = [
  { name: 'sp-min', width: 320, height: 800 },
  { name: 'sp', width: 375, height: 800 },
  { name: 'tablet', width: 768, height: 1024 },
];

const PATHS = ['/', '/about/', '/blog/'];

const findOverflowing = (page: Page) =>
  page.evaluate(() => {
    const limit = document.documentElement.clientWidth;

    const isClipped = (el: Element) => {
      for (let p = el.parentElement; p && p !== document.documentElement; p = p.parentElement) {
        const overflowX = getComputedStyle(p).overflowX;
        if (['hidden', 'clip', 'auto', 'scroll'].includes(overflowX)) return true;
      }
      return false;
    };

    return [...document.querySelectorAll('*')]
      .filter((el) => {
        const rect = el.getBoundingClientRect();
        if (rect.width === 0 || rect.height === 0) return false;
        if (rect.right <= limit + 1) return false;
        if (getComputedStyle(el).position === 'fixed') return false;
        if (isClipped(el)) return false;
        return true;
      })
      .map((el) => ({
        selector: el.tagName.toLowerCase() + (el.className ? `.${String(el.className).split(' ').join('.')}` : ''),
        right: Math.round(el.getBoundingClientRect().right),
      }));
  });

for (const vp of VIEWPORTS) {
  for (const path of PATHS) {
    test(`${vp.name} / ${path} で横スクロールが出ない`, async ({ page }) => {
      await page.setViewportSize({ width: vp.width, height: vp.height });
      await page.goto(path);

      const offenders = await findOverflowing(page);

      expect(offenders, `はみ出し要素: ${JSON.stringify(offenders, null, 2)}`).toEqual([]);
    });
  }
}
```

`expect` の第2引数にメッセージを渡しているので、落ちたときのログに**どの要素が何px はみ出したか**がそのまま出ます。「どこかで横スクロールが出ています」では直せませんが、「`div.hero__inner` が 428px（ビューポート 375px）」なら直せます。

## GitHub Actions で落とす

```yaml
name: layout
on: [pull_request]

jobs:
  overflow:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npx playwright test tests/overflow.spec.ts
```

`playwright.config.ts` の `webServer` にビルド済みサーバーの起動コマンドを書いておけば、`baseURL` 経由で `PATHS` の相対パスがそのまま解決されます。

## 残る限界

正直に書いておくと、この方法でも取りこぼす／過検知するケースはあります。

| ケース | 挙動 |
|---|---|
| 疑似要素（`::before` / `::after`）のはみ出し | `querySelectorAll` では取れないため検出できない |
| JS で後から挿入される要素 | `goto` 直後だと間に合わない。`waitForLoadState('networkidle')` か明示的な待機が必要 |
| `overflow-x: auto` の内部で意図せずはみ出している要素 | クリップ扱いで除外されるため見逃す（そこは横スクロールしてよい領域なので、多くの場合は妥当） |
| フォント読み込み前後でのレイアウトシフト | Web フォント適用後に幅が変わる場合がある。`document.fonts.ready` を待つと安定する |

疑似要素まで追うなら `getComputedStyle(el, '::after')` を個別に見る必要がありますが、実務では**まず実要素のはみ出しを潰すだけで大半が片付きます**。網羅性を上げるより、CI に載せて再発をゼロにするほうが効きます。

## まとめ

- Console スニペットは「見つける」道具としては優秀だが、**そのまま自動化すると誤検知で使い物にならない**
- 祖先の `overflow-x` と `position: fixed` を除外条件に足すと、真犯人だけが残る
- 判定が安定したら Playwright で複数ビューポートに展開し、CI で落とす
- `overflow-x: hidden` で蓋をするのは、原因を隠しているだけ

DevTools で手作業していた検証をコードに落とすと、その検証は二度と忘れられません。DevTools 側の検証手順（レスポンシブ・CSSの詳細度・キャッシュ・LCP）は [Chrome DevTools で仮説と検証を回す手順](https://codequest.work/chrome-devtools-verification-guide/) にまとめています。
