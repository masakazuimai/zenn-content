---
title: "チームボード開発記 #5 — URL共有と未読管理を改善した実装（iOS対応の落とし穴）"
emoji: "🔗"
type: "tech"
topics: ["React", "TypeScript", "UX", "チャットアプリ", "NextJS"]
published: false
publish_date: 2026-04-27
---

## はじめに

自作チームボード（[前回記事](https://zenn.dev/orectic/articles/custom-team-board-nextjs-supabase)）を実運用する中で、地味にストレスだったのが**URL共有**と**未読管理**です。

- 長いURLをそのまま貼ると読みにくい（でもMarkdownを毎回手書きするのは面倒）
- チャンネルを見ているのに未読バッジが消えない
- 逆に、スクロール上部にいるのに新着が自動既読されてしまう

本記事では、URL貼り付け時の自動変換ダイアログと、スクロール位置に基づく未読管理の実装を紹介します。iOS特有の罠も含めて。

## URL貼り付け→リンク変換ダイアログ

### 課題

チャットでURLを共有するとき、こうなりがちです。

```
https://docs.google.com/spreadsheets/d/1AbCdEfGhIjKlMnOpQrStUvWxYz/edit?gid=0#gid=0
```

80文字超のURLが1行を占有して、前後のメッセージが読みにくくなります。

Markdownリンク `[スプレッドシート](URL)` に変換すればいいのですが、毎回手書きするのは面倒。そこで、**長いURLを貼り付けたら自動でダイアログを出す**仕組みを実装しました。

### 仕様

1. テキスト入力時に40文字超のURLを検出
2. 「リンクに変換しますか？」ダイアログを表示
3. 表示テキストを入力 → Markdown形式に自動変換
4. キャンセルすればそのまま

```
入力: https://very-long-url.example.com/path/to/something?param=value
  ↓ ダイアログで「参考資料」と入力
出力: [参考資料](https://very-long-url.example.com/path/to/something?param=value)
```

### 実装：onChangeでのURL検出

```typescript
const inputRef = useRef<string>("")
const [linkDialogUrl, setLinkDialogUrl] = useState<string | null>(null)
const [showLinkDialog, setShowLinkDialog] = useState(false)

const handleInputChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
  const newValue = e.target.value
  const prevValue = inputRef.current

  // 新しく追加されたテキストを抽出
  const added = newValue.length > prevValue.length
    ? newValue.slice(prevValue.length)
    : ""

  // URLパターンの検出
  const urlPattern = /https?:\/\/[^\s]+/
  const match = added.match(urlPattern)

  if (match && match[0].length > 40) {
    setLinkDialogUrl(match[0])
    setShowLinkDialog(true)
  }

  inputRef.current = newValue
  setText(newValue)
}
```

### なぜonPasteではなくonChangeなのか

最初は`onPaste`イベントで実装していました。

```typescript
// 最初の実装（NG: iOSで不安定）
const handlePaste = (e: React.ClipboardEvent) => {
  const pasted = e.clipboardData.getData("text")
  if (isLongUrl(pasted)) {
    setShowLinkDialog(true)
  }
}
```

**iOSのSafariで`onPaste`が発火しないケースがありました。** 具体的には：

- キーボードの「ペースト」ボタンからの貼り付け → 発火する
- テキスト長押し→メニューからの「ペースト」 → 発火しないことがある
- ユニバーサルクリップボード（他デバイスからの貼り付け）→ 発火しない

`onChange`で前回値との差分を見る方式なら、**どの方法で入力されたかに関係なく**URLを検出できます。

### ダイアログUI

```tsx
{showLinkDialog && linkDialogUrl && (
  <div className="fixed inset-0 bg-black/30 z-50 flex items-center justify-center">
    <div className="bg-white rounded-xl p-6 mx-4 max-w-sm w-full shadow-lg">
      <p className="font-bold mb-2">リンクに変換</p>
      <p className="text-neutral-600 mb-4 text-base break-all">
        {linkDialogUrl.length > 60
          ? `${linkDialogUrl.slice(0, 60)}...`
          : linkDialogUrl}
      </p>
      <input
        type="text"
        placeholder="表示テキストを入力"
        className="w-full border rounded-lg px-3 py-2 mb-4"
        autoFocus
        onKeyDown={(e) => {
          if (e.key === "Enter") {
            applyLink(e.currentTarget.value)
          }
        }}
      />
      <div className="flex gap-2">
        <button
          onClick={() => setShowLinkDialog(false)}
          className="flex-1 py-2 rounded-lg bg-neutral-100"
        >
          そのまま
        </button>
        <button
          onClick={() => applyLink(displayText)}
          className="flex-1 py-2 rounded-lg bg-primary-500 text-white"
        >
          変換
        </button>
      </div>
    </div>
  </div>
)}
```

### リンク変換処理

```typescript
const applyLink = (displayText: string) => {
  if (!linkDialogUrl) return

  const markdown = displayText
    ? `[${displayText}](${linkDialogUrl})`
    : linkDialogUrl

  // テキスト内のURLをMarkdownリンクに置換
  const newText = text.replace(linkDialogUrl, markdown)
  setText(newText)
  inputRef.current = newText

  setShowLinkDialog(false)
  setLinkDialogUrl(null)
}
```

### メッセージ編集時の対応

新規投稿だけでなく、メッセージ編集時にもURLを貼り付けるケースがあります。同じ`handleInputChange`を共有することで、編集モードでもダイアログが機能します。

## Markdownリンクのレンダリング

メッセージ表示側で`[テキスト](URL)`を検出してリンクに変換します。

```typescript
const renderMessageText = (text: string): React.ReactNode[] => {
  const parts: React.ReactNode[] = []
  const linkRegex = /\[([^\]]+)\]\((https?:\/\/[^\s)]+)\)/g
  let lastIndex = 0
  let match: RegExpExecArray | null

  while ((match = linkRegex.exec(text)) !== null) {
    // リンク前のテキスト
    if (match.index > lastIndex) {
      parts.push(text.slice(lastIndex, match.index))
    }

    // リンク
    parts.push(
      <a
        key={match.index}
        href={match[2]}
        target="_blank"
        rel="noopener noreferrer"
        className="text-primary-500 underline hover:text-primary-700"
      >
        {match[1]}
      </a>
    )

    lastIndex = match.index + match[0].length
  }

  // 残りのテキスト
  if (lastIndex < text.length) {
    parts.push(text.slice(lastIndex))
  }

  return parts
}
```

## 未読管理の改善

### 課題1：見ているのに未読が消えない

チャンネルを開いているのに、新着メッセージで未読バッジが増え続けるバグ。

**原因：** スクロール位置を考慮せず、「チャンネルが開いている = 既読」としていたため、最下部にいない場合に既読処理が走らない。

### 課題2：上部にいるのに既読になる

逆に、古いメッセージをスクロールして読んでいるときに新着が来ると、見ていないのに既読になってしまう。

### 解決策：isAtBottomフラグ

「画面の最下部にいるかどうか」で既読判定を分岐します。

```typescript
const [isAtBottom, setIsAtBottom] = useState(true)
const [unreadCount, setUnreadCount] = useState(0)
const [showScrollButton, setShowScrollButton] = useState(false)

// スクロール位置の監視
const handleScroll = (e: React.UIEvent<HTMLDivElement>) => {
  const { scrollTop, scrollHeight, clientHeight } = e.currentTarget
  const atBottom = scrollHeight - scrollTop - clientHeight < 50
  setIsAtBottom(atBottom)

  if (atBottom) {
    // 最下部に到達 → 未読をクリア
    setUnreadCount(0)
    setShowScrollButton(false)
    markAllAsRead()
  }
}
```

### 新着メッセージの既読判定

```typescript
const handleNewMessage = (message: Message) => {
  if (isAtBottom) {
    // 最下部にいる → 自動既読 + 自動スクロール
    markAsRead(message.id)
    scrollToBottom()
  } else {
    // 上部にいる → 未読カウント + ↓ボタン表示
    setUnreadCount((prev) => prev + 1)
    setShowScrollButton(true)
  }
}
```

### 「↓新着N件」ボタン

```tsx
{showScrollButton && (
  <button
    onClick={() => {
      scrollToBottom()
      setUnreadCount(0)
      setShowScrollButton(false)
      markAllAsRead()
    }}
    className="fixed bottom-20 right-4 bg-primary-500 text-white
               px-4 py-2 rounded-full shadow-lg z-30"
  >
    ↓ 新着 {unreadCount}件
  </button>
)}
```

### チャンネル切り替え時のリセット

これが一番ハマったバグです。**チャンネルを切り替えたとき、前のチャンネルの`isAtBottom`状態が残っていた。**

前のチャンネルで上にスクロールした状態（`isAtBottom = false`）のまま新しいチャンネルに切り替えると、新しいチャンネルでも最下部にいるのに未読判定が「上部にいる」になり、未読バッジが消えない。

```typescript
// チャンネル切り替え時にリセット
useEffect(() => {
  setIsAtBottom(true)
  setUnreadCount(0)
  setShowScrollButton(false)
  scrollToBottom()
}, [currentChannelId])
```

**状態のリセット漏れ**は、チャット系アプリでは定番のバグパターンです。チャンネルIDが変わったら、スクロール関連の状態は全てリセットが鉄則。

## まとめ

| 改善 | 実装のポイント |
|---|---|
| URL自動変換 | `onChange`で差分検出（iOSのonPaste問題回避） |
| Markdownリンク | 正規表現でパース + Reactノードに変換 |
| 未読管理 | `isAtBottom`フラグで既読判定を分岐 |
| チャンネル切替 | 状態リセット漏れに注意 |

地味な改善ばかりですが、毎日使うツールでは**この地味さが効く**。URL共有と未読管理が安定したことで、チーム内の「あの情報どこ？」「読んだのに未読になってる」が解消されました。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
