---
title: "curl -A GPTBot で自分のSPAを見たら body が空だった — AIクローラーにCSRサイトは見えているのか"
emoji: "🕷️"
type: "tech"
topics: ["SEO", "AI", "Nextjs", "frontend", "curl"]
published: true
---

## 結論から

- AIクローラー（GPTBot / ClaudeBot / PerplexityBot）が **JavaScript を実行するかどうかは、どの公式ドキュメントにも書かれていない**
- 一方 Googlebot / Bingbot / Applebot は「JS実行のために Chromium / Edge / WebKit を使う」と**公式に明記している**
- つまり **CSR（クライアントサイドレンダリング）で本文を描画する SPA は、AIクローラーから本文が見えていない可能性がある**
- これは `curl` 一本で自分のサイトでも確かめられる

「SPA は SEO に不利」という話は2015年頃からありますが、Googlebot は進化してこの問題をほぼ解決しました。では **AI検索のクローラーはどうなのか？** を、手を動かして確かめた記録です。

## きっかけ：HTML が約 28KB しかない LP に遭遇した

国内のあるノーコード SPA サービスで作られたランディングページの「サーバーが返す生の HTML」を取得してみたところ、こうなっていました。

- レスポンスは **わずか約 28KB**
- `<body>` の中は**ほぼ空**。本文・見出し・画像・内部リンクが一切ない
- コンテンツは全部 JavaScript 実行後にクライアント側で描画される、典型的な CSR

ブラウザで見ると普通のきれいな LP なのに、サーバーが最初に返す HTML には中身がない。これは珍しくない構造です。問題は「**この空の HTML を、クローラーはどう扱うのか**」です。

## 自分のサイトで確かめる（コピペで動く）

ブラウザの「ページのソースを表示」でもいいですが、**クローラーになりきって**取得するのがポイントです。User-Agent を偽装します。

```bash
# 1. 普通に取得して、bodyに中身があるか見る
curl -s https://example.com/ | grep -o '<body[^>]*>.*</body>' | wc -c

# 2. Googlebot のふりをして取得
curl -s -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
  https://example.com/ > googlebot.html

# 3. GPTBot のふりをして取得
curl -s -A "Mozilla/5.0 (compatible; GPTBot/1.1; +https://openai.com/gptbot)" \
  https://example.com/ > gptbot.html

# 4. 2つのレスポンスが同じか比較（差分が出なければ Dynamic Rendering なし）
diff googlebot.html gptbot.html && echo "完全一致：UAで出し分けていない"
```

CSR の SPA だと、`googlebot.html` も `gptbot.html` も**初期 HTML は同じで body はほぼ空**になります。つまり「クローラー向けに中身を返す（Dynamic Rendering）」もしていない、という状態が可視化できます。

> 補足：`curl` が見るのは「JS 実行**前**の HTML」です。ここに本文があるか＝「JS を実行しないクローラーに本文が届くか」とほぼ同義になります。

## SEOクローラーはもう JS を実行する（公式明記あり）

ここが重要なところ。主要 SEO クローラーは、過去5年で「JS を実行する」と**公式に宣言**しています。

| クローラー | JS実行 | 公式の根拠 |
|---|---|---|
| Googlebot | ✅ | "Google Search runs JavaScript with an evergreen version of Chromium"（2019/05〜） |
| Bingbot | ✅ | Microsoft Edge ベースのレンダリングを採用と公式ブログで announce |
| Applebot | ✅ | "Applebot may render the content of your website within a browser"（About Applebot） |

Googlebot は「fetch してから、リソースに余裕ができた段階でヘッドレス Chromium がレンダリングする」**2段階インデックス**方式。だから今や CSR の SPA でも、最終的には本文をインデックスできます。「SPA は SEO に不利」は、Google に関しては概ね過去の話です。

## AIクローラーは「書いていない」

では AI検索が使うクローラーはどうか。公式ドキュメントを精査した結果がこれです。

| クローラー | JS実行の公式明記 |
|---|---|
| GPTBot | ❌ 記載なし |
| ClaudeBot | ❌ 記載なし |
| PerplexityBot | ❌ 記載なし |

「実行する」とも「実行しない」とも書かれていない。**ただ単に書かれていない**、というのが現状です。

ここは事実と推測を分けて読む必要があります。

**事実：**
- GPTBot / ClaudeBot / PerplexityBot は、いずれもレンダリングエンジンを明記していない
- User-Agent 文字列に `Chrome/W.X.Y.Z` のようなブラウザバージョン表記もない
- 対して Googlebot / Bingbot は2019年に「JS実行のため Chromium/Edge を採用した」と公式アナウンス済み

**推測（断定しない）：**
- 明記がない以上「JS を実行しない」とは**結論できない**
- だが「実行する」と仮定するのも**根拠がない**

SEO クローラー側が揃って「Chromium / Edge / WebKit で JS を実行する」と明示しているのに、AI クローラーだけが沈黙している。これを偶然と読むか、構造的な選択と読むか。少なくとも「**CSR の本文が AI に届く保証はない**」とは言えます。

## 実装する側の話：だから初期 HTML に意味を持たせる

ここからは作り手の視点です。AI クローラーが JS を実行するか不明な以上、安全側に倒すなら答えはシンプルで、

> **本文・見出し・構造化データを「JS 実行前の HTML」に載せておく**

これだけです。具体的には、

- CSR ではなく **SSR / SSG**（または ISR）で初期 HTML に本文を含める
- `<head>` の `title` / `meta` / **JSON-LD** はサーバー側で出力する（クライアント描画にしない）
- ファーストペイントの HTML に H1・本文・内部リンクが入っているかを、上の `curl` で定期チェックする

ちなみに、自分が作っている SEO 診断ツールはこの問題に正面から関わっていて、**取得した初期 HTML を解析する**設計（Cloudflare Workers 上で動く＝ブラウザのように JS を実行しない）にしています。なので「ブラウザで見れば中身があるのに、診断だと body 長や H1 がゼロ点」という結果が出ることがある。最初はバグかと思いましたが、これは**クローラーから見た現実に近い**わけです。作ってみて、この乖離を可視化できるのは便利でした。

## まとめ

- AI クローラーの JS 実行可否は**公式に不明**。CSR の SPA 本文は届かない可能性がある
- `curl -A` で、自分のサイトの「JS 実行前の HTML」に本文があるか今すぐ確認できる
- 迷ったら **初期 HTML に本文・JSON-LD を載せる**（SSR/SSG）。Google にも AI にも効く安全策

3社のクローラー公式ステートメント全文や、CSR サイトを SEO 診断にかけたときの挙動の詳細は、別途まとめた観測レポートに置いています。

→ [AIクローラーにSPAは見えているのか（観測レポート）](https://seo.codequest.work/ja/blog/spa-crawler-visibility-report)

自分のサイトの初期 HTML に本文が載っているか確認したい人は、[CodeQuest.work の SEO 診断](https://seo.codequest.work/ja/seo-check)（URL を入れるだけ・登録不要で3回無料）でも同じ観点でチェックできます。
