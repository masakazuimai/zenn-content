---
title: "競合サイトのSEOスコア、知っていますか？ — 数字で見る差の怖さ"
emoji: "📊"
type: "tech"
topics: ["SEO", "Web制作", "マーケティング", "TypeScript", "Lighthouse"]
published: true
---

## はじめに — 「うちのサイト、検索で何位ですか？」

クライアントや上司からこう聞かれたとき、即答できるでしょうか。

さらに厄介なのは次の質問です。

> 「競合は何位なんですか？うちと何が違うんですか？」

感覚ではなく **数字で差を示せるか** が、SEO対策の説得力を決めます。

この記事では、競合サイトとのSEO上の差をどう数値化するか、そのアプローチとコードを紹介します。

## SEOの「差」はどこに表れるか

検索順位は結果でしかありません。順位を決めている **要因** を比較しないと、何を改善すべきかわかりません。

比較すべき主なポイント：

| カテゴリ | 比較項目 | 差が出やすい部分 |
|---------|---------|----------------|
| メタ情報 | title / description | キーワードの含め方、文字数 |
| 構造化データ | JSON-LDの有無・種類 | リッチリザルト表示の差 |
| パフォーマンス | Core Web Vitals | LCP・CLSの数値差 |
| コンテンツ | 見出し構造・文字数 | 情報の網羅性 |
| 技術SEO | canonical・robots・sitemap | クロール効率 |

## メタ情報の比較を自動化する

まず最も基本的な、titleとdescriptionの比較です。

```typescript
// メタ情報を取得する関数
const fetchMetaInfo = async (url: string) => {
  const res = await fetch(url)
  const html = await res.text()

  const title = html.match(/<title>([^<]*)<\/title>/i)?.[1] ?? ''
  const description = html.match(
    /meta\s+name=["']description["']\s+content=["']([^"']*)/i
  )?.[1] ?? ''

  return {
    url,
    title,
    titleLength: title.length,
    description,
    descriptionLength: description.length,
  }
}
```

これを自社サイトと競合サイトの両方に実行すれば、titleの文字数やキーワードの含め方を横並びで比較できます。

```typescript
// 2サイトのメタ情報を比較
const compare = async (myUrl: string, competitorUrl: string) => {
  const [mine, theirs] = await Promise.all([
    fetchMetaInfo(myUrl),
    fetchMetaInfo(competitorUrl),
  ])

  return { mine, theirs }
}
```

ただし、これはあくまでメタ情報だけの比較です。

## Lighthouseスコアをプログラムで取得する

パフォーマンスの比較には、Lighthouseが使えます。Chrome DevToolsで手動実行するのが一般的ですが、Node.jsからプログラム実行も可能です。

```typescript
// Lighthouse をプログラムから実行（概念コード）
import lighthouse from 'lighthouse'
import chromeLauncher from 'chrome-launcher'

const runLighthouse = async (url: string) => {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] })

  const result = await lighthouse(url, {
    port: chrome.port,
    output: 'json',
  })

  await chrome.kill()

  // 主要スコアを抽出
  const categories = result?.lhr?.categories
  return {
    performance: categories?.performance?.score ?? 0,
    seo: categories?.seo?.score ?? 0,
    accessibility: categories?.accessibility?.score ?? 0,
  }
}
```

このスコアを自社と競合で並べると、こんな表が作れます。

| 指標 | 自社サイト | 競合A | 競合B |
|------|----------|------|------|
| Performance | 72 | 91 | 85 |
| SEO | 80 | 95 | 88 |
| Accessibility | 85 | 90 | 78 |

**数字で見ると「どこが負けているか」が一目瞭然です。**

## 構造化データの差を確認する

検索結果にFAQや評価星が表示されるかどうかは、構造化データの有無で決まります。競合に表示されて自社に表示されないなら、それだけでCTRに差がつきます。

```typescript
// JSON-LDのスキーマタイプを抽出
const extractSchemaTypes = (html: string): string[] => {
  const scripts = html.match(
    /<script\s+type=["']application\/ld\+json["']>([\s\S]*?)<\/script>/gi
  )

  if (!scripts) return []

  return scripts.flatMap(script => {
    const json = script.replace(/<\/?script[^>]*>/gi, '')
    try {
      const data = JSON.parse(json)
      return [data['@type']].flat().filter(Boolean)
    } catch {
      return []
    }
  })
}
```

競合が `FAQPage` や `HowTo` を設定しているのに自社が未設定なら、そこが改善ポイントです。

## 手動比較の限界

ここまでのコードを見て気づいた方もいると思いますが、**これを全項目・全ページ・複数競合で実行するのは相当な手間です。**

- メタ情報の比較
- 構造化データの比較
- Core Web Vitalsの比較
- 見出し構造の比較
- 画像最適化の比較
- 内部リンク構造の比較

1サイトのチェックだけでも大変なのに、競合と比較するとなると工数は倍以上です。

筆者は [SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=competitor-seo-score-gap) を使って、まず自社サイトのスコアを把握し、同じツールで競合サイトも診断して差分を確認しています。URLを入れるだけで45項目以上のチェックが走るので、比較の起点として便利です。

## 競合分析で意識すべきこと

数値の比較ができたら、次のステップは **優先順位をつけること** です。

1. **差が大きい項目** から改善する（スコア差20以上は最優先）
2. **影響度が高い項目** を優先する（構造化データ > alt属性）
3. **実装コストが低い項目** から着手する（メタタグ修正 > サイト構造変更）

全てを一度に直す必要はありません。「競合との差が最も大きく、かつ直しやすい項目」から始めるのが効率的です。

## まとめ

- SEOの改善は「感覚」ではなく「数字」で競合と比較すべき
- メタ情報・パフォーマンス・構造化データが比較しやすい3大領域
- 手動比較はコードを書けばできるが、全項目を網羅するのは非現実的
- まず自社サイトのスコアを把握し、競合との差分から優先順位をつける

「なんとなく負けている気がする」を「ここが具体的に負けている」に変えるだけで、改善のスピードは段違いに上がります。

---

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=competitor-seo-score-gap) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=competitor-seo-score-gap) — Web制作・SEO関連の技術情報サイト
