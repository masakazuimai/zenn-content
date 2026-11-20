---
title: "RealityKitで玉の仕掛けゲームを作った：10倍の大きさと自前の跳ね返り"
emoji: "🔔"
type: "tech"
topics: ["realitykit", "swift", "ios", "swiftui", "gamedev"]
published: true
---

RealityKit の物理で、ドミノ・坂・振り子・ベルを玉でつなぐゲームを作るとき、つまずきやすい点は3つでした。

1. **本物の大きさ（ドミノの高さ48mm）だと物理がうまく動かない** → 場面を10倍の大きさにし、重力も10倍にする
2. **跳ね返りの設定（restitution）が、重力で落ちた玉に効かない** → 当たりの知らせを合図に、跳ね返りを自分で付ける
3. **部品どうしのつなぎ目で玉が外れる** → 部品ごとに「入口」と「出口」を持たせ、置いた部品を出口に吸い付ける

この記事では、iOS ゲーム「からくりマーブル」を作ったときに試しのアプリで測った数字と、実際のコードでこの3つを説明します。最後に、玉が届かなかったときに「どこを直せばよいか」を出す作りも紹介します。

**対象読者**: SwiftUI は書けて、RealityKit の `ModelEntity`・`PhysicsBodyComponent` を少し触ったことがある方（中級向け）。

## 作ったもの

木の机に部品（坂・曲がるレール・らせん・筒・はね板・ドミノ・振り子・シーソー・ベルなど）を並べて「動かす」を押すと、ガラスの玉が転がり、最後のベルを鳴らせたら成功、というゲームです。

![組んだ装置。右上の玉がらせんと坂を下り、振り子とドミノを通ってベルへ向かう](/images/realitykit-marble-machine-physics/1-machine.jpg =300x)
*組んだ装置。右上のガラスの玉が、らせんと坂を下り、振り子とドミノを通ってベルへ向かう*

![動かしたあと。ドミノが倒れてベルが鳴り「クリア！」が出た画面](/images/realitykit-marble-machine-physics/2-clear.jpg =300x)
*動かしたあと。ドミノが倒れてベルが鳴ると「クリア！」になる*

- SwiftUI ＋ RealityKit（iOS 18 以上）
- 中の物理エンジンは PhysX（NVIDIA の物理エンジン。RealityKit の中で動いていることが、実行中の記録の文から分かりました）
- 玉はガラス玉・ピンポン球・スーパーボールの3種類（大きさ・重さ・跳ね方が違う）

最初に「RealityKit のままで作れるか」を確かめる小さな試しのアプリを作り、ドミノの数・大きさ・重力を変えて測りました。1つ目のつまずきは、この試しで見つかりました。

## 1. 本物の大きさでは、押したドミノが倒れない

ドミノを本物の大きさ（8×48×24mm）で30枚並べ、先頭を押しました。結果は次のとおりです（シミュレータ・iPhone 16）。

| 大きさ | 重力 | 倒れた数 | 波の速さ（本物の大きさに直して） |
|---|---|---|---|
| 1倍 | 9.81 m/s² | 0/30（先頭を15度傾けて置いても倒れない） | — |
| 10倍 | 9.81 m/s² | 30/30 | 0.28 m/s |
| **10倍** | **98.1 m/s²（10倍）** | **30/30** | **0.86 m/s** |

本物の大きさでは1枚も倒れません。振り子も、糸の長さ120mmに対して1倍では8.4mm（7%）も伸びてしまいました（10倍では0.55mm）。Apple の開発者フォーラムにも「小さすぎる物は思わぬ動きをする」「動きの小さい物は物理が眠る（計算を止める）」という回答があり、小さい物は苦手と見てよさそうです。

### 10倍にするなら、重力も10倍にする

10倍の大きさにすると倒れるようにはなりますが、波の速さが 0.28 m/s と、ゆっくりに見えます。これは物の倒れる時間が `√(長さ ÷ 重力)` に比例するためです。長さだけ10倍にすると時間は √10 ≒ 3.2 倍に延び、スローモーションになります。

**長さも重力も10倍にすれば、時間はそのまま**になります。実際、重力も10倍にした場面の波の速さ 0.86 m/s は、ドミノの波の論文（van Leeuwen「The Domino Effect」2004）の式から出した値（ドミノどうしの摩擦0.2で約0.87 m/s）とほぼ同じでした。

設定は、場面の一番上の Entity に `PhysicsSimulationComponent` を付けて重力を書き換えるだけです。

```swift
// 場面は10倍の大きさで作る（部品の寸法・位置は本物のメートル × s）
let s: Float = 10

var simulation = PhysicsSimulationComponent()
simulation.gravity = [0, -9.81 * s, 0]   // 重力も10倍
world.components.set(simulation)
```

玉の重さも「密度 × 場面の体積」で出し、ドミノやシーソーと同じ出し方にそろえました。こうすると重さの比が本物どおりになります。

```swift
static func ball(scale s: Float, kind: BallKind) -> ModelEntity {
    let radius = kind.radius * s               // 本物の半径（ガラス玉 12mm など）× 10
    let entity = ModelEntity(mesh: .generateSphere(radius: radius), materials: [glass])
    let shape = ShapeResource.generateSphere(radius: radius)
    let mass = kind.density * (4.0 / 3.0) * .pi * radius * radius * radius
    entity.components.set(CollisionComponent(shapes: [shape]))
    var body = PhysicsBodyComponent(shapes: [shape], mass: mass, material: kind.physics, mode: .dynamic)
    // ピンポン球は中が空洞＝回りにくさ（慣性モーメント）を 2/3·m·r² にする（詰まった球は 2/5）
    if kind == .pingpong {
        body.massProperties = PhysicsMassProperties(mass: mass, inertia: SIMD3(repeating: 2.0 / 3.0 * mass * radius * radius))
    }
    entity.components.set(body)
    entity.components.set(PhysicsMotionComponent())
    return entity
}
```

10倍＋重力10倍は、坂の玉でも確かめました。高さ99mmの坂を下りた玉の速さは 1.04 m/s で、滑らずに転がる球の計算 `√(10/7 × g × h)` ＝ 1.18 m/s より12%遅いだけでした（坂から床へ乗り移るときの当たりの分と見ています）。

大きさが決まったところで、次は玉が跳ねない問題です。

## 2. restitution が、落ちた玉に効かない

RealityKit では、跳ね返りの強さを `PhysicsMaterialResource` の `restitution`（反発係数。1に近いほどよく跳ねる）で決めます。ところが、10倍＋重力10倍の場面で玉を落とすと、0.3〜0.9 のどれにしても跳ねませんでした。重力なしで速く当てたときだけ効きます。

フォーラムにも「restitution を 1.0 にしても大きくエネルギーが減る」という報告があり、私は設定で合わせるのをやめ、跳ね返りを自分で付けることにしました。

### 当たりの知らせを合図に、次のコマで速さを書き換える

やり方は次の2段です。

1. `CollisionEvents.Began`（当たり始めの知らせ）で、玉が何かに当たったことだけ覚える
2. 次のコマで「当たる前の速さ」と「物理が止めたあとの速さ」を比べ、**速さが変わった向き**を当たった面の向きとみなし、その向きの分を e 倍で逆向きにする

```swift
@MainActor
final class BounceHelper {
    let ball: Entity
    let restitution: Float       // e：ガラス玉 0.5・ピンポン球 0.88・スーパーボール 0.9
    let minimumSpeed: Float      // これより遅い当たりは跳ねさせない（転がっているだけの玉が震えないように）
    private var lastVelocity: SIMD3<Float> = .zero
    private var pending: Float?

    func handle(_ event: CollisionEvents.Began) {
        guard event.entityA == ball || event.entityB == ball else { return }
        pending = restitution
    }

    // 毎コマ呼ぶ
    func track() {
        guard let velocity = ball.components[PhysicsMotionComponent.self]?.linearVelocity else { return }
        defer { lastVelocity = ball.components[PhysicsMotionComponent.self]?.linearVelocity ?? velocity }
        guard let e = pending else { return }
        pending = nil

        let change = velocity - lastVelocity            // 物理が実際に止めた分
        guard simd_length(change) > minimumSpeed else { return }
        let normal = simd_normalize(change)             // ＝当たった面の向き
        let incoming = simd_dot(lastVelocity, normal)   // 面に向かっていた速さ（負）
        guard incoming < -minimumSpeed else { return }
        let tangent = velocity - simd_dot(velocity, normal) * normal   // 面に沿った速さは残す
        var motion = ball.components[PhysicsMotionComponent.self] ?? PhysicsMotionComponent()
        motion.linearVelocity = tangent - e * incoming * normal
        ball.components.set(motion)
    }
}
```

`handle` は `scene.subscribe(to: CollisionEvents.Began.self)` で、`track` は毎コマの更新（`SceneEvents.Update`）で呼びます。

### 当たった場所から向きを出すと失敗した

最初は、知らせに入っている当たった場所（`event.position`）から面の向きを出していました。ところがこの場所は正確でなく、横向きの速さを逆にして玉があらぬ方へ飛ぶことがありました。「物理が実際に止めた向き」を使う形に変えてから、跳ね方が安定しました。

### はね板だけは、板の本当の向きを使う

傾いた板で玉を前へ飛ばす「はね板」では、上の推し量りでも外れることがありました。当たる前の向きのほんの少しの違いで、次の部品に届く・届かないが玉の種類ごとに入れ替わったのです。はね板だけは、板の Entity の向き（`orientation`）を面の向きとして使い、「上へ30度・当たる前の速さの e 倍」で飛ばす決まった形にしました。

```swift
let normal = pad.orientation(relativeTo: nil).act(SIMD3<Float>(0, 1, 0))
let across = simd_normalize(SIMD3(normal.x, 0, normal.z))
let speed = e * simd_length(lastVelocity)
motion.linearVelocity = across * (speed * 0.866) + SIMD3<Float>(0, speed * 0.5, 0)   // cos30°・sin30°
```

飛び方を式で決めたので、「どこまで飛ぶか」も同じ式で前もって計算できます。これが次の「部品のつなぎ」で効いてきます。

## 3. 部品に入口と出口を持たせて、つなぎ目で外れないようにする

物理が本物らしく動いても、部品の置き方が数ミリずれていると玉は溝の端に落ちます。遊ぶ人に数ミリ単位で置いてもらうのは無理なので、部品ごとに**入口（Port）と出口**を持たせました。

```swift
// 部品の中での位置（本物のメートル）と、玉の進む向き
struct Port {
    let position: SIMD3<Float>
    let direction: SIMD3<Float>
    var pushes = false   // 玉ではなく「押す力」が出る出口（倒れるドミノ・振れる振り子）

    var isHorizontal: Bool { abs(direction.y) < 0.5 }
}

enum PartPorts {
    static func entry(_ kind: PartKind) -> Port? {
        switch kind {
        case .ramp:   return Port(position: [-0.1, 0.105, 0], direction: [1, 0, 0])
        case .tube:   return Port(position: [0, 0.15, 0], direction: [0, -1, 0])   // 真下に落ちる入口
        case .domino: return Port(position: [-0.012, 0.012, 0], direction: [1, 0, 0])
        // …
        }
    }

    static func exit(_ kind: PartKind) -> Port? {
        switch kind {
        case .ramp:   return Port(position: [0.12, 0.011, 0], direction: [1, 0, 0])
        case .domino: return Port(position: [0.012, 0.012, 0], direction: [1, 0, 0], pushes: true)
        case .bell:   return nil   // ゴール
        // …
        }
    }

    // 置いた部品の向き（15度きざみ）と位置で、机の上の入口・出口に直す
    static func world(_ port: Port, of part: PlacedPart) -> Port {
        let turn = simd_quatf(angle: Float(part.rotation) * .pi / 12, axis: [0, 1, 0])
        return Port(position: part.origin + turn.act(port.position), direction: turn.act(port.direction), pushes: port.pushes)
    }
}
```

横向きの入口の高さは「溝に乗った玉の真ん中」にそろえました。出口も同じ決め方にしておくと、「前の出口の玉が、そのまま次の入口に入れる高さか」を数字で比べられます。

### 置いた部品を、近くの出口へ吸い付ける

部品を置く（動かす）たびに、近くの空いている出口を探し、入口がその出口の少し先・少し下に来るように位置を直します。

- 溝から溝（坂・曲がるレール・らせん）は、つなぎ目を 5mm につめる（あけると玉が溝の端に落ちる）
- そのほかは 25mm 先（玉が飛ぶ分）
- 押す出口（ドミノ・振り子）の次は、同じ床の高さのすぐ隣
- はね板の次は、2章の式で出した「飛ぶ遠さ」の所

```swift
// はね板で跳ねた玉が横へ飛ぶ遠さ。落とす高さ h から、上へ30度・e 0.9 で飛び出し、3cm 下で受ける
let speed = (2 * g * h).squareRoot() * 0.9
let up = speed * 0.5
let across = speed * 0.866
let time = (up + (up * up + 2 * g * 0.03).squareRoot()) / g
return across * time
```

部品はマス目に置きますが、マス目のままだと最大1.5cmずれて玉が外れました。そこで、マス目に合わせたうえで余りを `nudgeX`・`nudgeZ` に持たせ、ぴったり合わせています。

```swift
let exactX = best.point.x - offset.x
snapped.gridX = Int((exactX / cell).rounded())
snapped.nudgeX = exactX - Float(snapped.gridX) * cell   // マス目からのずれ
```

高さは段（決まった厚みの台）で上げ下げします。入口が出口より高くて届かないときは、0段のまま置き、「坂の出口が低すぎます。坂を2段上げるとつながります」のように知らせます。

## 4. 届かなかったときに、どこを直せばよいかを出す

組んで動かしても、たいていは1回目では届きません。ただ「失敗しました」と出すだけでは、遊ぶ人はどこを直せばよいか分かりません。そこで、玉が止まった所と、3章の入口・出口から手がかりを出すことにしました。

### 止まった・落ちたを見分ける

```swift
let position = ball.position(relativeTo: nil)
if position.y < -0.3 * s { endMiss(at: position, fell: true); return }   // 机の外へ落ちた

let speed = simd_length(ball.components[PhysicsMotionComponent.self]?.linearVelocity ?? .zero) / s
let waiting = pendingVelocity != nil || elevatorBusy || now.timeIntervalSince(playStarted) < 1
if speed < 0.02, !waiting {                                    // 本物に直して秒速2cm 未満が
    if let since = stillSince, now.timeIntervalSince(since) > 1.5 {   // 1.5秒続いたら止まった
        endMiss(at: position, fell: false)
    } else if stillSince == nil { stillSince = now }
} else {
    stillSince = nil
}
```

エレベーターで運ばれている間や、動かした直後の1秒は「止まった」に数えません。これを入れる前は、玉が待っているだけなのに失敗扱いになりました。

### 近い出口と、その次の入口の高さの差で案内する

止まった所に赤い印を置き、次の順で案内の文を決めます。

1. 止まった所にいちばん近い**出口**＝玉が最後に出た部品
2. その出口にいちばん近い**入口**＝次の部品
3. 次の入口のほうが高ければ「何段上げる」、離れていれば「近づける」
4. 出口から遠くへ飛んでいたら（はね板など）、最後に通った部品の名前で「向きを回すか、行き先に置き直そう」

ここで1つ失敗がありました。「次の部品を2段上げよう」と案内すると、その部品の入口が今度は前の部品より高くなり、そこで止まってしまうのです。案内のとおりに直したのに、また別のことを言われる形になりました。今は「前の部品も全部同じだけ上げる」と案内しています。

## うまくいかないとき

ここまでで扱わなかったものも含めて、試しと作る途中でつまずいた所をまとめます。

| 症状 | 原因 | 直し方 |
|---|---|---|
| 押したドミノが倒れない・振り子の糸が伸びる | 物が小さすぎる | 場面を10倍の大きさに、重力も10倍に（1章） |
| 10倍にしたら全部スローモーションに見える | 重力がそのまま | 重力も同じ倍率に（1章） |
| 玉が跳ねない | restitution が落ちた玉に効かない | 跳ね返りを自分で付ける（2章） |
| 速い玉（秒速5m）が壁を抜ける | すり抜け防止（`isContinuousCollisionDetectionEnabled`）がこの形では効かなかった | 玉の速さに上限を付け、部品の当たりの形を厚くする |
| 振り子の糸（つなぎ）が効かず玉が落ちる | 支点に当たりの形が無い | 支点にも `CollisionComponent` を付ける |
| `.static` から `.dynamic` に変えたその場で入れた速さが効かない | 切り替えのコマでは速さが入らない | 次のコマで `PhysicsMotionComponent` に入れる |
| 動かしても玉が落ちないことがある | 待っている間に玉の物理が眠った | 最初のコマで下向きに小さな速さを入れて起こす |
| 玉が低い所に当たり、ドミノが倒れずに滑る | 当たる高さが低い | 当てる高さは部品の作りで決める・木の摩擦を少し高めに（止まり 0.5） |

すり抜けは、これでも抜けるときは「前のコマから今のコマまでの線」で当たりを自分で調べる方法もあります。からくりマーブルでは、速さの上限と厚い当たりの形で足りました。

## まとめ

- RealityKit で机の上の小さな仕掛けを作るなら、**場面を10倍の大きさにし、重力も10倍**にすると、ドミノも振り子も本物の速さで動く
- 跳ね返りは restitution に頼らず、**当たりの知らせ＋次のコマの速さの差**で自分で付けると安定する。決まった跳び方をさせたい板は、板の向きから式で飛ばす
- 部品に**入口と出口**を持たせると、吸い付き・高さの知らせ・失敗の案内まで同じ数字で作れる

私は最初、物理エンジンの設定値を合わせれば済むと思っていました。実際には「設定で合わせるのをやめ、自分の式で決める」部分が増えるほど、遊ぶ人にとって分かりやすい動きになりました。式で決めた動きは、前もって計算でき、案内の文にも使えるからです。

## 作ったアプリ

この記事の仕組みで作ったのが、iOS ゲーム「からくりマーブル」です。くじで選んだ部品を木の机に並べ、ガラスの玉でベルを鳴らせたら成功です。無料で遊べます。

[からくりマーブル（App Store）](https://codequest.work/app/karakuri/zenn)

---

**参考**

- J.M.J. van Leeuwen「The Domino Effect」arXiv:physics/0401018（2004）https://arxiv.org/abs/physics/0401018
- Apple Developer Forums「Bouncy ball in RealityKit - game」https://developer.apple.com/forums/thread/791867
- Apple Developer Forums「visionOS RealityKit's physics simulation stops for certain entity.scale values」https://developer.apple.com/forums/thread/764728

---

**筆者について：** Web制作・SEOツール開発を行うフリーランス。[CodeQuest.work](https://codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=realitykit-marble-machine-physics) で活動中。
[Direbase（ディレベース）](https://seo.codequest.work/?utm_source=zenn&utm_medium=article&utm_campaign=realitykit-marble-machine-physics) — ディレクターも使っているSEO診断ツールを公開中。
