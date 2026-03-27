---
title: "llms.txtでAI検索に備える — 存在チェックと検証コードをTypeScriptで書く"
emoji: "🤖"
type: "tech"
topics: ["SEO", "AI", "ChatGPT", "llmstxt", "TypeScript"]
published: true
---

## AI検索の時代、サイトが「見つからない」問題

ChatGPT、Perplexity、Gemini — AI検索を使うユーザーが急増しています。

ところが、従来のSEO対策だけでは**AIにサイトの内容を正しく伝えられない**ケースが出てきました。

- Google検索では1ページ目に出るのに、ChatGPTの回答に自社サイトが引用されない
- Perplexityで業界名を聞くと、競合は出るのに自社が出ない
- Geminiに聞いても、古い情報や不正確な情報で紹介される

原因の一つが、**AIクローラーがサイトの構造を効率的に把握できていない**こと。人間向けのHTML構造だけでは、AIが「このサイトの主要ページはどれか」「何の会社か」を判断しにくいのです。

## llms.txtとは何か

**llms.txt**は、AIクローラー向けにサイトの概要と主要ページを伝えるためのMarkdownファイルです。`robots.txt`のAI版と考えるとわかりやすいでしょう。

```
/llms.txt          ← サイトルートに配置
/.well-known/llms.txt  ← こちらに置く場合もある
```

中身はシンプルなMarkdown形式です。

```markdown
# サイト名

> サイトの概要説明（1〜2文）

## 主要ページ

- [サービス紹介](https://example.com/service): サービスの詳細
- [料金プラン](https://example.com/pricing): 料金体系
- [ブログ](https://example.com/blog): 技術ブログ
```

AIクローラーはこのファイルを読むことで、**サイト全体を効率的に理解**できます。全ページをクロールする必要がなくなるため、正確な情報が引用されやすくなります。

## 自分のサイトにllms.txtがあるか検出する

まず、あなたのサイト（または競合サイト）にllms.txtが設置されているかを確認するコードです。

```typescript
// llms.txtの存在チェック
async function detectLlmsTxt(domain: string) {
  const paths = [
    `https://${domain}/llms.txt`,
    `https://${domain}/.well-known/llms.txt`,
  ]

  const results = await Promise.all(
    paths.map(async (url) => {
      try {
        const res = await fetch(url, { redirect: "follow" })
        if (!res.ok) return { url, exists: false }

        const text = await res.text()
        // HTMLが返ってきた場合は404ページの可能性
        if (text.trimStart().startsWith("<!") || text.trimStart().startsWith("<html")) {
          return { url, exists: false }
        }

        return { url, exists: true, content: text }
      } catch {
        return { url, exists: false }
      }
    })
  )

  return results.filter((r) => r.exists)
}
```

ポイントは**HTMLが返ってきた場合を除外する**処理です。多くのサイトでは存在しないパスにアクセスするとステータス200でカスタム404ページを返します。`text()`の先頭が`<!`や`<html`で始まる場合はllms.txtではないと判定しています。

## 取得したllms.txtの構造を検証する

ファイルが存在しても、中身が不十分では意味がありません。最低限チェックすべき項目を検証するコードです。

```typescript
interface LlmsTxtValidation {
  hasH1Title: boolean
  hasDescription: boolean
  markdownLinks: { text: string; url: string }[]
  linkCount: number
}

function validateLlmsTxt(content: string): LlmsTxtValidation {
  const lines = content.split("\n")

  // H1タイトルがあるか（# で始まる行）
  const hasH1Title = lines.some((line) => /^# .+/.test(line))

  // 説明文があるか（> で始まる引用行）
  const hasDescription = lines.some((line) => /^> .+/.test(line))

  // Markdownリンクの抽出
  const linkPattern = /\[([^\]]+)\]\(([^)]+)\)/g
  const markdownLinks: { text: string; url: string }[] = []

  for (const line of lines) {
    let match
    while ((match = linkPattern.exec(line)) !== null) {
      markdownLinks.push({ text: match[1], url: match[2] })
    }
  }

  return {
    hasH1Title,
    hasDescription,
    markdownLinks,
    linkCount: markdownLinks.length,
  }
}
```

これで以下がわかります。

| チェック項目 | 重要度 | 理由 |
|-------------|--------|------|
| H1タイトル | 必須 | AIがサイト名を認識するため |
| 説明文（引用） | 推奨 | サイトの概要をAIに伝える |
| Markdownリンク | 必須 | 主要ページへの導線 |
| リンク数 | 参考 | 少なすぎるとカバー不足、多すぎると優先度が不明確 |

## 対応状況 — まだほとんどのサイトが未設置

上のコードで実際にいくつかのサイトをチェックしてみると、差が見えてきます。

2025年3月時点では、海外の大手テック企業（Anthropic、Cloudflare等）は設置済みですが、**日本のサイトで対応しているところはごく少数**です。

つまり今設置すれば、AI検索での露出において競合より先行できます。自分のサイトと競合を上のスクリプトで比較してみてください。

## llms.txtを設置しないとどうなるか

設置しなくてもAIクローラーはサイトをクロールします。ただし以下のリスクがあります。

- **不正確な引用**: AIがサイトの古いページや重要でないページを参照する
- **情報の欠落**: 主要サービスが言及されない
- **競合に負ける**: 競合がllms.txtを設置している場合、AIの回答で優先される可能性

特に**ブランド名で検索されたとき**に影響が大きい。「〇〇とは？」とAIに聞かれて、競合の情報が先に出るのは避けたいところです。

## 設置のハードル

llms.txtの仕様自体はシンプルですが、実際に書こうとすると意外と手が止まります。

- サイトのどのページを載せるべきか？
- 説明文はどう書けばAIに伝わるか？
- Markdownリンクの形式は正しいか？
- `.well-known/`パスにも置くべきか？

手動で書くと「とりあえず主要ページを3つ並べただけ」になりがちで、十分な効果を発揮できません。

サイトの構造を分析して必要な情報を網羅したllms.txtを作成するには、**サイト全体の診断結果をもとに自動生成する**のが確実です。筆者が開発した[SEO CHECK](https://seo.codequest.work/ja/llms-txt-check)では、URLを入力するだけでllms.txtのチェックと推奨テンプレートの自動生成ができます。

## GEO・AIO・LLMO — 知っておくべき新しいSEO用語

最後に、llms.txt周辺で出てくる用語を整理しておきます。

| 用語 | 正式名 | 意味 |
|------|--------|------|
| GEO | Generative Engine Optimization | 生成AI検索への最適化 |
| AIO | AI Overview Optimization | GoogleのAI概要への最適化 |
| AEO | Answer Engine Optimization | 回答エンジンへの最適化 |
| LLMO | Large Language Model Optimization | LLM向けの最適化 |

呼び方は違いますが、本質は同じです。**「AIがあなたのサイトを正しく理解し、引用してくれるか」**。

llms.txtは、これらすべてに効く最もシンプルな施策です。まだ対応していないなら、今が先行者利益を取れるタイミングです。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — llms.txtのチェック・自動生成にも対応したSEO診断ツールを公開中。
