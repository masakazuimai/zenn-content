---
title: "フォーム離脱率を下げるEFO実装ガイド — HTML5 API＋JavaScriptで完結"
emoji: "📝"
type: "tech"
topics: ["javascript", "html", "form", "cro", "ux"]
published: true
published_at: "2026-05-13 09:00"
---

EFO（Entry Form Optimization）とは、問い合わせフォームや申し込みフォームの入力完了率を高める施策です。フォームはコンバージョンの最終関門であり、Baymard Instituteの調査（2024年）によるとECサイトのカート離脱率は平均70.19%に達しています。

EFOの施策はすべてJavaScriptとCSSで実装可能です。この記事では、フォーム離脱の原因と、コードで実装する具体的な改善手法を解説します。

---

## フォームで離脱が起きる5つの原因

Baymard Instituteの調査では、フォーム離脱の主な原因は以下の通りです。

- **入力項目が多すぎる**: ユーザーはフォームを見た瞬間に「面倒」と感じて離脱する
- **エラーが送信後にしか分からない**: 全項目を入力した後にエラーが出ると、やり直す気力を失う
- **何を入力すべきか分からない**: ラベルやプレースホルダーが不明確
- **入力形式の制約が厳しい**: 半角・全角の指定、ハイフンの有無など
- **進捗が見えない**: 長いフォームでゴールが見えないと途中で諦める

---

## EFOの6つの改善手法

### 1. フィールド数の削減

Formstackの調査（2023年）によると、フォームのフィールド数を削減するとCVRが平均25%向上します。本当に必要な項目だけに絞りましょう。

| 項目 | 問い合わせフォーム | 資料請求フォーム | 会員登録フォーム |
|---|---|---|---|
| 必須 | 名前・メール・内容 | 名前・メール・会社名 | メール・パスワード |
| 任意 | 電話番号・会社名 | 電話番号・役職 | 名前・電話番号 |
| 不要（削除候補） | 住所・FAX・部署 | 住所・FAX | 住所・生年月日 |

### 2. リアルタイムバリデーション

入力中にエラーを即座にフィードバックすることで、送信後のエラーによる離脱を防ぎます。HTML5のConstraint Validation APIを使えば、少ないコードで実装できます。

```javascript
const form = document.querySelector('#contact-form');

form.querySelectorAll('input, textarea').forEach(field => {
  field.addEventListener('blur', () => {
    const errorEl = field.nextElementSibling;

    if (!field.validity.valid) {
      field.classList.add('is-invalid');
      if (errorEl?.classList.contains('error-message')) {
        errorEl.textContent = field.validationMessage;
      }
    } else {
      field.classList.remove('is-invalid');
      if (errorEl?.classList.contains('error-message')) {
        errorEl.textContent = '';
      }
    }
  });
});```

参考: [クライアント側のフォームバリデーション - MDN Web Docs](https://developer.mozilla.org/ja/docs/Learn_web_development/Extensions/Forms/Form_validation)

### 3. ステップフォーム

長いフォームを複数ステップに分割し、進捗バーで現在地を表示します。Typeformの事例では、ステップフォームの導入でフォーム完了率が最大35%向上しています。

```javascript
function showStep(stepIndex, totalSteps, steps) {
  steps.forEach((step, i) => {
    step.style.display = i === stepIndex ? 'block' : 'none';
  });

  const progress = document.querySelector('.progress-bar');
  const percentage = ((stepIndex + 1) / totalSteps) * 100;
  progress.style.width = `${percentage}%`;
  progress.setAttribute('aria-valuenow', percentage);
}```

### 4. 入力支援（オートコンプリート）

HTML5の`autocomplete`属性を適切に設定するだけで、ブラウザの自動入力機能が有効になります。追加のJavaScriptは不要です。

```html
<input type="text" name="name" autocomplete="name" />
<input type="email" name="email" autocomplete="email" />
<input type="tel" name="tel" autocomplete="tel" />
<input type="text" name="organization" autocomplete="organization" />
<input type="text" name="postal-code" autocomplete="postal-code" />```

参考: [HTML autocomplete属性 - MDN Web Docs](https://developer.mozilla.org/ja/docs/Web/HTML/Attributes/autocomplete)

### 5. 入力形式の自動変換

電話番号の全角→半角変換、ハイフンの自動除去など、ユーザーの入力ミスをコード側で吸収します。

```javascript
function normalizeInput(value) {
  return value
    .replace(/[０-９]/g, s => String.fromCharCode(s.charCodeAt(0) - 0xFEE0))
    .replace(/[-ー−]/g, '')
    .trim();
}

document.querySelector('input[name="tel"]').addEventListener('blur', (e) => {
  e.target.value = normalizeInput(e.target.value);
});```

### 6. エラーメッセージの改善

デフォルトのブラウザエラーメッセージは分かりにくいため、カスタムメッセージに置き換えます。

| フィールド | デフォルト | 改善後 |
|---|---|---|
| メール | 「有効なメールアドレスを入力してください」 | 「メールアドレスに@が含まれていません」 |
| 電話番号 | 「パターンに一致しません」 | 「電話番号は数字のみで入力してください」 |
| 必須項目 | 「このフィールドは必須です」 | 「お名前を入力してください」 |

---

## EFOの効果を計測する

改善の効果を定量的に把握するため、GA4でフォームの各ステップをイベントとして送信します。

```javascript
// フォーム開始（最初のフィールドにフォーカス）
form.querySelector('input').addEventListener('focus', () => {
  gtag('event', 'form_start', { form_name: 'contact' });
}, { once: true });

// フォーム送信完了
form.addEventListener('submit', () => {
  gtag('event', 'form_submit', { form_name: 'contact' });
});```

参考: [GA4 イベントを送信する - Google Developers](https://developers.google.com/analytics/devguides/collection/ga4/events)

GA4の探索レポートで`form_start`と`form_submit`のイベント数を比較すれば、フォームの完了率（= form_submit / form_start）を算出できます。

---

## まとめ

この記事は [CodeQuest.work](https://codequest.work/efo-guide/) に掲載した記事を技術者向けに再構成したものです。

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

