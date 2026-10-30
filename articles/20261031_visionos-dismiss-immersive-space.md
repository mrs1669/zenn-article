---
title: "【Apple Vision Pro】Immersive Spaceを閉じた後のonDisappear、シミュレーターと実機で桁違いだった件"
emoji: "🚪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [visionOS, SwiftUI, RealityKit, AppleVisionPro, ImmersiveSpace]
published: false
---

## はじめに

Apple Vision Pro 向けに、人型の 3D モデルを数百体、骨格のアニメーションで動かす、処理の重い Immersive Space を持つアプリを作っています。

このアプリで、Immersive Space を閉じたあと続けて重い処理を始めると、実機で約 10 秒後に `0x8BADF00D` で強制終了されました。シミュレーターで時刻を測ると、`dismissImmersiveSpace()` の await が返ったあと、Immersive Space の中のビューの `onDisappear` が最大 780 ms 遅れて来ていました。そこで「片付けの途中に重い処理を重ねたのが原因」と考えて直しましたが、あとで実機で測ると、その遅れはほとんどありませんでした（ ﾟдﾟ）

この記事では、次の 3 つを、測った値と一緒にまとめます。

1. **実機で測った、Immersive Space が閉じる順番**（シーンの通知・`scenePhase`・`onDisappear`・await）
2. **シミュレーターでだけ起きる `onDisappear` の遅れ**（同じアプリで、シミュレーターは 370〜780 ms、実機は 2.9〜6.0 ms）
3. **実機での測り方**（公開の API で順番を記録する方法と、アプリを入れ直さずに os_log を読む方法）

**確かめた事実** と **推測** を分けて書きます。推測には【推測】、まだ確かめていないことには【未確認】と付けます。

### 環境

| | 版 |
| --- | --- |
| Xcode | 27.1（27A9275） |
| 実機 | Apple Vision Pro・visionOS 27.2 beta（24N120）。強制終了が起きたのは visionOS 27.0 beta |
| シミュレーター | Apple Vision Pro・visionOS 26.5（23O470） |

---

## 1. 実機で測った、Immersive Space が閉じる順番

### 測り方

操作なしで「Immersive Space を開く → 2 秒待つ → 閉じる」を 5 回くり返す、最小のアプリを作りました。中身は `RealityView` に箱を 200 個並べただけです。閉じるときに、次の時刻を記録します。

- `dismissImmersiveSpace()` を呼んだ時刻と、await が返った時刻
- シーンの通知（`UIScene.willDeactivateNotification`・`didEnterBackgroundNotification`・`didDisconnectNotification`）。Immersive Space のシーンは、ロールが `UISceneSessionRoleImmersiveSpaceApplication` のもの
- 中のビューの `scenePhase` の変化と `onDisappear`
- RealityKit の System の `update` が、await のあとにも呼ばれたか

:::details 最小のアプリの Immersive Space の View（クリックで開く）
```swift
import QuartzCore
import RealityKit
import SwiftUI

/// System が毎フレーム数えるための目印（await のあとに update が来るかを見る。System 側は省略）
struct ProbeComponent: Component {}

struct ProbeImmersiveView: View {
    @Environment(\.scenePhase) private var scenePhase

    var body: some View {
        RealityView { content in
            let root = Entity()
            root.components.set(ProbeComponent())
            let mesh = MeshResource.generateBox(size: 0.08)
            let material = SimpleMaterial(color: .orange, isMetallic: false)
            for index in 0..<200 {
                let box = ModelEntity(mesh: mesh, materials: [material])
                box.position = [Float(index % 20) * 0.12 - 1.2, Float(index / 20) * 0.12, 0]
                root.addChild(box)
            }
            root.position = [0, 1.0, -2.5]
            content.add(root)
        }
        .onAppear {
            print("onAppear", CACurrentMediaTime())
        }
        .onDisappear {
            print("onDisappear", CACurrentMediaTime())
        }
        .onChange(of: scenePhase) { old, new in
            print("scenePhase", old, "→", new, CACurrentMediaTime())
        }
    }
}

// App では次のように置く
// ImmersiveSpace(id: "space") {
//     ProbeImmersiveView()
// }
```
:::

### 結果（実機・5 回）

`dismissImmersiveSpace()` を呼んだ時刻を 0 とします。

| 出来事 | 呼んでからの時間 |
| --- | --- |
| シーンの `willDeactivate` | 12〜16 ms |
| 中のビューの `scenePhase` が inactive | 19〜22 ms |
| シーンの `didEnterBackground` | 420〜427 ms |
| 中のビューの `scenePhase` が background | 427〜436 ms |
| シーンの `didDisconnect` | 430〜439 ms |
| 中のビューの `onDisappear` | 434〜445 ms |
| **await が返る** | 434〜445 ms（`onDisappear` の直後。5 回とも同じミリ秒） |

await が返ったあとに、RealityKit の System の `update` は 1 回も呼ばれませんでした。

分かったことは次のとおりです。

- await は、シーンの切り離し（`didDisconnect`）と、中のビューの `onDisappear` が済んでから返る。Apple のドキュメントの説明（下の表）どおり👍
- 中のビューの `scenePhase` は、呼んで約 20 ms で inactive になる。閉じ始めたことをいちばん早く知れるのはここ
- 閉じる処理そのものに約 430 ms かかる。このあいだメインスレッドが重いと、閉じる処理が遅れる

### ほかの「閉じる」との違い

| API | 宣言 | 返るとき（ドキュメント） | 実機で測った値 |
| --- | --- | --- | --- |
| `DismissAction`（`@Environment(\.dismiss)`。iOS のシートなど） | `func callAsFunction()`（同期） | 閉じるよう頼んだらすぐ返る | — |
| `DismissWindowAction`（visionOS などのウィンドウ） | `func callAsFunction(id:)` など（同期） | 同上 | 3〜4 ms で返る。`onDisappear` はその 220〜226 ms あと |
| `DismissImmersiveSpaceAction` | `func callAsFunction() async` | システムが閉じ終えてから返る | 434〜445 ms で返る。`didDisconnect` と `onDisappear` のあと |

`DismissImmersiveSpaceAction` のドキュメントには、次の一文があります。

> The asynchronous call returns after the system finishes dismissing the space.
> （Apple Developer Documentation「[DismissImmersiveSpaceAction](https://developer.apple.com/documentation/swiftui/dismissimmersivespaceaction)」）

iOS の `dismiss()` やウィンドウの `dismissWindow()` は「頼んだらすぐ返る」ので、呼んだ直後はまだ閉じていない前提で使います。`dismissImmersiveSpace()` は async で、実機では中のビューの `onDisappear` まで待ってから返りました。

シミュレーターでも、この最小のアプリでは、await の時点で `onDisappear` は済んでいました。中身を 13 通りに変えて 5 回ずつ（計 65 回）測っても同じでした（2 章の表）。

---

## 2. シミュレーターでだけ起きる `onDisappear` の遅れ

### 処理の重いアプリでは、シミュレーターで遅れた

実際のアプリ（人型の 3D モデル約 200 体を、骨格のアニメーションで動かす Immersive Space）で、await が返ってから中のビューの `onDisappear` までの時間を測りました。

| | シミュレーター（visionOS 26.5） | 実機（visionOS 27.2 beta） |
| --- | --- | --- |
| 最小のアプリ | `onDisappear` は await より先 | `onDisappear` は await より先 |
| 処理の重いアプリ | `onDisappear` は await の **370〜780 ms** あと | `onDisappear` は await の **2.9〜6.0 ms** あと |

同じアプリなのに、シミュレーターと実機で桁が違いますw

シミュレーターの UIKit のログでは、処理の重いアプリでも、シーンの片付け（`Scene will invalidate`）は await の 3〜5 ms 前に済んでいました。遅れていたのは、そのあとの中のビューの `onDisappear` だけです。

```text
（シミュレーター・処理の重いアプリ。時刻は秒）
05.147  dismissImmersiveSpace() を呼ぶ
05.585  [UIKit] Scene will invalidate: UIWindowScene …
05.590  await が返る（この時点で onDisappear はまだ）
05.961  中のビューの onDisappear                    ← await の約 370 ms あと
```

### 最小のアプリで、遅れる条件を探した（シミュレーター）

処理の重いアプリにあって最小のアプリに無いものを、1 つずつ足して測りました。どれでも遅れは再現しませんでした。

| 足したもの | await の長さ | await の時点で `onDisappear` は |
| --- | --- | --- |
| なし（mixed / full） | 424〜444 ms | 済み |
| System の `update` で毎フレーム 8 ms メインスレッドを止める | 453〜587 ms | 済み |
| 閉じる操作を持つビューを、閉じる前に画面から外す | 420〜423 ms | 済み |
| RealityView の attachments | 421〜423 ms | 済み |
| 33 ms ごとに状態を変える `.task` | 422〜430 ms | 済み |
| ARKit のワールドトラッキング | 415〜459 ms | 済み |
| 重い素材（大きな 3D モデルと、骨格のアニメーションつきの人型モデル 68 体） | 421〜432 ms | 済み |
| 空間オーディオを 9 か所で鳴らす | 420〜424 ms | 済み |
| 閉じる 2.5 秒前に、没入の方式を full から mixed に切り替える | 427〜435 ms | 済み |

【推測】シミュレーターで重い中身を描いていると、SwiftUI がビューを片付ける処理が後回しになると考えていますが、どの条件で起きるのかは突き止められていません。

:::message
**シミュレーターで `onDisappear` が遅れる条件が分かった方がいたら、教えてもらえるとうれしいです。**
:::

### 強制終了の原因について

最初は、シミュレーターで見えた遅れから「await のあと、中身の片付けが終わる前に重い処理を始めたため」と考え、`onDisappear` を待ってから重い処理を始めるように直しました。しかし、実機ではこの遅れは数 ms しかなく、原因の説明としては成り立ちませんでした（直したつもりが、的外れだったわけです (´・ω・｀)）。

実機のクラッシュのログで分かっているのは次のことです。

- `FRONTBOARD 0x8BADF00D`、「scene-update watchdog transgression: exhausted real (wall clock) time allowance of 10.00 seconds」
- 落ちたときのメインスレッドは、閉じる途中の Immersive Space の中で、人型モデルの骨格のメッシュを変形していた（`DeformedDirectMeshSystem::update` → … → `Mesh.modifyVertices` → `makeMeshCollection`）

【推測】実機では、骨格のメッシュの変形がメインスレッドで重く（Immersive Space を開いているあいだのアプリの CPU は平均 67%）、閉じる処理（約 430 ms のシーンの更新）の前後にメインスレッドが詰まって、シーンの更新が 10 秒以内に終わらなかったと考えています。閉じたあとに始めた重い処理がどう関わったか、どのシーンの更新が見張りに引っかかったかは、ログからは分かりませんでした。

【未確認】変形する人型モデルの数を減らしたあと、実機で同じ流れを試しても強制終了されないかは、まだ確かめていません。

**シミュレーターで見えたタイミングのずれを、そのまま実機の原因にしてはいけない**、というのが今回のいちばんの学びです📝

---

## 3. 実機での測り方

### 3-1. 公開の API で、閉じる順番を記録する

シーンの通知と、中のビューの `scenePhase`・`onDisappear`、await の前後の時刻を、同じ時計で記録します。

```swift
// シーンの通知（Immersive Space は UISceneSessionRoleImmersiveSpaceApplication）
for (name, label) in [(UIScene.willDeactivateNotification, "willDeactivate"),
                      (UIScene.didEnterBackgroundNotification, "didEnterBackground"),
                      (UIScene.didDisconnectNotification, "didDisconnect")] {
    NotificationCenter.default.addObserver(forName: name, object: nil, queue: .main) { note in
        let role = (note.object as? UIScene)?.session.role.rawValue ?? "?"
        print(label, role, CACurrentMediaTime())
    }
}

// 中のビュー
ImmersiveSpace(id: "space") {
    ProbeView()
        .onChange(of: scenePhase) { old, new in print("scenePhase", old, new, CACurrentMediaTime()) }
        .onDisappear { print("onDisappear", CACurrentMediaTime()) }
}

// 閉じる側
print("dismiss call", CACurrentMediaTime())
await dismissImmersiveSpace()
print("dismiss return", CACurrentMediaTime())
```

実機を装着したまま操作しなくて済むよう、起動したら自動で開け閉めをくり返し、終わったら `exit(0)` する作りにしました。Mac から入れて起動し、標準出力を受け取ります。アプリへの起動引数は `--` の後ろに書きます。

```bash
xcrun devicectl device install app --device <端末> DismissTiming.app
xcrun devicectl device process launch --console --terminate-existing --device <端末> \
  com.example.DismissTiming -- -cycles 5 -style full
```

### 3-2. 入れてあるアプリを入れ直さずに、os_log を読む

処理の重いアプリは、閉じきるまで待った時間を os_log に出していました。環境変数 `OS_ACTIVITY_DT_MODE` を付けて起動すると、os_log が標準エラーにも出るので、`devicectl` の `--console` でそのまま受け取れます。アプリを入れ直したり、ログを足したビルドを作ったりしなくて済むので楽です✨

```bash
xcrun devicectl device process launch --console --terminate-existing --device <端末> \
  --environment-variables '{"OS_ACTIVITY_DT_MODE":"YES"}' <Bundle ID> -- <アプリの起動引数>
```

人の操作が要る場面（今回は、Immersive Space を閉じるボタンを押す）は、記録の上限の時間を長めにしておきます。上限で `devicectl` を止めると、そのあとのログは受け取れません。

### 3-3. 試しのアプリの署名

試しのアプリは、自分のチームの開発用の署名で入れました。`xcodebuild` に `-allowProvisioningUpdates` を付けなければ、手元にある開発用のプロファイル（実機を含む、Xcode が管理するワイルドカードのもの）だけが使われ、開発者アカウントに新しい App ID などは登録されません。

```bash
xcodebuild -project DismissTiming.xcodeproj -scheme DismissTiming \
  -destination 'generic/platform=visionOS' build \
  CODE_SIGN_STYLE=Automatic DEVELOPMENT_TEAM=<チーム ID>
```

---

## まとめ

- **実機では、`dismissImmersiveSpace()` の await は、シーンの `didDisconnect` と中のビューの `onDisappear` のあとに返る**（呼んでから約 430 ms）。中のビューの `scenePhase` は約 20 ms で inactive になる
- **シミュレーターでは、処理の重い Immersive Space だと、中のビューの `onDisappear` が await より最大 780 ms 遅れた**。実機では同じアプリで 3〜6 ms。最小のアプリではどちらも遅れない
- **タイミングは実機で測る**。公開の API の通知と `scenePhase` で順番を記録でき、入れてあるアプリの os_log も `OS_ACTIVITY_DT_MODE` で読める
- 閉じたあとの重い処理は `onDisappear` を目印に始めると、実機では待ち時間がほとんど増えず、シミュレーターでも順番が崩れない。ただし、閉じる処理の前後のメインスレッドを軽くすることのほうが大事（今回の強制終了は、こちらが原因だと考えている）

:::message
**シミュレーターで `onDisappear` が遅れる条件が分かった方がいたら、教えてもらえるとうれしいです。**
:::

最後に、ちょっとだけぼやかせてください🙏

AI 時代になって、いろいろ AI にお願いしてものを作るのがずいぶん楽になったと思っていたのですが、（当たり前ですがw）Apple Vision Pro は iOS 以上にシミュレーターと実機の挙動差があると感じる今日この頃です。

AI にお願いしながらご飯を食べたり、ほかのことをしようにも、実機をかぶっていないといけないので、かぶったままご飯を食べたり、サボろうとしたら急に画面が出てきて「わぁっ😲」となったり、おでこにかけて待ってみたり（Apple Vision Pro は外すと自動で閉じる仕様になっています）で、大変ですよね…(´；ω；｀)

## 参考文献

Apple Developer Documentation（2026-10-31 に参照）

- SwiftUI
  - [DismissImmersiveSpaceAction](https://developer.apple.com/documentation/swiftui/dismissimmersivespaceaction)
  - [OpenImmersiveSpaceAction](https://developer.apple.com/documentation/swiftui/openimmersivespaceaction)
  - [ImmersiveSpace](https://developer.apple.com/documentation/swiftui/immersivespace)
  - [DismissAction](https://developer.apple.com/documentation/swiftui/dismissaction)
  - [DismissWindowAction](https://developer.apple.com/documentation/swiftui/dismisswindowaction)
  - [ScenePhase](https://developer.apple.com/documentation/swiftui/scenephase)
  - [onDisappear(perform:)](https://developer.apple.com/documentation/swiftui/view/ondisappear(perform:))
- UIKit
  - [UIScene.willDeactivateNotification](https://developer.apple.com/documentation/uikit/uiscene/willdeactivatenotification)
  - [UIScene.didEnterBackgroundNotification](https://developer.apple.com/documentation/uikit/uiscene/didenterbackgroundnotification)
  - [UIScene.didDisconnectNotification](https://developer.apple.com/documentation/uikit/uiscene/diddisconnectnotification)
  - [UISceneSession.Role.immersiveSpaceApplication](https://developer.apple.com/documentation/uikit/uiscenesession/role-swift.struct/immersivespaceapplication)
- Xcode（クラッシュの調べ方）
  - [Addressing watchdog terminations](https://developer.apple.com/documentation/xcode/addressing-watchdog-terminations)
  - [Understanding the exception types in a crash report](https://developer.apple.com/documentation/xcode/understanding-the-exception-types-in-a-crash-report)
