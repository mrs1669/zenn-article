---
title: "【Xcode15】SwiftLintの導入でXcodeCloudがFailする際の解決法"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Xcode, SwiftLint, XcodeCloud, CI, iOS]
published: false
---

## 概要

新規でXcodeプロジェクトを作成してSwiftLintを導入したところ、Xcode CloudがFailする事象が発生しました。
原因がXcode15に起因するものであることが判明したので共有します。

## 環境
> Xcode Cloud Xcodeバージョン: Latest Release(Xcode 15.3 (15E204a))
> Xcode Cloud macOSバージョン: Latest Release(macOS Sonoma 14.4.1 (23E224))
> SwiftLintバージョン: 0.54.0

## Failする事象

GitHub上では

``` shell: GitHub エラー文
❌ Command PhaseScriptExecution failed with a nonzero exit code
```

👆のエラーで怒られています。これでは原因がわからないので、

-----

Xcode Cloudに詳細なログ見に行くと👇のようなログになっています。

どうやら、**Run xcodebuild build** の項目で、`xcodebuild`で失敗していそうです。

``` shell: Xcode Cloud Failコマンド
Run command: 'xcodebuild build -scheme {scheme名} -project /Volumes/workspace/repository/{プロジェクト名}.xcodeproj -destination generic/platform=iOS -derivedDataPath /Volumes/workspace/DerivedData -resultBundleVersion 3 -resultBundlePath /Volumes/workspace/resultbundle.xcresult -resultStreamPath /Volumes/workspace/tmp/resultBundleStream307889e9-507b-4a36-8258-761abffcd349.json -IDEPostProgressNotifications=YES CODE_SIGN_IDENTITY=- AD_HOC_CODE_SIGNING_ALLOWED=YES COMPILER_INDEX_STORE_ENABLE=NO -hideShellScriptEnvironment'
```

``` shell: Xcode Cloud エラー文
❌ Command exited with non-zero exit-code: 65
```
