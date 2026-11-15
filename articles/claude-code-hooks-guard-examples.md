---
title: "Claude Code の hooks 実例｜危ないコマンドの確認・push 前の鍵チェック・読まずに動いたら止める"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "ai", "hooks", "python", "git"]
published: false
publish_date: 2026-11-19
---

## はじめに

Claude Code の hooks（フック）は、Claude がツールを使う前後や、会話の始まり・終わりに、**自分のスクリプトを挟める仕組み**です。設定の書き方は公式の説明にありますが、「実際に何を見張らせると役に立つのか」は、使ってみないと分かりませんでした。

この記事では、筆者が毎日の運用で動かしている hooks を、**目的・設定・コードの要点・効かない所**まで書きます。どれも、うっかりの失敗が1回起きてから足したものです。

- 対象読者：Claude Code を毎日使っていて、settings.json を自分で書いたことがある人（中級者向け）
- 書いてあること：PreToolUse・SessionStart・Stop の実例と、git の pre-push フックとの分け方
- 書いていないこと：hooks の全イベントの網羅（公式の説明を見てください）

https://code.claude.com/docs/ja/hooks

運用の全体像（定期実行・完了報告・見張り役）は、前の記事に書きました。この記事はその中の「守りのフック」の節を、手順とコードまで深くしたものです。

https://zenn.dev/orectic/articles/claude-code-ai-org-daily-operations

## hooks の基本を3行で

細かい話の前に、この記事で使う3つの決まりだけ押さえます。

1. **入力は標準入力の JSON**：ツールの名前（`tool_name`）、ツールに渡す中身（`tool_input`）、会話の記録の場所（`transcript_path`）、作業フォルダ（`cwd`）などが入ってきます
2. **返し方は2通り**：終了コード 2 で止める（標準エラーの文が Claude に渡る）か、標準出力に JSON を出して「確認に回す」「止める」「文を足す」を指示するか
3. **登録は settings.json の `hooks`**：イベント名 → `matcher`（どのツールに効かせるか）→ 動かすコマンド

この記事で使うイベントは3つです。PreToolUse は「ツールを使う直前」、SessionStart は「会話の始まり」、Stop は「Claude が返事を終える直前」に動きます。

筆者の設定は、いまこうなっています（道筋は短くしてあります）。

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "python3 \"$HOME/.claude/hooks/bash-guard.py\"", "timeout": 10 }] },
      { "matcher": "Edit|Write|MultiEdit|NotebookEdit|Bash",
        "hooks": [{ "type": "command", "command": "python3 \"$HOME/.claude/hooks/preflight-guard.py\"", "timeout": 10 }] }
    ],
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "python3 \"$HOME/.claude/hooks/preflight-list.py\"", "timeout": 10 }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "python3 \"$HOME/.claude/hooks/japanese-reply-guard.py\"", "timeout": 10 }] }
    ]
  }
}
```

`timeout` は秒です。見張りが重いと毎回の操作が遅くなるので、どれも AI を呼ばず、文字の照合だけで済ませています。ここから1つずつ見ていきます。

## 実例1：危ないコマンドを、実行の前に人間の確認へ回す（PreToolUse）

最初に足したのは、取り返しのつかないコマンドの見張りです。

### 何を見るか

| 見るもの | 理由 |
|---|---|
| `rm -rf`（一時フォルダ・`node_modules`・`.next` などのビルドの置き場は除く） | 消したら戻らない |
| `git push --force`・`--force-with-lease`・`+ブランチ名` | 相手の履歴を消す |
| `git reset --hard` | 書きかけの変更が消える |
| `sudo` | 管理者の権限で動く |
| 鍵のファイル（`.env`・`.dev.vars`・`token.json` など）の中身を画面に出す `cat`・`grep`・`head` など | 値が会話の記録とログに残る |

最後の行が、いちばん効いています。Claude は「設定が入っているか確かめます」と言って、`.env` を `cat` しがちです。画面に出た値は会話の記録に残り、あとから消せません。

### 止め方は「deny」ではなく「ask」

見つけたときの返し方は、止める（`deny`）ではなく、**確認に回す**（`ask`）にしました。

```python
print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "permissionDecision": "ask",
        "permissionDecisionReason": "危ないコマンドの見張り: git reset --hard（書きかけの変更が消える）。実行してよいか確かめてください。",
    }
}, ensure_ascii=False))
```

`deny` にすると、Claude は別の書き方で同じことをしようとします。`ask` なら、人間が理由を読んで1回押せば進めます。**本当に要る操作は止めず、うっかりだけを止める**のが狙いです。

### コマンドの読み方で詰まった所

素直に `if "rm -rf" in command` と書くと、すぐに漏れが出ました。実際に直した所です。

- **区切りで分ける**：`cd x && rm -rf y` のように、`;`・`&&`・`||`・`|` でつながった後ろのコマンドも見る。Python の `shlex` に `punctuation_chars=";&|"` を渡すと、引用符の中の `|` を区切りと間違えません
- **中で動くコマンドも見る**：`bash -c "…"`・`ssh 先 "…"`・`find … -exec rm -rf {} +`・`xargs rm -rf` の中身を取り出して、同じ判定にかける
- **変数を置き換える**：`S=/private/tmp/work; rm -rf "$S"` は安全な置き場なので通す。中身の分からない変数（`rm -rf "$DIR"`）は安全とみなさない
- **値が画面に出るかで分ける**：`grep -c API_KEY .env`（件数だけ）や `$(cat .env)`（変数に入るだけ）は通す。`cat .env | head` は値が出るので止める

コードは区切りの処理だけで100行を超えました。それでも「全部の書き方を拾う」ことは目指していません。理由は後の「効かない所」に書きます。

## 実例2：push の前に、送るコミットの鍵を確かめる（git の pre-push）

2つ目は Claude Code の hooks ではなく、**git の pre-push フック**です。ここを分けたのには理由があります。

### なぜ Claude Code の hooks ではないのか

push は、Claude の会話からだけでなく、定期実行のシェルや Python のスクリプトからも走ります。Claude Code の PreToolUse は Claude の Bash にしか効かないので、**git 自身に見張らせる**ほうが漏れません。

```zsh
#!/bin/zsh
# .git/hooks/pre-push（共通のスクリプトへのリンクを置く）
ZERO=0000000000000000000000000000000000000000
STATUS=0
while read -r LOCAL_REF LOCAL_SHA REMOTE_REF REMOTE_SHA; do
  [ "$LOCAL_SHA" = "$ZERO" ] && continue
  if [ "$REMOTE_SHA" = "$ZERO" ]; then
    RANGE="$LOCAL_SHA --not --remotes"   # 初めての push：どのリモートにも無いコミットだけ
  else
    RANGE="$REMOTE_SHA..$LOCAL_SHA"
  fi
  gitleaks git --log-opts="$RANGE" --redact --no-banner . >&2 || STATUS=1
done
exit $STATUS
```

ポイントは2つです。

- **送るコミットの範囲だけを見る**：`--not --remotes` で「まだどのリモートにも無いコミット」に絞ります。履歴全体を毎回見ると遅く、昔の誤検知で毎回止まります
- **`--redact` で値を伏せる**：見つけた鍵の値を画面に出すと、見張り自身が鍵を会話の記録に残してしまいます。ファイル名と種類だけ出せば足ります

### 足したきっかけ

ある日、自分の2コミットを push したら、**同じフォルダで別の会話が作っていた11コミットも一緒に出ていきました**。あとで中身を確かめて問題はありませんでしたが、確かめる前に送っていました。複数の会話や定期実行が同じフォルダを触る運用では、「自分が作ったものだけを送る」前提が崩れます。

このフックは「送る分を全部見る」ので、だれが作ったコミットでも止められます。

## 実例3：決まりを読まずに書き換えたら止める（SessionStart と PreToolUse の組）

3つ目は、2つのフックを組にしたものです。いちばん新しく、いちばん効き目を感じています。

### 困っていたこと

筆者はプロジェクトごとに、決まりと引き継ぎのメモ（MEMORY.md など）を置いています。ところが、別のフォルダで開いた会話に仕事を頼むと、**そのプロジェクトのメモを読まずに作業を始める**ことがありました。実際に、「SNS に貼るリンクは短縮 URL にする」という決まりを知らない会話が、計測用の長い URL をそのまま投稿しました。

「読んでから始めて」と CLAUDE.md に書いても、守られる回と守られない回がありました。そこで、仕組みで止めることにしました。

### 組み方

| フック | すること |
|---|---|
| SessionStart（`preflight-list.py`） | 作業フォルダから、そのプロジェクトで先に読むファイルの一覧を作り、会話の頭に差し込む |
| PreToolUse（`preflight-guard.py`） | 最初のファイルの書き換え・push・deploy の前に、会話の記録にその一覧を読んだ跡があるかを見る。無ければ止めて一覧を返す |

SessionStart は、標準出力の `additionalContext` に入れた文を、会話の頭に足せます。

```python
print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "SessionStart",
        "additionalContext": "【作業の前の確かめ】手を付ける前に、次を Read してください：\n- …/MEMORY.md\n- …/CLAUDE.md",
    }
}, ensure_ascii=False))
```

PreToolUse の側は、`transcript_path` の会話の記録（1行1件の JSONL）を読み、Read の道具の `file_path` か、Bash のコマンドの中に、そのファイルの道筋があるかを探します。

### 速くするための工夫

会話の記録は長くなると数十MBになります。毎回頭から読むと、操作のたびに待たされます。そこで、**前に読んだ位置を小さな状態ファイルに持ち、そこから先だけ読む**ようにしました。一度そろえば、その会話では二度と見ません。

### 通すもの

止める対象を絞らないと、作業が回らなくなります。次のものは見ずに通しています。

- 一時フォルダ・作業メモの置き場だけの書き換え
- 無人の定期実行（`claude -p`）と、サブエージェント（読むべきものは呼ぶ側が渡す）
- メモの無いフォルダ

## 実例4：最後の返事が英語なら書き直させる（Stop）

最後は小さなものです。Stop は、Claude が返事を終える直前に動きます。

筆者は日本語で返事をもらう設定にしていますが、長い作業のあと、最後の返事だけ英語になることがありました。そこで、最後の返事から**コードの囲み・行内のコード・URL を除いた文**で、日本語の字の割合が15%を切ったら止めます。

```python
if data.get("stop_hook_active"):
    return 0  # 同じ返事で2回目は止めない（止まり続けないため）
...
print(json.dumps({
    "decision": "block",
    "reason": "最後の返事が英語になっています。同じ中身を日本語で書き直してください。",
}, ensure_ascii=False))
```

`stop_hook_active` を見るのが大事です。これを見ないと、書き直した返事もまた止めてしまい、**会話が終わらなくなります**。

## 4つに共通する決まり：見張りが壊れたら通す

ここまでの4つは、どれも同じ形で終わります。

```python
def main():
    try:
        payload = json.load(sys.stdin)
        hits = reasons(payload["tool_input"]["command"])
    except Exception:
        return  # 何も出さずに終わる＝通す
```

**見張り自体のバグで、毎日の作業が全部止まる**ほうが、うっかりを1回見逃すより被害が大きいからです。見張りは何本もの会話と定期実行のすべての操作に挟まるので、1つの例外で全部が止まります。この形なら、見張りが壊れても作業は止まりません。

その代わり、見張りを直したら、**止めたいコマンドと通したいコマンドを並べて流す**確かめをしています。

```bash
echo '{"tool_input":{"command":"cd app && rm -rf src"}}' | python3 bash-guard.py   # ask が出る
echo '{"tool_input":{"command":"rm -rf node_modules"}}' | python3 bash-guard.py   # 何も出ない
```

## 効かない所

hooks は便利ですが、何でも守れるわけではありません。分かっている穴を書いておきます。

- **わざと抜ける手は塞がない**：変数を何段も重ねる、スクリプトのファイルを書いてから動かす、などは見ていません。目的はうっかりを止めることで、悪意のある操作を防ぐ仕組みではありません。そちらは権限の設定（`permissions` の `deny`）や、サンドボックスで分けるものです
- **Bash 以外の道具は、matcher に書いたものしか見ない**：MCP の道具から外に出る操作は、道具の名前を1つずつ matcher に足す必要があります
- **git の pre-push は `--no-verify` で抜けられる**：これも同じく、うっかりの見張りです
- **AI を使わないので、意味は分からない**：「この `rm -rf` は本当に要るか」は判断できません。だから止めずに `ask` にしています

## うまく動かないとき

作るときに詰まった所です。

| 症状 | 原因 | 直し方 |
|---|---|---|
| フックが動いていないように見える | settings.json の JSON が壊れている／`matcher` のツール名の綴りが違う | `/hooks` で読み込まれているかを見る。手で `echo '{…}' \| python3 フック.py` を流して出力を見る |
| 日本語の理由が文字化けする | `json.dumps` の既定で `\uXXXX` になっている | `ensure_ascii=False` を付ける |
| 操作のたびに遅い | 会話の記録を毎回全部読んでいる／外の API を呼んでいる | 読んだ位置を覚えて差分だけ読む。AI や通信は使わない |
| Stop のフックで会話が終わらない | `stop_hook_active` を見ていない | 2回目は止めずに通す |
| `deny` したのに別の書き方で同じことをする | 理由が伝わっていない | `ask` にして人間が判断する。止めるなら理由を具体的に書く |

## これから足すなら、この順番

1. 鍵のファイルの中身を画面に出す命令を `ask` にする（いちばん被害が大きく、いちばん起きやすい）
2. git の pre-push で、送るコミットだけ gitleaks にかける
3. `rm -rf` と `push --force` を `ask` にする
4. 決まりを読まずに動く失敗が1回起きたら、SessionStart と PreToolUse の組を足す

全部を最初から作る必要はありません。**失敗が1回起きたら、その操作だけを止める**のが、結果的にいちばん外れの少ない足し方でした。

## 使っているテンプレートについて

この記事の bash-guard（危ないコマンドの確認）と pre-push（鍵の確かめ）は、個人の値を抜いて、AI組織キットというテンプレートに入れています。Claude Code を役割を分けて毎日回すための、決まり・部署の定義・定期実行の型をまとめた筆者の有料の商品です（Anthropic の公式品ではありません）。

https://kit.orecticdesign.com/z

この記事のコードだけでも、同じものは作れます。まずは鍵のファイルの見張りから試してみてください。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=claude-code-hooks-guard-examples) で活動中。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=claude-code-hooks-guard-examples) — ディレクターも使っているSEO診断ツールを公開中。
