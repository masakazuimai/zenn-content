---
title: "いまさらながら diff チェッカーをバージョンアップした話"
emoji: "🔍"
type: "tech"
topics: ["javascript", "diff", "vanillajs", "githubactions", "初心者"]
published: true
---

## AI に diff させるの、トークンもったいなくない？

プログラミングを学んでいると、こんな場面によく出くわします。

「答えのコードと自分のコードを見比べたい。どこが違うんだろう？」

そのとき ChatGPT や Claude に「このコード比べて」と貼り付けていませんか？

気持ちはわかります。でも冷静に考えると、**diff は確定的な処理**です。どこが違うかは計算すれば一瞬でわかる。それをわざわざ LLM に推論させて、トークンを消費するのは明らかに過剰です。

しかも LLM は diff が不得意なケースもあります。長いコードを貼ると途中を省略したり、「ここが違いますね」と言いながら実は見落としていたり。

**差分確認は専用ツールに任せる。AI はもっと高度な判断に使う。**

そう思って、diff チェッカーを本格的にリニューアルしました。

https://codequest.work/generator/diff-checker/

---

## 初学者が差分確認で困ること

プログラミングを学んでいる方が「コードが動かない」と相談してくると、たいていの原因は細かいミスです。

- `border` が `boder` になっている（タイポ）
- インデントがズレている
- 全角スペースが混入している（IME の誤操作）
- セミコロンが抜けている

こういったミスは、答えのコードと自分のコードを並べて比べれば一瞬で発見できます。でも「どうやって比べるか」がわからない。

git の `diff` コマンドは初学者には難しい。VS Code の差分表示は同一ファイルの比較向け。AI に貼り付けるとトークンを消費する。

**そのギャップを埋めるツールが必要**でした。

---

## 何を改善したか

### ① 並列表示（サイドバイサイド）に変更

最大の変更点です。

もともとは unified diff（`+`・`-` 記号による表示）を採用していましたが、これは初学者には読みにくい。

```
- border-radius: 50%
+ border-radius: 5px
```

`+` が追加で `-` が削除、というルールを知らないと読めません。

今回は左右に並べて色で示す形に変えました。

| 左パネル（sample code） | 右パネル（my code） |
|---|---|
| border-radius: 50% | border-radius: 5px |

赤い行 → 元のコード。緑の行 → 自分のコード。説明しなくても伝わります。

### ② ハイブリッド diff で文字レベルのハイライト

行単位の差分だけだと、どの文字が違うかわかりません。

たとえば `border` と `boder` の違い。行ごと赤くしても、その行のどこが間違いか探す必要があります。

今回は3段階の diff を組み合わせました。

- 行レベル：`Diff.diffLines()` で変更行を検出
- 単語レベル：`Diff.diffWords()` で変更単語を特定
- 文字レベル：`Diff.diffChars()` で削除された文字だけを精密にマーク

結果：

```
左（sample code）：bor[d]er  ← d だけ赤
右（my code）    ：[boder]   ← 単語丸ごと緑
```

`border` の `d` が消えて `boder` になったことが一目でわかります。

### ③ 全角スペース検出

初学者あるある問題です。IME で日本語入力中に全角スペース（`　`）が混入するケース。

普通の diff ツールだと全角スペースは見えません。しかし混入したコードはエラーになります。

今回は Unicode のプライベート領域（`\uE000`）をセンチネル文字として使い、全角スペースを正確に検出・マーキングしました。

```javascript
const FW = "\uE000";

const normalizeCode = (text, ignoreWhitespace) => {
  let normalized = text
    .replace(/\r\n/g, "\n")
    .replace(/\r/g, "\n")
    .replace(/\u3000/g, FW); // 全角スペース → センチネル
  if (!ignoreWhitespace) return normalized;
  return normalized.split("\n").map((l) => l.trimEnd()).join("\n");
};
```

センチネルは空白ではないので `diffWords` がトークン境界として扱わず、全角スペースが正しい側（my code）にだけ □ マークで表示されます。

### ④ ファイルのドラッグ＆ドロップ

テキストエリアにファイルを直接ドロップするとその内容が読み込まれます。`FileReader API` を使ったシンプルな実装です。

### ⑤ コード品質の改善

1ファイル構成を `index.html`・`style.css`・`app.js` に分割。ペースト処理を非推奨の `execCommand` から Selection API に移行。Prettier を削除（整形による偽差分を排除）。

---

## GitHub Actions で FTP 自動デプロイ

```yaml
- name: FTP Deploy
  uses: SamKirkland/FTP-Deploy-Action@v4.3.4
  with:
    server: ${{ secrets.FTP_SERVER }}
    username: ${{ secrets.FTP_USERNAME }}
    password: ${{ secrets.FTP_PASSWORD }}
    local-dir: ./public/
    server-dir: ${{ secrets.FTP_SERVER_DIR }}
```

`main` への push で自動的に ConoHa WING へデプロイされます。Secrets に FTP 情報を登録するだけで動きます。

---

## SEO・AEO・LLMO 対応

せっかくリニューアルするので構造化データも整備しました。

- `WebApplication` スキーマ
- `FAQPage` スキーマ（強調スニペット対策）
- `HowTo` スキーマ（使い方ページ）
- `llms.txt`（ChatGPT・Perplexity・Claude への引用対策）

AI が引用するとき「このツールは何か」を正確に伝えるための整備です。

---

## 使ってみてください

**CodeDiff Checker**
https://codequest.work/generator/diff-checker/

- 登録不要・インストール不要
- ブラウザだけで動作
- 入力したコードは外部サーバーに送信されない

「答えのコードと自分のコードをどうやって比べればいいかわからない」という方に渡せるツールを目指しました。

diff に AI を使っていたトークン、もっと価値あることに使いましょう。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
