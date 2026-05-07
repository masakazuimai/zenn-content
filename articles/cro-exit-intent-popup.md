---
title: "離脱防止ポップアップをexit-intent＋dialog要素でコード実装する"
emoji: "🚪"
type: "tech"
topics: ["javascript", "html", "cro", "ux", "frontend"]
published: true
published_at: "2026-05-13 09:00"
---

離脱防止ポップアップとは、ユーザーがページを閉じようとした瞬間にオファーやメッセージを表示し、離脱を防ぐCRO施策です。exit-intentと呼ばれるマウスの動きをJavaScriptで検出して実装します。

OptinMonsterの事例では、exit-intentポップアップでメール登録率が最大600%向上したケースが報告されています。この記事では、離脱防止ポップアップをコードで実装する方法と、ユーザー体験を損なわないための設計ポイントを解説します。

---

## 離脱防止ポップアップが有効なケース

| ケース | 表示するオファー例 | 期待効果 |
|---|---|---|
| ECサイトのカート離脱 | 「今なら送料無料」「10%OFFクーポン」 | カート復帰率の向上 |
| SaaSの料金ページ離脱 | 「無料トライアル14日間」「導入事例を見る」 | トライアル登録率の向上 |
| ブログ記事の離脱 | 「関連資料を無料ダウンロード」「メルマガ登録」 | リード獲得 |
| LPの離脱 | 「限定価格は本日まで」「無料相談を予約」 | CVR向上 |

---

## JavaScriptで実装するexit-intent検出

exit-intentの検出原理は、マウスカーソルがブラウザのビューポート上端を超えた瞬間を`mouseleave`イベントで捕捉するものです。

### 基本的なexit-intent検出

```javascript
function initExitIntent(callback) {
  const handleMouseLeave = (e) => {
    if (e.clientY <= 0) {
      callback();
      document.removeEventListener('mouseleave', handleMouseLeave);
    }
  };

  setTimeout(() => {
    document.addEventListener('mouseleave', handleMouseLeave);
  }, 5000);
}```

参考: [mouseleave イベント - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/API/Element/mouseleave_event)

`setTimeout`で5秒間の遅延を入れることで、ページ読み込み直後の誤検出を防ぎます。

### 表示頻度の制御

同じユーザーに何度もポップアップを表示するのは逆効果です。localStorageで表示済みフラグを管理し、一定期間は再表示しないようにします。

```javascript
function shouldShowPopup(popupId, intervalDays) {
  const key = `exit_popup_${popupId}`;
  const lastShown = localStorage.getItem(key);

  if (lastShown) {
    const daysSince = (Date.now() - Number(lastShown)) / (1000 * 60 * 60 * 24);
    if (daysSince < intervalDays) {
      return false;
    }
  }
  return true;
}

function markPopupShown(popupId) {
  localStorage.setItem(`exit_popup_${popupId}`, String(Date.now()));
}```

参考: [Web Storage API - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/API/Web_Storage_API/Using_the_Web_Storage_API)

### モーダルUIの実装

HTML5の`<dialog>`要素を使えば、アクセシビリティに配慮したモーダルを少ないコードで実装できます。

```html
<dialog id="exit-popup">
  <h2>ちょっと待ってください！</h2>
  <p>今なら無料で資料をダウンロードできます。</p>
  <a href="/download" class="cta-button">無料ダウンロード</a>
  <button onclick="this.closest('dialog').close()">閉じる</button>
</dialog>```

```javascript
initExitIntent(() => {
  if (shouldShowPopup('lead-magnet', 7)) {
    document.getElementById('exit-popup').showModal();
    markPopupShown('lead-magnet');
    gtag('event', 'exit_popup_shown', { popup_id: 'lead-magnet' });
  }
});```

参考: [dialog要素 - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/HTML/Element/dialog)

---

## モバイルでの離脱防止

モバイルデバイスではマウスカーソルがないため、exit-intentが使えません。代替として以下の手法があります。

| 手法 | トリガー条件 | 向いているケース |
|---|---|---|
| スクロール率トリガー | ページの70%以上をスクロールした後、上方向にスクロールした時 | ブログ記事、LP |
| 滞在時間トリガー | ページに30秒以上滞在した後 | 料金ページ、比較ページ |
| 非アクティブ検出 | 10秒以上操作がない時 | フォームページ |

---

## UXを損なわないための3つのルール

---

- **閉じるボタンを分かりやすく配置する**: 小さなXボタンだけでは不十分。「閉じる」テキストボタンを必ず用意する
- **同じユーザーに繰り返し表示しない**: 最低7日間は再表示しない。表示頻度が高いとブランドイメージを損なう
- **ページ読み込み直後に表示しない**: 最低5秒はコンテンツを読ませてから表示する。即座のポップアップはGoogleのインタースティシャルガイドラインにも抵触する

---

## まとめ

この記事は [CodeQuest.work](https://codequest.work/exit-popup-guide/) に掲載した記事を技術者向けに再構成したものです。

CROシリーズの他の記事:
- [CRO完全ガイド — コードで実装する8つの手法](https://codequest.work/cro-guide/)
- [ABテストのやり方 — コード実装方法](https://codequest.work/ab-test-guide/)
- [EFO（フォーム最適化）実装ガイド](https://codequest.work/efo-guide/)
- [離脱防止ポップアップの実装方法](https://codequest.work/exit-popup-guide/)
- [マイクロコピー改善でCVRを上げる方法](https://codequest.work/microcopy-guide/)
- [ファーストビュー改善の方法](https://codequest.work/firstview-guide/)
- [Core Web Vitalsをコードで最適化](https://codequest.work/pagespeed-guide/)
- [ソーシャルプルーフの実装パターン](https://codequest.work/social-proof-guide/)
- [Webパーソナライズの実装方法](https://codequest.work/personalize-guide/)

