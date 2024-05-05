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

## Failする事象

GitHub上では
``` shell: GitHub エラー文
❌ Command PhaseScriptExecution failed with a nonzero exit code
```

Xcode Cloudに詳細なログ見に行くと👇のようなログになっています。
