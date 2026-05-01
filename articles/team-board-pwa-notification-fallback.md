---
title: "チームボード開発記 #4 — PWAプッシュ通知が届かない問題を解決した（フォールバック設計）"
emoji: "🔔"
type: "tech"
topics: ["PWA", "ServiceWorker", "WebPush", "NextJS", "TypeScript"]
published: true
---

## はじめに

自作チームボード（[前回記事](https://zenn.dev/orectic/articles/custom-team-board-nextjs-supabase)）にPWAプッシュ通知を実装し、チームに展開したところ「**通知が来ないときがある**」という報告が相次ぎました。

調査の結果、Web Push APIの購読状態とService Workerのライフサイクルが複雑に絡み合っていることが原因でした。

本記事では、PWAプッシュ通知の**フォールバック設計**と、実装中に遭遇した**2重通知バグの原因と修正**を解説します。

## 前提

| 項目 | 詳細 |
|---|---|
| フレームワーク | Next.js 14（App Router） |
| 通知方式 | Web Push API + Service Worker |
| 対象 | PWAとしてインストール済み or ブラウザで開いている状態 |
| バックエンド | Supabase + Next.js API Routes |

## 通知が届かないケースの整理

PWAプッシュ通知が届かない原因は主に3つありました。

| ケース | 原因 | 発生頻度 |
|---|---|---|
| Service Workerが未登録 | ブラウザのストレージクリア・PWA未インストール | 時々 |
| プッシュ購読の期限切れ | ブラウザ/OS側の自動失効 | 稀 |
| 通知権限が未許可 | ユーザーが許可していない・ブロック済み | 初回のみ |

特に「Service Workerが未登録だが、アプリはブラウザで開いている」というケースが問題でした。Web Push APIは使えないが、ブラウザのNotification APIは使える状態です。

## フォールバック設計

以下の優先順位で通知を送ります。

```
1. PWAプッシュ通知（Web Push API）
   ↓ 購読なし
2. ブラウザ通知（Notification API）
   ↓ 権限なし
3. アプリ内通知（通知パネル + ファビコンバッジ）
```

### 実装

```typescript
const sendNotification = async (
  title: string,
  body: string,
  userId: string
) => {
  // 1. PWAプッシュ購読があるか確認
  const subscription = await getPushSubscription(userId)

  if (subscription) {
    // PWAプッシュ通知を送信（サーバーサイド）
    await fetch("/api/push-send", {
      method: "POST",
      body: JSON.stringify({ subscription, title, body }),
    })
    return
  }

  // 2. ブラウザ通知にフォールバック
  if ("Notification" in window && Notification.permission === "granted") {
    new Notification(title, {
      body,
      icon: "/icon-192x192.png",
    })
    return
  }

  // 3. アプリ内通知は別ルートで常に処理される（Supabase Realtime経由）
}
```

### サーバーサイドのプッシュ送信

```typescript
// /api/push-send/route.ts
import webpush from "web-push"

webpush.setVapidDetails(
  "mailto:your@email.com",
  process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
)

export async function POST(request: Request) {
  const { subscription, title, body } = await request.json()

  try {
    await webpush.sendNotification(
      subscription,
      JSON.stringify({ title, body })
    )
    return Response.json({ success: true })
  } catch (error: unknown) {
    // 購読が失効していた場合、DBから削除
    if (error instanceof webpush.WebPushError && error.statusCode === 410) {
      await deleteSubscription(subscription.endpoint)
    }
    return Response.json({ success: false }, { status: 500 })
  }
}
```

**410 Goneのハンドリングが重要です。** プッシュ購読はブラウザやOS側で予告なく失効することがあります。410が返ってきたらDBから購読情報を削除し、次回からフォールバックに回るようにします。

## 2重通知の罠

### 発生した問題

フォールバック実装後、「**同じ通知が2回来る**」というバグが発生しました。

### 最初の（間違った）実装

```typescript
// NG: Service Workerのcontrollerで判定していた
const sendNotification = async (title: string, body: string, userId: string) => {
  if (navigator.serviceWorker?.controller) {
    // SWが制御中 → プッシュ通知
    await sendPushNotification(userId, title, body)
  } else {
    // SWなし → ブラウザ通知
    new Notification(title, { body })
  }
}
```

### 原因

Service Workerには以下のライフサイクルがあります。

```
インストール → アクティベーション → 制御開始（controller !== null）
```

問題は、**PWAプッシュ購読は持っているが、`navigator.serviceWorker.controller`がまだnull**というタイミングが存在することでした。

1. ユーザーがPWAをインストール済み → プッシュ購読がDBにある
2. ページを開いた直後、SWがまだアクティベーション中 → `controller`はnull
3. 上記のコードはブラウザ通知を送る
4. 直後にSWがアクティブになり、サーバーからのプッシュも届く
5. **2重通知**

### 修正

`controller`ではなく、**プッシュ購読の有無**で判定するように変更しました。

```typescript
// OK: 購読の有無で判定
const sendNotification = async (title: string, body: string, userId: string) => {
  const subscription = await getPushSubscription(userId)

  if (subscription) {
    // 購読あり → プッシュ通知のみ（SWの状態に関わらず）
    await sendPushNotification(userId, title, body)
    return
  }

  // 購読なし → ブラウザ通知
  if ("Notification" in window && Notification.permission === "granted") {
    new Notification(title, { body })
  }
}
```

購読がDBに存在する = いずれプッシュ通知が届く。SWの`controller`がnullでも、購読が有効ならプッシュだけ送り、ブラウザ通知は送らない。これで2重通知が解消しました。

## PWAバッジのIndexedDB永続化

### 課題

ファビコンとタブタイトルで未読バッジを表示していましたが、PWAがバックグラウンドに回ると状態が消えてしまう問題がありました。

### 解決策

IndexedDBでバッジカウントを永続化し、フォアグラウンド復帰時に復元します。

```typescript
// バッジカウントの永続化
const saveBadgeCount = async (count: number) => {
  const db = await openDB("team-board", 1, {
    upgrade(db) {
      db.createObjectStore("state")
    },
  })
  await db.put("state", count, "badgeCount")
}

// フォアグラウンド復帰時に復元
const restoreBadgeCount = async (): Promise<number> => {
  const db = await openDB("team-board", 1)
  const count = await db.get("state", "badgeCount")
  return count ?? 0
}

// visibilitychange で復元
document.addEventListener("visibilitychange", async () => {
  if (document.visibilityState === "visible") {
    const count = await restoreBadgeCount()
    updateFaviconBadge(count)
    updateTitleBadge(count)
  }
})
```

PWAのBadging APIが使える環境では`navigator.setAppBadge(count)`も併用します。

```typescript
const updateBadge = async (count: number) => {
  // IndexedDBに永続化
  await saveBadgeCount(count)

  // ファビコン + タブタイトル
  updateFaviconBadge(count)
  updateTitleBadge(count)

  // PWA Badging API（対応環境のみ）
  if ("setAppBadge" in navigator) {
    if (count > 0) {
      await navigator.setAppBadge(count)
    } else {
      await navigator.clearAppBadge()
    }
  }
}
```

## 通知設定画面

チームメンバーごとに通知の受信設定を用意しています。

```typescript
type NotificationSettings = {
  readonly pushEnabled: boolean     // プッシュ通知のON/OFF
  readonly mentionOnly: boolean     // @メンションのみ通知
  readonly muteChannels: string[]   // ミュートチャンネル一覧
}
```

全体のON/OFFだけでなく、**メンションのみ**や**チャンネル単位のミュート**ができるようにしたことで、「通知が多すぎる」という声がなくなりました。

## まとめ

| 問題 | 解決策 |
|---|---|
| 通知が届かない | PWAプッシュ → ブラウザ通知の段階的フォールバック |
| 2重通知 | SWのcontrollerではなく購読の有無で判定 |
| 購読の失効 | 410 Goneで自動クリーンアップ |
| バッジが消える | IndexedDBで永続化 + visibilitychangeで復元 |

PWA通知の実装で一番ハマるのは**Service Workerのライフサイクルと通知経路の不整合**です。「通知が来ない」「2回来る」は同じ原因の表と裏で、**購読の有無を唯一の判定基準にする**ことで両方解消できます。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=team-board-pwa-notification-fallback) で活動中。
[SEO CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=team-board-pwa-notification-fallback) — ディレクターも使っているSEO診断ツールを公開中。
