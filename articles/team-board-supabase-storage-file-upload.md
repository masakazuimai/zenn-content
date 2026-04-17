---
title: "Supabase Storageでチャットアプリにファイル添付を実装した話 — 日本語ファイル名の罠と自動クリーンアップ"
emoji: "📎"
type: "tech"
topics: ["Supabase", "NextJS", "TypeScript", "ファイルアップロード", "個人開発"]
published: true
---

## はじめに

Next.js + Supabaseで自作したチームボード（[前回記事](https://zenn.dev/orectic/articles/custom-team-board-nextjs-supabase)）を実運用していて、最初に出た要望が「**ファイルを送れるようにしてほしい**」でした。

初期実装では画像をBase64でDBに直接埋め込んでいましたが、データベースが急速に肥大化。Supabase Storageへの移行を決断しました。

本記事では、Supabase Storageを使ったファイル添付機能の実装を解説します。途中で踏んだ**日本語ファイル名のエンコード問題**と、メッセージ削除時の**ファイル自動クリーンアップ**の設計も含めて紹介します。

## 技術構成

| 項目 | 選定 |
|---|---|
| フロントエンド | Next.js 14（App Router） |
| ストレージ | Supabase Storage |
| DB | Supabase（PostgreSQL） |
| ファイルサイズ上限 | 50MB |

## Base64埋め込みの問題

最初の実装では画像をBase64に変換し、メッセージのJSON内に直接保存していました。

```typescript
// 初期実装（NG）: Base64をDBに保存
const fileData = await toBase64(file)
await supabase.from("messages").insert({
  text: "画像を送ります",
  file: { name: file.name, data: fileData } // 巨大なBase64文字列
})
```

小さい画像なら問題ありませんが、1MBの画像でBase64は約1.37MBの文字列になります。チームで1日10枚画像を送るだけで、月に400MB以上DBが膨らむ計算です。

Supabase無料枠のDB容量は500MB。**運用1ヶ月で枠を使い切るペース**でした。

## Supabase Storageへの移行

### バケットの作成

Supabase Dashboardで`attachments`バケットを作成します。公開バケットにして、署名なしでURLアクセスできるようにしました（チーム内ツールのため）。

```sql
-- RLS ポリシー例
CREATE POLICY "Authenticated users can upload"
ON storage.objects FOR INSERT
TO authenticated
WITH CHECK (bucket_id = 'attachments');

CREATE POLICY "Anyone can view"
ON storage.objects FOR SELECT
TO public
USING (bucket_id = 'attachments');
```

### アップロード処理

```typescript
const uploadFile = async (file: File, channelId: string): Promise<string> => {
  const fileId = crypto.randomUUID()
  const ext = file.name.split(".").pop()
  const path = `${channelId}/${fileId}.${ext}`

  const { error } = await supabase.storage
    .from("attachments")
    .upload(path, file, { upsert: false })

  if (error) throw new Error(`Upload failed: ${error.message}`)

  const { data } = supabase.storage
    .from("attachments")
    .getPublicUrl(path)

  return data.publicUrl
}
```

メッセージにはURLだけを保存します。

```typescript
await supabase.from("messages").insert({
  text: "画像を送ります",
  file: { name: file.name, url: publicUrl } // URLだけ
})
```

DB容量の増加は、1メッセージあたり数百バイトのURL文字列のみ。ファイル本体はStorageに置くので、**DB肥大化の問題が解消**されました。

## 日本語ファイル名の罠

### 問題

当初は `channelId/元ファイル名` というパスでアップロードしていました。

```typescript
// 最初の実装（NG）
const path = `${channelId}/${file.name}`
```

すると日本語ファイル名で以下の問題が発生しました。

```
アップロード時のパス: attachments/ch-001/議事録_2026-04.pdf
公開URL: attachments/ch-001/%E8%AD%B0%E4%BA%8B%E9%8C%B2_2026-04.pdf
```

Supabase StorageのAPIが返すパスと、公開URLのエンコード済みパスが一致しないケースがあり、**ファイルの削除や参照でエラーが起きる**ことがありました。

### 解決策：IDベースのパス

ファイル名にUUIDを使い、元のファイル名はメッセージのメタデータとして保持します。

```typescript
const fileId = crypto.randomUUID()
const ext = file.name.split(".").pop()
const path = `${channelId}/${fileId}.${ext}`
// → attachments/ch-001/a1b2c3d4-e5f6-7890-abcd-ef1234567890.pdf
```

```typescript
// メッセージには元のファイル名を保持（表示用）
file: {
  name: file.name,        // "議事録_2026-04.pdf"（表示用）
  url: publicUrl,          // IDベースのURL（参照用）
}
```

**パスにはASCII文字のみ使う。** これはSupabase Storageに限らず、クラウドストレージ全般で有効なプラクティスです。

## ファイルサイズ制限

50MBの制限を設け、超過時はGoogle Drive等の外部共有を案内しています。

```typescript
const MAX_FILE_SIZE = 50 * 1024 * 1024 // 50MB

const handleFileSelect = (file: File) => {
  if (file.size > MAX_FILE_SIZE) {
    showToast(
      "50MBを超えるファイルはGoogleドライブ等で共有してください",
      "error"
    )
    return
  }
  uploadFile(file, currentChannelId)
}
```

50MBにした理由は、Supabase無料枠のStorage容量（1GB）と、チームの実際の利用頻度から逆算しました。週に数回の資料共有なら、1GBあれば数ヶ月は持ちます。

## Base64からの後方互換

既存のBase64メッセージが壊れないよう、表示側で両方のフォーマットに対応しています。

```tsx
const FilePreview = ({ file }: { file: MessageFile }) => {
  // URL形式（新）
  if (file.url) {
    return isImageFile(file.name)
      ? <img src={file.url} alt={file.name} className="max-w-xs rounded-lg" />
      : <a href={file.url} download={file.name}>{file.name}</a>
  }

  // Base64形式（旧）— 後方互換
  if (file.data) {
    return <img src={`data:image/*;base64,${file.data}`} alt={file.name} />
  }

  return null
}
```

## メッセージ削除時の自動クリーンアップ

メッセージを削除したとき、添付ファイルがStorageに残り続けるのは無駄です。

```typescript
const deleteMessage = async (message: Message) => {
  // 添付ファイルがあればStorageから削除
  if (message.file?.url) {
    const path = extractPathFromUrl(message.file.url)
    await supabase.storage.from("attachments").remove([path])
  }

  // メッセージ本体を削除
  await supabase.from("messages").delete().eq("id", message.id)
}

// URLからStorageパスを抽出
const extractPathFromUrl = (url: string): string => {
  const storageUrl = supabase.storage.from("attachments").getPublicUrl("").data.publicUrl
  return url.replace(storageUrl, "")
}
```

**注意点：** Storageの削除に失敗してもメッセージの削除は続行します。孤立ファイルが残る可能性はありますが、メッセージが消えないよりマシです。定期的なクリーンアップバッチを入れる場合は、messagesテーブルに存在しないファイルを検索すれば対応できます。

## アバター画像

同じ仕組みで、ユーザーアバターもSupabase Storageに保存しています。

```typescript
const uploadAvatar = async (
  accountId: string,
  file: File
): Promise<string> => {
  const path = `avatars/${accountId}`

  // upsertで既存アバターを上書き
  const { error } = await supabase.storage
    .from("attachments")
    .upload(path, file, { upsert: true })

  if (error) throw new Error(`Avatar upload failed: ${error.message}`)

  const { data } = supabase.storage
    .from("attachments")
    .getPublicUrl(path)

  await supabase
    .from("accounts")
    .update({ avatar_url: data.publicUrl })
    .eq("id", accountId)

  return data.publicUrl
}
```

アバターは1ユーザー1ファイルなので、パスに`accountId`をそのまま使い、`upsert: true`で上書きします。UUID方式のファイル添付とは設計が異なるポイントです。

表示側は、アバターURLがあれば画像、なければ頭文字のフォールバック。

```tsx
{account.avatarUrl ? (
  <img
    src={account.avatarUrl}
    alt={account.name}
    className="w-8 h-8 rounded-full object-cover"
  />
) : (
  <div
    className="w-8 h-8 rounded-full flex items-center justify-center text-white"
    style={{ backgroundColor: colorPalette[account.colorIndex] }}
  >
    {account.name.charAt(0)}
  </div>
)}
```

## まとめ

| 問題 | 解決策 |
|---|---|
| DB肥大化 | Supabase StorageにファイルURLだけ保存 |
| 日本語ファイル名 | UUIDベースのパスに変更 |
| 孤立ファイル | メッセージ削除時に自動クリーンアップ |
| Base64互換 | 表示側で両フォーマット対応 |

Supabase Storageは無料枠でも1GB使えるので、小規模チームのファイル共有には十分です。実装で一番ハマったのは日本語ファイル名の問題で、**パスにASCII以外を使わない**というルールを最初から決めておくと事故を防げます。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/) で活動中。
[SEO CHECK](https://seo.codequest.work/) — ディレクターも使っているSEO診断ツールを公開中。
