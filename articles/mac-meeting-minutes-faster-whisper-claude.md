---
title: "Mac だけで会議の議事録を作る｜ScreenCaptureKit で録音・faster-whisper で文字起こし・claude -p でまとめ"
emoji: "🎙️"
type: "tech"
topics: ["macos", "swift", "whisper", "python", "claudecode"]
published: true
---

## はじめに

オンライン会議の議事録を、有料の AI の議事録機能に頼らず、**Mac の中だけで作る仕組み**を組みました。流れは次の3段です。

1. **録音**：ScreenCaptureKit（macOS の画面・音の収録の仕組み）で、相手の声（Mac から出る音）と自分の声（マイク）を別々の wav に録る
2. **文字起こし**：faster-whisper（Whisper を軽く速く動かすライブラリ）で、Mac の中で文字にする
3. **議事録**：`claude -p`（Claude Code を1回だけ動かして結果を返す使い方）に文字起こしを渡し、決まったこと・やることの形にまとめる

録音と文字起こしは Mac の外へ出ません。外に出るのは、議事録をまとめるときに Claude へ渡す文字起こしの文だけです。

この記事では、3段それぞれの**一般的な参考コード**（Swift・Python・シェル）を載せ、通しで作れるようにします。途中で、**録音を止めたのにオレンジのマイクの印が消えない**という、気づきにくい問題にも当たったので、その原因と直し方も書きます。

### 対象読者

- Mac（macOS 15 以降）で、Google Meet や Zoom の議事録を手元で作りたい人
- Swift・Python・シェルを読める中級者（Xcode のプロジェクトは使わず、`swiftc` でコマンドを1本作ります）

### 筆者がこれを作った理由

有料の AI の議事録機能は便利ですが、月額がかかるうえ、録音そのものが外のサービスに置かれます。手元にはすでに Whisper のモデルと Claude Code があったので、「録音さえ Mac の中でできれば、残りはつなぐだけ」と考えたのが始まりです。いちばん手間がかかったのは、予想どおり録音の部分でした。そこで、まず録音から見ていきます。

## 全体の形

```
[会議アプリ] ─音→ ScreenCaptureKit ─┬─ system.wav（相手）
              マイク ───────────────┘─ mic.wav（自分）
                       ↓ ffmpeg で 16kHz・1ch に揃えて混ぜる
                  mix16k.wav ─→ faster-whisper ─→ 文字起こし.md（時刻・相手／自分）
                       ↓ claude -p（標準入力で渡す）
                  議事録.md
```

| 段 | 道具 | 外に出るもの |
|---|---|---|
| 録音 | Swift の自作コマンド（ScreenCaptureKit） | なし |
| 文字起こし | Python＋faster-whisper（CPU で動く） | なし |
| 議事録 | `claude -p` | 文字起こしの文 |
| 起動・停止 | シェル＋launchd（macOS の常駐の仕組み） | なし |

## 1. 録音：ScreenCaptureKit で相手の声とマイクを別々に録る

Mac で「相手の声」を録る方法としては、BlackHole などの仮想オーディオドライバーを入れて音の出口を付け替えるやり方がよく知られています。ただ、ドライバーを入れると、会議のたびに出力先を切り替える手間が増えます。

macOS 15 からは、ScreenCaptureKit が**システムの音とマイクを同じストリームで、別々の出力として**受け取れるようになりました（`SCStreamConfiguration.captureMicrophone` と出力の種類 `.microphone`）。ドライバーは要りません。

### 録音のコマンド（参考コード）

`rec.swift` として保存します。引数は保存先のフォルダだけです。SIGTERM（終了の合図）を受けたらファイルを閉じて終わります。

```swift:rec.swift
import AVFoundation
import ScreenCaptureKit

let outDir = URL(fileURLWithPath: CommandLine.arguments.dropFirst().first ?? ".", isDirectory: true)

// 1本の wav に書き足していく係。最初の音が来たときの形式でファイルを作る
final class TrackWriter {
    private let url: URL
    private var file: AVAudioFile?
    init(_ url: URL) { self.url = url }

    func append(_ sample: CMSampleBuffer) {
        guard let desc = CMSampleBufferGetFormatDescription(sample) else { return }
        let format = AVAudioFormat(cmAudioFormatDescription: desc)
        let frames = AVAudioFrameCount(CMSampleBufferGetNumSamples(sample))
        guard frames > 0,
              let pcm = AVAudioPCMBuffer(pcmFormat: format, frameCapacity: frames) else { return }
        pcm.frameLength = frames
        let status = CMSampleBufferCopyPCMDataIntoAudioBufferList(
            sample, at: 0, frameCount: Int32(frames), into: pcm.mutableAudioBufferList)
        guard status == noErr else { return }
        do {
            if file == nil {
                file = try AVAudioFile(forWriting: url, settings: format.settings,
                                       commonFormat: format.commonFormat,
                                       interleaved: format.isInterleaved)
            }
            try file?.write(from: pcm)
        } catch {
            FileHandle.standardError.write("write failed: \(error)\n".data(using: .utf8)!)
        }
    }

    func close() { file = nil }  // AVAudioFile は解放されたときにヘッダーを書いて閉じる
}

final class Recorder: NSObject, SCStreamOutput, SCStreamDelegate {
    let remote = TrackWriter(outDir.appendingPathComponent("system.wav"))
    let local = TrackWriter(outDir.appendingPathComponent("mic.wav"))
    private let lock = NSLock()
    private var done = false

    func stream(_ stream: SCStream, didOutputSampleBuffer sample: CMSampleBuffer,
                of type: SCStreamOutputType) {
        lock.lock(); defer { lock.unlock() }
        guard !done else { return }
        switch type {
        case .audio: remote.append(sample)       // Mac から出る音＝相手
        case .microphone: local.append(sample)   // マイク＝自分
        default: break                           // 画面の映像は捨てる
        }
    }

    func stream(_ stream: SCStream, didStopWithError error: Error) {
        FileHandle.standardError.write("stopped: \(error)\n".data(using: .utf8)!)
        finish(1)
    }

    func finish(_ code: Int32) {
        lock.lock(); done = true; remote.close(); local.close(); lock.unlock()
        exit(code)
    }
}

let recorder = Recorder()
var current: SCStream?

// SIGTERM・SIGINT を受けたら、収録を止めてからファイルを閉じる
signal(SIGTERM, SIG_IGN)
signal(SIGINT, SIG_IGN)
let handlers = [SIGTERM, SIGINT].map { sig -> DispatchSourceSignal in
    let source = DispatchSource.makeSignalSource(signal: sig, queue: .main)
    source.setEventHandler {
        Task {
            try? await current?.stopCapture()
            recorder.finish(0)
        }
    }
    source.resume()
    return source
}

Task {
    do {
        let content = try await SCShareableContent.excludingDesktopWindows(
            false, onScreenWindowsOnly: false)
        guard let display = content.displays.first else { exit(1) }
        let filter = SCContentFilter(display: display, excludingApplications: [],
                                     exceptingWindows: [])
        let config = SCStreamConfiguration()
        config.capturesAudio = true
        config.excludesCurrentProcessAudio = true
        config.captureMicrophone = true          // macOS 15 以降
        config.sampleRate = 48000
        config.channelCount = 2
        config.width = 2                         // 映像は使わないので最小に
        config.height = 2
        config.minimumFrameInterval = CMTime(value: 1, timescale: 1)

        let stream = SCStream(filter: filter, configuration: config, delegate: recorder)
        let audioQueue = DispatchQueue(label: "audio")
        try stream.addStreamOutput(recorder, type: .audio, sampleHandlerQueue: audioQueue)
        try stream.addStreamOutput(recorder, type: .microphone, sampleHandlerQueue: audioQueue)
        try stream.addStreamOutput(recorder, type: .screen,
                                   sampleHandlerQueue: DispatchQueue(label: "screen"))
        try await stream.startCapture()
        current = stream
        print("recording: \(outDir.path)")
    } catch {
        FileHandle.standardError.write("start failed: \(error)\n".data(using: .utf8)!)
        exit(1)
    }
}
_ = handlers
dispatchMain()
```

ポイントは3つです。

- **映像の出力も登録しておく**：音だけ欲しくても、`.screen` の出力先が無いと、届いた映像のフレームの行き先が無いというエラーがログに出ることがあります。2×2・1秒に1枚にして、受け取ったら捨てています
- **相手とマイクを別のファイルにする**：あとで「どちらが話したか」を決める材料になります（3章）
- **`excludesCurrentProcessAudio`**：このコマンド自身の音は録らない設定です。通知音などを鳴らす作りにしても混ざりません

### マイクの許可の文を埋め込んで作る

コマンドラインの道具がマイクを使うには、許可を求める理由の文（`NSMicrophoneUsageDescription`）が要ります。アプリの形にしなくても、Info.plist をバイナリの中に埋め込めば足ります。

```xml:Info.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleIdentifier</key><string>com.example.meeting-rec</string>
  <key>CFBundleName</key><string>meeting-rec</string>
  <key>NSMicrophoneUsageDescription</key><string>会議の自分の声を録音して議事録を作るため</string>
</dict>
</plist>
```

```bash
swiftc -O rec.swift -o meeting-rec \
  -Xlinker -sectcreate -Xlinker __TEXT -Xlinker __info_plist -Xlinker Info.plist
codesign -s - -f -i com.example.meeting-rec meeting-rec
```

`codesign -s -` は自分の Mac の中だけで通る署名（アドホック署名）です。識別子（`-i`）を毎回同じにしておくと、macOS の許可をこの識別子で覚えてくれます。ただし**作り直すと許可を取り直すことになる**ので、録音のコマンドはできるだけ触らず、変更はシェルや Python の側に寄せました。

録音ができたので、次はこれをどう起動・停止するかです。ここに、許可の持ち主という落とし穴があります。

## 2. 起動と停止：launchd から動かして許可の持ち主を揃える

macOS の「画面とシステムオーディオの収録」と「マイク」の許可は、**だれがそのコマンドを起動したか**で持ち主が決まります。ターミナルから直接 `./meeting-rec` を動かすと、許可を求められるのはターミナル（や、ターミナルを動かしている Claude Code など）になり、その道具全体に収録の許可を渡すことになります。

そこで、launchd（macOS が常駐プロセスを管理する仕組み）に一時的な登録を作って起動しました。こうすると、親がいない状態で `meeting-rec` 本体が起動するので、**許可の持ち主が `meeting-rec` 自身**になります。

```bash:start.sh
#!/bin/zsh
# 録音を始める。止めるのは stop.sh
set -u
ROOT=${0:A:h}
LABEL=com.example.meeting-rec
DOMAIN=gui/$(id -u)

if launchctl print $DOMAIN/$LABEL >/dev/null 2>&1; then
  echo "もう録音中です"; exit 1
fi

DIR=$ROOT/meetings/$(date +%Y-%m-%d-%H%M)
mkdir -p $DIR
cat >$DIR/rec.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>$LABEL</string>
  <key>ProgramArguments</key><array><string>$ROOT/meeting-rec</string><string>$DIR</string></array>
  <key>RunAtLoad</key><true/>
  <key>StandardErrorPath</key><string>$DIR/rec.log</string>
</dict></plist>
EOF

launchctl bootstrap $DOMAIN $DIR/rec.plist || exit 1
echo $DIR >$ROOT/.recording
sleep 3
if ! launchctl print $DOMAIN/$LABEL | grep -q "state = running"; then
  echo "すぐ止まりました: $DIR/rec.log" >&2
  launchctl bootout $DOMAIN/$LABEL 2>/dev/null; rm -f $ROOT/.recording; exit 1
fi
echo "録音中: $DIR"
```

初めて動かすと、macOS が「画面とシステムオーディオの収録」と「マイク」の2つの許可を `meeting-rec` に求めます。両方許可したあと、一度止めてもう一度始めると録れるようになります。

止める側は、SIGTERM を送ってファイルを閉じさせ、登録を外します。

```bash:stop.sh（前半）
#!/bin/zsh
set -u
ROOT=${0:A:h}
LABEL=com.example.meeting-rec
DOMAIN=gui/$(id -u)
DIR=$(cat $ROOT/.recording 2>/dev/null)

launchctl kill SIGTERM $DOMAIN/$LABEL
for i in {1..20}; do
  launchctl print $DOMAIN/$LABEL 2>/dev/null | grep -q "state = running" || break
  sleep 0.5
done
launchctl bootout $DOMAIN/$LABEL 2>/dev/null
rm -f $ROOT/.recording
```

これで録音は止まります。……と思っていたのですが、ここで1つ問題が残りました。

## 3. 止めたのにオレンジのマイクが消えない：replayd の原因と直し

録音を止めたあとも、メニューバーにオレンジのマイクの印（マイクが使われている合図）が残ることがありました。`meeting-rec` はもう終わっているのに、です。使う側からすると「まだ録られているのでは」と不安になります。筆者も使い始めた日に2回、これで手を止めました。

### 原因：収録の裏方がマイクを放していなかった

今どのプロセスがマイクを使っているかを、Core Audio（macOS の音の土台の仕組み）に聞いてみると、残っていたのは `meeting-rec` ではなく `/usr/libexec/replayd` でした。replayd は、画面収録を裏で受け持つ macOS のプロセスです。ScreenCaptureKit でマイクを録ると、マイクの入力は実際にはこの replayd が開いています。手元の環境では、`stopCapture()` のあとも replayd が入力を開いたままになる回があり、それがオレンジの印の正体でした。

### マイクを使っているプロセスを調べるコマンド

最近の macOS の Core Audio には、「音を使っているプロセスの一覧」（`kAudioHardwarePropertyProcessObjectList`）を聞く仕組みがあります。そこから、入力（マイク）を使っているものの pid を出す小さなコマンドを作りました。

```swift:mic_users.swift
import CoreAudio

func address(_ selector: AudioObjectPropertySelector) -> AudioObjectPropertyAddress {
    AudioObjectPropertyAddress(mSelector: selector,
                               mScope: kAudioObjectPropertyScopeGlobal,
                               mElement: kAudioObjectPropertyElementMain)
}

func uint32(_ object: AudioObjectID, _ selector: AudioObjectPropertySelector) -> UInt32 {
    var addr = address(selector)
    var value: UInt32 = 0
    var size = UInt32(MemoryLayout<UInt32>.size)
    AudioObjectGetPropertyData(object, &addr, 0, nil, &size, &value)
    return value
}

let system = AudioObjectID(kAudioObjectSystemObject)
var listAddr = address(kAudioHardwarePropertyProcessObjectList)
var size: UInt32 = 0
AudioObjectGetPropertyDataSize(system, &listAddr, 0, nil, &size)
var processes = [AudioObjectID](repeating: 0,
                                count: Int(size) / MemoryLayout<AudioObjectID>.size)
AudioObjectGetPropertyData(system, &listAddr, 0, nil, &size, &processes)

// 入力（マイク）を動かしているプロセスの pid を1行ずつ出す
for process in processes where uint32(process, kAudioProcessPropertyIsRunningInput) != 0 {
    print(Int32(bitPattern: uint32(process, kAudioProcessPropertyPID)))
}
```

```bash
swiftc -O mic_users.swift -o mic_users
```

### 止めたあとに、replayd だけを放させる

`stop.sh` の後半で、マイクを使っているプロセスの中に replayd がいれば終了させます。replayd は macOS が必要なときに起動し直す裏方なので、次に録音を始めるときにはまた立ち上がります。**名前を確かめて replayd だけを対象にする**のが大事です（会議アプリなど、ほかのマイクの使い手は止めません）。

```bash:stop.sh（後半）
# 画面収録の裏方（replayd）がマイクを開いたままだと、オレンジの印が残る
sleep 1
for pid in $($ROOT/mic_users 2>/dev/null); do
  if [ "$(ps -o comm= -p $pid 2>/dev/null)" = "/usr/libexec/replayd" ]; then
    kill $pid 2>/dev/null && echo "replayd にマイクを放させました"
  fi
done

# 文字起こしと議事録づくりは裏で続ける
nohup $ROOT/process.sh $DIR >>$DIR/process.log 2>&1 &
echo "録音を止めました。議事録を作っています: $DIR"
```

会議がもう終わっているタイミングで流すので、会議アプリのマイクとぶつかることもありません。これでオレンジの印は止めるたびに消えるようになりました。録音の段はここまでです。次は、録った2本の音を文字にします。

## 4. 文字起こし：faster-whisper で混ぜた音を1回だけ文字にする

### 最初は2本を別々に文字にして、時刻の順がずれた

最初は、相手の wav と自分の wav をそれぞれ Whisper にかけて、あとで時刻順に並べ直す作りにしました。ところが、Whisper の区切り（セグメント）の始まりと終わりの時刻は2本の間で細かくは揃わないので、並べ直すと発言の順がずれ、読むと会話の流れがおかしくなりました。

そこで、**2本を混ぜた音を1回だけ文字にし、各区切りの時間帯で、どちらの音が大きかったかで話し手を決める**形に変えました。時刻は1本の文字起こしの中で決まるので、順番はずれません。

### 音を揃えて混ぜる（ffmpeg）

Whisper は 16kHz・1ch の音を前提にしているので、先に ffmpeg（音声・動画の変換の道具）で揃えます。

```bash
ffmpeg -y -loglevel error -i system.wav -ac 1 -ar 16000 system16k.wav
ffmpeg -y -loglevel error -i mic.wav    -ac 1 -ar 16000 mic16k.wav
ffmpeg -y -loglevel error -i system16k.wav -i mic16k.wav \
  -filter_complex "amix=inputs=2:duration=longest:normalize=0" -ac 1 -ar 16000 mix16k.wav
```

`normalize=0` を付けないと、`amix` が音量を入力の数で割るため、全体が小さくなります。

### 文字起こしと話し手の付け方（参考コード）

```python:transcribe.py
"""使い方: python transcribe.py 出力.md 混ぜた音.wav 相手=system16k.wav 自分=mic16k.wav"""
import sys
import wave

import numpy as np
from faster_whisper import WhisperModel

RATE = 16000
MODEL = "mobiuslabsgmbh/faster-whisper-large-v3-turbo"


def read_wav(path: str) -> np.ndarray:
    with wave.open(path, "rb") as w:
        return np.frombuffer(w.readframes(w.getnframes()), dtype=np.int16).astype(np.float32)


def rms(samples: np.ndarray, start: float, end: float) -> float:
    part = samples[int(start * RATE):int(end * RATE)]
    return float(np.sqrt(np.mean(part ** 2))) if part.size else 0.0


def hms(sec: float) -> str:
    s = int(sec)
    return f"{s // 3600:02d}:{s % 3600 // 60:02d}:{s % 60:02d}"


def main() -> int:
    out_path, mix_path, *pairs = sys.argv[1:]
    tracks = [(name, read_wav(path)) for name, path in (p.split("=", 1) for p in pairs)]

    model = WhisperModel(MODEL, device="cpu", compute_type="int8")
    segments, _ = model.transcribe(
        mix_path,
        language="ja",
        beam_size=5,
        vad_filter=True,                  # 無音を飛ばす（無音での作り話を減らす）
        condition_on_previous_text=False, # 同じ文の繰り返しを減らす
    )

    rows: list[dict] = []
    for seg in segments:
        text = seg.text.strip()
        if not text:
            continue
        speaker = max(tracks, key=lambda t: rms(t[1], seg.start, seg.end))[0]
        last = rows[-1] if rows else None
        if last and last["speaker"] == speaker and seg.start - last["end"] < 1.5:
            rows[-1] = {**last, "end": seg.end, "text": f'{last["text"]} {text}'}
        else:
            rows.append({"start": seg.start, "end": seg.end, "speaker": speaker, "text": text})

    with open(out_path, "w", encoding="utf-8") as f:
        f.write("# 文字起こし\n\n")
        f.writelines(f'[{hms(r["start"])}] {r["speaker"]}: {r["text"]}\n' for r in rows)
    return 0 if rows else 1


if __name__ == "__main__":
    sys.exit(main())
```

- **モデル**：large-v3-turbo を `int8` で CPU に載せています。GPU が無くても、手元の Mac では5分弱の会議が文字起こしから議事録まで約2分で終わりました
- **`vad_filter=True`**：VAD（声のある所だけを見つける処理）で無音を飛ばします。無音の所で Whisper が文を作ってしまう失敗が減ります
- **同じ話し手の続きはまとめる**：1.5秒以内に同じ人の区切りが続いたら1行にし、読みやすくしています

話し手は「相手／自分」の2つだけです。相手が3人いても全員「相手」になります。人ごとに分けたいときは話者分離（pyannote など）を足す手がありますが、モデルの登録と数 GB の追加が要るので、今回は入れていません。議事録では「だれが言ったか」より「何が決まったか」が大事なので、2つで足りています。

文字起こしができれば、あとは読みやすい形にまとめるだけです。ここで Claude を使います。

## 5. 議事録：claude -p に形を決めて渡す

`claude -p` は、Claude Code を対話なしで1回だけ動かし、結果を標準出力に返す使い方です。指示を引数に、文字起こしを標準入力に渡します。

```bash
claude -p "$(cat minutes-prompt.md)" < 文字起こし.md > 議事録.md.tmp \
  && [ -s 議事録.md.tmp ] && mv 議事録.md.tmp 議事録.md
```

一時ファイルに書いてから移すのは、途中で失敗したときに空の議事録が残らないようにするためです。

指示の文（プロンプト）は、出してほしい形をはっきり書きます。特に効いたのは、**分からないことを埋めさせない**一文でした。

```markdown:minutes-prompt.md
標準入力に、オンライン会議の文字起こしがあります。各行は「[時刻] 話し手: 発言」です。
話し手は「相手」（Mac から出た音）と「自分」（マイク）の2つだけです。
自動の文字起こしなので、聞き間違い・同じ文の繰り返し・声の重なりがあります。

次の形の議事録を日本語の Markdown で書いてください。出力は本文だけにし、前置きは書かないでください。
ファイルやツールは使わないでください。

# 議事録（主な話題を一言で）

## 決まったこと
- 1行に1つ。決まっていないことは書かない

## やること
| やること | 担当 | 期日 |
|---|---|---|
（担当と期日が会話に出なければ「未定」と書く。推測で埋めない）

## 話した要点
- 数字・固有名詞・理由を残す

## 確かめたほうがよいこと
- 聞き間違いの疑いがある数字・名前・日付と、結論が出ずに終わった話
```

最後の「確かめたほうがよいこと」の節が、思った以上に役に立ちました。合成音声で作った約5分の2人の会議で試したところ、Whisper が「機能の数」を「昨日の数」と聞き間違えた箇所を、Claude がこの節に聞き間違いの疑いとして拾っていました。決まったこと・やることの担当も、台本と合っていました。**文字起こしの誤りを直させるのではなく、怪しい所を人に見せる**ほうが、議事録としては安全です。

ここまでで、録音から議事録までが1本につながりました。最後に、実際に使ってみてつまずいた所をまとめます。

## うまくいかないとき

| 症状 | 原因 | 直し方 |
|---|---|---|
| 許可の画面が出ず、すぐ止まる | ターミナルから直接起動していて、許可の持ち主がターミナルになっている | launchd から起動する（2章）。システム設定の「プライバシーとセキュリティ」で持ち主を確かめる |
| 作り直したら録れなくなった | 署名が変わり、前の許可と別物と見なされた | 同じ識別子で署名し、許可を入れ直す。録音のコマンドはなるべく作り直さない |
| `system.wav` が無い・無音 | 会議の音が鳴っていない／`capturesAudio` が false | 会議の音が実際に鳴っているか、設定を確かめる |
| 止めたのにオレンジのマイクが残る | replayd がマイクを開いたまま | 3章の `mic_users` で確かめ、replayd だけ終了させる |
| 「自分」の行に相手の発言が混じる | スピーカーの音がマイクに入っている | ヘッドホンかイヤホンを使う |
| 議事録が空 | `claude -p` が失敗した（ログイン切れなど） | 文字起こしは残っているので、`claude -p` の部分だけやり直す |
| 長い会議で終わらない | CPU での文字起こしは会議の長さに比例する | 目安は1時間の会議で20分ほど。止めたら裏で回して、終わったら通知を出す |

もう1つ、技術ではない注意として、**録る前に会議の相手へ「議事録のために録音します」と伝えて**ください。手元で完結する仕組みでも、録音していることは相手に分かるようにしておくのが前提です。

## まとめ

- 録音は ScreenCaptureKit（macOS 15 以降）で、相手の声とマイクを別々の wav に。ドライバーは要らない
- 許可の持ち主を録音のコマンド本体にするため、launchd から起動する
- 止めたあとにオレンジのマイクが残るのは replayd。Core Audio でマイクの使い手を調べ、replayd だけを放させる
- 文字起こしは2本を混ぜて faster-whisper で1回だけ。話し手は区切りごとの音の大きさで決める
- 議事録は `claude -p` に形を決めて渡し、推測で埋めさせず、怪しい所は「確かめたほうがよいこと」に出させる

月額の議事録機能をやめても、議事録の質は落ちませんでした。いちばんの違いは、録音が Mac の外に出ないことです。

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=mac-meeting-minutes-faster-whisper-claude) で活動中。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=mac-meeting-minutes-faster-whisper-claude) — ディレクターも使っているSEO診断ツールを公開中。
