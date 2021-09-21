---
title: Unity中的特殊文件夹
date: '2021/9/19 22:43:44'
updated: '2021/9/19 22:59:56'
tags: []
category:
  - Unity
  - Base
mathjax: true
toc: false
abbrlink: 74e7a2cc
---
# Editor
Editor文件夹可以在根目录下，也可以在子目录里，只要叫Editor就可以。Editor下面放的所有资源文件或者脚本文件都不会被打进发布包中，并且脚本也只能在编辑时使用。一般会把一些工具类的脚本放在这里，或者是一些编辑时用的DLL。 比如我们现在要做类似技能编辑器，那么编辑器的代码放在这里是再好不过了，因为实际运行时我们只需要编辑器生成的文件，而不需要编辑器的核心代码。
<!--more-->

# Editor Default Resources
注意中间是有空格的，它必须放在Project视图的根目录下，如果你想放在/xxx/xxx/Editor Default Resources 这样是不行的。你可以把编辑器用到的一些资源放在这里，比如图片、文本文件、等等。它和Editor文件夹一样都不会被打到最终发布包里，仅仅用于开发时使用。你可以直接通过EditorGUIUtility.Load去读取该文件夹下的资源。

# Plugins
如果做手机游戏开发一般 andoird 或者 ios 要接一些sdk 可以把sdk依赖的库文件放在这里，比如 .so .jar 文件。这样打完包以后就会自动把这些文件打在你的包中。

# Resources
可以在根目录下，也可以在子目录里，只要名子叫Resources就可以。比如目录：/xxx/xxx/Resources  和 /Resources 是一样的，无论多少个叫Resources的文件夹都可以。Resources文件夹下的资源不管你用还是不用都会被打包。

Resource.Load ：编辑时和运行时都可以通过Resource.Load来直接读取。
Resources.LoadAssetAtPath() ：它可以读取Assets目录下的任意文件夹下的资源，它可以在编辑时或者编辑器运行时用，但是它不能在真机上用，它的路径是”Assets/xx/xx.xxx” 必须是这种路径，并且要带文件的后缀名。
AssetDatabase.LoadAssetAtPath()：它可以读取Assets目录下的任意文件夹下的资源，它只能在编辑时用。它的路径是”Assets/xx/xx.xxx” 必须是这种路径，并且要带文件的后缀名。

在电脑上开发的时候尽量来用Resource.Load() 或者 Resources.LoadAssetAtPath() ，假如手机上选择一部分资源要打assetbundle，一部分资源Resource.Load()。在做.apk或者.ipa的时候，现在都是用脚本来自动化打包，在打包之前可以用AssetDatabase.MoveAsset()把已经打包成assetbundle的原始文件从Resources文件夹下移动出去在打包，这样打出来的运行包就不会有多余的文件了。打完包以后再把移动出去的文件夹移动回来。

# StreamingAssets
这个文件夹下的资源也会全都打包在.apk或者.ipa。它和Resources的区别是，Resources会压缩文件，但是它不会压缩。并且它是一个只读的文件夹，在程序运行时只读。它在各个平台下的路径是不同的，不过你可以用Application.streamingAssetsPath 它会根据当前的平台选择对应的路径。
有些游戏为了让所有的资源全部使用assetbundle，会把一些初始的assetbundle放在StreamingAssets目录下，运行程序的时候在把这些assetbundle拷贝在Application.persistentDataPath目录下，如果这些assetbundle有更新的话，那么下载到新的assetbundle在把Application.persistentDataPath目录下原有的覆盖掉。
因为Application.persistentDataPath目录是应用程序的沙盒目录，所以打包之前是没有这个目录的，直到应用程序在手机上安装完毕才有这个目录。
StreamingAssets目录下的资源都是不压缩的，所以它比较大会占空间，比如你的应用装在手机上会占用100M的容量，那么你又在StreamingAssets放了一个100M的assetbundle，那么此时在装在手机上就会在200M的容量。

# Gizmos
Gizmos文件夹存放用Gizmos.DrawIcon方法使用的贴图、图标资源。放在Gizmos文件夹中的贴图资源可以直接通过名称使用，可以被Editor作为gizmo画在屏幕上。

# 参考资料
[Unity中的特殊文件夹](https://blog.csdn.net/baidu_35080512/article/details/78618293)