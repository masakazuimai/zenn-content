---
title: "WordPress管理画面からSEO診断できるプラグインを作った"
emoji: "🔌"
type: "tech"
topics: ["WordPress", "PHP", "SEO", "API", "プラグイン"]
published: true
---

## 作ったもの

WordPress管理画面からワンクリックでSEOスコアを診断できるプラグインです。

URLを入力して「診断する」を押すだけで、100点満点のSEOスコアと4カテゴリ（構造化データ・基本SEO・コンテンツ・技術SEO）の内訳が表示されます。

![SEO診断結果の画面](/images/wordpress-seo-check-plugin/screenshot.png)

**GitHub**: [masakazuimai/codequest-seo-check-plugin](https://github.com/masakazuimai/codequest-seo-check-plugin)

## なぜ作ったか

自社で運営している[SEO診断ツール（CodeQuest.work SEO_CHECK）](https://seo.codequest.work)があります。ブラウザでURLを入力して診断するWebアプリです。

ただ、WordPressでサイトを運営しているユーザーにとって「別のサイトを開いて、URLをコピペして、診断する」という手順は面倒です。管理画面に組み込めば、ログインしている状態からワンクリックで自分のサイトを診断できます。

もう一つの理由は**配布チャネルの拡大**です。WordPress公式ディレクトリに掲載されれば、SEO診断ツールの存在を知らない人にもリーチできます。

## 技術構成

```
WordPress管理画面（PHP）
  ↓ wp_remote_post
外部API（Cloudflare Workers + Hono）
  ↓ HTML解析・スコア計算
診断結果をJSON返却
  ↓
管理画面にスコア表示（jQuery + CSS）
```

プラグイン自体は**PHPのみ**で、外部ライブラリは使っていません。SEO診断のロジックはすべてAPI側にあるため、プラグインは「APIを呼んで結果を表示する」だけのシンプルな構成です。

## APIとの通信

WordPressからの外部API呼び出しには `wp_remote_post` を使います。cURLを直接使うとサーバー環境による互換性の問題が出るため、WordPress標準の HTTP API を使うのが鉄則です。

```php
// サンプル: 外部APIにPOSTリクエストを送る
function my_plugin_call_api( $url ) {
    $response = wp_remote_post(
        'https://api.example.com/check',
        array(
            'headers' => array(
                'Content-Type' => 'application/json',
            ),
            'body'    => wp_json_encode( array( 'url' => $url ) ),
            'timeout' => 60,
        )
    );

    if ( is_wp_error( $response ) ) {
        return $response; // WP_Errorをそのまま返す
    }

    $code = wp_remote_retrieve_response_code( $response );
    $body = wp_remote_retrieve_body( $response );
    $data = json_decode( $body, true );

    if ( 200 !== $code ) {
        return new WP_Error(
            'api_error',
            isset( $data['message'] )
                ? sanitize_text_field( $data['message'] )
                : '不明なエラー'
        );
    }

    return $data;
}
```

ポイント:
- `wp_json_encode` でリクエストボディを生成（`json_encode` ではなく）
- レスポンスの取得は `wp_remote_retrieve_response_code` / `wp_remote_retrieve_body`
- エラーメッセージは `sanitize_text_field` でサニタイズしてから使う

## APIキー認証

APIキーを使った認証は `X-API-Key` ヘッダーで送ります。

```php
// サンプル: APIキーをヘッダーに付与
$headers = array( 'Content-Type' => 'application/json' );

$api_key = get_option( 'my_plugin_api_key', '' );
if ( ! empty( $api_key ) ) {
    $headers['X-API-Key'] = $api_key;
}

$response = wp_remote_post( $endpoint, array(
    'headers' => $headers,
    'body'    => wp_json_encode( $payload ),
    'timeout' => 60,
) );
```

APIキーの保存には `get_option` / `update_option` を使い、Settings APIで管理します。設定画面では `type="password"` で表示し、平文が画面上に見えないようにしています。

## セキュリティ対策

wp.org公式ディレクトリの審査では、セキュリティが厳しくチェックされます。最低限必要な対策をまとめます。

### 1. nonce検証

AJAXリクエストにはnonceを付与し、サーバー側で検証します。CSRF対策です。

```php
// JS側にnonceを渡す
wp_localize_script( 'my-plugin-js', 'myPluginData', array(
    'ajaxUrl' => admin_url( 'admin-ajax.php' ),
    'nonce'   => wp_create_nonce( 'my_plugin_nonce' ),
) );

// AJAX処理の冒頭で検証
function my_plugin_ajax_handler() {
    check_ajax_referer( 'my_plugin_nonce', 'nonce' );

    // ... 処理
}
add_action( 'wp_ajax_my_plugin_action', 'my_plugin_ajax_handler' );
```

### 2. 権限チェック

管理者のみが実行できるようにします。

```php
if ( ! current_user_can( 'manage_options' ) ) {
    wp_send_json_error( array( 'message' => '権限がありません' ) );
}
```

### 3. 入力サニタイズ・出力エスケープ

```php
// 入力: URLをサニタイズ
$url = esc_url_raw( wp_unslash( $_POST['url'] ) );

// 出力: HTMLエスケープ
echo esc_html( $title );
echo esc_attr( $value );
echo esc_url( $link );
```

これは「外部APIのレスポンス」にも適用します。APIが返す値を信頼せず、すべてサニタイズしてからフロントに渡します。

### 4. uninstall.php

プラグイン削除時にDBに残したオプションを掃除します。

```php
<?php
// uninstall.php
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

delete_option( 'my_plugin_api_key' );
```

### 5. ABSPATH チェック

全PHPファイルの先頭で、WordPressから読み込まれていることを確認します。直接アクセスを防ぎます。

```php
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
```

## 設計判断: 全機能を入れず「入口」に絞る

自社のSEO診断ツールにはキーワード調査・競合比較・Core Web Vitals測定など多くの機能があります。これらをすべてプラグインに入れることもできましたが、あえて**SEOスコア診断だけ**に絞りました。

理由:
- 管理画面に機能を詰め込むとUXが悪くなる
- プラグインが重くなる
- メンテナンスコストが増える

代わりに、診断結果の下に「キーワード調査」「競合比較」などへの**導線リンク**を設置しています。プラグインを「入口」にして、本体サービスに送客する構造です。

Yoast SEOやRank Mathなど、成功しているプラグインも同じ戦略を取っています。基本機能は無料で提供し、高度な機能はSaaSに誘導するモデルです。

## まとめ

- 外部APIを呼ぶだけのシンプルなプラグインでも、十分な価値を提供できる
- wp.orgのセキュリティ要件（nonce・エスケープ・権限チェック）は最初から入れる
- プラグインに全機能を入れず「入口」に絞る設計が、ユーザーにも開発者にも良い

WordPressプラグインの開発は、SaaSの新しい配布チャネルとして有効です。既にAPIを持っているなら、プラグインは薄いクライアントとして作るのが効率的です。

**GitHub**: [masakazuimai/codequest-seo-check-plugin](https://github.com/masakazuimai/codequest-seo-check-plugin)
**SEO診断ツール**: [CodeQuest.work SEO_CHECK](https://seo.codequest.work)
**制作・開発**: [CodeQuest.work](https://codequest.work/) — Web制作・SEOツール開発のフリーランスサイト
