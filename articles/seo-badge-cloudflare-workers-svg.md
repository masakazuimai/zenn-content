---
title: "SEOスコアバッジをCloudflare Workers + SVG動的生成で作った話"
emoji: "🏅"
type: "tech"
topics: ["CloudflareWorkers", "Hono", "SVG", "SEO", "NextJS"]
published: true
---

## はじめに --- 「SEOスコアを証明できないか？」

SEO診断ツールを開発していて、ユーザーから「スコアをサイト上で見せたい」という声をもらいました。

Web制作者なら分かると思いますが、自分が手がけたサイトのSEO品質を**第三者ツールのスコアで証明できる**のは営業上かなり強い。Google PageSpeed Insightsのスコアをスクショで貼る人は多いですが、リアルタイムに検証可能なバッジとなると話は別です。

そこで作ったのが、**85点以上を達成したサイトに埋め込めるSEOバッジ**です。reCAPTCHAのようにサイト右下にフローティング表示され、ホバーでスコアが展開、クリックで検証ページに飛べます。

この記事では、追加コスト0円で実現した技術的な仕組みを解説します。

## 完成イメージ

サイト右下に青いアイコンが常駐し、ホバーすると右にスライド展開して「SEO_CHECK 92点」のように表示されます。reCAPTCHAバッジを見慣れている人なら、すぐにイメージできると思います。

- **通常時**: 青いグラデーションの正方形アイコン（「SEO」の文字入り）
- **ホバー時**: 右にスライドして白背景のテキストエリアが展開
- **クリック時**: 検証ページに遷移（ドメイン・スコア・最終チェック日を表示）

## アーキテクチャ

```
[ユーザーのサイト]
  └─ 埋め込みHTML（インラインCSS付き）
       ├─ SVGバッジ画像 ← Cloudflare Workers で動的生成
       └─ 検証リンク → /badge/verify/:token

[管理側]
  ├─ domain_badges テーブル（D1）
  │    ├─ badge_token (UUID)
  │    ├─ badge_enabled (0/1)
  │    ├─ domain
  │    └─ latest_score
  └─ ダッシュボード（ON/OFF切り替え）
```

ポイントは以下の3つです。

1. **外部サービス不要**: shields.ioやBadgen等を使わず、Workers上でSVGを直接生成
2. **サーバー側でON/OFF制御**: バッジを無効にすると1x1の透明SVGを返すだけ。埋め込みコードの貼り替え不要
3. **ドメイン毎にトークン管理**: 複数サイトを運営しているユーザーに対応

## Workers上でSVGを動的生成する

shields.ioスタイルのバッジをWorkers上で生成しています。HonoのルーティングでSVGを返すだけなので、非常にシンプルです。

```typescript
// イメージコード（実際の実装とは異なります）

/** スコアに応じた色を決定 */
function getColor(score: number): string {
  if (score >= 90) return "#4c1"
  if (score >= 60) return "#dfb317"
  return "#e05d44"
}

/** SVGバッジを文字列で生成 */
function generateBadge(label: string, score: number): string {
  const color = getColor(score)
  // テンプレートリテラルでSVGを組み立て
  return `<svg xmlns="http://www.w3.org/2000/svg" ...>
    <rect ... fill="#555"/>        <!-- ラベル背景 -->
    <rect ... fill="${color}"/>    <!-- スコア背景 -->
    <text>${label}</text>
    <text>${score}</text>
  </svg>`
}
```

要点は、SVGがXML文字列なのでテンプレートリテラルで組み立てるだけということ。画像処理ライブラリもCanvasも不要で、Workersの無料プランでも余裕で動きます。

## ON/OFFの仕組み --- 透明SVGで切り替える

ユーザーがダッシュボードからバッジを無効にしたとき、埋め込みコードを差し替えてもらう必要はありません。APIが返すSVGを切り替えるだけです。

```typescript
// イメージコード（実際の実装とは異なります）

app.get("/image/:token", async (c) => {
  const badge = await db.findByToken(token)

  // 無効 or スコアなし → 1x1透明SVG
  if (!badge || !badge.enabled) {
    return c.body(TRANSPARENT_SVG, 200, svgHeaders)
  }

  // 有効 → スコア入りSVGを返す
  return c.body(generateBadge("SEO Score", badge.score), 200, svgHeaders)
})
```

考え方はシンプルで、`enabled`が`false`なら1x1の透明SVGを返すだけ。埋め込みコードは同じURLのままなので、ユーザーに貼り替えてもらう必要はありません。

## フローティングバッジのフロントエンド実装

埋め込み先のサイトに貼るHTMLは、CSSをインラインで含めて依存ゼロにしています。ただし自サイト（Next.js）ではTailwind CSSで実装しています。

```tsx
// UIイメージ（実際の実装とは異なります）

<a className="group fixed bottom-24 right-6 flex items-center h-14
              transition-all duration-500">
  {/* アイコン（常時表示） */}
  <div className="w-14 h-14 rounded-xl group-hover:rounded-r-none
                  bg-gradient-to-br from-[#1a32d0] to-[#3730a3]
                  transition-all duration-500 ...">
    <span className="text-white font-bold">SEO</span>
  </div>
  {/* 展開テキスト（ホバーで出現） */}
  <div className="overflow-hidden w-0 group-hover:w-48
                  transition-all duration-500">
    <div className="bg-white border rounded-r-xl h-14 ...">
      <span>SEO_CHECK</span>
      <span className="text-blue-600 font-bold">{score}点</span>
    </div>
  </div>
</a>
```

ポイントは`group-hover`でコンテナのホバー状態をトリガーにし、`w-0`→`w-48`のwidth transitionでスライド展開すること。JSでのアニメーション制御は不要です。

## 検証ページでスコアの信頼性を担保する

バッジをクリックすると `/badge/verify/:token` に遷移し、以下の情報を表示します。

- 対象ドメイン
- SEOスコア
- 最終チェック日時

APIから返すJSONはシンプルです。

```typescript
// イメージコード（実際の実装とは異なります）

app.get("/verify/:token", async (c) => {
  const badge = await db.findByToken(token)

  if (!badge || !badge.enabled) {
    return c.json({ valid: false })
  }

  return c.json({
    valid: true,
    domain: badge.domain,
    score: badge.score,
    checkedAt: badge.checkedAt,
  })
})
```

検証ページがあることで、スクリーンショットの偽造やスコアの詐称を防げます。「本当にこのスコアなの？」という疑問にワンクリックで答えられるのがポイントです。

## なぜこの機能を作ったか（マーケティング的背景）

技術的な話だけでなく、ビジネス上の狙いも書いておきます。

**1. ユーザーの自発的な宣伝になる**

バッジを埋め込んでくれたサイトは、そのまま当ツールの広告塔になります。85点以上という条件があるため、「高品質なサイトが使っているツール」というブランディングにもなります。

**2. 85点の壁がモチベーションになる**

「あと3点でバッジがもらえる」という状況は、追加の改善アクションを促します。結果としてツールの利用頻度が上がり、有料プランへの転換率も上がります。

**3. 追加コストがほぼゼロ**

SVGをテンプレートリテラルで生成するだけなので、画像処理サービスやCDNの追加コストは発生しません。D1のクエリもトークンでの単純な主キー検索なので、負荷も最小限です。

## 運用コストまとめ

| 項目 | コスト |
|------|--------|
| SVG生成 | Cloudflare Workers無料枠内 |
| データ管理 | Cloudflare D1無料枠内 |
| CDN/キャッシュ | Cloudflare標準機能 |
| 外部API | なし |
| **合計** | **0円** |

## まとめ

- Cloudflare Workers上でSVGをテンプレートリテラルで動的生成すれば、外部サービスなしでバッジ機能が作れる
- ON/OFFは「スコア入りSVG vs 1x1透明SVG」の切り替えだけ。コードの貼り替え不要
- 検証ページでスコアの信頼性を担保し、マーケティングツールとしても機能させる
- 追加コスト0円で、ユーザーの自発的な宣伝を促す仕組みになる

Cloudflare Workers + D1の組み合わせは、この手の「軽いけど動的な機能」にぴったりです。同じようなバッジ機能を作りたい方の参考になれば幸いです。

---

この記事で紹介したバッジ機能は [CodeQuest.work SEO_CHECK](https://seo.codequest.work) で実際に使えます。無料プランでもSEO診断は利用可能なので、まずは自分のサイトをチェックしてみてください。
