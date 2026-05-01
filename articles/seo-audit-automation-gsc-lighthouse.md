---
title: "Search Console × Lighthouse × 自作スクリプトで回す SEO 監査の自動化"
emoji: "🤖"
type: "tech"
topics: ["seo", "googlesearchconsole", "lighthouse", "typescript", "githubactions"]
published: true
---

## はじめに

SEO 監査を手動でやると、だいたい 3 ヶ月で続かなくなります。

理由は単純で、**毎週同じ指標を眺める作業に飽きる**からです。さらに、異常が起きてから気づくまでのラグが長く、「先週急落していたのに今日まで気づかなかった」が起こります。

そこで私が運用しているのが、**Search Console API × Lighthouse × 自作の差分スクリプト**を組み合わせた監査パイプラインです。週次で自動実行され、**変化があったページだけ**を Slack に通知します。

本記事では、このパイプラインの構成と実装のポイント、そして運用で見えたハマりどころをまとめます。

## 監査パイプラインの全体像

構成はこの 3 層です。

1. **Search Console API** — クエリ・ページ単位の impressions/clicks/ctr/position を週次で取得
2. **Lighthouse** — 監視対象の重要ページに対して Performance/SEO/Accessibility をスコアリング
3. **自作スクリプト** — 前週比で「**有意に悪化したページ**」だけを抽出して通知

ポイントは、**全ページを毎回通知しない**ことです。数百ページあるサイトで「全ページの前週比レポート」を送っても、誰も読みません。**変化があったものだけを出す**のが継続運用のコツです。

## 1. Search Console API で週次データを取る

まず Search Console のデータ取得から。`googleapis` パッケージを使います。

```typescript
import { google } from "googleapis"

type SearchAnalyticsRow = {
  readonly page: string
  readonly clicks: number
  readonly impressions: number
  readonly ctr: number
  readonly position: number
}

async function fetchWeeklyStats(
  siteUrl: string,
  startDate: string,
  endDate: string
): Promise<readonly SearchAnalyticsRow[]> {
  const auth = new google.auth.GoogleAuth({
    keyFile: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    scopes: ["https://www.googleapis.com/auth/webmasters.readonly"],
  })

  const webmasters = google.webmasters({ version: "v3", auth })

  const response = await webmasters.searchanalytics.query({
    siteUrl,
    requestBody: {
      startDate,
      endDate,
      dimensions: ["page"],
      rowLimit: 1000,
    },
  })

  return (response.data.rows ?? []).map((row) => ({
    page: row.keys?.[0] ?? "",
    clicks: row.clicks ?? 0,
    impressions: row.impressions ?? 0,
    ctr: row.ctr ?? 0,
    position: row.position ?? 0,
  }))
}
```

認証はサービスアカウントでも OAuth でも良いですが、**CI から回すならサービスアカウント**が楽です。Search Console 側でそのサービスアカウントのメールアドレスをプロパティに追加しておく必要がある点だけ注意してください。

## 2. 前週比の差分を計算する

取得した 2 週分のデータを突き合わせて、**悪化したページだけ**を抽出します。

```typescript
type PageDiff = {
  readonly page: string
  readonly clicksChange: number
  readonly positionChange: number
  readonly severity: "critical" | "warning" | "info"
}

function calcDiff(
  current: readonly SearchAnalyticsRow[],
  previous: readonly SearchAnalyticsRow[]
): readonly PageDiff[] {
  const prevMap = new Map(previous.map((r) => [r.page, r]))

  return current
    .map((curr): PageDiff | null => {
      const prev = prevMap.get(curr.page)
      if (!prev) return null

      const clicksChange = curr.clicks - prev.clicks
      const positionChange = curr.position - prev.position

      const severity = judgeSeverity(prev, curr)
      if (severity === null) return null

      return { page: curr.page, clicksChange, positionChange, severity }
    })
    .filter((d): d is PageDiff => d !== null)
}

function judgeSeverity(
  prev: SearchAnalyticsRow,
  curr: SearchAnalyticsRow
): PageDiff["severity"] | null {
  const clicksDropRate = (curr.clicks - prev.clicks) / Math.max(prev.clicks, 1)
  const positionDrop = curr.position - prev.position

  if (prev.clicks >= 10 && clicksDropRate <= -0.5) return "critical"
  if (positionDrop >= 5 && prev.position <= 20) return "warning"
  if (clicksDropRate <= -0.2 && prev.clicks >= 5) return "info"

  return null
}
```

ここで大事なのは、**しきい値を「絶対値」と「割合」の両方で持つ**ことです。

- クリック数が 100 → 80 は**割合では -20% だが、絶対値では影響大**
- クリック数が 2 → 1 は**割合では -50% だが、ノイズ**

このため `prev.clicks >= 10` のような**最小ベースライン**を入れないと、弱小ページのノイズで通知が埋まります。

## 3. Lighthouse で重要ページだけ詳細計測

Search Console のデータは「検索経由の結果」なので、原因の切り分けには Lighthouse の技術スコアが必要です。ただし、**全ページを毎週 Lighthouse で回すのは過剰**なので、以下の方針にします。

- 全ページは Search Console で俯瞰
- **異常が出たページだけ** Lighthouse で深掘り
- トップページ・主要ランディングページだけは毎週固定で計測

実装は `lighthouse` パッケージを直接呼ぶのがシンプルです。

```typescript
import lighthouse from "lighthouse"
import * as chromeLauncher from "chrome-launcher"

type LighthouseScores = {
  readonly performance: number
  readonly seo: number
  readonly accessibility: number
  readonly bestPractices: number
}

async function runLighthouse(url: string): Promise<LighthouseScores> {
  const chrome = await chromeLauncher.launch({ chromeFlags: ["--headless"] })

  try {
    const result = await lighthouse(url, {
      port: chrome.port,
      output: "json",
      onlyCategories: ["performance", "seo", "accessibility", "best-practices"],
    })

    if (!result) throw new Error("Lighthouse returned no result")

    const categories = result.lhr.categories
    return {
      performance: Math.round((categories.performance?.score ?? 0) * 100),
      seo: Math.round((categories.seo?.score ?? 0) * 100),
      accessibility: Math.round((categories.accessibility?.score ?? 0) * 100),
      bestPractices: Math.round((categories["best-practices"]?.score ?? 0) * 100),
    }
  } finally {
    await chrome.kill()
  }
}
```

## 4. GitHub Actions で週次実行

実行基盤は GitHub Actions のスケジュールワークフローが手軽です。

```yaml
name: SEO Audit Weekly

on:
  schedule:
    - cron: "0 0 * * 1"  # 毎週月曜 0:00 UTC
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - name: Run audit
        env:
          GSC_SITE_URL: ${{ secrets.GSC_SITE_URL }}
          GOOGLE_APPLICATION_CREDENTIALS_JSON: ${{ secrets.GOOGLE_APPLICATION_CREDENTIALS_JSON }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: npm run audit
```

サービスアカウントの JSON は、環境変数で渡して実行時にファイル化する形で問題ありません。

## 5. 運用でハマった 4 つのポイント

### ハマり 1: Search Console の 2 日ディレイ

Search Console のデータは**直近 2 日分が未確定**です。昨日分を見に行くと 0 件が返ってくることがあります。**3 日前基準で取得する**ようにしました。

### ハマり 2: Lighthouse のスコアブレ

Lighthouse の Performance スコアは**同じ URL でも毎回 ±5 程度ブレ**ます。1 回だけ計測して前週比を出すと、ノイズで誤通知が増えます。**3 回計測して中央値を使う**のが安定します。

### ハマり 3: API クォータ

Search Console API は 1 日あたりのクエリ数に上限があります。大量ページを扱うサイトでは、**ページ単位の dimensions を 1 回、クエリ単位を 1 回**、のように呼び出しを分けて通る量に調整します。

### ハマり 4: 「悪化」だけ見ていると機会損失に気づかない

これは運用して気づいた点ですが、悪化アラートだけだと、**伸びているページに気づかず放置**します。「前週比で順位が 10 位以上改善したページ」も通知に含めると、そのページに内部リンクを集中させる判断ができるようになります。

## 使い分けの整理

| レイヤー | ツール | 役割 |
| --- | --- | --- |
| 全体俯瞰 | Search Console API | 全ページの週次トレンド |
| 詳細診断 | Lighthouse | 異常ページの技術スコア |
| 差分抽出 | 自作スクリプト | ノイズを除いた通知 |
| 全体監査 | [SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-audit-automation-gsc-lighthouse) | サイト単位の 45 項目一括診断 |

日々の定点観測は自作パイプラインで回しつつ、**「サイト全体として SEO 的に抜けがないか」の棚卸しは SEO CHECK にまとめて任せる**、という使い分けが今のところ最も運用負荷が軽いです。

## まとめ

1. 手動 SEO 監査は続かない。**差分通知ベースで自動化**する
2. Search Console API で全体を俯瞰し、Lighthouse で異常ページだけ深掘り
3. しきい値は「絶対値 × 割合」の両方で持つ
4. 悪化だけでなく**改善も通知**すると、機会損失を拾える
5. 全体の棚卸しは月 1 回、別ツールで一括チェックする

週次で回し始めると、**「気づいたら順位が落ちていた」が消えます**。SEO は先回りできた分だけ勝てる領域なので、監視の自動化は費用対効果がかなり高い投資です。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-audit-automation-gsc-lighthouse) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=seo-audit-automation-gsc-lighthouse) — ディレクターも使っているSEO診断ツールを公開中。
