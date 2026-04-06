---
title: "SEOスコアバッジをCloudflare Workers + SVG動的生成で実装した設計と判断"
emoji: "🏅"
type: "tech"
topics: ["CloudflareWorkers", "Hono", "SVG", "SEO", "NextJS"]
published: true
---

## 作ったもの

サイトに埋め込めるSEOスコアバッジ。reCAPTCHAのようにサイト右下に常駐し、ホバーでスコアが展開、クリックで第三者検証ページに飛べます。

[CodeQuest.work SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-badge-cloudflare-workers-svg) で85点以上を達成したドメインに発行されます。

この記事では、まず**コピペで動くミニマルなサンプル**を示した上で、実プロダクトで直面した**設計判断とトレードオフ**を解説します。

## まず動かす：WorkersでSVGバッジを返すサンプル

Cloudflare Workersで動的にSVGバッジを生成する最小構成です。`wrangler init`後にそのまま貼れば動きます。

```typescript
// src/index.ts — Honoで動くSVGバッジAPI
import { Hono } from "hono"

const app = new Hono()

function badgeSvg(label: string, value: string, color: string): string {
  const lw = label.length * 7 + 10
  const vw = value.length * 7 + 10
  const w = lw + vw
  return `<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="20">
    <rect width="${lw}" height="20" fill="#555"/>
    <rect x="${lw}" width="${vw}" height="20" fill="${color}"/>
    <g fill="#fff" font-family="Verdana" font-size="11" text-anchor="middle">
      <text x="${lw / 2}" y="14">${label}</text>
      <text x="${lw + vw / 2}" y="14">${value}</text>
    </g>
  </svg>`
}

app.get("/badge/:score", (c) => {
  const score = Number(c.req.param("score"))
  const color = score >= 90 ? "#4c1" : score >= 60 ? "#dfb317" : "#e05d44"
  return c.body(badgeSvg("SEO Score", String(score), color), 200, {
    "Content-Type": "image/svg+xml",
    "Cache-Control": "public, max-age=3600",
  })
})

export default app
```

`curl localhost:8787/badge/92` でSVGが返ってきます。shields.ioと同じ見た目のバッジが、外部サービスなしで生成できます。

これが基本形。ここから先が「実プロダクトでどう判断したか」の話です。

## 判断1：shields.io vs 自前生成

最初はshields.ioのエンドポイントを使う案もありました。

| | shields.io | 自前生成 |
|---|---|---|
| 実装コスト | ほぼゼロ | SVGテンプレート作成が必要 |
| 外部依存 | あり（ダウン時にバッジ消える） | なし |
| カスタマイズ | 限定的 | 完全自由 |
| レスポンス速度 | shields.io → Workers → ユーザー | Workers → ユーザー |

**選んだのは自前生成**。理由は2つ。

1. ユーザーのサイトに表示するバッジが外部サービス障害で消えるのは許容できない
2. SVGはXML文字列なのでテンプレートリテラルで十分。ライブラリ不要で実装コストも低い

shields.ioは素晴らしいサービスですが、「他人のサイトに埋め込まれるもの」に外部依存を入れるのはリスクが高すぎました。

## 判断2：ON/OFFをどう実現するか

ユーザーがダッシュボードからバッジを非公開にしたとき、2つの方法があります。

**案A**: 埋め込みコードを2種類（公開用/非公開用）発行し、貼り替えてもらう

**案B**: 同じURLのまま、APIが返すSVGを切り替える

```typescript
// 案Bの実装イメージ
const TRANSPARENT = '<svg xmlns="http://www.w3.org/2000/svg" width="1" height="1"/>'

app.get("/image/:token.svg", async (c) => {
  const badge = await db.findByToken(token)

  if (!badge?.enabled) {
    return c.body(TRANSPARENT, 200, { "Content-Type": "image/svg+xml" })
  }

  return c.body(badgeSvg("SEO Score", String(badge.score), color), 200, {
    "Content-Type": "image/svg+xml",
  })
})
```

**選んだのは案B**。理由は明確で、ユーザーにHTMLの貼り替えを要求するのは現実的ではないからです。

特にWordPressのテンプレートやフッターに埋め込んでいる場合、「OFFにしたいからコード差し替えてください」は手間がかかりすぎる。1x1の透明SVGを返せば、ブラウザ上では何も表示されません。

### ハマりポイント：Cache-Controlとの戦い

ここで問題が起きました。`Cache-Control: public, max-age=3600`を付けていたため、OFFにしてもCDNキャッシュが残っている間はバッジが表示され続けます。

かといって`no-cache`にすると、人気サイトに埋め込まれた場合にWorkersへのリクエストが毎回走る。

最終的には`max-age=300`（5分）に落ち着きました。ON/OFFの反映が5分遅延する代わりに、通常時のリクエスト数を抑える。ユーザーに「反映まで数分かかります」と伝えれば許容範囲です。

## 判断3：1ユーザー1バッジ vs ドメイン毎

最初は`users`テーブルに`badge_token`を1つ持つ設計でした。しかしすぐに問題が見えました。

- Web制作者は複数サイトを運営している
- クライアントサイトと自社サイトでON/OFFを分けたい
- 最新チェックのスコアが別サイトのものに上書きされる

そこで`domain_badges`テーブルを分離しました。

```sql
CREATE TABLE domain_badges (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    domain TEXT NOT NULL,
    badge_token TEXT NOT NULL,
    badge_enabled INTEGER DEFAULT 0,
    latest_score INTEGER,
    latest_checked_at DATETIME,
    UNIQUE(user_id, domain)
);
```

### トレードオフ：スコアをどこから取るか

バッジ画像のリクエスト時に毎回`check_history`テーブルをJOINしてスコアを取る案と、`domain_badges.latest_score`にキャッシュしておく案。

JOINの方がデータは常に最新ですが、バッジ画像は外部サイトから大量にリクエストされる可能性があります。D1のクエリ数を節約するため、キャッシュ方式を採用し、SEOチェック実行時に`latest_score`を更新する設計にしました。

```typescript
// チェック実行後にキャッシュを更新（イメージ）
await db.prepare(
  "UPDATE domain_badges SET latest_score = ?, latest_checked_at = CURRENT_TIMESTAMP WHERE user_id = ? AND domain = ?"
).bind(score, userId, domain).run()
```

デメリットは「チェック実行」と「バッジ更新」が別々のタイミングになること。チェックルートにバッジ更新ロジックが混ざるので凝集度は下がりますが、パフォーマンスを優先しました。

## 判断4：埋め込みHTMLの設計

埋め込みコードをどう設計するか。3つの案がありました。

| | `<img>`タグだけ | `<script>`埋め込み | HTML+インラインCSS |
|---|---|---|---|
| 導入の手軽さ | 最も簡単 | やや抵抗あり | 簡単 |
| デザインの自由度 | なし | 高い | 中程度 |
| セキュリティ懸念 | なし | 外部JS実行への抵抗 | なし |

**選んだのはHTML+インラインCSS**。

`<script>`は「他人のJSを自サイトで実行する」ことへの心理的抵抗が大きい。`<img>`だけだとフローティング表示やホバー展開ができない。HTML+インラインCSSなら、依存ゼロでデザインも実現でき、コードを読めば何をしているか一目瞭然です。

```css
/* ホバーでスライド展開するCSS */
#seo-check-badge:hover #seo-badge-text {
  max-width: 192px; /* 0 → 192px のtransitionで展開 */
}
```

CSSだけでアニメーションが完結するので、JSが無効な環境でも壊れません。

## まとめ

| 判断ポイント | 選択 | 理由 |
|---|---|---|
| バッジ生成 | 自前SVG | 外部依存排除 |
| ON/OFF | 透明SVG差し替え | ユーザーのコード貼り替え不要 |
| データ設計 | ドメイン毎テーブル | 複数サイト運営に対応 |
| スコア取得 | キャッシュ | 外部リクエスト耐性 |
| 埋め込み形式 | HTML+インラインCSS | JS不要・依存ゼロ |

どれも「技術的に正しい方」ではなく「ユーザーにとって面倒が少ない方」を選んでいます。バッジは一度貼ったら忘れられるべきもので、運用コストをユーザーに転嫁しない設計が重要でした。

---

この記事で紹介したバッジ機能は [CodeQuest.work SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-badge-cloudflare-workers-svg) で実際に使えます。85点以上を達成すると、ダッシュボードからバッジを取得できます。

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-badge-cloudflare-workers-svg) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-badge-cloudflare-workers-svg) — Web制作・SEO関連の技術情報サイト
