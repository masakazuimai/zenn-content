---
title: "Reactで長押しアクションシートを実装する — スマホチャットアプリのUX改善"
emoji: "📱"
type: "tech"
topics: ["React", "TypeScript", "モバイル", "UX", "PWA"]
published: false
publish_date: 2026-04-21
---

## はじめに

自作チームボード（[前回記事](https://zenn.dev/orectic/articles/custom-team-board-nextjs-supabase)）をスマホで使ったとき、致命的な問題がありました。

**メッセージに対する操作が何もできない。**

PC版ではホバーでアクションバー（返信・リアクション・編集・削除）が表示されますが、スマホにホバーはありません。リアクションも返信もできず、テキストを読むだけのアプリになっていました。

LINEやiMessageのように**長押しでメニューを出す**方式を実装し、スマホでも快適に操作できるようにした実装を紹介します。

## 完成形のイメージ

メッセージを500ms長押しすると、画面下部からアクションシートがスライドイン。

| アクション | 説明 |
|---|---|
| リアクション | 絵文字6種（👍❤️😊🎉👀✅）から選択 |
| 返信 | スレッド返信 |
| ブックマーク | 後で見返す |
| ピン留め | チャンネルに固定 |
| 全文コピー | テキストをクリップボードに |
| テキスト選択 | 部分選択モードに切替 |
| 編集 | 自分のメッセージのみ表示 |
| 削除 | 自分のメッセージのみ表示 |

## 長押し検出の実装

`touchstart`でタイマーを開始し、500ms経過したらアクションシートを表示。`touchend`/`touchmove`でキャンセルします。

```typescript
const longPressTimer = useRef<NodeJS.Timeout | null>(null)
const touchStartPos = useRef<{ x: number; y: number } | null>(null)

const handleTouchStart = (e: React.TouchEvent, message: Message) => {
  const touch = e.touches[0]
  touchStartPos.current = { x: touch.clientX, y: touch.clientY }

  longPressTimer.current = setTimeout(() => {
    setActionSheetMessage(message)
    setShowActionSheet(true)
  }, 500)
}

const handleTouchMove = (e: React.TouchEvent) => {
  if (!touchStartPos.current || !longPressTimer.current) return

  const touch = e.touches[0]
  const dx = Math.abs(touch.clientX - touchStartPos.current.x)
  const dy = Math.abs(touch.clientY - touchStartPos.current.y)

  // 10px以上動いたらスクロールと判定してキャンセル
  if (dx > 10 || dy > 10) {
    clearTimeout(longPressTimer.current)
    longPressTimer.current = null
  }
}

const handleTouchEnd = () => {
  if (longPressTimer.current) {
    clearTimeout(longPressTimer.current)
    longPressTimer.current = null
  }
}
```

**`touchmove`でのキャンセルが重要です。** これがないと、スクロール中に指が止まった瞬間にアクションシートが開いてしまいます。10pxの閾値で「タップの微小なブレ」と「スクロール」を区別しています。

## iOSネイティブメニューの抑制

長押しするとiOSのネイティブ選択メニュー（コピー・調べる・共有）が出てきます。これを抑制しないと、自前のアクションシートとネイティブメニューが重なります。

```css
.message-item {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}
```

ただし、これだけだと**テキストのコピーが一切できなくなります**。そこでアクションシートに「全文コピー」と「テキスト選択」の2つのアクションを用意しました。

### 全文コピー

```typescript
const handleCopyText = async (message: Message) => {
  await navigator.clipboard.writeText(message.text)
  showToast("コピーしました")
  setShowActionSheet(false)
}
```

### テキスト選択モード

```typescript
const [selectableMessageId, setSelectableMessageId] = useState<string | null>(null)

const handleEnableSelect = (message: Message) => {
  setSelectableMessageId(message.id)
  setShowActionSheet(false)
}
```

```tsx
<div
  className={`message-item ${
    selectableMessageId === message.id ? "select-mode" : ""
  }`}
>
  {message.text}
</div>
```

```css
.message-item.select-mode {
  -webkit-touch-callout: default;
  -webkit-user-select: text;
  user-select: text;
}
```

テキスト選択モードに入ると、そのメッセージだけ`user-select: text`に切り替わり、通常通りテキスト選択ができるようになります。

## アクションシートコンポーネント

画面下部からスライドインするシートのUI実装です。

```tsx
const ActionSheet = ({
  message,
  currentUser,
  onClose,
  onReaction,
  onReply,
  onBookmark,
  onPin,
  onCopy,
  onSelect,
  onEdit,
  onDelete,
}: ActionSheetProps) => {
  const isOwnMessage = message.user.accountId === currentUser.accountId

  return (
    <>
      {/* 背景オーバーレイ */}
      <div
        className="fixed inset-0 bg-black/30 z-40"
        onClick={onClose}
      />

      {/* シート本体 */}
      <div className="fixed bottom-0 left-0 right-0 bg-white rounded-t-2xl z-50
                      animate-slide-up safe-area-bottom">
        {/* リアクション行 */}
        <div className="flex justify-around px-4 py-3 border-b border-neutral-200">
          {["👍", "❤️", "😊", "🎉", "👀", "✅"].map((emoji) => (
            <button
              key={emoji}
              onClick={() => onReaction(emoji)}
              className="text-2xl p-2 rounded-full hover:bg-neutral-100
                         active:bg-neutral-200 transition-colors"
            >
              {emoji}
            </button>
          ))}
        </div>

        {/* アクション一覧 */}
        <div className="py-2">
          <ActionItem label="返信" onClick={onReply} />
          <ActionItem label="ブックマーク" onClick={onBookmark} />
          <ActionItem label="ピン留め" onClick={onPin} />
          <ActionItem label="全文コピー" onClick={onCopy} />
          <ActionItem label="テキスト選択" onClick={onSelect} />
          {isOwnMessage && (
            <>
              <ActionItem label="編集" onClick={onEdit} />
              <ActionItem
                label="削除"
                onClick={onDelete}
                variant="danger"
              />
            </>
          )}
        </div>

        {/* キャンセル */}
        <div className="px-4 pb-4">
          <button
            onClick={onClose}
            className="w-full py-3 rounded-xl bg-neutral-100 text-neutral-700
                       font-medium active:bg-neutral-200"
          >
            キャンセル
          </button>
        </div>
      </div>
    </>
  )
}

const ActionItem = ({
  label,
  onClick,
  variant = "default",
}: {
  label: string
  onClick: () => void
  variant?: "default" | "danger"
}) => (
  <button
    onClick={onClick}
    className={`w-full text-left px-6 py-3 active:bg-neutral-100
      ${variant === "danger" ? "text-red-600" : "text-neutral-900"}`}
  >
    {label}
  </button>
)
```

### スライドインアニメーション

```css
@keyframes slide-up {
  from {
    transform: translateY(100%);
  }
  to {
    transform: translateY(0);
  }
}

.animate-slide-up {
  animation: slide-up 0.25s ease-out;
}

/* iPhoneのセーフエリア対応 */
.safe-area-bottom {
  padding-bottom: env(safe-area-inset-bottom);
}
```

## PC / スマホの出し分け

PCではホバーアクションバー、スマホではタッチイベントで切り替えます。

```tsx
<div
  className="group relative"
  onTouchStart={(e) => handleTouchStart(e, message)}
  onTouchMove={handleTouchMove}
  onTouchEnd={handleTouchEnd}
>
  {/* メッセージ本文 */}
  <MessageContent message={message} />

  {/* PC: ホバーで表示 — md以上のみ */}
  <div className="hidden md:group-hover:flex absolute -top-3 right-2
                  bg-white border rounded-lg shadow-sm">
    <HoverActionBar message={message} />
  </div>
</div>
```

Tailwindの`md:`ブレイクポイントで切り替え。768px未満ではホバーアクションバーは非表示で、長押しのみ。768px以上ではホバーが使えるので両方動作します。

## リアクション絵文字の常時展開

もう1つ改善したのが、リアクション絵文字パネルの表示方式です。

従来はホバーで展開していましたが、スマホ対応を機に**常時展開**に変更しました。

```tsx
{/* 改善前: ホバーで展開 */}
<div className="hidden group-hover:flex">
  {emojis.map(/* ... */)}
</div>

{/* 改善後: 常時展開 */}
<div className="flex gap-1 mt-1">
  {message.reactions.map((reaction) => (
    <button
      key={reaction.emoji}
      onClick={() => toggleReaction(message.id, reaction.emoji)}
      className="flex items-center gap-1 px-2 py-0.5 rounded-full
                 bg-neutral-100 hover:bg-neutral-200"
    >
      <span>{reaction.emoji}</span>
      <span className="text-neutral-600">{reaction.count}</span>
    </button>
  ))}
</div>
```

常時展開にすることで：
- スマホでもPCでも同じ見た目・操作感
- 「誰がどのリアクションをしたか」が一目で分かる
- ホバー前提のUIが減り、アクセシビリティも向上

## スマホ横スクロールの修正

長いURLやコードを含むメッセージで、画面全体が横スクロールしてしまう問題もありました。

```css
.message-content {
  overflow-x: hidden;
  max-width: 100%;
  word-break: break-word;
}

.message-content pre {
  overflow-x: auto;
  max-width: 100%;
}
```

メッセージ全体は横スクロール禁止、コードブロック内だけスクロール可能にします。

## まとめ

| 改善 | 実装 |
|---|---|
| 長押し検出 | touchstart + 500msタイマー + touchmoveキャンセル |
| iOSメニュー抑制 | CSSの`-webkit-touch-callout: none` |
| テキストコピー | 全文コピー + テキスト選択モードで代替 |
| PC/スマホ切替 | Tailwindのmd:ブレイクポイントで出し分け |
| リアクション | ホバー展開 → 常時展開 |

スマホ対応で一番苦労したのはiOSのネイティブメニューとの共存でした。完全に抑制するとテキスト選択ができなくなるので、「選択モード」という逃げ道を用意するのがポイントです。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
