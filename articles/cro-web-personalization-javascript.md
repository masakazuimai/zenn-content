---
title: "WebパーソナライズをJavaScript＋localStorageだけで実装する方法"
emoji: "🎯"
type: "tech"
topics: ["javascript", "cro", "marketing", "webdev", "frontend"]
published: false
published_at: "2026-05-18 09:00"
---

Webパーソナライズとは、ユーザーの属性や行動に基づいて表示内容を動的に変更するCRO施策です。McKinseyの調査（2023年）によると、パーソナライズを実施している企業はそうでない企業と比較して売上が40%高い傾向にあります。

高度なパーソナライズにはサーバーサイドの仕組みが必要ですが、クライアントサイドのパーソナライズはJavaScriptだけで実装可能です。この記事では、コードで実現できるパーソナライズの手法と具体的な実装方法を解説します。

---

## コードで実装できる4つのパーソナライズ

| 条件 | 実装例 | 使用技術 | 難易度 |
|---|---|---|---|
| 流入元 | Google広告からの流入者に広告連動CTAを表示 | URLパラメータ（utm_source） | 低 |
| 訪問回数 | 初回訪問者に初回限定オファーを表示 | localStorage | 低 |
| デバイス | モバイルユーザーに電話CTAを表示 | メディアクエリ / User-Agent | 低 |
| 閲覧履歴 | 過去に見たカテゴリの関連コンテンツを表示 | localStorage + JavaScript | 中 |

---

## 流入元別のコンテンツ出し分け

UTMパラメータを使って、流入元に応じたメッセージを表示します。広告のランディングページで特に効果的です。

```javascript
function getUtmParam(param) {
  const url = new URL(window.location.href);
  return url.searchParams.get(param);
}

function personalizeBySource() {
  const source = getUtmParam('utm_source');
  const headline = document.querySelector('.hero-headline');
  const cta = document.querySelector('.hero-cta');

  const config = {
    google: {
      headline: 'Google広告からお越しの方へ — 初回相談無料',
      cta: '無料相談を予約する'
    },
    facebook: {
      headline: 'SNSで話題のサービスを体験しませんか？',
      cta: '無料で試してみる'
    }
  };

  const personalized = config[source];
  if (personalized && headline && cta) {
    headline.textContent = personalized.headline;
    cta.textContent = personalized.cta;
  }
}

personalizeBySource();
```

参考: [URLSearchParams - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/API/URLSearchParams)

---

## 訪問回数に応じた表示変更

初回訪問者にはサービス概要を、リピーターには直接的なCTAを表示します。

```javascript
function getVisitCount() {
  const key = 'visit_count';
  const count = Number(localStorage.getItem(key) || 0) + 1;
  localStorage.setItem(key, String(count));
  return count;
}

const visitCount = getVisitCount();

if (visitCount === 1) {
  // 初回: サービス概要を強調
  document.querySelector('.welcome-banner').style.display = 'block';
} else if (visitCount >= 3) {
  // 3回目以降: 直接的なCTAを表示
  document.querySelector('.returning-cta').style.display = 'block';
}
```

参考: [Web Storage API - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/API/Web_Storage_API/Using_the_Web_Storage_API)

---

## 閲覧履歴に基づくレコメンド

ユーザーが過去に閲覧したページのカテゴリを記録し、関連コンテンツを表示します。

```javascript
function recordPageView(category) {
  const key = 'viewed_categories';
  const viewed = JSON.parse(localStorage.getItem(key) || '[]');

  if (!viewed.includes(category)) {
    const updated = [...viewed, category].slice(-10);
    localStorage.setItem(key, JSON.stringify(updated));
  }
}

function getRecommendedCategory() {
  const viewed = JSON.parse(localStorage.getItem('viewed_categories') || '[]');
  const frequency = {};

  viewed.forEach(cat => {
    frequency[cat] = (frequency[cat] || 0) + 1;
  });

  return Object.entries(frequency)
    .sort((a, b) => b[1] - a[1])
    .map(entry => entry[0])[0] || null;
}
```

---

## パーソナライズの注意点

---

- **SEOへの影響**: クライアントサイドのパーソナライズはJavaScriptで動的に変更するため、Googlebot にはデフォルトコンテンツが見えます。SEO上重要なテキスト（h1・メタタグ等）はサーバーサイドで出力しましょう
- **プライバシー**: localStorageはCookieと同様にユーザーのブラウザに保存されるデータです。個人を特定する情報は保存せず、カテゴリ・訪問回数など匿名情報のみを扱いましょう
- **フォールバック**: JavaScript無効環境やlocalStorage未対応ブラウザでも、デフォルトコンテンツが正常に表示されるようにしましょう

---

## まとめ

この記事は [CodeQuest.work](https://codequest.work/personalize-guide/) に掲載した記事を技術者向けに再構成したものです。

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

