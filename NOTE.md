# Android 热加载笔记

## 0x00

所有代码加载的核心在于一句 `loaer.loadClass()`，以下内假定于读者理解这一点

## 0x01

自定义一个 ClassLoader，可以完全自定义，可以直接用 DexClassLoader，etc.

使这个 ClassLoader 自动用于 Application 的 class 加载，可用以下若干种方法达成：

* 替换掉 `currentActivityThread.mPackages[0].mClassLoader` 为自己的 ClassLoader（需要反射）

* 利用 ClassLoader 的双亲委托机制替换掉 `Application.getClassLoader()` 的 parent（需要反射）(instant run 原理)

* `DexFile.loadDex()` 未测试，API 26 已废弃

## 0x02

资源加载部分

layout 的 xml 可以用 [anko](https://github.com/Kotlin/anko) 绕过

多语言:

* 重写 `Resources`，反射调用 `addAssetPath`

* 外部 JSON 追加到 dex 尾部，释放 dex 的同时释放掉 json（图种原理）
