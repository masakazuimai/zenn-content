---
title: "PHPを更新してphp -vも新しいのに、Webは古い版を返していた話"
emoji: "🐘"
type: "tech"
topics: ["php", "phpfpm", "nginx", "security", "docker"]
published: false
publish_date: 2026-11-28
---

2026年10月21日の PHP2Shell（CVE-2026-109618・CVE-2026-109619）と、11月19日のセキュリティリリースが続きました。サーバーの PHP を上げて `php -v` に新しい版が出れば、それで済んだと思いがちです。ところが Docker で確かめると、ディスクの PHP は新しいのに、Web からのリクエストには古い PHP が答えている状態を作れました。

この記事は、PHP のサーバーを自分で更新している方（VPS・Docker・自前のサーバー）向けです。結論を先に書きます。

- `php -v` が見ているのは **CLI**（端末から動かす PHP）だけで、Web のリクエストを処理する PHP-FPM（Web サーバーから受けた処理を PHP に渡す常駐の仕組み）とは別に確かめる
- OS のパッケージの PHP は、版番号が古いままでも直っていることがある。番号ではなく、パッケージの末尾と変更履歴で判断する
- パッケージを上げても、FPM を起動し直すまでは古い本体が動き続けることがある

確かめた環境は、PHP 公式の FPM イメージ、Debian 13、Ubuntu 24.04 のコンテナです。

## 修正版の早見表

まず「どの版なら直っているか」を押さえておきます。上流（php.net）の版の対応は次のとおりです（php.net の2026年のニュース、JPCERT/CC の注意喚起 JPCERT-AT-2026-0033 による）。

| 系統 | 10月21日（PHP2Shell）の修正版 | 11月19日の修正版 |
|---|---|---|
| 8.5 | 8.5.12 | 8.5.13 |
| 8.4 | 8.4.27 | 8.4.28 |
| 8.3 | 8.3.36 | 8.3.37 |
| 8.2 | 8.2.35 | 8.2.36（2026年12月31日でサポート終了） |
| 8.6 | — | 8.6.0（11月19日に正式版） |
| 8.1 以前 | 修正版なし（サポート終了） | 修正版なし（サポート終了） |

PHP2Shell の2件は認証なしのリモートコード実行につながり、2026年10月26日付で CISA の悪用が確認された脆弱性の一覧（Known Exploited Vulnerabilities カタログ）にも載りました。11月分は、セッション・FastCGI・エラーログなど、当たる条件が問題ごとに違います。

ただ、この表と見比べる「版」をどこから取るかで迷いました。ここからが本題です。

## 迷った所1：php -v と Web の PHP は別物だった

表と照らす版は、端末の `php -v` で取るのが一番手軽です。ところが PHP の入口は1つではありません。

- **CLI**: 端末やバッチから動かす PHP
- **PHP-FPM**: nginx などから FastCGI（Web サーバーと PHP をつなぐ決まった通信の形）でリクエストを受ける PHP
- **mod_php**: Apache に組み込んだ PHP
- **php-cgi**: CGI・FastCGI の構成で動く PHP

10月の PHP2Shell の公告は、Web のリクエストを処理する側（PHP-FPM・mod_php・php-cgi・FrankenPHP・LiteSpeed）を対象にしています。つまり見るべきは Web の入口なのに、`php -v` は CLI の答えしか返しません。

版だけでなく、設定の値も入口ごとに違いました。nginx から `php:8.4.28-fpm` へリクエストを渡す構成で、FPM のプール（FPM が受け持つ設定のまとまり）に次の1行を足しました。

```ini
php_admin_value[session.upload_progress.enabled] = 1
```

すると、CLI で読んだ値は `0` のまま、nginx を通した応答では `1` でした。11月の CVE-2026-115560 はこの「アップロード進捗」が有効なことが条件の1つなので、CLI だけ見ると「無効だから当たらない」と読み違えます。

Web 側の値は、確かめたい入口に小さな PHP のページを置いて読むのが確実でした。

```php
<?php
header('Content-Type: text/plain; charset=utf-8');
echo 'PHP_VERSION=', PHP_VERSION, "\n";
echo 'PHP_SAPI=', PHP_SAPI, "\n";
echo 'session.upload_progress.enabled=', var_export(ini_get('session.upload_progress.enabled'), true), "\n";
```

`PHP_SAPI`（PHP がどの入口で動いているかの名前）で、確かめたのがどの入口かが分かります。このページは調べる間だけ置き、アクセス元を絞り、終わったら消します。

## 迷った所2：Ubuntu は 8.3.6 のままでも直っていた

Web の入口の版が取れても、それを表と比べるだけでは判断を誤る場合がありました。OS のパッケージの PHP です。

ディストリビューション（Debian・Ubuntu など）は、上流の版番号を変えずに、修正だけをパッケージに取り込むことがあります（バックポート＝新しい版の修正を古い版へ移植すること）。Docker で確かめた2つは、版の見え方がはっきり違いました。

| 環境 | 入れたパッケージ | php -v の表記 | 変更履歴にあった修正 |
|---|---|---|---|
| Debian 13（trixie） | 8.4.27-1~deb13u1 から 8.4.28-1~deb13u1（trixie-security）へ | 更新後は 8.4.28 | 8.4.27-1 に PHP2Shell の2件、8.4.28-1~deb13u1 に11月分 |
| Ubuntu 24.04 | php8.3-cli の 8.3.6-0ubuntu0.24.04.14 | 8.3.6 のまま | 0ubuntu0.24.04.13 に PHP2Shell の2件、.14 に11月分の CVE |

Debian は上流と同じ番号になるので表と比べやすい一方、Ubuntu は `8.3.6` と出るので、表だけ見ると「8.3.37 より古い＝危ない」と見えます。実際には、パッケージ末尾の `0ubuntu0.24.04.14` と変更履歴の該当の行が判断の材料でした。JPCERT/CC も、ディストリビューションの PHP は提供元の情報もあわせて確かめるよう呼びかけています。

なお、ここで確かめた Ubuntu のパッケージは CLI 用です。FPM を使うサーバーでは、Web 側が使うパッケージでも同じ照合が要ります。

## 迷った所3：更新したのに古い FPM が答えていた

パッケージで直っていることが確かめられても、それで終わりではありませんでした。一番驚いたのはここです。

Debian 13 のコンテナで、php-fpm8.4 を 8.4.27 で起動してから apt で 8.4.28 に上げました。このコンテナには systemd（サービスの起動と停止を受け持つ仕組み）が無く、更新のときに FPM は再起動されませんでした。

| 見た場所 | 更新の直後（再起動なし） | master に USR2 を送ったあと |
|---|---|---|
| ディスクの実行ファイル | `php -v`・`php-fpm8.4 -v` とも 8.4.28 | — |
| Web 経由の応答 | `PHP_VERSION=8.4.27`、進捗の設定は `'1'` | 8.4.28、進捗の設定は `'0'` |
| master の `/proc/<PID>/exe` | `/usr/sbin/php-fpm8.4 (deleted)` | `(deleted)` が消えた |

master は FPM の親のプロセスで、子のワーカーを束ねています。`php-fpm8.4 -v` はそのとき呼んだディスクのファイルの版なので新しく出ますが、すでに動いていた master はメモリに載った古い本体のままでした。`(deleted)` は「元のファイルはもう置き換わった」という印です。USR2（FPM に再読み込みをさせる合図）を送ると、両方の表示が新しい版に変わりました。

systemd のある環境なら、対象のサービスを起動し直します。サービス名の系統は自分の環境に合わせます。

```bash
sudo systemctl restart php8.4-fpm
```

## 結局、3つの層を順に見ることにした

ここまでの3つの迷いは、見る層をそろえると整理できました。順番は次のとおりです。

1. **版**: CLI だけでなく、Web サーバーが渡す先の PHP の版を取り、表の同じ系統と比べる
2. **パッケージ**: OS のパッケージなら、版の末尾まで見て、変更履歴で対象の CVE が入った版と突き合わせる
3. **動いている本体**: FPM の master の `/proc/<PID>/exe` に `(deleted)` が無いかと、Web 経由で返る `PHP_VERSION` を見る

設定の条件（アップロード進捗・例外の引数・FastCGI の持続接続など）も、3の Web の入口で読みます。CLI の値は CLI の記録として扱う、と決めておくと迷いません。

## うまくいかないとき

3つの層を見るつもりでも、つまずきやすい所がありました。

- **`ps` が無くて master が見つからない**: PHP 公式の FPM イメージには `ps` がありませんでした。procps を入れるか、ホスト側からプロセスを見ます
- **`/proc/<PID>/exe` が読めない**: 本体の場所は root でないと読めないことがあります。`sudo` で見ます
- **CLI と Web で設定の値が違う**: 不具合ではなく、FPM のプールの上書きです。`php-fpm -tt` の出力にプールで足した設定が出るので、その URL がどのプールに渡るかと合わせて読みます
- **公式イメージで php.ini が見つからない**: 確かめた公式 FPM イメージは、読み込んだ php.ini が `(none)` で、組み込みの既定値で動いていました。見本の `php.ini-production` は `/usr/local/etc/php/` にあるだけです。そのため `zend.exception_ignore_args` は既定の `0` で、`php:8.5.12-fpm` では CVE-2026-129474 の公告にある2つの設定条件（`fatal_error_backtraces=1` と `zend.exception_ignore_args=0`）がそろっていました。8.5.13 で直っています
- **アップロード進捗を切れば更新しなくてよいと考えてしまう**: 進捗の無効化は、セッションの経路への緩和です。今回の公告に対応するには更新が要ります。11月の修正版は進捗を既定で無効にしますが、明示的に有効にした設定のほうが優先されるので、Web 側で値を読みます

## 更新のあとにやったこと

最後の層まで確かめたあとも、PHP2Shell のように悪用が確認された脆弱性は、過去の記録も見ます。JPCERT/CC が挙げるのは、不自然に長い・形のおかしい `PHPSESSID` の Cookie、不審な `multipart/form-data` とその前後のメモリ上限のエラーや HTTP 500、ワーカーの予期しない異常終了です。

FPM の子プロセスをシグナルで落とすと、ログには次の形で出ました。

```text
WARNING: [pool www] child 7 exited on signal 11 (SIGSEGV) after 4.883770 seconds from start
```

`exited on signal` を目印に数え、出ていれば元のログを開いて時間帯を照らします。痕跡が無くても、攻撃を受けていないとは言い切れない点は覚えておきます。

PHP 8.2 は2026年12月31日でサポートが終わります。8.2 のサーバーは今回の修正を入れたうえで、サポート中の系統への移行を計画しておくと安心です。

版・パッケージ・動いている本体を一度に洗い出すシェルスクリプトと、CVE ごとの当たる条件と緩和策の表は、元の記事にまとめています。

[PHPの脆弱性の確認と対応｜CLIとFPMの版・再起動まで](https://codequest.work/php-vulnerability-check/?utm_source=zenn&utm_medium=article&utm_campaign=php-fpm-update-old-binary-cli-web-gap)

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=php-fpm-update-old-binary-cli-web-gap) で活動中。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=php-fpm-update-old-binary-cli-web-gap) — ディレクターも使っているSEO診断ツールを公開中。
