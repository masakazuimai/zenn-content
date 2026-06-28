---
title: "Macの⌘とWindowsのCtrlを1つのデータで両対応させるショートカット表示の設計"
emoji: "⌨️"
type: "tech"
topics: ["JavaScript", "frontend", "UI", "個人開発", "Web"]
published: true
---

## 結論から

キーボードショートカットを画面に表示するとき、Mac の `⌘C` と Windows の `Ctrl+C` を**どう両対応させるか**が地味に悩ましいです。素朴にやると Mac 用と Windows 用で2セットのデータを持つ羽目になります。

これを1つのデータで済ませる設計をまとめます。要点は4つです。

- ショートカットは `["mod", "C"]` のような **OS非依存トークン**で持つ（`mod` / `alt` / `shift` は論理的な修飾キー）
- 表示するときだけ **Mac辞書 / Windows辞書** に通して `mod → ⌘` / `mod → Ctrl` に変換する
- ユーザーのキー入力を取得するときは `keydown` で拾い、`e.metaKey || e.ctrlKey` を `mod` に**統合**する
- OS の自動判定は**初期値だけ**にして、最終的にはユーザーが選んだ結果を `localStorage` に保存する

この記事は、自分で作ったショートカット表示ツールの設計メモです。コードは要点が見えるように汎用化して載せます。

## なぜ「⌘と Ctrl 問題」が起きるのか

同じ「コピー」でも、Mac では `⌘ + C`、Windows では `Ctrl + C` と表記が違います。`shift` は Mac だと `⇧`、Windows だと `Shift`。`Delete` は Mac で `⌫`、Windows で `Del`。

ここで「Mac 用の文字列」と「Windows 用の文字列」を別々にデータとして持つと、ショートカットが増えるたびに**両方を二重メンテ**することになります。どちらか片方を直し忘れて表記がズレる、という事故も起きます。

解きたい問題はシンプルで、**データは1つにして、表示のときだけ OS に合わせて変換したい**、ということです。

## データは OS 非依存トークンで持つ

そこで、ショートカットのキーを「論理トークンの配列」で表現します。

```js
// 「コピー」も「取り消し」も、データは OS 非依存
const copy = { label: "コピー", keys: ["mod", "C"] };
const undo = { label: "取り消し", keys: ["mod", "Z"] };
const redo = { label: "やり直し", keys: ["mod", "shift", "Z"] };
```

`mod` は「Mac の ⌘ / Windows の Ctrl にあたる主修飾キー」、`alt` は「⌥ / Alt」、`shift` は「⇧ / Shift」を表す**論理名**です。実際の記号は持たせません。

記号への変換は、表示用の辞書を OS ごとに用意して、表示の瞬間に行います。

```js
const MAC = { mod: "⌘", alt: "⌥", shift: "⇧", Delete: "⌫", Enter: "↩" };
const WIN = { mod: "Ctrl", alt: "Alt", shift: "Shift", Delete: "Del", Enter: "Enter" };

const renderKeys = (keys, os) => {
  const map = os === "mac" ? MAC : WIN;
  return keys
    .map((k) => `<kbd>${map[k] ?? k}</kbd>`) // 辞書になければ ?? でそのまま（"C" など）
    .join("+");
};
```

```js
renderKeys(["mod", "shift", "Z"], "mac"); // <kbd>⌘</kbd>+<kbd>⇧</kbd>+<kbd>Z</kbd>
renderKeys(["mod", "shift", "Z"], "win"); // <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Z</kbd>
```

ポイントは **データは1つ、辞書は OS の数だけ**という形です。対応 OS を増やしたくなっても、データには手を入れず辞書を1つ足すだけで済みます。`map[k] ?? k` にしておくと、辞書に載っていない普通のキー（`C` や `F5`）はそのまま表示されます。

## キーボードから直接ショートカットを取得する

「一覧にないキーを自分で登録する」機能を付けるなら、ユーザーが実際に押したキーを `keydown` で拾います。ここがいちばんハマりやすいので、丁寧に見ていきます。

```js
let capturing = false; // 「今からキーを取得する」モードのフラグ
const pending = { mod: false, alt: false, shift: false, key: "" };

document.addEventListener("keydown", (e) => {
  if (!capturing) return;

  e.preventDefault(); // ① ブラウザ標準のショートカットを止める

  // ② 修飾キー単体は「まだ確定じゃない」ので待つ
  if (["Meta", "Control", "Alt", "Shift"].includes(e.key)) return;

  pending.mod = e.metaKey || e.ctrlKey; // ③ ⌘ と Ctrl を mod に統合
  pending.alt = e.altKey;
  pending.shift = e.shiftKey;
  pending.key = normalizeKey(e.key); // ④ e.key を表示用に正規化

  capturing = false; // 1つ取れたらキャプチャ終了
});
```

### ① `preventDefault()` しないとブラウザに横取りされる

`⌘ + S`（保存）や `⌘ + W`（タブを閉じる）など、ブラウザやOSが先に反応するショートカットがあります。取得モード中は `e.preventDefault()` で標準動作を止めないと、登録する前にダイアログが開いたりタブが閉じたりします。

### ② 修飾キー単体の `keydown` を弾く

`⌘ + C` を押すとき、ユーザーは普通「先に ⌘ を押し下げて、次に C」を押します。このとき `⌘` を押し下げた瞬間にも `keydown` が飛びます。`e.key` が `"Meta"` / `"Control"` / `"Alt"` / `"Shift"` のときは**まだ本体キーが来ていない**ので、確定せずに `return` して次の入力を待ちます。

### ③ `e.metaKey || e.ctrlKey` で論理 `mod` に寄せる

ここが両対応の肝です。Mac で ⌘ を押すと `e.metaKey` が、Windows で Ctrl を押すと `e.ctrlKey` が `true` になります。これを **OR で1つの `mod` に畳む**ことで、「どっちの環境で押されたか」を気にせず同じトークンに落とせます。`e.metaKey` と `e.ctrlKey` を別々に持つと、また OS 分岐が増えてしまいます。

### ④ `e.key` を表示用に正規化する

`e.key` は押したキーの「文字」が入りますが、そのままでは表示に向かない値も来ます。

```js
const normalizeKey = (k) => {
  if (k === " ") return "Space";
  const map = {
    ArrowLeft: "←", ArrowRight: "→", ArrowUp: "↑", ArrowDown: "↓",
    Backspace: "Delete",
  };
  if (map[k]) return map[k];
  return k.length === 1 ? k.toUpperCase() : k; // "a" → "A"、"F5" はそのまま
};
```

注意したいのは、**`e.key` は Shift を押していないと小文字で来る**ことです（`a` キーなら `"a"`）。表示は大文字に揃えたいので `toUpperCase()` します。スペースは `" "` という見えない値なので `"Space"` に、矢印キーは `"ArrowLeft"` のような長い名前で来るので記号に変換します。`F5` のような複数文字のキーはそのまま使います。

:::message
`e.key`（押された文字）と `e.code`（物理的なキー位置、例 `"KeyA"`）は別物です。ゲームの WASD 操作のように**キーの位置**が大事なら `e.code`、今回のように**表示する文字**が大事なら `e.key` を使うのが素直です。
:::

## 取得した入力をトークンに組み立てる

`pending` に溜めた状態を、最初に作った表示トークンの形へ組み立てます。

```js
const pendingTokens = () => {
  const t = [];
  if (pending.mod) t.push("mod");
  if (pending.alt) t.push("alt");
  if (pending.shift) t.push("shift");
  if (pending.key) t.push(pending.key);
  return t; // 例: ["mod", "shift", "Z"]
};
```

これで `renderKeys()` にそのまま渡せます。**入力（keydown 取得）と表示（辞書変換）が同じトークン形式を共有している**ので、間に変換層を挟まずに済み、コードが素直になります。修飾キーを `mod → alt → shift → 本体キー` の順で積むと、表示順も安定します。

## OS 判定はユーザーの上書きを前提にする

初期表示をどちらの OS に合わせるかは、自動で当たりを付けます。

```js
const detectOS = () =>
  /Mac|iPhone|iPad/i.test(navigator.platform || navigator.userAgent) ? "mac" : "win";
```

ただし `navigator.platform` は[MDN でも非推奨（deprecated）](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/platform)とされています。将来的には `navigator.userAgentData.platform` への移行が想定されますが、これも対応ブラウザが限られます。

なので自動判定は**あくまで初期値**と割り切り、画面に Mac / Windows の切り替えトグルを置いて、**ユーザーが選んだ結果を正**にします。外部キーボードを繋いだ Windows、ブラウザを偽装している環境など、自動判定が外れるケースは普通にあるからです。

## 選択と並び順を `localStorage` に保存する

選んだ OS や、お気に入り・並び順は `localStorage` に保存しておくと、次に開いたときも同じ状態で使えます。

```js
const STORAGE_KEY = "shortcut-state";

const save = (state) => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
};

const load = () => {
  const fresh = () => ({ os: detectOS(), favorites: [], order: [] });
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return fresh();
    const parsed = JSON.parse(raw);
    return {
      os: parsed.os === "mac" || parsed.os === "win" ? parsed.os : detectOS(),
      favorites: Array.isArray(parsed.favorites) ? parsed.favorites : [],
      order: Array.isArray(parsed.order) ? parsed.order : [],
    };
  } catch {
    return fresh(); // 壊れた JSON / プライベートモード等は初期化
  }
};
```

`localStorage` は `JSON.parse` が壊れた値で例外を投げたり、プライベートモードで弾かれたりするので、読み込みは必ず `try/catch` で包みます。さらに、保存された値が**期待した形か（`os` が `"mac"` / `"win"` か、配列か）をバリデート**して、想定外なら初期値に戻すと、古いデータ形式が残っていても壊れません。

## まとめ

- ショートカットは `["mod", "alt", "shift", キー]` の **OS 非依存トークン**で持ち、表示の瞬間だけ OS 辞書で `⌘` / `Ctrl` に変換する
- `keydown` でキーを取得する4つの勘所：`preventDefault()` / 修飾キー単体を弾く / `e.metaKey || e.ctrlKey` を `mod` に統合 / `e.key` を正規化
- `e.key`（文字）と `e.code`（物理位置）は使い分ける。表示なら `e.key` + 正規化
- OS の自動判定は初期値だけにして、ユーザーの選択を `localStorage` に永続化する
- **入力と表示で同じトークン形式を共有**すると、変換層が減ってコードが素直になる

「データ1つ・辞書を OS の数だけ」という形にしておくと、後からアプリやプラットフォームを足すのも怖くありません。キー操作を扱う UI を作るときの土台として、参考になれば。

## 作ったツール

この設計で、Photoshop・Illustrator・Figma・Excel・Word・PowerPoint のショートカットをアプリ別に一覧できるチートシートを作りました。よく使うキーを ★ で集めて自分専用の「マイ一覧」にし、Mac / Windows 表記を切り替えて、印刷・PDF・PNG 画像として書き出せます。

- ツール本体: https://codequest.work/generator/shortcut-cheatsheet/
- 使い方ガイド: https://codequest.work/shortcut-cheatsheet-tool/

登録不要・ブラウザ完結なので、手元のチートシートが欲しい方はそのまま使えますし、「自分でも組み込みたい」方はこの記事の設計がそのまま使えます。
