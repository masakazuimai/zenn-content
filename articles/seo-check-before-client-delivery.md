---
title: "納品したサイト、SEO大丈夫？ — クライアントに聞かれる前にチェックすべき5項目"
emoji: "🔍"
type: "tech"
topics: ["SEO", "Web制作", "フリーランス", "NextJS", "TypeScript"]
published: true
---

## はじめに — 「SEOどうなってますか？」が一番怖い

Web制作のフリーランスをやっていると、納品後にこんな連絡が来ることがあります。

> 「サイト公開したんですけど、検索に全然出てこないんですが...」

デザインは褒められた。表示速度も問題ない。でも **SEOの基本設定が抜けていた** — これは実際によくある話です。

問題は、SEOの不備は **納品直後には気づかれない** こと。Googleにインデックスされて数週間〜数ヶ月後に「検索に出ない」と言われ、信頼を失うパターンです。

この記事では、納品前にチェックすべきSEOの最低限の項目と、簡単な検出コードを紹介します。

## 納品前に最低限チェックすべき5項目

### 1. メタタグ（title / description）

最も基本的で、最も忘れられやすい項目です。

```typescript
// メタタグの存在チェック（簡易版）
const checkMetaTags = (html: string) => {
  const hasTitle = /<title>.+<\/title>/i.test(html)
  const hasDescription = /meta\s+name=["']description["']/i.test(html)

  return {
    title: hasTitle,
    description: hasDescription,
  }
}
```

titleが空、descriptionが未設定のまま納品しているケースは意外と多いです。特にNext.jsのApp Routerでは `metadata` オブジェクトの設定を忘れがちです。

**チェックポイント：**
- 全ページにtitleが設定されているか
- titleが60文字以内か（超えるとGoogle検索結果で切れる）
- descriptionが120文字前後で設定されているか
- 各ページでtitle/descriptionが固有か（コピペになっていないか）

### 2. OGPタグ

SNSでシェアされたときの見え方を決めるタグです。クライアントが自社サイトをSNSでシェアしたとき、画像もタイトルも出ないと「このサイト大丈夫？」となります。

```typescript
// OGPタグの検出
const checkOGP = (html: string) => {
  const ogTags = ['og:title', 'og:description', 'og:image', 'og:url']

  return ogTags.map(tag => ({
    tag,
    exists: new RegExp(`property=["']${tag}["']`, 'i').test(html),
  }))
}
```

**特に `og:image` の未設定は致命的です。** シェアされたときにリンクだけの味気ない表示になります。

### 3. 見出し構造（h1〜h3）

h1が2つある、h1がない、h2を飛ばしてh3がある — こうした見出し構造の崩れは、Googleのクロール品質に影響します。

```typescript
// 見出し構造のチェック
const checkHeadings = (html: string) => {
  const h1Count = (html.match(/<h1[\s>]/gi) || []).length
  const hasH2 = /<h2[\s>]/gi.test(html)

  return {
    h1Count,      // 1が理想
    hasH2,        // h1の次にh2があるか
    isValid: h1Count === 1 && hasH2,
  }
}
```

CMSやコンポーネントの組み合わせで、意図せずh1が複数出ることはよくあります。

### 4. 構造化データ（JSON-LD）

リッチリザルト（検索結果にFAQや評価星が表示される）を出すために必要です。設定しなくても検索には出ますが、**CTR（クリック率）に大きな差が出ます。**

```typescript
// 構造化データの存在チェック
const checkStructuredData = (html: string) => {
  const jsonLdMatches = html.match(
    /<script\s+type=["']application\/ld\+json["']>([\s\S]*?)<\/script>/gi
  )

  return {
    exists: jsonLdMatches !== null,
    count: jsonLdMatches?.length ?? 0,
  }
}
```

ただし、構造化データは「あるかないか」だけでなく「正しいか」が重要です。必須プロパティの漏れ、スキーマタイプの選択ミスなどは手動では見つけにくいポイントです。

### 5. canonical URL

同じコンテンツが複数URLでアクセスできる場合、Googleに「正規のURLはこれです」と伝えるタグです。設定漏れは重複コンテンツ扱いの原因になります。

```typescript
// canonical URLのチェック
const checkCanonical = (html: string) => {
  const match = html.match(
    /<link\s+rel=["']canonical["']\s+href=["']([^"']+)["']/i
  )

  return {
    exists: match !== null,
    url: match?.[1] ?? null,
  }
}
```

www有無、末尾スラッシュの有無、httpとhttpsの混在 — これらが放置されているサイトは多いです。

## 「5項目だけ」では足りない現実

ここまで5つの基本項目を紹介しましたが、実際のSEOチェックは **もっと多岐にわたります。**

- 画像のalt属性は全て設定されているか
- 内部リンクの構造は適切か
- モバイル対応は問題ないか
- ページ速度（Core Web Vitals）は基準を満たしているか
- robots.txtでクロールをブロックしていないか
- サイトマップは正しく生成されているか

これを毎回手動でやるのは現実的ではありません。

筆者は自作の [SEO CHECK](https://seo.codequest.work/) で、URLを入れるだけで45項目以上を一括チェックしています。納品前の最終確認として使うと、抜け漏れを防げます。

## 納品フローに組み込むのがベスト

SEOチェックは「思い出したときにやる」だと漏れます。納品フローの一部に組み込むのがおすすめです。

| フェーズ | やること |
|---------|---------|
| 開発中 | メタタグ・OGPをコンポーネントに組み込む |
| ステージング | 全ページのSEOチェックを実行 |
| 納品前 | チェック結果をクライアントに共有 |
| 納品後1週間 | Google Search Consoleでインデックス状況を確認 |

チェック結果をレポートとして見せるだけで、クライアントからの信頼度が変わります。「SEOもちゃんと見てくれている」という安心感は、次の案件にもつながります。

## まとめ

- 納品後に「検索に出ない」と言われるのはSEO設定漏れが原因
- 最低限チェックすべきは **メタタグ・OGP・見出し構造・構造化データ・canonical** の5項目
- ただし実際のSEO項目は45以上あり、手動では非現実的
- 納品フローにSEOチェックを組み込むことで品質を担保できる

「コードは書けるけどSEOは自信がない」というエンジニアほど、ツールで仕組み化するのが効果的です。

---

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-check-before-client-delivery) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-check-before-client-delivery) — Web制作・SEO関連の技術情報サイト
