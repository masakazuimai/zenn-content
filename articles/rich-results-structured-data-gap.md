---
title: "Googleのリッチリザルトに出るサイトと出ないサイト、何が違うのか"
emoji: "⭐"
type: "tech"
topics: ["SEO", "構造化データ", "JSON-LD", "Google", "NextJS"]
published: false
---

## はじめに — 同じ1位でもクリック率が違う

Google検索で同じ順位に表示されていても、**クリック率（CTR）に2〜3倍の差** がつくことがあります。

違いは「リッチリザルト」の有無です。

- FAQ（よくある質問）が検索結果に展開表示される
- 評価の星マーク（★★★★☆）が表示される
- パンくずリストが階層表示される
- レシピの調理時間やカロリーが表示される

これらは全て **構造化データ（JSON-LD）** を正しく設定しているかどうかで決まります。

## リッチリザルトの種類と効果

Googleがサポートしているリッチリザルトの主なタイプです。

| タイプ | 表示内容 | 向いているサイト |
|-------|---------|---------------|
| FAQ | 質問と回答の展開 | サービスサイト、LP |
| HowTo | 手順のステップ表示 | チュートリアル、ブログ |
| Article | 公開日・著者・サムネイル | メディア、ブログ |
| Product | 価格・在庫・レビュー | ECサイト |
| BreadcrumbList | パンくずリスト | 全サイト |
| LocalBusiness | 住所・営業時間・地図 | 店舗ビジネス |

**設定していないサイトは、これらの表示枠を全て競合に譲っています。**

## JSON-LDの基本構造

構造化データは `<script type="application/ld+json">` タグでHTMLに埋め込みます。

最も基本的なArticleスキーマの例：

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "記事タイトル",
  "author": {
    "@type": "Person",
    "name": "著者名"
  },
  "datePublished": "2025-01-15",
  "image": "https://example.com/image.jpg"
}
```

FAQPageスキーマの例：

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "質問テキスト",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "回答テキスト"
      }
    }
  ]
}
```

構造自体はシンプルですが、**問題は「正しく書けているか」の判定** です。

## 自サイトの構造化データを検出する

まず、自分のサイトに構造化データが入っているかを確認するコードです。

```typescript
// ページ内のJSON-LDを全て抽出
const extractJsonLd = (html: string) => {
  const pattern = /<script\s+type=["']application\/ld\+json["']>([\s\S]*?)<\/script>/gi
  const results: Array<{ type: string; valid: boolean }> = []
  let match

  while ((match = pattern.exec(html)) !== null) {
    try {
      const data = JSON.parse(match[1])
      const types = [data['@type']].flat().filter(Boolean)
      results.push({ type: types.join(', '), valid: true })
    } catch {
      results.push({ type: 'パースエラー', valid: false })
    }
  }

  return results
}
```

これで「構造化データがあるか」「何のタイプか」はわかります。

## 「ある」だけでは不十分

構造化データが存在しても、リッチリザルトに表示されないケースがあります。

**よくある原因：**

| 問題 | 具体例 |
|------|-------|
| 必須プロパティの欠落 | Articleにheadlineがない |
| プロパティの型が違う | datePublishedが不正な日付形式 |
| スキーマタイプの選択ミス | ブログ記事にProduct型を使用 |
| ネストの構造エラー | @contextの位置が不正 |
| Googleの非対応型 | サポート外のスキーマを使用 |

**「構造化データがある」と「リッチリザルトに表示される」の間には大きなギャップがあります。**

Google公式の[リッチリザルトテスト](https://search.google.com/test/rich-results)で1ページずつ確認できますが、サイト全体を確認するのは手間です。

## Next.jsでの構造化データ実装

Next.js（App Router）で構造化データを追加する場合、`<script>` タグを直接埋め込みます。

```tsx
// app/blog/[slug]/page.tsx
const ArticlePage = ({ article }) => {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: article.title,
    author: { '@type': 'Person', name: article.author },
    datePublished: article.publishedAt,
    image: article.ogImage,
  }

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      {/* 記事本文 */}
    </>
  )
}
```

ここで問題になるのは、**どのページにどのスキーマタイプが最適か** の判断です。トップページはOrganization？WebSite？サービスページはProduct？SoftwareApplication？

この選択を間違えると、構造化データを入れても効果が出ません。

## 構造化データの優先順位

全ページに一気に導入するのは大変なので、優先順位をつけます。

1. **BreadcrumbList** — 全ページに共通で入れられる。実装コスト低
2. **Article** — ブログ・メディアがあるなら必須
3. **FAQPage** — サービスページのCTR向上に直結
4. **Organization / LocalBusiness** — ブランド検索のナレッジパネル表示
5. **Product** — ECサイトなら最優先

「自分のサイトにどのスキーマが必要で、正しく設定されているか」を一括で確認できると効率的です。筆者は [SEO CHECK](https://seo.codequest.work/) で構造化データの検出・必須プロパティのチェック・スキーマタイプの判定を自動化しています。構造化データだけで40点満点の診断カテゴリになっているので、かなり詳しく見てくれます。

## まとめ

- リッチリザルトはCTRに2〜3倍の差を生む
- JSON-LDで構造化データを設定すれば表示対象になる
- ただし「存在するだけ」ではダメで、正しい構造・必須プロパティが必要
- Next.jsでは `<script type="application/ld+json">` で直接埋め込む
- スキーマタイプの選択を間違えると効果が出ない

検索順位を上げるのは時間がかかりますが、構造化データの追加は **即日反映される可能性がある** 数少ないSEO施策です。競合がリッチリザルトを表示しているなら、今すぐ対応する価値があります。

---

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=rich-results-structured-data-gap) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=rich-results-structured-data-gap) — Web制作・SEO関連の技術情報サイト
