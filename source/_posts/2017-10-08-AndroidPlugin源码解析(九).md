---
title: AndroidPlugin源码解析(九)
date: 2017-10-08
tags: AndroidPlugin
---

## 前言

前面我们分了8篇讲解整个Android Apk的打包过程，可以说我们中间几篇文章都是只见树木不见森林，研究了其中的细节，但缺少对这个过程的整体把握，这篇文章就是要填充缺少的整体图。

## 综述

到目前为止，Android的打包框架从Ant&ADT渐渐转移到了Gradle :

1. Build Tools 19.0.0 之前，Ant&ADT作为最主要的框架
2. Build Tools 19.0.0以后，开始使用Gradle作为打包过程的框架

Android编译源码的方案从最初的Java Tool Chain新加了Jack & Jill Tool Chain，现在又准备废弃Jack & Jill Tool Chain了。

1. Build Tools version 21.1.1之后，加入了Jack & Jill Tool Chain
2. 已经准备废弃Jack & Jill Tool Chain，未来通过javac和dx工具的修改提供Java8的支持。

### 经典打包过程

图:

![android_build.png](/img/androidBuild/android_build.png)

步骤如下:

1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
2. 处理.aidl文件，生成对应的Java接口文件
3. 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
5. 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk
6. 通过JarSigner工具，对上面的apk进行debug或release签名
7. 通过zipalign工具，将签名后的apk进行对齐处理。

### 使用Gradle后

**这就是我们前面重点讲的内容，先看一下图**

打包过程图: 

![build_process_gradle.png](/img/androidBuild/build_process_gradle.png)

使用Gradle后最详细的打包图:[本图来源](https://sites.google.com/a/android.com/tools/tech-docs/new-build-system/build-workflow)

![Android_Build_Process.svg](/img/androidBuild/Android_Build_Process.svg)

这幅图中，我们没有分析`llvm-rs-cc`，`AIDL`，`NDK`周边的文件，剩下的我们都有提到。

看完前面的过程再来看这幅图，不得不说这个图画的真的好，尤其注意对应左下角的图示说明，就可以看出Google是如何对这样一个打包过程进行分层的，把哪些工作交给哪部分去做。

这对我们整体去设计一个大型的程序也很有帮助，将某些工作交给更合适的工具去做，而不是一个工具做所有的工作。

#### Jack & Jill 

Jack & Jill Tool Chain 只在Gradle框架中存在，但是因为看到[Google打算废弃Jack消息](https://android-developers.googleblog.com/2017/03/future-of-java-8-language-feature.html)，
这就是我不在前面的文章中分析Jack & Jill的原因了，放两张图，稍微看一下就好。

![JackSitesDiagram.jpg](/img/androidBuild/JackSitesDiagram.jpg)

### Proguard

最后，两张图来了解一下proguard在这两个Tool Chain中在哪一步执行。

对javac之后执行proguard的图：

![javac_proguard_dex.png](/img/androidBuild/javac_proguard_dex.png)

Jack & Jill Proguard执行流程:

![jack_jill_proguard.png](/img/androidBuild/jack_jill_proguard.png)

看完了这个流程整体架构图，下面我们就看看gradle需要做哪些工作来把apk安装到手机上。

## Task执行依赖

之前在Gradle学习博客中，我们可以知道，对于gradle来说，整个Tasks是一个图，执行某一个Task的时候会先执行它的依赖，所以我们打包的时候只要调用`install Task`即可，

~~~ java
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
~~~

从这里粘贴的各个`dependsOn`方法可以看出，`scope.getSourceGenTask()`和`installTask`只有依赖其他Task，没有被其他Task依赖。
</br>
`installTask`就不讲了，这就是应该我们开发完成后主动触发的。
</br>
那么`scope.getSourceGenTask()`是什么时候被触发的呢？

原来，是我们在AS中点击sync的时候触发的，这个时候会执行`[:app:generateDebugSources]`，这样就触发了`scope.getSourceGenTask()`，因为它依赖`generateBuildConfigTask`，所以这个时候也会生成我们最终使用的`BuildConfig.java`文件，也会检测`Manifest`的合法性


这样，我们就可以通过调用`installTask`，让这个Task帮我们调用所有它依赖的Task，来完成整个Apk的打包签名安装工作，
不过AS的gradle并不是只调用`installTask`而已，我觉得应该在`installTask`中途做了一个hook，加入了GUI，否则正常调用`installTask`的话，不会出现那个选择安装到哪个手机的对话框的。

这样我们整个打包流程点和面都齐全了，这个系列的文章暂时就告一段落了，接下来有空会选择填前面文章中留下的坑，比如`classes如何变成dex的`，`aapt是如何处理那些资源的`等...