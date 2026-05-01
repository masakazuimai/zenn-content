---
title: "個人開発iOSヘルスケアアプリのセキュリティ設計 ― 痛風管理アプリで実践した9つの防御策"
emoji: "🛡️"
type: "tech"
topics: ["iOS", "Swift", "Security", "SwiftUI", "StoreKit"]
published: true
---

## はじめに

痛風・高尿酸血症の食事管理に特化したiOSアプリ「痛風管理（TsuKanCalendar）」を個人開発しています。健康データを扱うアプリは攻撃対象になりやすく、個人開発でもセキュリティ設計は避けて通れません。

https://codequest.work/gout-app-tsukankalendar-v200/?utm_source=zenn&utm_medium=article&utm_campaign=ios-health-app-security-tsukankalendar

この記事では、約2,600行・18ファイルのSwiftUIアプリで実践した**9つのセキュリティ設計判断**を、実コードとともに解説します。

## アプリの概要

| 項目 | 内容 |
|------|------|
| アーキテクチャ | SwiftUI + MVVM |
| データ永続化 | UserDefaults + iCloud (NSUbiquitousKeyValueStore) |
| 課金 | StoreKit 2（広告削除 ¥200） |
| 広告 | Google AdMob |
| 外部API | なし（完全オフライン動作） |

## 1. バックエンドを持たない設計 ― 最大の防御

最も効果的なセキュリティ対策は**攻撃面（Attack Surface）を減らすこと**です。

このアプリはバックエンド サーバーを一切持ちません。食品データベースはバンドル内のJSONファイル、ユーザーデータはデバイスローカル＋iCloudのみです。

```swift
// 食品データはバンドル内JSONから読み込み（ネットワーク不要）
private func loadJSON() {
    guard let url = Bundle.main.url(
        forResource: "food_database", withExtension: "json"
    ),
    let data = try? Data(contentsOf: url),
    let decoded = try? JSONDecoder().decode(
        [FoodCategory].self, from: data
    ) else {
        logger.error("食品データの読み込みに失敗")
        return
    }
    categories = decoded
}
```

**これにより排除できる脅威:**

- SQLインジェクション、APIの認証バイパス → サーバーがないので発生しない
- 中間者攻撃（MITM） → ユーザーデータの通信が発生しない
- サーバー侵害によるデータ漏洩 → サーバーがないので不可能
- DDoS攻撃 → 対象がない

:::message
「バックエンド不要なアプリにバックエンドを作らない」は当たり前に聞こえますが、「将来のため」にAPIサーバーを立てがちな個人開発者は多いです。不要なインフラは攻撃面そのものです。
:::

## 2. StoreKit 2のトランザクション検証 ― 課金バイパスの防止

課金処理は最も狙われるポイントです。StoreKit 2の`Transaction.verified`を使い、**Appleが署名した正規トランザクションのみ**を信頼します。

```swift
func purchase() {
    guard let product = products.first(where: {
        $0.id == productID
    }) else { return }

    Task {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            // ✅ Apple署名の検証済みトランザクションのみ受け入れ
            if case .verified = verification {
                grantEntitlement()
            } else {
                // ❌ 検証失敗 → 不正な購入として拒否
                alertMessage = "購入の検証に失敗しました"
            }
        case .userCancelled:
            break
        case .pending:
            alertMessage = "購入が保留中です"
        @unknown default:
            alertMessage = "予期しないエラーが発生しました"
        }
    }
}
```

さらに、アプリ起動時に`Transaction.currentEntitlements`で購入状態を再検証しています。

```swift
func checkEntitlements() async {
    for await result in Transaction.currentEntitlements {
        if case .verified(let transaction) = result,
           transaction.productID == productID {
            grantEntitlement()
            return
        }
    }
    // エンタイトルメントなし → 権限を無効化
    revokeEntitlement()
}
```

**ポイント:**

- `.unverified`を信頼しない（改ざんされたレシートを拒否）
- 起動時に毎回検証（ローカルストレージの改ざんに対応）
- サーバーサイド検証なしでも、StoreKit 2のデバイスレベル署名検証で十分な信頼性

:::message alert
StoreKit 1時代は`SKPaymentTransaction`のレシートをサーバーで検証する必要がありました。StoreKit 2では`VerificationResult`がJWS（JSON Web Signature）でAppleが署名しており、クライアントだけで安全に検証できます。
:::

## 3. ビルド環境による広告IDの分離

広告ユニットIDの漏洩自体は直接的な脅威ではありませんが、本番IDがデバッグビルドに混入すると**テストクリックによるAdMobアカウントBAN**のリスクがあります。

```swift
static var adUnitID: String {
    #if DEBUG
    // デバッグビルド → テスト用ID
    return AdConstants.testUnitID
    #else
    if Bundle.main.appStoreReceiptURL?
        .lastPathComponent == "sandboxReceipt" {
        // TestFlight → テスト用ID
        return AdConstants.testUnitID
    } else {
        // App Store → 本番ID
        return AdConstants.productionUnitID
    }
    #endif
}
```

**3段階の分離:**

| 環境 | 判定方法 | 使用ID |
|------|----------|--------|
| Xcode Debug | `#if DEBUG` コンパイラフラグ | テスト |
| TestFlight | `sandboxReceipt` チェック | テスト |
| App Store | 上記以外 | 本番 |

`#if DEBUG`はコンパイル時に評価されるため、リリースバイナリにテストID判定コードが含まれません。TestFlightの判定は`appStoreReceiptURL`のパスで行い、ランタイムで正確に環境を識別します。

## 4. 入力バリデーション ― ユーザー入力を信頼しない

ユーザーがプリン体量を手動入力できる機能では、必ずバリデーションを行います。

```swift
// 手動データ追加時のバリデーション
func addItem() {
    // ✅ 空文字チェック + 数値変換チェック
    guard let value = Int(inputText), !nameText.isEmpty else {
        return  // 不正な入力は無視
    }
    let newItem = DataItem(name: nameText, value: value)
    // ...
}
```

数値入力がある画面すべてで同様のパターンを適用しています。

```swift
// カスタム数値入力
guard let amount = Int(customInput), amount > 0 else { return }
```

**バリデーション設計方針:**

- `Int()`/`Double()`の型変換失敗を`guard`で早期リターン
- 空文字・ゼロ以下の値を拒否
- 不正入力時はエラーメッセージではなく**静かに無視**（攻撃者に情報を与えない）

:::message
iOSアプリではWebほどインジェクション攻撃のリスクは低いですが、不正な値がデータベースに混入すると表示崩れやクラッシュの原因になります。「信頼境界（Trust Boundary）はUI入力」と考えて設計しています。
:::

## 5. iCloudデータ同期の安全設計

UserDefaultsとiCloudの双方向同期では、**同期対象キーをホワイトリスト方式で限定**しています。

```swift
// 同期対象キーの判定（ホワイトリスト方式）
private func shouldSyncKey(_ key: String) -> Bool {
    allowedPrefixes.contains(where: { key.hasPrefix($0) }) ||
    allowedExactKeys.contains(key)
}
```

`allowedPrefixes`と`allowedExactKeys`に同期すべきキーだけを列挙し、それ以外は一切同期しません。

これにより以下を防ぎます。

- **意図しないデータの同期:** アプリ内部状態やデバッグフラグがiCloudに漏洩しない
- **同期ストーム:** 無関係なキーの変更がiCloud通知を連鎖させない
- **容量超過:** `NSUbiquitousKeyValueStore`の1MBクォータを無駄に消費しない

iCloud変更通知のハンドリングでは、変更理由（`reason`）を精査して適切に対応します。

```swift
@objc private func cloudDidChange(_ notification: Notification) {
    guard let userInfo = notification.userInfo,
          let reason = userInfo[
              NSUbiquitousKeyValueStoreChangeReasonKey
          ] as? Int else { return }

    switch reason {
    case NSUbiquitousKeyValueStoreServerChange,
         NSUbiquitousKeyValueStoreInitialSyncChange:
        syncFromCloud()
    case NSUbiquitousKeyValueStoreQuotaViolationChange:
        logger.warning("iCloudストレージ容量超過")
        // ⚠️ 容量超過時は同期せず警告のみ
    case NSUbiquitousKeyValueStoreAccountChange:
        syncFromCloud()
    default:
        break
    }
}
```

## 6. HealthKit権限の最小化原則

HealthKitは**最小権限の原則（Principle of Least Privilege）** を厳格に適用しています。

```swift
func requestAuthorization() async {
    guard isAvailable else { return }

    // 読み取り: 体重とBMIのみ
    let readTypes: Set<HKObjectType> = [
        HKObjectType.quantityType(forIdentifier: .bodyMass)!,
        HKObjectType.quantityType(forIdentifier: .bodyMassIndex)!,
    ]

    // 書き込み: 体重のみ（BMIは読み取り専用）
    let writeTypes: Set<HKSampleType> = [
        HKSampleType.quantityType(forIdentifier: .bodyMass)!,
    ]

    do {
        try await healthStore.requestAuthorization(
            toShare: writeTypes, read: readTypes
        )
        isAuthorized = true
    } catch {
        logger.error("HealthKit認証失敗: \(error.localizedDescription)")
    }
}
```

**設計判断:**

| 判断 | 理由 |
|------|------|
| 読み取りは体重・BMIのみ | 痛風管理に必要最小限のデータ |
| 書き込みは体重のみ | BMI書き込みは不要（体重から自動計算される） |
| 歩数・心拍等は要求しない | 機能に不要なデータへのアクセスはApp Store審査でもリジェクト対象 |

:::message
HealthKitの権限は一度許可すると「設定 > ヘルスケア > データアクセス」からしか取り消せません。ユーザーの信頼を裏切らないよう、必要最小限のデータのみ要求することが重要です。
:::

## 7. スレッドセーフティ ― データ競合の防止

`@MainActor`を全ViewModelに適用し、データ競合を構造的に防止しています。

```swift
@MainActor
final class TrackingViewModel: ObservableObject {
    @Published var items: [TrackedItem] = []
    @Published private(set) var dailySummary: [String: Int] = [:]
    // ...
}
```

**`@MainActor`がセキュリティに効く理由:**

- `@Published`プロパティへの同時書き込みを防止 → データ破損を回避
- iCloud同期の変更通知がメインスレッドで処理される → UI状態の一貫性を保証
- 課金状態の更新が直列化される → 中間状態を排除

さらにイミュータブルパターンの徹底で、参照経由の意図しない変更を防いでいます。

```swift
// ❌ ミュータブルな更新（データ競合のリスク）
// items[i].quantity += delta

// ✅ イミュータブルな更新（新しいオブジェクトを生成）
func updateQuantity(for item: TrackedItem, delta: Int) {
    guard let i = items.firstIndex(where: {
        $0.id == item.id
    }) else { return }
    let newQuantity = max(0, items[i].quantity + delta)
    if newQuantity == 0 {
        items = items.filter { $0.id != item.id }
    } else {
        var updated = items
        updated[i] = TrackedItem(
            food: item.food, quantity: newQuantity
        )
        items = updated
    }
}
```

## 8. CSVエクスポートのインジェクション対策

データエクスポート機能では、CSV Injection（数式インジェクション）を防止しています。

CSVファイルをExcelで開く際、セルの先頭が`=`や`@`だと数式として実行される可能性があります。ユーザーが自由入力できるフィールドがあるため、全テキストフィールドにエスケープ処理を適用しています。

```swift
// CSV Injection対策: Excel等で数式実行されるのを防止
private func escapeCSV(_ value: String) -> String {
    let dangerous = ["=", "+", "-", "@", "\t", "\r"]
    if dangerous.contains(where: { value.hasPrefix($0) }) {
        return "'\(value)"  // 先頭にシングルクォートで数式実行を抑止
    }
    if value.contains(",") || value.contains("\"") || value.contains("\n") {
        return "\"\(value.replacingOccurrences(of: "\"", with: "\"\""))\""
    }
    return value
}
```

**対策のポイント:**

- `=`, `+`, `-`, `@` で始まる値にシングルクォートを付与 → Excelの数式実行を抑止
- カンマ・ダブルクォート・改行を含む値をRFC 4180準拠でエスケープ
- エクスポート時にすべてのテキストフィールドに適用

## 9. iOSデータ保護による保存データの暗号化

UserDefaultsに保存されるデータは、iOSの**Data Protection**機構により端末ロック時に自動暗号化されます。

| 保護レベル | 暗号化タイミング | 本アプリの対象データ |
|-----------|-----------------|-------------------|
| Complete Protection | デバイスロック直後 | HealthKitデータ（OS管理） |
| Complete Unless Open | ファイルクローズ時 | - |
| After First Unlock | 初回ロック解除まで | UserDefaults（デフォルト） |

バックエンドを持たない設計により、データがネットワーク上を流れる経路が存在しません。iCloud同期はAppleのエンドツーエンド暗号化基盤上で行われるため、転送中のデータも保護されています。

:::message
アプリレイヤーでCryptoKitによる追加暗号化を検討する場合もありますが、**鍵管理の複雑さ**がかえってセキュリティリスクになり得ます。OS標準の保護機構を正しく活用する方が、個人開発では堅実な選択です。
:::

## まとめ

| 対策 | 防ぐ脅威 | 実装コスト |
|------|----------|-----------|
| バックエンドなし設計 | サーバー攻撃全般 | 低（設計判断） |
| StoreKit 2署名検証 | 課金バイパス | 低（API標準機能） |
| ビルド環境別広告ID | AdMobアカウントBAN | 低 |
| 入力バリデーション | データ破損・クラッシュ | 低 |
| iCloud同期ホワイトリスト | データ漏洩・同期異常 | 中 |
| HealthKit最小権限 | 不要なデータ収集 | 低（設計判断） |
| @MainActor + イミュータブル | データ競合・破損 | 中 |
| CSVインジェクション対策 | 数式実行攻撃 | 低 |
| iOSデータ保護活用 | 端末盗難時のデータ漏洩 | 低（OS標準） |

個人開発でも、設計判断の段階でセキュリティを組み込めば大きなコスト増にはなりません。「攻撃面を減らす」「入力を信頼しない」「権限は最小限」― この3原則を意識するだけで、多くの脅威を構造的に排除できます。

健康データを預かるアプリとして、ユーザーの信頼に応える設計を続けていきます。

---

**SEOスコアチェックツール**: [SEO_CHECK](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=ios-health-app-security-tsukankalendar) — RINIAディレクターツール。
**制作・開発**: [CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=ios-health-app-security-tsukankalendar) — Web制作・SEO関連の技術情報サイト
