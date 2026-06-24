---
title: "Puter.jsでAPIキー不要のAIチャットを作り、BYOKで複数AIを共通化する"
emoji: "🗒️"
type: "tech"
topics: ["Puter", "AI", "JavaScript", "frontend", "Claude"]
published: true
---

## 結論から

- **Puter.js を `<script>` で読み込むだけで、APIキーなしにブラウザから GPT / Claude / Gemini 系を呼べる**（`puter.ai.chat()`）
- 無料で動くのは Puter の **「User Pays（利用者が払う）」モデル**。利用料を開発者ではなくエンドユーザーの Puter アカウント側が負担するので、初回に無料サインインが要る
- 精度・速度・モデルを自分で握りたい場面は **BYOK（Bring Your Own Key）**。Gemini / Groq / Claude を**同じ `callAI()` インターフェースに包む**と、切り替えが `switch` 1か所で済む
- スマホ Safari は**サインインのポップアップがブロックされて固まる**。最初の応答にタイムアウトを置いて「別のAIに切り替えて」と案内する fallback を入れておくと安定する

この記事は、付箋に質問を書くと AI が答える小さなメモアプリを作ったときの実装メモです。完成物は記事末尾に置いておきます。

## Puter.js とは（なぜキー不要で動くのか）

[Puter](https://developer.puter.com/) は、ブラウザから無料で AI を呼び出せるオープンソースのクラウドサービスです（ソースは [GitHub: HeyPuter/puter](https://github.com/HeyPuter/puter)）。

通常、AI を Web アプリに組み込むには各社の API キーを取得し、**キーを隠すためにサーバーを立てて中継**する必要があります。Puter はこの中継とコスト負担を肩代わりしてくれるので、スクリプトを 1 行読み込むだけで AI を呼べます。

```html
<!-- これだけでAIが使える。サーバーもAPIキーも不要 -->
<script src="https://js.puter.com/v2/"></script>

<script>
  const reply = await puter.ai.chat("日本の首都はどこ？");
  console.log(reply); // -> AIの回答テキスト
</script>
```

無料で動く理由は **User Pays モデル**です。AI の利用料を、アプリの開発者ではなく**アプリを使う人の Puter アカウント側が負担**します。だからこそ初回利用時に無料の Puter サインインが求められます。開発者は API コストもキー管理も気にせず組み込める、という分担です。

## `puter.ai.chat` をストリーミングで使う

回答を 1 文字ずつ表示したいので、`{ stream: true }` を付けます。返り値は async iterator になり、`part.text` を繋いでいくだけです。モデルを指定したいときは `{ model }` を足します。

```js
async function callPuter({ model, messages, onChunk }) {
  const options = { stream: true };
  if (model) options.model = model; // 例: "claude-sonnet-4"

  const response = await puter.ai.chat(messages, options);

  let full = "";
  for await (const part of response) {
    const chunk = part?.text ?? "";
    if (chunk) {
      full += chunk;
      onChunk?.(full); // 累積テキストをUIに反映
    }
  }
  return full;
}
```

`messages` は `[{ role: "user" | "assistant", content }]` の配列なので、**会話履歴をそのまま渡せばマルチターン**になります。ここが後で BYOK 側と揃えるポイントになります。

## 4プロバイダを1つの `callAI()` に包む

「キー不要の Puter」と「自分のキーで使う Gemini / Groq / Claude」を UI から切り替えたいので、呼び出し口を 1 つに統一します。**入り口の引数を揃えて、中で `switch` するだけ**です。

```js
// callAI({ provider, model, messages, apiKey, onChunk }) -> Promise<string>
export async function callAI({ provider, model, messages, apiKey, onChunk }) {
  switch (provider) {
    case "puter":  return callPuter({ model, messages, onChunk });
    case "gemini": return callGemini({ model, messages, apiKey });
    case "groq":   return callGroq({ model, messages, apiKey });
    case "claude": return callClaude({ model, messages, apiKey });
    default: throw new Error(`未知のプロバイダ: ${provider}`);
  }
}
```

UI 側は `callAI()` だけを知っていればよく、プロバイダが増えても `switch` に 1 行足すだけで済みます。`messages` の形（`role` / `content` の配列）を全プロバイダで共通にしておくのがコツで、各社のクセは下の各関数の中だけに閉じ込めます。

## BYOK：各プロバイダの fetch 実装の違い

BYOK では各社の API を**ブラウザから直接 `fetch`** します。キーは `localStorage` に置き、送信先は各社 API だけにします。3 社で地味に作法が違うので、そこだけ吸収します。

### Gemini（role を変換して contents で送る）

Gemini は `messages` ではなく `contents`、`assistant` ではなく `model` という命名です。マッピングが必要です。

```js
async function callGemini({ model, messages, apiKey }) {
  const url =
    `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`;

  const contents = messages.map((m) => ({
    role: m.role === "assistant" ? "model" : "user", // ここが変換
    parts: [{ text: m.content }],
  }));

  const res = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ contents }),
  });
  const data = await res.json();
  return (data?.candidates?.[0]?.content?.parts || [])
    .map((p) => p.text || "").join("");
}
```

### Groq（OpenAI 互換でそのまま投げられる）

Groq は OpenAI 互換なので、`messages` を**変換なしでそのまま**渡せます。推論が速いのも利点です。

```js
async function callGroq({ model, messages, apiKey }) {
  const res = await fetch("https://api.groq.com/openai/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
    body: JSON.stringify({ model, messages }), // そのまま
  });
  const data = await res.json();
  return data?.choices?.[0]?.message?.content ?? "";
}
```

### Claude（ブラウザ直叩きには専用ヘッダが要る）

Anthropic の API をブラウザから直接叩くときは、`anthropic-dangerous-direct-browser-access: true` を付けないと CORS で弾かれます。`max_tokens` も必須です。

```js
async function callClaude({ model, messages, apiKey }) {
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-api-key": apiKey,
      "anthropic-version": "2023-06-01",
      "anthropic-dangerous-direct-browser-access": "true", // これが必要
    },
    body: JSON.stringify({ model, max_tokens: 4096, messages }),
  });
  const data = await res.json();
  return (data?.content || [])
    .filter((b) => b.type === "text").map((b) => b.text).join("");
}
```

> ヘッダ名のとおり「dangerous」です。キーがブラウザに露出するのを承知のうえで、**自分のキーを自分の端末で使う BYOK だから許される**形であって、不特定多数に配るアプリで他人のキーを扱う用途には向きません。

## スマホで Puter が固まる問題と対策

実機で一番ハマったのがこれです。キー不要の Puter は初回にサインインのポップアップを出しますが、**スマホ（特に Safari）ではポップアップがブロックされ、応答が来ないまま固まる**ことがあります。

対策として、**最初のチャンクが一定時間来なければ打ち切って案内する**タイムアウトを噛ませました。

```js
const FIRST_RESPONSE_TIMEOUT = 30000;
let firstSeen = false;

const timeout = new Promise((_, reject) => {
  setTimeout(() => {
    if (!firstSeen) {
      reject(new Error(
        "Puterから応答がありません。ポップアップを許可してサインインするか、" +
        "別のAI（自分の無料キー）に切り替えてください。"
      ));
    }
  }, FIRST_RESPONSE_TIMEOUT);
});

// 本体側は最初のチャンクで firstSeen = true にしてタイマーを止める
return Promise.race([run(), timeout]);
```

`Promise.race` で「本体の実行」と「タイムアウト」を競争させ、最初のチャンクが来たら `firstSeen = true` にしてタイマーを無効化します。固まったときは「ポップアップを許可」か「Gemini（無料キー）など別 AI に切り替え」を案内すれば、ユーザーは詰まらずに済みます。`callAI()` で口を揃えてあるので、**プロバイダの切り替え自体は 1 行**で終わります。

## データはすべて localStorage に置く

このアプリはログイン不要にしたかったので、メモ・会話・API キーを**すべて `localStorage` に置き、サーバーには一切送りません**。薄いラッパーで JSON 化とエラー処理だけ 1 か所に集約しておくと扱いやすいです。

```js
export function saveJson(key, value) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
    return true;
  } catch (error) {
    console.error("保存に失敗:", key, error);
    return false; // 容量超過などは握りつぶしてUIは止めない
  }
}
```

裏返しに、**端末・ブラウザ間で同期されない／データを消すと消える**というトレードオフがあります。AI に送るのは質問本文だけ、API キーは各社 API にだけ送る、という線引きを UI でも明示しておくと安心です。

## 作ったアプリ

この実装で、付箋に質問を書いて送信すると AI が答える小さなメモアプリにしました。付箋ごとに「AI に聞く」と「ふつうのメモ」を切り替えられて、保存は自動（localStorage）です。

- アプリ本体: https://codequest.work/generator/ai-shirabe-memo/
- 使い方とコンセプトの解説: https://codequest.work/ai-shirabe-memo-app/

「とりあえず AI 付きメモが欲しい」ならそのまま使えますし、「自分でも組み込みたい」ならこの記事の `callAI()` の形がそのまま使えます。

## まとめ

- Puter.js は `<script>` 1 行 + `puter.ai.chat()` で**キーなしに AI を呼べる**。無料の裏側は User Pays モデル
- 複数プロバイダは **`callAI({ provider, ... })` で口を 1 つに**まとめ、各社のクセ（Gemini の role 変換 / Groq の OpenAI 互換 / Claude の専用ヘッダ）は各関数に閉じ込める
- スマホの**ポップアップ詰まりはタイムアウト + 別 AI へ切り替え**で回避
- ログイン不要にするなら localStorage 完結。同期されないトレードオフは UI で明示

`messages` の形さえ全プロバイダで揃えておけば、AI の追加も差し替えも怖くありません。手元の小さなツールから、ぜひ試してみてください。
