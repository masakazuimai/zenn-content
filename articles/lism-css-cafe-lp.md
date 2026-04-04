---
title: "Lism CSSだけでカフェLPを作ってみた — CDN1行で始めるユーティリティCSS"
emoji: "☕"
type: "tech"
topics: ["css", "lismcss", "html", "webdesign", "frontend"]
published: true
---

## はじめに

**Lism CSS**というCSSフレームワークをご存知でしょうか？

Tailwind CSSのようなユーティリティファーストのアプローチでありながら、独自のレイアウトシステムとデザイントークンを備えた新しいCSSフレームワークです。

今回は、Lism CSSのCDN版（v0.0.13）を使って、**架空のイタリアンカフェ「Trattoria LUCE」のLP**を1ファイルで作ってみました。

![完成イメージ](https://codequest.work/wp-content/uploads/2026/04/スクリーンショット-2026-04-03-8.15.58.jpg)

## Lism CSSとは

Lism CSSは、以下の特徴を持つCSSフレームワークです。

- **ユーティリティクラス**: `-ai:c`（align-items: center）、`-g:20`（gap）など短いクラス名
- **レイアウトプリミティブ**: `l--stack`、`l--flex`、`l--columns`、`l--center`など
- **デザイントークン**: `--c-main`、`--c-base`、`--c-text`などCSS変数で管理
- **CDN対応**: `<link>`タグ1行で導入可能

```html
<link href="https://cdn.jsdelivr.net/npm/lism-css@0.0.13/dist/css/main.css" rel="stylesheet" />
```

## 完成するもの

以下のセクションで構成されるカフェLPを作ります。

1. **ヘッダー**（固定ナビ + スクロール時の背景変化）
2. **ファーストビュー**（全画面ヒーロー）
3. **CONCEPT**（2カラム：画像 + テキスト）
4. **MENU**（3カラム：メニュー価格表）
5. **CHEF**（2カラム：テキスト + 画像）
6. **COURSE**（3カラム：コース料金カード）
7. **ACCESS + RESERVATION**（2カラム：店舗情報 + 予約フォーム）
8. **CTAバナー**（背景画像 + CTA）
9. **フッター**（2カラム）

## Step 1: ベースとカラートークンの設定

Lism CSSはデザイントークンをCSS変数で管理しています。これを上書きするだけで、テーマをカスタマイズできます。

今回はダークテーマ × ゴールドのカフェらしい配色にします。

```css
:root {
  /* Lism トークン上書き */
  --c-main: #c9a84c;            /* ゴールド — ブランドカラー */
  --c-accent: #d4b45e;          /* ゴールド明るめ — ホバー用 */
  --c-base: #111111;            /* ページ背景（黒） */
  --c-base-2: #1a1a1a;          /* カード・フッター背景 */
  --c-base-3: #222222;          /* セクション交互背景 */
  --c-text: #ffffff;            /* メインテキスト（白） */
  --c-text-2: rgba(255,255,255,0.65); /* サブテキスト */
  --c-divider: rgba(255,255,255,0.08); /* 罫線・ボーダー */
  --c-link: #c9a84c;            /* リンク色 */
}
```

ポイントは、Lism CSSが用意している`--c-main`や`--c-base`といったトークン名をそのまま使うこと。フレームワーク内部のユーティリティクラス（`-c:main`、`-bgc:base-2`など）がそのまま機能します。

## Step 2: レイアウトプリミティブを理解する

Lism CSSの最大の特徴は**レイアウトプリミティブ**です。よく使うものを紹介します。

### `l--stack`：縦積みレイアウト

子要素を縦方向に並べます。`-g:`でギャップを指定。

```html
<div class="l--stack -g:20">
  <p>テキスト1</p>
  <p>テキスト2</p>
</div>
```

### `l--flex`：横並びレイアウト

Flexboxの横並び。`-ai:c`で中央揃え、`-jc:sb`で両端揃え。

```html
<div class="l--flex -ai:c -jc:sb">
  <span>左</span>
  <span>右</span>
</div>
```

### `l--columns`：カラムレイアウト

`--cols`でカラム数を指定。レスポンシブ対応済み。

```html
<div class="l--columns -g:20" style="--cols: 3">
  <div>カラム1</div>
  <div>カラム2</div>
  <div>カラム3</div>
</div>
```

### `l--center`：中央配置

要素を縦横中央に配置します。ヒーローセクションに最適。

```html
<section class="l--center" style="min-height: 100dvh;">
  <h1>中央に表示</h1>
</section>
```

## Step 3: ヘッダーを作る

固定ナビゲーションをLismのユーティリティだけで組みます。

```html
<header class="l--flex -ai:c -jc:sb has--gutter -py:20"
        style="position: fixed; top: 0; left: 0; right: 0; z-index: 100;">
  <a href="#" class="font-display -td:n -c:text">LUCE</a>
  <nav class="l--flex -g:30 -ai:c">
    <a href="#concept" class="-td:n -c:text-2 -hov:fade">CONCEPT</a>
    <a href="#menu" class="-td:n -c:text-2 -hov:fade">MENU</a>
    <a href="#reserve" class="btn-reserve -py:10 -px:30">RESERVE</a>
  </nav>
</header>
```

**使っているクラスの意味：**

| クラス | 意味 |
|---|---|
| `l--flex` | display: flex |
| `-ai:c` | align-items: center |
| `-jc:sb` | justify-content: space-between |
| `has--gutter` | 左右にガター（余白） |
| `-py:20` | padding-block（上下余白） |
| `-td:n` | text-decoration: none |
| `-c:text-2` | color: var(--c-text-2) |
| `-hov:fade` | ホバー時にフェード効果 |

## Step 4: CONCEPTセクション — 2カラムレイアウト

画像とテキストを横並びにする典型的なセクションです。

```html
<section class="is--container -container:m has--gutter -py:50">
  <div class="l--columns -g:50 -ai:c" style="--cols: 2">
    <!-- 画像 -->
    <div class="img-zoom -ov:h" style="aspect-ratio: 3/4;">
      <img src="..." alt="店内の雰囲気" class="-w:100%"
           style="height: 100%; object-fit: cover;" />
    </div>
    <!-- テキスト -->
    <div class="l--stack -g:25">
      <p class="-c:main">コンセプト</p>
      <h2 class="font-serif">光が集まる場所で、本物のイタリアを。</h2>
      <p class="-c:text-2">シェフが現地の食材と伝統的な調理法にこだわり...</p>
    </div>
  </div>
</section>
```

`is--container -container:m`で最大幅を制限し、`has--gutter`で左右余白を確保。`l--columns`で2カラムに分割しています。

## Step 5: MENUセクション — 3カラム価格表

メニューの価格表は、ドットリーダーで品名と価格をつなぐデザインです。

```html
<div class="l--columns -g:20" style="--cols: 3">
  <div class="l--stack -g:25 -p:40 -bd -bdc:divider">
    <p class="font-display -ta:c -c:main -fz:l">Antipasto</p>
    <div class="gold-line" style="margin-inline: auto;"></div>
    <div class="l--stack -g:20">
      <div class="menu-row">
        <span>カプレーゼ ディ ブッラータ</span>
        <span class="menu-dots"></span>
        <span class="font-display -c:main -fz:l">1,800</span>
      </div>
    </div>
  </div>
  <!-- Pasta, Dolce も同様 -->
</div>
```

ドットリーダーはカスタムCSSで実装しています。

```css
.menu-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
}
.menu-dots {
  flex: 1;
  border-bottom: 1px dotted var(--c-divider);
  margin: 0 12px;
  min-width: 20px;
}
```

## Step 6: COURSEセクション — カード型3カラム

画像 + テキスト + 価格のカード型レイアウトです。

```html
<div class="l--columns -g:20" style="--cols: 3">
  <div class="l--stack -bd -bdc:divider">
    <div class="img-zoom -ov:h" style="aspect-ratio: 16/10;">
      <img src="..." alt="ランチコース" class="-w:100%"
           style="height: 100%; object-fit: cover;" />
    </div>
    <div class="l--stack -g:15 -p:30 -ai:c -ta:c">
      <p class="font-display -c:main -fz:s">LUNCH</p>
      <p class="font-serif -fw:500 -fz:l">Pranzo コース</p>
      <p class="-c:text-2">前菜・パスタ・デザート・カフェの全4品</p>
      <p class="font-display -c:main" style="font-size: 2rem;">
        ¥2,800<span class="-fz:s">/税込</span>
      </p>
    </div>
  </div>
</div>
```

`-bd`でボーダーを追加し、`-bdc:divider`でボーダーカラーを指定。中央のカードだけゴールドのボーダーにして差別化しています。

## Step 7: フェードインアニメーション

Intersection Observerを使った軽量なスクロールフェードインです。

```css
.fade-in {
  opacity: 0;
  transform: translateY(30px);
  transition: opacity 0.8s ease, transform 0.8s ease;
}
.fade-in.is-visible { opacity: 1; transform: translateY(0); }
.delay-1 { transition-delay: 0.1s; }
.delay-2 { transition-delay: 0.2s; }
.delay-3 { transition-delay: 0.3s; }
```

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      entry.target.classList.add('is-visible');
    }
  });
}, { threshold: 0.15 });

document.querySelectorAll('.fade-in').forEach((el) => observer.observe(el));
```

外部ライブラリ不要で、要素が画面に入ったタイミングでふわっと表示されます。`delay-1`〜`delay-3`でカラム要素を順番に表示させると、リッチな印象になります。

## Lism CSSを使ってみた感想

### 良かった点

- **クラス名が短い**: `-ai:c`、`-g:20`など、Tailwindより短く書ける
- **レイアウトプリミティブが便利**: `l--stack`、`l--columns`、`l--center`だけでほとんどのレイアウトが組める
- **デザイントークンの上書きが簡単**: CSS変数を書き換えるだけでテーマを変更できる
- **CDN1行で使える**: ビルド環境不要で試せる

### 気になった点

- **情報が少ない**: まだ新しいフレームワークなので、公式ドキュメント以外の情報がほとんどない
- **バージョンが0.x**: 破壊的変更の可能性があるため、本番利用はバージョン固定が必須
- **レスポンシブの細かい制御**: Tailwindの`md:`、`lg:`のようなブレイクポイント指定に慣れていると、別のアプローチが必要

## まとめ

Lism CSSは、Tailwindとも従来のCSSフレームワークとも違う独自のアプローチで、特に**レイアウト構築の効率**が高いと感じました。

今回のカフェLPはHTML1ファイル（約620行）で完結しています。CDN読み込み + CSS変数の上書き + レイアウトプリミティブの組み合わせだけで、ここまでのLPが作れるのは素直に面白いです。

デモのソースコードはGitHubで公開しています。

https://github.com/masakazuimai/Copying_practice/tree/main/advanced/007

興味がある方はぜひ触ってみてください。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=lism-css-cafe-lp) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=lism-css-cafe-lp) — ディレクターも使っているSEO診断ツールを公開中。
