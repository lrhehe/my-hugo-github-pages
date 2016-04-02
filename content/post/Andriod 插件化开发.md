---
categories:
- 技术文章

date: "2016-04-02T12:24:04+08:00"

description:

images:

tags: ["Android开发","插件化开发"]

title: "Android 插件化开发"

toc: true
---

## 一、背景

在[猿辅导](www.yudanfudao.com) android app 的实际开发维护中遇到两个问题：

1. 代码维护问题：代码主要有两个部分，直播相关和直播无关。开发人员已经分为客户端和直播两个小组，但是两边的代码还在一个仓库中，不方便维护。
2. 上线问题：每次上线，都需要两个小组的人员一起守候到很晚（现在采用了灰度发布，好很多了）。

而理想情况是：

1. 代码分离，各自维护一个仓库，客户端依赖直播库

   为了代码分离，我们将所有直播相关内容拆分出来，得到直播库。
2. 独立实时上线，直播库可随时上线，而不需要客户端发布新版

   为了独立实时上线，需要将直播部分做成猿辅导 app 的插件。


## 二、现存方案比较


下图是来自 [Small](https://github.com/wequick/Small) 框架作者给出的[各个现有框架在 9 个方面的支持情况](https://github.com/wequick/Small/tree/master/Android)：

|                | DyLA | DiLA | ACDD  | DyAPK | DPG  | APF   | Small |
| -------------- | ---- | ---- | ----- | ----- | ---- | ----- | ----- |
| 加载非独立插件[1]     | ×    | x    | √     | √     | ×    | √     | √     |
| 加载.so插件        | ×    | ×    | ! [2] | ×     | ×    | ×     | √     |
| Activity生命周期   | √    | √    | √     | √     | ×    | √     | √     |
| Service动态注册    | ×    | ×    | √     | ×     | ×    | √     | x [3] |
| 资源分包共享[4]      | ×    | ×    | ! [5] | ![5]  | ×    | ! [6] | √     |
| 公共插件打包共享[7]    | ×    | ×    | ×     | ×     | ×    | ×     | √     |
| 支持AppCompat[8] | ×    | ×    | ×     | ×     | ×    | ×     | √     |
| 支持本地网页组件       | ×    | ×    | ×     | ×     | ×    | ×     | √     |
| 支持联调插件[9]      | ×    | x    | ×     | ×     | ×    | ×     | √     |

> [1] 独立插件：一个完整的apk包，可以独立运行。比如从你的程序跑起淘宝、QQ，但这加载起来是要闹哪样？非独立插件：依赖于宿主，宿主是个壳，插件可使用其资源代码并分离之以最小化，这才是业务需要嘛。
>
> -- _“所有不能加载非独立插件的插件化框架都是耍流氓”_。


> [2] ACDD加载.so用了Native方法(libdexopt.so)，不是Java层，源码似乎未共享。
>
> [3] Service更新频度低，可预先注册在宿主的manifest中，如果没有很好的理由说服我，现不支持。
>
> [4] 要实现宿主、各个插件资源可互相访问，需要对他们的资源进行分段处理以避免冲突。
>
> [5] 这些框架修改aapt源码、重编、覆盖SDK Manager下载的aapt，我只想说_“杀(wan)鸡(de)焉(kai)用(xin)牛(jiu)刀(hao)”_。 Small使用gradle-small-plugin，在后期修改二进制文件，实现了_**PP**_段分区。


> [6] 使用public-padding对资源id的_**TT**_段进行分区，分开了宿主和插件。但是插件之间无法分段。
>
> [7] 除了宿主提供一些公共资源与代码外，我们仍需封装一些业务层面的公共库，这些库被其他插件所依赖。公共插件打包的目的就是可以单独更新公共库插件，并且相关插件不需要动到。


> [8] AppCompat: Android Studio默认添加的主题包，Google主推的Metrial Design包也依赖于此。大势所趋。
>
> [9] 联调插件：使用Android Studio调试宿主时，可直接在插件代码中添加断点调试。


我们需要：

1. 支持 native 库 (猿辅导直播用到 native 库)
2. 支持非独立插件并支持动态更新插件 (直播库是非独立插件，需要独立实时上线)
3. 迁移简单方便 (避免引入过多的坑)
4. 活跃度高 (方便学习，交流，和解决问题)

最后，选定 Small 框架进行进一步了解。



## 三、插件化原理

### Android 相关基础

要想去弄明白 Android 插件化的原理，需要先了解一些 Android 基础知识。一个是 Android 打包 apk 的流程，另外一个是 Android 加载 apk 的流程。

#### 打包流程

可以参见：

[Build System Overview](http://developer.android.com/sdk/installing/studio-build.html)

[android Apk打包过程概述_android是如何打包apk的](http://blog.csdn.net/jason0539/article/details/44917745)

简单来说，一个apk主要是两部分，代码和资源。使用 aapt 命令可以方便查看其中内容：

```
> aapt l sample.apk
AndroidManifest.xml
assets/bundle.json
res/anim/abc_fade_in.xml
...
resources.arsc
classes.dex
META-INF/MANIFEST.MF
META-INF/CERT.SF
META-INF/CERT.RSA
```

class.dex 文件是代码部分。

而资源复杂一点，可以参考老罗的一篇博客：[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)

简单来说就是资源分为 res 和 assets。

assets 和 res 里面的 raw 类型，还有二进制图片文件会保持不变，打包到 apk 中。

res 中的所有资源都会被分配一个 id，layout 中使用 @+id 的也会生成一个 id，保存为 resources.arsc 打包进 apk。之后，所有 res 中的 xml 文件中的对应的地方会被对应的 id 替换，并重新编译成二进制文件。

#### 加载过程

主要为代码加载和资源加载

代码加载可以参考：[安卓高手之路之ClassLoader(二)](http://daojin.iteye.com/blog/1847093)

也可以继续参考老罗的博客：[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)

简单来说就是：

代码加载可以通过 DexClassLoader 加载 class.dex  (系统启动应用程序略有不同，可细看上面的文章)

资源加载通过 AssetManager 加载资源文件

### 要解决的问题

插件化简单说就是从一个主应用 (宿主) 中去启动其他未在系统安装的 apk 插件或者库文件。所以会有下面一些要解决的问题：

1. 动态加载插件中的 class
2. 动态加载插件中的资源，这里要保证能够和宿主还有其他已加载的插件的兼容
3. 启动未注册的 activity
4. 生命周期管理

### 常见解决方案

1. 动态加载插件中的 class

   使用 DexClassLoader 加载对应插件的代码

   可以参考 Small 项目：[Dynamic load classes](https://github.com/wequick/Small/wiki/Android-dynamic-load-classes)

2. 动态加载插件中的资源

   方案1： 使用 AssetManager 加载对应的插件的资源文件，不过为了保证宿主还有各插件之间的资源文件 id 不冲突，在编译阶段，修改了输出文件中的资源文件 id。可以参考 Small 项目：[Gradle-Small-Plugin](https://github.com/wequick/Small/tree/master/Android/DevSample/buildSrc)

   可以参考 Small 项目：[Dynamic load resources](https://github.com/wequick/Small/wiki/Android-dynamic-load-resources)

   方案2： 新建一套 AssetManager 和  Resources，在插件 Activity 打开的时候，将

3. 启动未注册的 activity 和生命周期管理

   方案1：使用代理 Activity，可以参考 [Dynamic-load-Apk](https://github.com/singwhatiwanna/dynamic-load-apk) 项目

   方案2：瞒天过海，简单讲就是预先注册一些 Activity (比如A, A1, ...) 作为壳，然后在准备启动插件 Actvity B 的时候，将其替换成 A 传给系统，但是在系统去生成 A 的时候，生成一个 B 出来，传给系统。这样就实现了未注册 Activity 的启动，也保证了其生命周期。不过在启动一些特殊 flag 的 Activity 时，比如 singleInstance，singleTask 和 singleTop 的 activity，存在限制。

   可以参考 Small 项目：[Dynamic register activities](https://github.com/wequick/Small/wiki/Android-dynamic-register-activities)





##四、针对模块拆分和动态更新的最简方案

根据代码分离，实时上线的需求，设计了下面的方案：

### 针对场景

1. 模块拆分：将工程中模块（application 或者 library）打包成独立插件 （plugin），宿主（Host）启动后进行加载
2. 动态更新：可下载插件，对原插件进行动态更新（重启应用可生效）

### 原理

1. 使用 DexClassLoader 加载插件代码
2. 给每个插件建立一套 AssetManager 和  Resources，借鉴 [Dynamic-load-Apk](https://github.com/singwhatiwanna/dynamic-load-apk) 项目
3. 代理 Instrumentation（借鉴 [Small](https://github.com/wequick/Small) 项目），在启动插件 Activity 的时候设置对应的资源

### 优势

采用最简单的方案，引入最少的坑

1. 不用对现有代码做修改
2. 不用对资源做修改
3. 原生的 Activity 启动方式（存在局限，见局限1）

### 局限

1. 插件的 Activity 和 Service 要在宿主中进行注册
2. 由于每个插件一套独立资源，宿主，插件之间不能够互相访问资源



具体参见：[AndroidHostPlugin](https://github.com/lrhehe/AndroidHostPlugin)