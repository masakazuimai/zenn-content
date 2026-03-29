---
title: "MEO対策、Webサイト側は大丈夫？ローカルSEO診断で見落としを防ぐ"
emoji: "📍"
type: "tech"
topics: ["SEO", "LocalSEO", "MEO", "構造化データ", "Web制作"]
published: true
---

## Googleビジネスプロフィールだけで満足していませんか？

地域ビジネスのWeb制作を請けると、必ずと言っていいほど「MEO対策もお願いします」と言われます。

Googleビジネスプロフィール（GBP）の最適化はもはや定番。写真を追加して、営業時間を設定して、口コミを集めて......。ここまではほとんどの制作者がやっています。

ところが、**Webサイト側のローカルSEO対策**となると、意外と抜けが多い。

- 構造化データにLocalBusinessスキーマを入れていない
- 電話番号はあるけど`tel:`リンクになっていない
- Googleマップの埋め込みを忘れている
- タイトルや見出しに地域名が入っていない

GBPとWebサイトの情報が一致していることはGoogleも重視しています。GBPだけ完璧でも、Webサイト側がスカスカでは片手落ちです。

## 何をチェックすべきか：5つの診断項目

地域ビジネスのWebサイトで最低限押さえるべきポイントを5つに整理しました。

### 1. LocalBusiness構造化データ

JSON-LD形式の`LocalBusiness`スキーマは、Googleに「このサイトは地域の事業者です」と伝える最も確実な方法です。

チェックすべき必須プロパティは以下の通りです。

| プロパティ | 内容 |
|-----------|------|
| `name` | 店舗・事業者名 |
| `url` | WebサイトURL |
| `logo` | ロゴ画像 |
| `sameAs` | SNSアカウント等 |
| `address` | 住所（PostalAddress） |
| `telephone` | 電話番号 |

「構造化データは入れてある」と思っていても、`Organization`だけで`LocalBusiness`が抜けているケースは結構あります。WordPressのSEOプラグインでも、デフォルトではLocalBusinessにならない場合が多いので注意が必要です。

```json
// LocalBusinessスキーマの基本構造（イメージ）
{
  "@context": "https://schema.org",
  "@type": "LocalBusiness",
  "name": "サンプル歯科医院",
  "url": "https://example.com",
  "telephone": "+81-3-1234-5678",
  "address": {
    "@type": "PostalAddress",
    "addressLocality": "渋谷区",
    "addressRegion": "東京都"
  }
}
```

### 2. 電話番号のtel:リンク

スマートフォンからの閲覧が大半を占める今、電話番号がタップで発信できないのは機会損失です。

テキストで電話番号を書いているだけでは不十分。`<a href="tel:03-1234-5678">` のようにリンクにしておく必要があります。

地味ですが、特に飲食店・クリニック・士業など「今すぐ電話したい」ユーザーが多い業種では、これだけでコンバージョンが変わります。

### 3. 住所の記載

ページ内に住所がテキストとして記載されているかどうか。画像だけに住所を入れていたり、Googleマップの埋め込みだけで済ませていたりするケースがあります。

テキストとして住所が存在することで、検索エンジンはサイトの所在地を正しく認識できます。構造化データの`address`と一致させるのが理想です。

### 4. Googleマップの埋め込み

`<iframe>`でGoogleマップを埋め込んでいるかどうか。アクセスページだけでなく、トップページのフッターにも設置しておくと、ユーザーの利便性とローカルシグナルの両方に効きます。

Google マップの埋め込みは、ユーザーにとって「場所がすぐわかる」という直接的な価値があると同時に、Googleに対しても所在地情報を補強するシグナルになります。

### 5. 地域キーワードの配置

`<title>` タグ、meta description、`<h1>` タグに地域名が含まれているかどうか。

「渋谷区 歯科医院」「新宿 税理士」のように、**地域名 + 業種**の組み合わせがページの主要な要素に入っていることが、ローカル検索での露出に直結します。

トップページだけでなく、サービスページや料金ページにも地域名を入れておくと、より多くのキーワードでの流入が期待できます。

## 手動チェックは現実的か

ここまで5項目を挙げましたが、これを毎回手動で確認するのは面倒です。

- ソースを開いてJSON-LDを探す
- `tel:`リンクの有無をCtrl+Fで検索
- Googleマップのiframeがあるか確認
- titleタグに地域名が入っているか確認

1サイトならまだしも、複数のクライアントサイトを抱えていると、抜け漏れが出ます。

### 自動チェックの仕組み

技術的には、HTMLを取得して以下のようなチェックを走らせるだけです。

```typescript
// ローカルSEO診断のチェック処理（イメージコード）

function checkLocalBusiness(html: string) {
  // JSON-LDスクリプトタグからLocalBusinessを探す
  const jsonLdScripts = extractJsonLd(html)
  const localBiz = jsonLdScripts.find(
    (s) => s["@type"] === "LocalBusiness"
  )

  if (!localBiz) return { found: false }

  // 必須プロパティの充足率を計算
  const required = ["name", "url", "telephone", "address"]
  const present = required.filter((p) => localBiz[p])

  return {
    found: true,
    completeness: present.length / required.length,
  }
}
```

```typescript
// tel:リンクの検出（イメージコード）
function checkTelLink(html: string) {
  return /href=["']tel:/i.test(html)
}

// Googleマップ埋め込みの検出（イメージコード）
function checkGoogleMap(html: string) {
  return /google\.com\/maps/i.test(html)
}
```

個々のチェックはシンプルですが、5項目をまとめて一発で確認できると、制作ワークフローに組み込みやすくなります。

## 納品前チェックリストとして使う

ローカルSEO診断が最も活きるのは、**地域ビジネスのサイト納品前**です。

制作の最終段階で以下を確認するフローを入れておけば、「後からMEO対策が必要になった」「構造化データが入っていなかった」という手戻りを防げます。

| チェック項目 | 対応が必要なケース |
|-------------|-------------------|
| LocalBusiness構造化データ | 未設置、またはプロパティ不足 |
| tel:リンク | 電話番号がテキストのみ |
| 住所記載 | 画像のみ、またはマップのみ |
| Googleマップ | 未埋め込み |
| 地域キーワード | title/h1に地域名なし |

クライアントへの報告時にも「ローカルSEOの5項目すべてクリアしています」と言えると、**制作の品質を具体的に示せます**。

## まとめ

- MEO対策はGBPだけでなく、**Webサイト側のローカルSEO**も重要
- チェックすべきは5項目：構造化データ、tel:リンク、住所、Googleマップ、地域キーワード
- 手動確認は手間がかかるので、自動チェックをワークフローに組み込むと効率的
- 納品前チェックリストとして使えば、**手戻り防止と品質アピール**の両方に効く

[CodeQuest.work SEO_CHECK](https://seo.codequest.work) のSEO診断にローカルSEO診断セクションを追加しました。URLを入力するだけで5項目を一括チェックできます。地域ビジネスのサイト制作に関わる方は、ぜひ試してみてください。

---

**SEO診断ツール**: [CodeQuest.work SEO_CHECK](https://seo.codequest.work)
**制作・開発**: [CodeQuest.work](https://codequest.work/) — Web制作・SEO関連の技術情報サイト
