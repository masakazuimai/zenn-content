---
title: "Next.js + @react-pdf/renderer で日本語PDFレポートを生成する"
emoji: "📄"
type: "tech"
topics: ["nextjs", "react", "typescript", "pdf"]
published: true
---

## はじめに

Next.jsで「診断結果をPDFでダウンロードしたい」という要件が出たとき、まず候補に上がるのが **jsPDF** と **@react-pdf/renderer** の2つです。

### jsPDF と @react-pdf/renderer の違い

| | jsPDF | @react-pdf/renderer |
|---|---|---|
| レイアウト | 座標を直接指定 | FlexboxベースのReactコンポーネント |
| 日本語対応 | `addFont` でBase64変換が必要 | `Font.register` でパス指定できる |
| TypeScript | 型定義が薄め | 充実している |
| 複雑なレイアウト | つらい | コンポーネントに分割できる |
| ファイルサイズ | 小さめ | やや大きめ |

jsPDFを最初に使ってみたのですが、「表紙・サマリー・テーブル・チャートを組み合わせた複数ページ構成」になると、`y` 座標を手動で管理し続けるのが厳しくなります。また日本語フォントを埋め込むには、ttfをArrayBufferとして読み込んでBase64に変換する処理が必要で、その管理もひと手間です。

一方、@react-pdf/renderer は **Reactコンポーネントとして書けること**、**`Font.register` がシンプル**なこと、**Flexboxでレイアウトできる**ことが決め手になりました。ただし、SVGまわりでかなりハマったので、その記録も含めてまとめます。

---

## セットアップ

### インストール

```bash
npm install @react-pdf/renderer
# または
pnpm add @react-pdf/renderer
```

### フォントファイルを配置する

日本語フォント（NotoSansJP）を `public/fonts/` に置きます。

```
public/
  fonts/
    NotoSansJP-Medium.ttf
```

[Google Fonts](https://fonts.google.com/noto/specimen/Noto+Sans+JP) からttfをダウンロードして配置するか、`@fontsource/noto-sans-jp` のnpmパッケージからコピーしてもOKです。

### register-fonts.ts を作る

フォント登録は複数回呼ばれると無駄なので、登録済みフラグで制御します。

```typescript
// lib/pdf/register-fonts.ts
import { Font } from "@react-pdf/renderer"
import path from "path"

let registered = false

export function registerFonts() {
  if (registered) return
  Font.register({
    family: "NotoSansJP",
    src: path.join(process.cwd(), "public/fonts/NotoSansJP-Medium.ttf"),
  })
  registered = true
}
```

`path.join(process.cwd(), ...)` を使うのがポイントです。`public/fonts/...` の相対パスだと、サーバーサイドで動かしたときにパスが解決できずエラーになります。

---

## 基本的なPDFコンポーネント

@react-pdf/renderer はDOMではなく独自のコンポーネント体系を持ちます。主なものは下記の通りです。

| コンポーネント | 役割 |
|---|---|
| `Document` | PDF全体のルート |
| `Page` | 1ページ分。`size="A4"` でA4 |
| `View` | divのような箱。Flexboxで配置 |
| `Text` | テキスト。fontFamily必須 |
| `StyleSheet.create` | スタイル定義 |

```tsx
import { Document, Page, View, Text, StyleSheet } from "@react-pdf/renderer"

const styles = StyleSheet.create({
  page: {
    fontFamily: "NotoSansJP",  // ← フォント指定必須
    fontSize: 10,
    padding: 40,
    backgroundColor: "#FFFFFF",
  },
  heading: {
    fontSize: 16,
    fontWeight: "bold",
    marginBottom: 12,
  },
})

export function SampleDocument() {
  return (
    <Document title="サンプルPDF">
      <Page size="A4" style={styles.page}>
        <View>
          <Text style={styles.heading}>SEO診断レポート</Text>
          <Text>診断結果を日本語で表示できます。</Text>
        </View>
      </Page>
    </Document>
  )
}
```

`fontFamily: "NotoSansJP"` をページレベルで指定しておくと、子要素全体に継承されます。

---

## SVGチャート描画でハマったこと

ここが一番の本題です。スコアをビジュアルに表示するために、SVGで円形のプログレスチャートを実装しようとして、2つの大きな壁にぶつかりました。

### ハマりポイント1: SVG `<Text>` で `textAnchor` や `fontSize` がtype errorになる

@react-pdf/renderer には `Svg` `Circle` `Path` `Text` 等のSVGコンポーネントが用意されています。ただし、SVG内の `<Text>` は通常のHTMLと同名なので、importが競合します。

```tsx
// こうすると "Text" が通常のText（PDF用）と競合する
import { Text } from "@react-pdf/renderer"
import { Svg, Text } from "@react-pdf/renderer"  // エラー
```

`as` でaliasを付けて回避できますが…

```tsx
import { Text as SvgText } from "@react-pdf/renderer"
```

今度は `textAnchor` や `fontSize` をpropsに渡すと**型エラー**になります。SVGのText要素として書いているのに、これらのSVG標準属性が型定義に含まれていないのです。

**解決策: SVGの上にView/Textを `position: "absolute"` で重ねる**

SVGには図形だけを描いて、テキストの配置はReactPDFのコンポーネント側で行う、という分離が一番スッキリします。

```tsx
import { View, Text, Svg, Circle } from "@react-pdf/renderer"

export function ScoreCircle({ score, maxScore, size = 100 }: Props) {
  return (
    // position: "relative" の親コンテナ
    <View style={{ width: size, height: size, alignItems: "center", justifyContent: "center" }}>
      {/* SVGには図形だけ */}
      <Svg width={size} height={size} style={{ position: "absolute" }}>
        <Circle cx={size / 2} cy={size / 2} r={size / 2 - 8}
          stroke="#E2E8F0" strokeWidth={6} fill="none" />
      </Svg>
      {/* テキストはPDFのTextコンポーネントで絶対配置 */}
      <Text style={{ fontSize: size * 0.28, fontWeight: "bold" }}>{score}</Text>
      <Text style={{ fontSize: size * 0.12, color: "#64748B" }}>/ {maxScore}</Text>
    </View>
  )
}
```

`<Svg style={{ position: "absolute" }}>` として親Viewの後ろに置き、その上にTextを積むことで、SVGの型エラーを避けながら日本語テキストも正しく表示できます。

---

### ハマりポイント2: `Circle` で `strokeDashoffset` が使えない

進捗インジケーターを実装するとき、通常のSVGでは `strokeDasharray` と `strokeDashoffset` の組み合わせで円弧を描きます。

```tsx
// ブラウザSVGではこれでOK
<circle
  strokeDasharray={`${circumference}`}
  strokeDashoffset={`${circumference * (1 - pct)}`}
/>
```

しかし @react-pdf/renderer の `<Circle>` は `strokeDashoffset` がTypeエラーになります（型定義に存在しない）。

**解決策: `Path` + `describeArc` 関数で円弧を直接描く**

三角関数で始点と終点の座標を計算して、SVGのArcコマンドでパスを生成します。

```typescript
function describeArc(
  cx: number,
  cy: number,
  r: number,
  startAngle: number,
  endAngle: number
): string {
  const x1 = cx + r * Math.cos(startAngle)
  const y1 = cy + r * Math.sin(startAngle)
  const x2 = cx + r * Math.cos(endAngle)
  const y2 = cy + r * Math.sin(endAngle)
  const largeArc = Math.abs(endAngle - startAngle) > Math.PI ? 1 : 0
  // M: 始点へ移動、A: 楕円弧
  return `M ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 1 ${x2} ${y2}`
}
```

使い方はこうなります。

```tsx
import { View, Text, Svg, Circle, Path } from "@react-pdf/renderer"
import { BRAND } from "../styles"

export function ScoreCircle({ score, maxScore, size = 100 }: Props) {
  const cx = size / 2
  const cy = size / 2
  const r = size / 2 - 8
  const pct = maxScore > 0 ? score / maxScore : 0
  const color = pct >= 0.8 ? BRAND.good : pct >= 0.6 ? BRAND.warning : BRAND.error

  const startAngle = -Math.PI / 2  // 12時の位置からスタート
  const endAngle = startAngle + 2 * Math.PI * pct

  return (
    <View style={{ width: size, height: size, alignItems: "center", justifyContent: "center" }}>
      <Svg width={size} height={size} viewBox={`0 0 ${size} ${size}`} style={{ position: "absolute" }}>
        {/* 背景の灰色リング */}
        <Circle cx={cx} cy={cy} r={r} stroke="#E2E8F0" strokeWidth={6} fill="none" />
        {/* スコアに応じた円弧（Pathで描画） */}
        {pct > 0 && (
          <Path
            d={describeArc(cx, cy, r, startAngle, endAngle)}
            stroke={color}
            strokeWidth={6}
            fill="none"
            strokeLinecap="round"
          />
        )}
      </Svg>
      <Text style={{ fontSize: size * 0.28, fontWeight: "bold", color: BRAND.dark }}>{score}</Text>
      <Text style={{ fontSize: size * 0.12, color: BRAND.textLight }}>/ {maxScore}</Text>
    </View>
  )
}
```

ゲージチャート（半円）の場合は、`startAngle` を `Math.PI`、`endAngle` を `0` にすれば下半分をくり抜いた半円になります。

```typescript
// ゲージチャート用のdescribeArc（y軸が反転するため符号を変える）
function describeArc(cx, cy, r, startAngle, endAngle): string {
  const x1 = cx + r * Math.cos(startAngle)
  const y1 = cy - r * Math.sin(startAngle)  // ← y軸の向きに注意
  const x2 = cx + r * Math.cos(endAngle)
  const y2 = cy - r * Math.sin(endAngle)
  const largeArc = endAngle - startAngle > Math.PI ? 1 : 0
  return `M ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 0 ${x2} ${y2}`
}

// 使用例
const bgPath = describeArc(cx, cy, r, Math.PI, 0)      // 背景の半円
const valuePath = describeArc(cx, cy, r, sweepAngle, Math.PI)  // 値の半円
```

`describeArc` の向きはSVGの座標系に依存するため、全円と半円でy軸の符号が変わる点に注意してください。

---

## API Route経由でPDFを配信する

Next.js App RouterのAPI Routeからサーバーサイドでバイナリを返す実装です。

```typescript
// app/api/generate-pdf/route.ts
import { NextResponse } from "next/server"
import { renderToBuffer } from "@react-pdf/renderer"
import { registerFonts } from "@/lib/pdf/register-fonts"
import { ProposalDocument } from "@/lib/pdf/proposal-document"

export async function POST(request: Request) {
  try {
    const body = await request.json()

    // フォント登録（サーバー起動後の初回のみ実行される）
    registerFonts()

    // ReactコンポーネントをBufferに変換
    const buffer = await renderToBuffer(
      ProposalDocument({ results: body.results, date: body.date })
    )

    // Buffer → Uint8Array への変換が必要
    const uint8 = new Uint8Array(buffer)

    return new NextResponse(uint8, {
      headers: {
        "Content-Type": "application/pdf",
        "Content-Disposition": `attachment; filename="report.pdf"`,
      },
    })
  } catch (error) {
    console.error("PDF generation failed:", error)
    return NextResponse.json({ error: "PDF generation failed" }, { status: 500 })
  }
}
```

`renderToBuffer` の戻り値は `Buffer` ですが、`NextResponse` に渡すには `Uint8Array` に変換する必要があります。`new Uint8Array(buffer)` の一行が抜けると、バイナリが正しく送信されず壊れたPDFが返ってくるのでご注意ください。

クライアント側のダウンロード処理はこうなります。

```typescript
async function downloadPDF(data: ReportData) {
  const response = await fetch("/api/generate-pdf", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  })

  if (!response.ok) throw new Error("PDF生成に失敗しました")

  const blob = await response.blob()
  const url = URL.createObjectURL(blob)
  const a = document.createElement("a")
  a.href = url
  a.download = "seo-report.pdf"
  a.click()
  URL.revokeObjectURL(url)
}
```

---

## 実際の出力例

今回の実装では、以下の5ページ構成のPDFが生成されます。

1. **表紙ページ**: ロゴ・タイトル・URL・診断日時
2. **エグゼクティブサマリー**: 総合スコア（円形チャート）、カテゴリ別スコアカード
3. **チェック項目一覧**: 全診断項目の結果テーブル（good/warning/badのステータス付き）
4. **改善提案**: badとwarningの項目を優先度付きでリストアップ
5. **Core Web Vitals**: LCP・CLS・INPのゲージチャート、モバイル/デスクトップ比較

各ページはコンポーネントに分割されているため、ページを増減させたいときは `<Document>` に子コンポーネントを追加・削除するだけで対応できます。これが @react-pdf/renderer を選んだ最大の恩恵です。

---

## まとめ

### @react-pdf/rendererのメリット

- **コンポーネントで分割できる**: 複数ページ構成でも管理しやすい
- **FlexboxでレイアウトできるのでCSSの知識が活きる**
- **日本語フォント登録が `Font.register` 1行で済む**
- **TypeScriptの型が充実している**

### 注意点

- SVGの `<Text>` は `textAnchor`/`fontSize` 等がtype errorになる → View/Textオーバーレイで解決
- `strokeDashoffset` は型定義にない → `Path` + `describeArc` で円弧を自前で描く
- `renderToBuffer` の戻り値は `Buffer`。`NextResponse` に渡す前に `Uint8Array` 変換が必要
- `Font.register` はサーバーサイドのみ動作。`process.cwd()` で絶対パスを指定する

SVGの制約はやや面倒ですが、慣れると三角関数でパスを計算する方が自由度は高いです。

---

実際にこの実装を使っているのが、SEO診断ツールの [CodeQuest.work SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=react-pdf-renderer-japanese-pdf) です。診断結果をPDFレポートとして書き出す機能（有料プラン）で、この記事で紹介したコンポーネント構成がそのまま動いています。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=react-pdf-renderer-japanese-pdf) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=react-pdf-renderer-japanese-pdf) — ディレクターも使っているSEO診断ツールを公開中。
