---
title: "変数は正しいのに分岐だけ違う — ExtendScriptの三項演算子が壊れていた"
emoji: "🐛"
type: "tech"
topics: ["javascript", "photoshop", "adobe", "debug", "automation"]
published: true
---

Photoshopをスクリプトから動かしてバナーを組んでいたときの話です。テキストを中央揃えにしたのに、3つとも右端に寄りました。

原因は自分のコードではなく、**処理系のバグ**でした。特定するまでに踏んだ手順と、最小再現コードを残します。

## 何をしていたか

macOSの `osascript` でPhotoshopにExtendScript（`.jsx`）を送り込み、1920×1080のデザインを組んでいました。テキストレイヤーを作って、指定に応じて左・中央・右へ配置する関数です。

```javascript
function addText(lname, content, fontPS, sizePt, colorHex, y, tracking, mode) {
  var t = doc.artLayers.add();
  t.kind = LayerKind.TEXT;
  var ti = t.textItem;
  ti.font = fontPS;
  ti.contents = content;
  ti.size = sizePt;
  ti.position = [60, y];

  var b = t.bounds, w = b[2].value - b[0].value;
  var x = (mode === "center") ? (W - w) / 2
        : (mode === "right")  ? (W - MARGIN - w)
        : MARGIN;

  t.translate(x - b[0].value, 0);
  return t;
}
```

`mode` に `"center"` を渡した3つのテキストが、すべて右揃えで配置されました。

## 第1段階：値を疑う

まず引数が正しく届いているかを見ます。ログを仕込みました。

```javascript
log.push("[debug] " + lname + " mode=" + mode + " typeof=" + (typeof mode));
```

出力はこうです。

```
[debug] cast mode=center typeof=string
cast: 幅=343 x=794-1137
```

`mode` は `center`、型も `string`。ここで「文字列オブジェクトになっていて `===` が false になるのでは」という仮説が立ちますが、`typeof` は `object` ではなく `string` を返しています。

## 第2段階：算術を疑う

次に、計算結果そのものを見ます。キャンバス幅 `W` は1200、テキスト幅 `w` は343でした。

中央揃えなら `(1200 - 343) / 2 = 428.5` になるはずです。ところが実際の左端は **794**。

794という数字がどこから来たのかを逆算します。

```
1200 - 63 - 343 = 794
```

`W - MARGIN - w`、つまり**右揃えの分岐の値**でした。3つとも同じで、logo（幅613）は `1200-63-613 = 524`、tagline（幅235）は `1200-63-235 = 902`。すべて右揃えの式にぴったり一致します。

つまり `mode === "center"` が false と評価され、`mode === "right"` が true と評価されている。値は `"center"` なのに、です。

## 第3段階：自分のコードを疑う

ここで「生成したファイルが壊れているのでは」と考えました。このスクリプトはPythonで組み立てて書き出していたので、置換ミスで別の文字列になっている可能性があります。

行をそのまま `repr()` で出力しました。

```python
for i, line in enumerate(open("script.jsx", encoding="utf-8"), 1):
    if "var x =" in line:
        print(i, repr(line))
```

```
89 '  var x = (mode === "center") ? (W - w)/2 : (mode === "right") ? (W - Math.round(108*S) - w) : Math.round(108*S);\n'
```

見えない文字も混入していません。コードは意図どおりでした。

ここで初めて、処理系を疑う段階に入ります。

## 第4段階：最小再現に落とす

デザインもPhotoshopのAPIも関係ない形まで削ります。書き方だけを変えた4つの関数を並べて、同じ入力を通しました。

```javascript
function a(m){ return (m === "center") ? "A" : (m === "right") ? "B" : "C"; }  // 素のネスト
function b(m){ return (m === "center") ? "A" : ((m === "right") ? "B" : "C"); } // 内側に括弧
function c(m){ return (m == "center")  ? "A" : (m == "right")  ? "B" : "C"; }   // == でネスト
function d(m){ return (m === "center") ? "A" : "X"; }                           // ネストなし

a("center") + a("right") + a("left");
b("center") + b("right") + b("left");
c("center") + c("right") + c("left");
d("center") + d("right");
```

結果です。

```
ネスト素   : BBC
括弧付き   : ABC
== でネスト: BBC
ネスト無し : AX
$.build=82.4 / version=4.5.12
```

`a("center")` が **"B"** を返しています。期待は `"A"` です。

## 何が壊れているのか

整理するとこうなります。

| 書き方 | 結果（期待は `ABC`） |
|---|---|
| `a ? "A" : b ? "B" : "C"`（括弧なしのネスト） | **`BBC`** ❌ |
| `a ? "A" : (b ? "B" : "C")`（内側に括弧） | `ABC` ✅ |
| `==` に変えて括弧なしネスト | **`BBC`** ❌ |
| ネストしない三項 | 正常 ✅ |

わかることが3つあります。

1. **`===` は壊れていない**。単体の比較 `m === "center"` は正しく `true` を返します。`==` でも同じ症状が出るので、比較演算子は無関係です
2. **壊れているのは、括弧を付けずにネストした条件演算子の評価**です。JavaScriptの条件演算子は右結合なので `a ? b : (c ? d : e)` と解釈されるべきですが、そうなっていません
3. **内側を括弧で囲めば正常に動きます**

筆者の環境は `$.version` が `4.5.12`、`$.build` が `82.4` でした。ExtendScriptはES3相当の古い処理系です。

## 既報を確認する

「自分だけが踏んだのか」を確認しました。

[Adobeコミュニティのバグ報告フォーラムに2025年7月24日付の投稿](https://community.adobe.com/t5/after-effects-bugs/bug-in-extendscript-s-conditional-ternary-operator/idi-p/15428397)があり、`true ? a : true ? b : c;` が `a` ではなく `b` を返すと報告されています。既知の挙動でした。

一方で、**Adobe公式ドキュメントにこの挙動を記載したものは見つかりませんでした**。`JavaScript Tools Guide` を検索しても `ternary` `conditional operator` `precedence` はヒットしません。このガイドはECMA仕様への**追加機能**だけを記述していて、仕様からの逸脱は文書化していないようです。

つまり、公式ドキュメントを読み込んでも防げません。踏むまで気づけない類のバグです。

## もうひとつ踏んだ罠：`name` が代入できない

同じセッションで、別の変数まわりの罠も踏みました。4パターンの色違いを書き出すループが、なぜか1ファイルしか出力せず、しかもファイル名が `out-Adobe Photoshop.png` になっていました。

```javascript
// ダメな例
var name = variants[v][0];   // "A" のはず
doc.saveAs(new File(dir + "/out-" + name + ".png"), ...);
```

ExtendScriptをPhotoshopで実行すると、**グローバルスコープがアプリケーションオブジェクトそのもの**になります。トップレベルの `var name = ...` は `app.name` への代入になり、このプロパティは読み取り専用なので**代入が黙って無視され**、参照すると `"Adobe Photoshop"` が返ります。

エラーは出ません。`version` や `parent` も同じ性質を持ちます。関数の引数として使う分にはローカル変数になるので問題ありませんが、トップレベルでは避けたほうが安全です。

## 得られた手順

処理系のバグを疑うまでに、こういう順で確認するのが結果的に速かったです。

1. **値を見る** — ログを仕込んで、引数が期待どおり届いているか
2. **計算を逆算する** — 出てきた数字が「どの式なら成立するか」を割り出す。今回は794という値が右揃えの式にぴったり一致したことで、分岐が違うと確定できた
3. **生成物を `repr()` で見る** — 自動生成したコードなら、見えない文字や置換ミスを潰す
4. **最小再現に落とす** — ドメイン固有のAPIをすべて剥がし、書き方の違いだけを並べて比較する
5. **既報を探す** — 自分だけの問題か、既知かを確認する

3までで自分のコードが正しいと確定したので、4に進めました。逆に言うと、**1〜3を飛ばして「処理系が悪い」と決めつけると、たいてい自分のバグです**。

## 対処

括弧なしのネスト三項演算子を使わず、`if` 文で書きます。

```javascript
var x;
if (mode === "center")     { x = (W - w) / 2; }
else if (mode === "right") { x = W - MARGIN - w; }
else                       { x = MARGIN; }
```

ES3世代の処理系を相手にするときは、モダンな圧縮した書き方に頼らず素直に書くのが、結局いちばん安全でした。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=extendscript-nested-ternary-bug) で活動中。
Photoshopをスクリプトから操作する実践的な手順は [ClaudeでPhotoshopを自動操作する方法](https://codequest.work/claude-photoshop-extendscript/?utm_source=zenn&utm_medium=article&utm_campaign=extendscript-nested-ternary-bug) にまとめています。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=extendscript-nested-ternary-bug) — ディレクターも使っているSEO診断ツールを公開中。
