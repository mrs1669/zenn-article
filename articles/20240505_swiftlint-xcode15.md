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

:::message alert
Command PhaseScriptExecution failed with a nonzero exit code
:::

👆のエラーで怒られています。これでは原因がわからないので、

-----

Xcode Cloudに詳細なログを見にいくと👇のようなログになっています。
![xcodecloud-log](https://raw.githubusercontent.com/mrs1669/zenn-article/main/Resources/Images/20240505_swiftlint-xcode15/xcodecloud-log.png)

どうやら、**Run xcodebuild build** の項目で、`xcodebuild`で失敗していそうです。

``` shell: Xcode Cloud Failコマンド
Run command: 'xcodebuild build -scheme {scheme名} -project /Volumes/workspace/repository/{プロジェクト名}.xcodeproj -destination generic/platform=iOS -derivedDataPath /Volumes/workspace/DerivedData -resultBundleVersion 3 -resultBundlePath /Volumes/workspace/resultbundle.xcresult -resultStreamPath /Volumes/workspace/tmp/resultBundleStream307889e9-507b-4a36-8258-761abffcd349.json -IDEPostProgressNotifications=YES CODE_SIGN_IDENTITY=- AD_HOC_CODE_SIGNING_ALLOWED=YES COMPILER_INDEX_STORE_ENABLE=NO -hideShellScriptEnvironment'
```

:::message alert
Command PhaseScriptExecution failed with a nonzero exit code
:::

-----

これでも原因はわからないので、詳細なログをダウンロードして確認します。
成果物 -> **Logs for {アプリ名} build** を確認してみます。

zipファイルになっているので解凍すると、5個ほどログファイルが入っていて今回は `xcodebuild-build.log` をチェックします。

5000行程度のログファイルなので、探すのが大変ですが `fail` のワードで探したらエラー箇所がすぐ見つかりました。

``` shell: Xcode Cloud ログ
2024-05-04T19:00:55.692446439Z	PhaseScriptExecution Run\ SwiftLint /Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build/Script-4FAB6FF02BE1914D00B2F33A.sh (in target '{hogehoge}' from project '{hogehoge}')
2024-05-04T19:00:55.692455732Z	    cd /Volumes/workspace/repository
2024-05-04T19:00:58.183364314Z	    /usr/bin/sandbox-exec -D SCRIPT_INPUT_FILE_0\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build/Script-4FAB6FF02BE1914D00B2F33A.sh -D SCRIPT_INPUT_ANCESTOR_0\=/ -D SCRIPT_INPUT_ANCESTOR_1\=/Volumes -D SCRIPT_INPUT_ANCESTOR_2\=/Volumes/workspace -D SCRIPT_INPUT_ANCESTOR_3\=/Volumes/workspace/DerivedData -D SCRIPT_INPUT_ANCESTOR_4\=/Volumes/workspace/DerivedData/Build -D SCRIPT_INPUT_ANCESTOR_5\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex -D SCRIPT_INPUT_ANCESTOR_6\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build -D SCRIPT_INPUT_ANCESTOR_7\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos -D SCRIPT_INPUT_ANCESTOR_8\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build -D SRCROOT\=/Volumes/workspace/repository -D PROJECT_DIR\=/Volumes/workspace/repository -D OBJROOT\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex -D SYMROOT\=/Volumes/workspace/DerivedData/Build/Products -D DSTROOT\=/tmp/{hogehoge}.dst -D SHARED_PRECOMPS_DIR\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/PrecompiledHeaders -D CACHE_ROOT\=/var/folders/ny/j3hng33n10l70j416fhpdm3h0000gn/C/com.apple.DeveloperTools/15.3-15E204a/Xcode -D CONFIGURATION_BUILD_DIR\=/Volumes/workspace/DerivedData/Build/Products/Debug-iphoneos -D CONFIGURATION_TEMP_DIR\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos -D LOCROOT\=/Volumes/workspace/repository -D LOCSYMROOT\=/Volumes/workspace/repository -D INDEX_PRECOMPS_DIR\=/Volumes/workspace/DerivedData/Index.noindex/PrecompiledHeaders -D INDEX_DATA_STORE_DIR\=/Volumes/workspace/DerivedData/Index.noindex/DataStore -D TEMP_DIR\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build -D TARGET_TEMP_DIR\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build -D DERIVED_FILE_DIR\=/Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build/DerivedSources -f /Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build/d166db88713498a5149eb12fb327e521.sb /bin/sh -c /Volumes/workspace/DerivedData/Build/Intermediates.noindex/{hogehoge}.build/Debug-iphoneos/{hogehoge}.build/Script-4FAB6FF02BE1914D00B2F33A.sh
2024-05-04T19:00:58.183476957Z	Linting Swift files in current working directory
2024-05-04T19:00:58.183489414Z	Error: No lintable files found at paths: ''
2024-05-04T19:00:58.183496736Z	Command PhaseScriptExecution failed with a nonzero exit code
```

> **Error: No lintable files found at paths: ''**
