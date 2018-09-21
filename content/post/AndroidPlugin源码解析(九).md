---
title: AndroidPlugin源码解析-(九)
date: 2017-10-08
tags: ["AndroidPlugin"]
---

## 前言

前面我们分了 8 篇讲解整个 Android Apk 的打包过程，可以说我们中间几篇文章都是只见树木不见森林，研究了其中的细节，但缺少对这个过程的整体把握，这篇文章就是要填充缺少的整体图。

## 综述

到目前为止，Android 的打包框架从 Ant&ADT 渐渐转移到了 Gradle :

1. Build Tools 19.0.0 之前，Ant&ADT 作为最主要的框架
2. Build Tools 19.0.0 以后，开始使用 Gradle 作为打包过程的框架

Android 编译源码的方案从最初的 Java Tool Chain 新加了 Jack & Jill Tool Chain，现在又准备废弃 Jack & Jill Tool Chain 了。

1. Build Tools version 21.1.1 之后，加入了 Jack & Jill Tool Chain
2. 已经准备废弃 Jack & Jill Tool Chain，未来通过 javac 和 dx 工具的修改提供 Java8 的支持。

### 经典打包过程

图:

![android_build.png](/img/androidBuild/android_build.png)

步骤如下:

1. 通过 aapt 打包 res 资源文件，生成 R.java、resources.arsc 和 res 文件（二进制 & 非二进制如 res/raw 和 pic 保持原样）
2. 处理.aidl 文件，生成对应的 Java 接口文件
3. 通过 Java Compiler 编译 R.java、Java 接口文件、Java 源文件，生成.class 文件
4. 通过 dex 命令，将.class 文件和第三方库中的.class 文件处理生成 classes.dex
5. 通过 apkbuilder 工具，将 aapt 生成的 resources.arsc 和 res 文件、assets 文件和 classes.dex 一起打包生成 apk
6. 通过 JarSigner 工具，对上面的 apk 进行 debug 或 release 签名
7. 通过 zipalign 工具，将签名后的 apk 进行对齐处理。

### 使用 Gradle 后

**这就是我们前面重点讲的内容，先看一下图**

打包过程图:

![build_process_gradle.png](/img/androidBuild/build_process_gradle.png)

使用 Gradle 后最详细的打包图:[本图来源](https://sites.google.com/a/android.com/tools/tech-docs/new-build-system/build-workflow)

![Android_Build_Process.svg](/img/androidBuild/Android_Build_Process.svg)

这幅图中，我们没有分析`llvm-rs-cc`，`AIDL`，`NDK`周边的文件，剩下的我们都有提到。

看完前面的过程再来看这幅图，不得不说这个图画的真的好，尤其注意对应左下角的图示说明，就可以看出 Google 是如何对这样一个打包过程进行分层的，把哪些工作交给哪部分去做。

这对我们整体去设计一个大型的程序也很有帮助，将某些工作交给更合适的工具去做，而不是一个工具做所有的工作。

#### Jack & Jill

Jack & Jill Tool Chain 只在 Gradle 框架中存在，但是因为看到[Google 打算废弃 Jack 消息](https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html)，
这就是我不在前面的文章中分析 Jack & Jill 的原因了，放两张图，稍微看一下就好。

![JackSitesDiagram.jpg](/img/androidBuild/JackSitesDiagram.jpg)

### Proguard

最后，两张图来了解一下 proguard 在这两个 Tool Chain 中在哪一步执行。

对 javac 之后执行 proguard 的图：

![javac_proguard_dex.png](/img/androidBuild/javac_proguard_dex.png)

Jack & Jill Proguard 执行流程:

![jack_jill_proguard.png](/img/androidBuild/jack_jill_proguard.png)

看完了这个流程整体架构图，下面我们就看看 gradle 需要做哪些工作来把 apk 安装到手机上。

## Task 执行依赖

之前在 Gradle 学习博客中，我们可以知道，对于 gradle 来说，整个 Tasks 是一个图，执行某一个 Task 的时候会先执行它的依赖，所以我们打包的时候只要调用`install Task`即可，

```java
installTask.dependsOn(tasks, variantScope.getAssembleTask());
variantOutputScope.getAssembleTask().dependsOn(tasks, zipAlignTask);
zipAlignTask.dependsOn(tasks, packageApp);
packageApp.dependsOn(tasks, stream.getDependencies());
packageApp.dependsOn(tasks, validateSigningTask);
packageApp.dependsOn(tasks, variantScope.getMergeAssetsTask());
packageApp.dependsOn(tasks, variantOutputScope.getProcessResourcesTask());
packageApp.optionalDependsOn(
      tasks,
      variantOutputScope.getShrinkResourcesTask(),
      variantOutputScope.getVariantScope().getJavaCompilerTask(),
      javacTask,
      variantOutputData.packageSplitResourcesTask,
      variantOutputData.packageSplitAbiTask);
javacTask.dependsOn(tasks, javacIncrementalSafeguard);
variantOutputScope.getProcessResourcesTask().dependsOn(tasks,
                    scope.getMergeResourcesTask());
generateBuildConfigTask.dependsOn(tasks, scope.getCheckManifestTask());
scope.getSourceGenTask().dependsOn(tasks, generateBuildConfigTask.getName());
mergeResourcesTask.dependsOn(tasks, scope.getPrepareDependenciesTask(), scope.getResourceGenTask());
scope.getResourceGenTask().dependsOn(tasks, generateResValuesTask);
```

从这里粘贴的各个`dependsOn`方法可以看出，`scope.getSourceGenTask()`和`installTask`只有依赖其他 Task，没有被其他 Task 依赖。
</br>
`installTask`就不讲了，这就是应该我们开发完成后主动触发的。
</br>
那么`scope.getSourceGenTask()`是什么时候被触发的呢？

原来，是我们在 AS 中点击 sync 的时候触发的，这个时候会执行`[:app:generateDebugSources]`，这样就触发了`scope.getSourceGenTask()`，因为它依赖`generateBuildConfigTask`，所以这个时候也会生成我们最终使用的`BuildConfig.java`文件，也会检测`Manifest`的合法性

所以，我们可以通过调用`installTask`，让这个 Task 帮我们调用所有它依赖的 Task，来完成整个 Apk 的打包签名安装工作，
不过 AS 的 gradle 并不是只调用`installTask`而已，我觉得应该在`installTask`中途做了一个 hook，加入了 GUI，否则正常调用`installTask`的话，不会出现那个选择安装到哪个手机的对话框的。

这样我们整个打包流程点和面都齐全了，这个系列的文章暂时就告一段落了，接下来有空会选择填前面文章中留下的坑，比如`classes如何变成dex的`，`aapt是如何处理那些资源的`等...
