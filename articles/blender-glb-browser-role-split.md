---
title: "Blenderで決めた動きを、Webでそのまま再生する｜glbに入れるもの・ブラウザで足すもの"
emoji: "🪐"
type: "tech"
topics: ["blender", "threejs", "claudecode", "webgl", "3d"]
published: true
---

Webサイトに3Dの演出を入れるとき、最初に迷うのは「どこまでをBlenderで作り、どこからをブラウザで作るか」です。

結論から書きます。**最後まで同じ形で再生したい「一度きりの動き」はglbに入れ、ずっと続く動き・光・画面の部品はブラウザで足す**と、作り直しが楽になります。この記事では、筆者の個人サイト [orecticdesign.com](https://orecticdesign.com/) のファーストビュー（FV）を例に、この線をどう引いたかを書きます。

![orecticdesign.comのファーストビュー。真鍮の渾天儀が組み上がった状態](https://codequest.work/wp-content/uploads/2026/10/claude-code-blender-fv-live.jpg)
*本番のファーストビュー。真鍮の渾天儀（こんてんぎ）が外側から集まって組み上がります*

## 作ったもの：組み上がる渾天儀

FVでは、真鍮の渾天儀が外側から集まって組み上がります。渾天儀は、天球の動きを輪の組み合わせで表した昔の天文道具です。

| 項目 | 内容 |
|---|---|
| 形と組み上がる動き | Blender 5.2.2で作成（Claude Codeの自作スキルから画面なしで実行） |
| 書き出し | glb（Draco圧縮・約300KB） |
| 再生 | three.js 0.170の GLTFLoader＋DRACOLoader で読み、AnimationMixer で再生 |
| 動きの長さ | 30fps・96フレーム（3.2秒） |

glbは、3Dの形・材質・動きを1ファイルにまとめたglTF形式のバイナリ版です。Draco圧縮は形のデータを小さくする圧縮方式で、読む側にDRACOLoaderが必要になります。

## 線引きの結果：glbに入れたもの・ブラウザで足したもの

先に全体を表にします。

| 置き場所 | 入れたもの |
|---|---|
| **glb（Blender）** | 輪・パネル・台座・芯の形とマテリアル、組み上がりの動き |
| **three.js** | 組み上がった後に輪を回し続ける動き、環境光、電灯の影（VSM）、光のにじみ（UnrealBloom）、霧、床と壁 |
| **HTML** | ナビのラベル |

判断の軸は3つでした。

1. **決まった形で終わる動きか、終わらない動きか**
2. **画面の大きさや状況で変えたいものか**
3. **文字として読ませる・操作させるものか**

以下、それぞれ説明します。

### 1. 一度きりの動きはBlender、終わらない動きはブラウザ

組み上がりは「3.2秒で始まり、完成形で止まる」動きです。始まりと終わりが決まっていて、毎回同じに見えてほしいので、Blenderでキーフレームを打ってglbに入れました。

一方、組み上がった後に輪が回り続ける動きには終わりがありません。これをBlenderで作ろうとすると、ループの継ぎ目を合わせる手間が増えますし、速さを変えたくなるたびに書き出し直しです。ブラウザなら毎フレーム角度を少し足すだけで済み、速さも数値1つで変えられます。

### 2. 光と空気はブラウザ側で調整する

環境光・影・にじみ・霧は、Blenderでレンダリングした見た目をそのままglbに持ち込めるわけではありません。glbに入るのは形とマテリアルが中心で、見え方の多くはブラウザ側の設定で決まります。

そこで、光と空気はthree.js側の担当と割り切りました。影はVSM（ぼかしの効く影の方式）、光のにじみはUnrealBloom（明るい部分をにじませる後処理）、それに霧と床・壁を足しています。ブラウザ上で数値を変えながら確認できるので、Blenderを開き直すより早く詰められます。

### 3. 文字はHTMLに残す

ナビのラベルは3Dの中に入れず、HTMLで重ねています。HTMLの文字なら、読み上げ・選択・リンクがふつうに働き、フォントや大きさもCSSで調整できます。3Dに文字を焼き込むと、こうした性質がすべて失われます。

## Blenderで付けた名前が、ブラウザとの受け渡し口になる

役割分担で一番効いたのが、**Blenderで付けたオブジェクト名がglbに残る**ことです。

three.js側では、名前で部品を探して手を加えています。

- 名前が `Core` の芯 → 地図を貼った球に差し替える
- 名前が `Ring` で始まる輪 → 組み上がった後に回し続ける

つまり、名前が「Blender側」と「ブラウザ側」の約束になります。Blenderで形を作り直しても、名前さえ変えなければブラウザ側のコードはそのまま動きます。

名前で探す部分を汎用の形で書くと、次のようになります。

```js
// gltf は GLTFLoader で読み込んだ結果
const core = gltf.scene.getObjectByName("Core");

const rings = [];
gltf.scene.traverse((obj) => {
  if (obj.name.startsWith("Ring")) rings.push(obj);
});
```

芯を差し替えたのは、試し描きの段階で芯の見た目が決まっていなかったからです。下の画像は、芯がまだ深緑の多面体だった頃のBlenderの試し描きです。

![Blenderでの試し描き。芯が深緑の多面体だった段階](https://codequest.work/wp-content/uploads/2026/10/claude-code-blender-render.jpg)
*Blenderでの試し描き。この時点では芯が深緑の多面体でした*

芯はブラウザ側で差し替える前提にしておいたので、Blender側では芯の見た目を作り込まずに済みました。「あとで差し替える部品には決まった名前だけ付けておく」と決めておくと、Blenderでの作業が軽くなります。

## 動きの決め方：止まり方は動画で見る

組み上がりの終わり方は、行き過ぎて揺れる動きを付けず、ぴたりと止まる形にしました。真鍮の重い道具が組み上がる場面なので、揺れるより止まるほうが合うと考えたためです。

ここで大事だったのは、**動きは静止画では判断できない**ことです。数フレームを切り出して見ても、速さの変化や止まり方の印象は分かりません。そこで低解像度の動画に書き出して、実際の速さで見てから決めました。低解像度ならレンダリングが軽く、案を何度も見比べられます。

## 動きを減らしたい人には、完成形から見せる

OSで「視差効果を減らす」などの設定（`prefers-reduced-motion`）をしている人には、組み上がりを飛ばして完成形から表示しています。

組み上がりをglbのアニメーションとして持っているので、これはブラウザ側で再生位置を最後に送るだけで実現できます。汎用の形で書くと次のとおりです。

```js
const reduceMotion = matchMedia("(prefers-reduced-motion: reduce)").matches;

const clip = gltf.animations[0];
const action = mixer.clipAction(clip);
action.setLoop(THREE.LoopOnce, 1);
action.clampWhenFinished = true; // 最後のフレームで止める
action.play();

if (reduceMotion) mixer.setTime(clip.duration); // 完成形から表示する
```

動きをBlender側に寄せても、「見せるかどうか」の判断はブラウザ側に残せます。これも役割分担の利点です。

## 受け渡しで気をつけること：上の向きが違う

BlenderとglTF（three.js）では、上を向く軸が違います。

| | 上向きの軸 |
|---|---|
| Blender | Z |
| glTF / three.js | Y |

glTF書き出しの `export_yup` は既定でTrueなので、書き出し時に変換されます。その結果、**Blenderの座標 (x, y, z) は three.js では (x, z, -y)** になります。

ブラウザ側でカメラや光の位置をBlenderの値に合わせるときは、この変換を通してから置きます。変換を忘れると、カメラが真上から覗いたり、影が横に伸びたりします。

## Claudeから動かすなら、本番はスクリプトで作る

この渾天儀は、Claude Codeの自作スキルからBlenderを画面なしで動かして作りました。ClaudeからBlenderを動かす方法は、大きく3つあります。

| 方法 | 概要 |
|---|---|
| 公式Blenderコネクタ | Blender Lab製。Anthropicが2026-04-28に「Claude for Creative Work」で発表。Blender 5.1以上 |
| MCP for Blender（旧blender-mcp） | コミュニティ製・MITライセンス |
| 画面なしスクリプト | `blender -b -P script.py` でPythonを実行 |

本番の素材はスクリプト方式で作りました。スクリプトなら作り直しが1コマンドで済み、glbの書き出し設定もファイルの中に固定できます。役割分担を決めても、glb側の書き出し設定が毎回変わると、ブラウザ側との約束が崩れるためです。

MCPは、対話しながら形を試す段階に向いています。ただし、どちらのMCPもAIが書いたPythonを確認なしで実行します（公式ページに「without any guards」とあります）。使う前に、作業用のファイルで試すなどの備えをしておくと安心です。

## まとめ

- 始まりと終わりが決まった一度きりの動きはglbに入れる
- 終わらない動き・光・空気はブラウザで足す
- 文字はHTMLに残す
- Blenderで付けた名前を、ブラウザとの約束にする
- 動きを減らす設定の人には、再生位置を最後に送って完成形から見せる
- Blenderの (x, y, z) は three.js では (x, z, -y)

Blender側のglb書き出し手順、three.jsで再生する最小コード、書き出しでつまずく点は、本体の記事で詳しく解説しています。

https://codequest.work/claude-code-blender-glb-threejs/?utm_source=zenn&utm_medium=article&utm_campaign=blender-glb-browser-role-split

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=blender-glb-browser-role-split) で活動中。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=blender-glb-browser-role-split) — ディレクターも使っているSEO診断ツールを公開中。
