---
title: "Core Web Vitalsが悪いとどうなるか — スコア改善のアプローチ"
emoji: "⚡"
type: "tech"
topics: ["SEO", "パフォーマンス", "NextJS", "Lighthouse", "TypeScript"]
published: true
---

## はじめに — 「遅いサイト」はGoogleに嫌われる

2021年からGoogleは **Core Web Vitals（CWV）** を検索ランキングの要因に組み込んでいます。

つまり、サイトの表示速度が遅い＝検索順位が下がる可能性がある、ということです。

「うちのサイトは大丈夫だろう」と思っていても、実際に計測すると意外なスコアが出ることがあります。特にモバイルでの数値は、PCと大きく異なるケースが多いです。

## Core Web Vitalsとは

3つの指標でユーザー体験を数値化したものです。

| 指標 | 正式名称 | 計測対象 | 良好の基準 |
|------|---------|---------|----------|
| **LCP** | Largest Contentful Paint | 最大コンテンツの表示速度 | 2.5秒以下 |
| **INP** | Interaction to Next Paint | 操作への応答速度 | 200ms以下 |
| **CLS** | Cumulative Layout Shift | レイアウトのズレ量 | 0.1以下 |

**LCPが遅い** = ページの表示が遅い
**INPが遅い** = ボタンを押しても反応が遅い
**CLSが大きい** = 読んでいる途中にレイアウトがガタガタ動く

どれもユーザーが「このサイト使いにくい」と感じる直接的な原因です。

## CWVが悪いとどうなるか

### 検索順位への影響

Googleは公式に「CWVはランキングシグナルの一つ」と明言しています。ただし、コンテンツの質ほどの影響力はありません。

**実際の影響イメージ：**
- コンテンツの質が同等の競合2サイト → CWVが良い方が上位
- CWVが悪くてもコンテンツが圧倒的なら上位は維持
- ただしCWVが極端に悪い場合はペナルティに近い影響も

### 直帰率への影響

Googleの調査によると：
- 表示速度が1秒→3秒になると **直帰率が32%増加**
- 1秒→5秒で **直帰率が90%増加**
- 1秒→10秒で **直帰率が123%増加**

検索順位以前に、**ユーザーがページを見てくれない** という問題が起きます。

## JavaScriptでCWVを計測する

`web-vitals` ライブラリを使うと、実際のユーザー環境でCWVを計測できます。

```typescript
import { onLCP, onINP, onCLS } from 'web-vitals'

// 各指標を計測してコンソールに出力
onLCP(metric => {
  console.log('LCP:', metric.value, 'ms')
})

onINP(metric => {
  console.log('INP:', metric.value, 'ms')
})

onCLS(metric => {
  console.log('CLS:', metric.value)
})
```

これを本番環境に仕込んで分析ツールに送ると、**実ユーザーのCWVデータ** が蓄積されます。Lighthouseのラボデータとは異なる、リアルな数値が見えます。

## Next.jsでよくあるCWVの問題と対策

### LCPの改善 — 画像最適化

LCPが遅い原因の多くは **画像の読み込み** です。

```tsx
// NG: 通常のimgタグ（最適化なし）
<img src="/hero.png" alt="ヒーロー画像" />

// OK: next/imageで自動最適化
import Image from 'next/image'

<Image
  src="/hero.png"
  alt="ヒーロー画像"
  width={1200}
  height={630}
  priority  // LCP対象の画像にはpriorityを付ける
/>
```

`priority` を付けると、Next.jsがその画像をプリロードしてくれます。ファーストビューに表示される画像には必ず付けましょう。

### CLSの改善 — レイアウトシフト防止

CLSが悪化する典型的なパターン：

```tsx
// NG: サイズ未指定の画像（読み込み後にレイアウトが動く）
<img src="/banner.png" alt="バナー" />

// OK: width/heightを事前に指定
<Image
  src="/banner.png"
  alt="バナー"
  width={728}
  height={90}
/>
```

フォントの読み込みでもCLSは発生します。

```tsx
// next/fontでフォントのレイアウトシフトを防止
import { Noto_Sans_JP } from 'next/font/google'

const notoSansJP = Noto_Sans_JP({
  subsets: ['latin'],
  display: 'swap',    // フォント読み込み中はシステムフォントを表示
  preload: true,
})
```

### INPの改善 — 重い処理の分離

ボタンクリック時に重い処理が走ると、INPが悪化します。

```typescript
// NG: メインスレッドで重い処理を実行
const handleClick = () => {
  const result = heavyCalculation(data)  // UIがフリーズ
  setResult(result)
}

// OK: requestIdleCallbackで処理を分離
const handleClick = () => {
  setLoading(true)
  requestIdleCallback(() => {
    const result = heavyCalculation(data)
    setResult(result)
    setLoading(false)
  })
}
```

## 計測はできても「何を直すか」が難しい

CWVの計測自体は上記のコードで可能です。しかし、**スコアが悪いとわかった後に何を直すべきか** の判断は簡単ではありません。

- LCPが遅いのは画像のせい？サーバーレスポンスのせい？レンダリングブロックのせい？
- CLSが高いのはどの要素が原因？
- 他のSEO項目は問題ないか？

CWVだけ改善しても、メタタグや構造化データが抜けていたら意味がありません。筆者は [SEO CHECK](https://seo.codequest.work/) でCWVを含む45項目以上を一括チェックし、どこから手をつけるべきかの優先順位を判断しています。

## まとめ

- Core Web Vitals（LCP / INP / CLS）はGoogleのランキング要因
- 表示速度の悪化は直帰率の大幅な増加に直結する
- `web-vitals` ライブラリで実ユーザーの計測が可能
- Next.jsでは `next/image` の `priority` やフォント最適化が効果的
- 計測→原因特定→優先順位付けの流れで改善する

パフォーマンス改善は地味な作業ですが、検索順位・直帰率・コンバージョン全てに影響します。まず自分のサイトの現状スコアを把握するところから始めてみてください。

---

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=core-web-vitals-impact-and-fix) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=core-web-vitals-impact-and-fix) — Web制作・SEO関連の技術情報サイト
