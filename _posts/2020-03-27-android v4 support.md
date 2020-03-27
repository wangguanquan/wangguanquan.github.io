---
layout: post
title: 解决Android Studio无法导入android.support.v4.app
categories: Blog
description: 记一下IDE升级导致项目编译错误
keywords: android, android.support.v4.app
---

## 找原因

在对老项目添加功能的时候随手把Android Studio升级到最新版`3.6.1`，结果这一手残行为导致工程编译错误，具体就是找不到`android.support.v4.app`包。

新版本的Android默认使用`androidx`包代替原有`support`包，我们打开`gradle.properties`文件可以看到如下两个配置

```
# AndroidX package structure to make it clearer which packages are bundled with the
# Android operating system, and which are packaged with your app's APK
# https://developer.android.com/topic/libraries/support-library/androidx-rn
android.useAndroidX=true
# Automatically convert third-party libraries to use AndroidX
android.enableJetifier=true
```

## 解决

- 如果不想动原有代码可以将以上两个配置改为`false`即可
- 长远来看我们应该将原有的`support`包改为`androidx`包

build.gradle添加如下引用以使用androidx

```
implementation 'androidx.appcompat:appcompat:1.1.0'
```