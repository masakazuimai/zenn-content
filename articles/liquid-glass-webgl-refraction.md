---
title: "CSSのbackdrop-filterでは作れない「屈折するガラス」をWebGLシェーダーで作る"
emoji: "🔮"
type: "tech"
topics: ["WebGL", "GLSL", "frontend", "JavaScript", "CSS"]
published: true
---

## 結論から

- CSS の `backdrop-filter` でできるのは**ぼかし・明度・彩度などのフィルター**まで。**背景を歪ませる「屈折」は仕様上できない**
- iOS 26 の Liquid Glass が「ガラスっぽい」のは、半透明やぼかしではなく**背景がレンズのように歪んで見える**から
- これを Web で再現するには、背景を WebGL の canvas に描画し、**フラグメントシェーダーでガラス領域のピクセルをずらす**のが定番
- 形（角丸矩形）の判定は **SDF（符号付き距離関数）**、屈折の向きは**距離場の勾配（＝法線）**で決める

この記事では、実際に動かしている GLSL シェーダーを分解しながら、屈折ガラスの作り方を順番に説明します。最後に、パラメータをいじってコードを書き出せるツールにした話もします。

## なぜ CSS では屈折が作れないのか

`backdrop-filter: blur(10px)` は「背景をぼかす」ことはできますが、**背景のピクセルを別の位置から持ってくる（＝歪ませる）ことはできません**。フィルター関数（blur / brightness / saturate …）は、あくまで背景の各ピクセルにその場で処理をかけるものだからです。

本物のガラス越しの景色は、縁に近いほど大きく曲がって見えます。これはガラスがレンズとして働き、背後の光を屈折させているためです。**「背景を位置ごとにずらす」処理が必要**で、これは CSS の表現範囲の外。だから CSS だけで作ったガラスは、半透明でぼけてはいても、どこか平面的に見えるわけです。

そこで WebGL の出番になります。フラグメントシェーダーなら「この画素には、背景のどの座標の色を持ってくるか」を1ピクセル単位で自由に決められます。これが屈折表現の核心です。

## 全体の構成

やることはシンプルで、次の3つだけです。

1. 画面全体を覆う1枚の三角形（fullscreen triangle）を描く
2. その上で、背景画像をテクスチャとして貼る
3. フラグメントシェーダーで、**ガラス領域だけ背景サンプル位置をずらして**屈折・色収差・ぼかし・スペキュラを合成する

頂点シェーダーは uv を渡すだけです。

```glsl
attribute vec2 uv;
attribute vec2 position;
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = vec4(position, 0.0, 1.0);
}
```

以降はすべてフラグメントシェーダーの話になります。

## ステップ1: 背景を cover で敷く

まず背景画像を `object-fit: cover` 相当で画面いっぱいに配置します。canvas と画像のアスペクト比から uv をスケールするだけです。

```glsl
vec2 coverUv(vec2 fragPx) {
  vec2 uv = fragPx / uResolution;
  float canvasAspect = uResolution.x / uResolution.y;
  float imageAspect  = uImageResolution.x / uImageResolution.y;
  vec2 scale = canvasAspect > imageAspect
    ? vec2(1.0, imageAspect / canvasAspect)
    : vec2(canvasAspect / imageAspect, 1.0);
  return (uv - 0.5) * scale + 0.5;
}
```

ガラスの外側はこの背景をそのまま表示し、内側だけ後述の屈折処理をかけます。

## ステップ2: ガラスの形を SDF で定義する

ガラスの形（角丸矩形）は **SDF（符号付き距離関数）** で表します。ある画素がガラスの内側なら負、外側なら正の「距離」を返す関数です。角丸矩形の SDF は Inigo Quilez の有名な式が使えます。

```glsl
// 角丸矩形の符号付き距離（内側で負）
float sdRoundRect(vec2 p, vec2 b, float r) {
  vec2 q = abs(p) - b + r;
  return min(max(q.x, q.y), 0.0) + length(max(q, 0.0)) - r;
}
```

`p` はガラス中心からの相対座標、`b` は半径（half size）、`r` は角丸です。なぜマスク（内外の二値）ではなく距離を使うのかというと、**距離があると「縁からどれだけ内側か」が連続値で分かり、屈折やアンチエイリアスに使える**からです。

## ステップ3: 距離場の勾配 ＝ 法線で「縁」を検出する

屈折は「縁の向き」に沿って起こします。その向き（外向き法線）は、**SDF の勾配**を数値微分で求められます。

```glsl
float e = 1.5;
vec2 grad = vec2(
  sdRoundRect(p + vec2(e, 0.0), uGlassHalf, uRadius) - sdRoundRect(p - vec2(e, 0.0), uGlassHalf, uRadius),
  sdRoundRect(p + vec2(0.0, e), uGlassHalf, uRadius) - sdRoundRect(p - vec2(0.0, e), uGlassHalf, uRadius)
);
grad = normalize(grad + 1e-6);
```

さらに「縁からの入り込み具合」を 0〜1 で出し、縁ほど強く曲がる曲率を作ります。

```glsl
float band  = clamp(-d / max(uEdge, 1.0), 0.0, 1.0); // 0=縁, 1=内側の平坦部
float curve = pow(1.0 - band, 1.6);                  // 縁で大きく、中央で0
```

`curve` が屈折・色収差・スペキュラすべての「縁ほど強く」を決める共通の係数になります。

## ステップ4: 縁ほど強く背景をずらす ＝ 屈折

ここが本丸です。法線方向 `grad` に、縁での強さ `curve` を掛けて、背景のサンプル位置をずらします。

```glsl
float maxDisp = uEdge * 0.9;
vec2 disp = grad * curve * uRefraction * maxDisp;

vec3 glassColor = sampleBlur(fragPx + disp, uBlur); // ずらした位置の背景を取る
```

`uRefraction` を上げるほど縁の歪みが大きくなり、ガラスの厚み感が出ます。中央付近は `curve` がほぼ 0 なので歪まず、縁に近いほど大きく曲がる——本物のレンズと同じ振る舞いになります。

## ステップ5: 色収差（RGB をずらす）

リアルなレンズは、波長で屈折率が違うため縁に色のにじみ（色収差）が出ます。これは **RGB を別々のオフセットでサンプリング**すれば再現できます。

```glsl
vec2 caOff = grad * curve * uAberration * 14.0;

vec3 glassColor;
glassColor.r = sampleBlur(fragPx + disp + caOff, uBlur).r;
glassColor.g = sampleBlur(fragPx + disp,         uBlur).g;
glassColor.b = sampleBlur(fragPx + disp - caOff, uBlur).b;
```

R と B を逆方向にずらすのがポイントです。強くしすぎると安っぽくなるので、効果は「言われないと気づかない」程度に抑えるのがコツです。

## ステップ6: フロスト（すりガラス）の多点ぼかし

すりガラス感はぼかしで出します。WebGL には CSS の blur がないので、**周囲を複数回サンプリングして平均**します。ここでは中心＋12方向の固定カーネルにしています。

```glsl
vec3 sampleBlur(vec2 fragPx, float radiusPx) {
  vec2 baseUv = coverUv(fragPx);
  if (radiusPx < 0.5) return texture2D(uTexture, baseUv).rgb;
  // 12方向の単位ベクトルを用意（一部抜粋）
  // dirs[0]=(1,0) dirs[4]=(0.7,0.7) ... の計12方向
  vec3 sum = texture2D(uTexture, baseUv).rgb;
  float wsum = 1.0;
  for (int i = 0; i < TAPS; i++) {
    vec2 off = dirs[i] * radiusPx / uResolution;
    sum += texture2D(uTexture, baseUv + off).rgb;
    wsum += 1.0;
  }
  return sum / wsum;
}
```

ガウシアンほど厳密ではありませんが、UI のアクセント用途なら固定カーネルで十分きれいです。タップ数を増やすほど滑らかになりますが、その分重くなるトレードオフです。

## ステップ7: スペキュラ（縁の光）とアンチエイリアス

最後に縁の鏡面ハイライトを足します。法線を擬似的に 3D 化し、ライト方向との内積で光らせます。

```glsl
vec3 n = normalize(vec3(grad * curve, 0.6));
vec3 lightDir = normalize(vec3(-0.6, 0.75, 0.55)); // 左上から
float spec = pow(max(dot(n, lightDir), 0.0), 12.0);
float rim  = smoothstep(0.0, 0.9, curve);
glassColor += spec * uSpecular * rim * 1.4;
```

そして、ガラスと背景の境界を 1px でなめらかに合成します。SDF があるとアンチエイリアスも `smoothstep` 一発です。

```glsl
float coverage = 1.0 - smoothstep(-1.0, 1.0, d);
gl_FragColor = vec4(mix(bgColor, glassColor, coverage), 1.0);
```

これで屈折ガラスの見た目が完成します。

## ogl で動かす最小セットアップ（JS側）

シェーダーを動かすランタイムは、軽量 WebGL ライブラリ [ogl](https://github.com/oframe/ogl) を CDN から読むのが手軽です。fullscreen triangle は `Triangle` 一発で作れます。

```js
import { Renderer, Triangle, Program, Mesh, Texture }
  from 'https://cdn.jsdelivr.net/npm/ogl@1.0.11/+esm'

const canvas = document.getElementById('liquid-glass')
const renderer = new Renderer({ canvas, dpr: Math.min(devicePixelRatio, 2), alpha: false })
const gl = renderer.gl

const geometry = new Triangle(gl)
const texture  = new Texture(gl, { generateMipmaps: false })
const program  = new Program(gl, {
  vertex, fragment,
  uniforms: {
    uTexture:    { value: texture },
    uResolution: { value: [1, 1] },
    uGlassHalf:  { value: [0, 0] },
    uRefraction: { value: 0.7 },
    uAberration: { value: 0.4 },
    // …他のパラメータも uniform で渡す
  },
})
const mesh = new Mesh(gl, { geometry, program })

const img = new Image()
img.crossOrigin = 'anonymous'
img.onload = () => { texture.image = img; resize() }
img.src = BACKGROUND_URL
```

背景画像を別ドメインから読む場合は `img.crossOrigin = 'anonymous'`（と配信側の CORS 許可）を忘れずに。これがないとテクスチャとして使えません。

## DPR（Retina）対応の注意

ハマりやすいのが解像度です。シェーダー内の座標はすべて**実ピクセル（CSS px × dpr）**で計算しているので、JS 側で uniform に渡すときも dpr を掛ける必要があります。

```js
function resize() {
  const w = canvas.clientWidth, h = canvas.clientHeight
  renderer.setSize(w, h)
  const dpr = renderer.dpr
  const U = program.uniforms
  U.uResolution.value = [w * dpr, h * dpr]
  U.uGlassHalf.value  = [GLASS.halfW * dpr, GLASS.halfH * dpr]
  U.uEdge.value       = GLASS.edge * dpr
  U.uBlur.value       = GLASS.blur * dpr
  renderer.render({ scene: mesh })
}
```

ここを CSS px のまま渡すと、Retina で歪み量やぼかし量が半分になってしまい「Mac だと薄いのにスマホだと濃い」みたいな差が出ます。

## パラメータをいじってコードを書き出せるツールにした

ここまでのパラメータ（屈折強度・ぼかし・縁の光・色収差・縁の厚み・角丸・ティント）は、数値を手で詰めるのがかなり面倒です。そこで、**スライダーで調整してプレビューを見ながら、決まったらそのまま貼れるコードを書き出せるツール**にしました。

- ツール: https://codequest.work/generator/liquid-glass-generator/
- 使い方ガイド: https://codequest.work/liquid-glass-generator-tool/

出力されるのは、この記事のシェーダーをそのまま埋め込んだ ogl 製の自己完結スニペットです。プレビューと出力が**同じシェーダー文字列を共有**しているので、見た目がズレません。

## まとめ

- `backdrop-filter` では屈折は作れない。背景を「位置ごとにずらす」には WebGL のフラグメントシェーダーが要る
- 形は **SDF（角丸矩形の符号付き距離）**、屈折の向きは**距離場の勾配＝法線**で決める
- 縁ほど強い係数 `curve` を、屈折・色収差・スペキュラで共有すると一貫した質感になる
- ぼかしは多点サンプリング、境界は `smoothstep` でアンチエイリアス
- DPR を掛け忘れると Retina で見た目が変わるので注意

SDF と勾配が分かれば、丸・ピル・カードなど形を変えるのも `sdRoundRect` の引数を変えるだけです。ガラス以外（水滴・凸レンズなど）にも応用が効くので、ぜひ手元の画像で試してみてください。
