---
title: AndroidPlugin源码解析-(七)
date: 2017-09-10
tags: ["AndroidPlugin"]
---

忽视了 Aidl, Shader, Ndk, Jni, Jack, DataBinding, StripNativeLibrary, Split, InstantRun, Lint 这些 Task 之后，我们就只剩下面三个重要的 Task 了：

- createJavacTask(tasks, variantScope);
- createPostCompilationTasks(tasks, variantScope);
- createPackagingTask(tasks, variantScope, true /_publishApk_/, fullBuildInfoGeneratorTask);

这篇文章我们来分析前面两个 Task

### createJavacTask

这个 Task 的方法为:

```java
public AndroidTask<? extends JavaCompile> createJavacTask(
        @NonNull final TaskFactory tasks,
        @NonNull final VariantScope scope) {
    final BaseVariantData<? extends BaseVariantOutputData> variantData = scope.getVariantData();
    // 这是AndroidJavaCompileTask的前置Task，如果生成的源码有变动，则清空AndroidJavaCompile的输出目录，强制重新编译
    // 这个文件为  build/intermediates/incremental-safeguard/tag.txt
    AndroidTask<IncrementalSafeguard> javacIncrementalSafeguard = androidTasks.create(tasks,
            new IncrementalSafeguard.ConfigAction(scope));

    final AndroidTask<? extends JavaCompile> javacTask = androidTasks.create(tasks,
            new JavaCompileConfigAction(scope));
    scope.setJavacTask(javacTask);
    javacTask.dependsOn(tasks, javacIncrementalSafeguard);

    setupCompileTaskDependencies(tasks, scope, javacTask);

    // Create jar task for uses by external modules.
    // 还没遇见过，暂时忽视
    // 生成的文件路径在 build/intermediates/classes-jar/${buildType}/class.jar
    if (variantData.getVariantDependency().getClassesConfiguration() != null) {
        AndroidTask<Jar> packageJarArtifact =
                androidTasks.create(tasks, new PackageJarArtifactConfigAction(scope));
        packageJarArtifact.dependsOn(tasks, javacTask);
    }

    return javacTask;
}
```

看到这里相当于存在 3 个 Task，但是我们只要关心`javacTask`即可，

第一个`javacIncrementalSafeguard`,就是类似标识位的东东，

最后一个`packageJarArtifact`，因为我还没遇到使用外部 modules 的情况，所以也忽略，

大家如果在打包完成的 build 路径下看到我提到的路径，就可以在这里查看情况。

好的，那么我们来仔细看一下`javacTask`

```java
public void execute(@NonNull final AndroidJavaCompile javacTask) {
    scope.getVariantData().javacTask = javacTask;
    // compileSdkVersion >= 24，需要用JDK 1.8及以上版本
    javacTask.compileSdkVersion = scope.getGlobalScope().getExtension().getCompileSdkVersion();
    javacTask.mBuildContext = scope.getInstantRunBuildContext();

    // We can't just pass the collection directly, as the instanceof check in the incremental
    // compile doesn't work recursively currently, so every ConfigurableFileTree needs to be
    // directly in the source array.
    for (ConfigurableFileTree fileTree : scope.getVariantData().getJavaSources()) {
        javacTask.source(fileTree);
    }

    // Set boot classpath if we don't need to keep the default.  Otherwise, this is added as
    // normal classpath.
    javacTask.getOptions().setBootClasspath(
            Joiner.on(File.pathSeparator).join(
                    scope.getGlobalScope().getAndroidBuilder()
                            .getBootClasspathAsStrings(false)));


    javacTask.setDestinationDir(scope.getJavaOutputDir()); // build/intermediates/classes/

    javacTask.setDependencyCacheDir(scope.getJavaDependencyCache()); // build/intermediates/dependency-cache/

    CompileOptions compileOptions = scope.getGlobalScope().getExtension().getCompileOptions();

    AbstractCompilesUtil.configureLanguageLevel(
            javacTask,
            compileOptions,
            scope.getGlobalScope().getExtension().getCompileSdkVersion(),
            scope.getVariantConfiguration().getJackOptions().isEnabled());

    javacTask.getOptions().setEncoding(compileOptions.getEncoding());

    GlobalScope globalScope = scope.getGlobalScope();
    Project project = scope.getGlobalScope().getProject();

    CoreAnnotationProcessorOptions annotationProcessorOptions =
            scope.getVariantConfiguration().getJavaCompileOptions()
                    .getAnnotationProcessorOptions();


    checkNotNull(annotationProcessorOptions.getIncludeCompileClasspath());
    Collection<File> processorPath =
            Lists.newArrayList(
                    scope.getVariantData().getVariantDependency()
                            .resolveAndGetAnnotationProcessorClassPath(
                                    annotationProcessorOptions.getIncludeCompileClasspath(),
                                    scope.getGlobalScope().getAndroidBuilder().getErrorReporter()));


    if (!processorPath.isEmpty()) {
        if (Boolean.TRUE.equals(annotationProcessorOptions.getIncludeCompileClasspath())) {
            processorPath.addAll(javacTask.getClasspath().getFiles());
        }
        javacTask.getOptions().getCompilerArgs().add("-processorpath");
        javacTask.getOptions().getCompilerArgs().add(FileUtils.joinFilePaths(processorPath));
    }
    if (!annotationProcessorOptions.getClassNames().isEmpty()) {
        javacTask.getOptions().getCompilerArgs().add("-processor");
        javacTask.getOptions().getCompilerArgs().add(
                Joiner.on(',').join(annotationProcessorOptions.getClassNames()));
    }
    if (!annotationProcessorOptions.getArguments().isEmpty()) {
        for (Map.Entry<String, String> arg :
                annotationProcessorOptions.getArguments().entrySet()) {
            javacTask.getOptions().getCompilerArgs().add(
                    "-A" + arg.getKey() + "=" + arg.getValue());
        }
    }

    // Create directory for output of annotation processor.
    javacTask.doFirst(task -> {
        FileUtils.mkdirs(scope.getAnnotationProcessorOutputDir()); // build/generated/source/apt
    });
    javacTask.getOptions().getCompilerArgs().add("-s");
    javacTask.getOptions().getCompilerArgs().add(
            scope.getAnnotationProcessorOutputDir().getAbsolutePath());
}
```

看下来这个代码，这里就是在给 javac 命令添加参数，看到`setDestinationDir()`方法，说明 javac 之后生成的代码路径为`build/intermediates/classes/`。
先往回看代码:

```java
javacTask.getOptions().setBootClasspath(
        Joiner.on(File.pathSeparator).join(
            scope.getGlobalScope().getAndroidBuilder()
                .getBootClasspathAsStrings(false)));
```

这个方法就是设置一些很基础的 jar 包的 classpath，比如`android.jar`。

这个代码通过查找 Sdk location 来获取这些 jar 包的位置，至于使用的是哪个版本，则由`compileSdkVersion`决定，

所以，升级`compileSdkVersion`，就能使用最新的`android.jar`的 sdk，

还有，如果你想使用暴露隐藏方法的`android.jar`，替换对应`compileSdkVersion`的`android.jar`即可。具体路径在`${sdkVersion}/platforms/android-${compileSdkVersion}`。
</br>

往下继续看代码，可以看到一个叫做`processorPath`的列表，`processorPath`这个列表是怎么来的呢？就是我们在`build.gradle`里面`dependencies`里面声明的`annotationProcessor`。

假设你的项目里有`butterknife`之类的东东，所以这个列表不会为空，所以会为 javac 添加如下参数`-processorpath ${processorPath}`,

继续看，发现又添加了具体的 apt 处理类的参数:
`-processor ${processorClassName1,processorClassName2...}`，

最后，如果你为某个`annotationProcessor`设置了参数，就会为 javac 再加入新的参数`-A ${key}=${value}`,

为某个`annotationProcessor`设置了参数的方式为(以`butterknife`为例)

`javaCompileOptions.annotationProcessorOptions.arguments['butterknife.debuggable'] = 'false'`

所以，这个地方给 javac 添加的参数类似`-A butterknife.debuggable=false`。
</br>

最后，要在执行 javac 之前做一些处理，我们可以看到，就是创建`build/generated/source/apt`这个文件夹，如果存在，删除之后再重新创建。创建好此文件夹后，就可以加入`-s ${annotationProcessorOutputDir}`,

这样，无论是需要 apt 生成的代码，还是最终需要 javac 生成的代码都配置好了，如果想知道每个参数的具体含义，请参考`javac -help`
</br>

剩下的就是调用 javac 命令来执行了，因为这个属于`JavaCompile` task，这个 Task 是 gradle 自带的，所以到这里`createJavacTask`方法就结束了。
</br>

接下来这个 Task 并不是打包过程的一步，放在这里是因为如果你需要打包一个 Jar 包给别人，那么这里是最合适执行此 Task 的地方，因为这个 Task 要依赖`javacTask`。

这个 Task 的任务就是把 Javac 生成的那些文件，R 文件，BuildConfig 文件打包，所以这里也没有写出这个 Task 的名字，就直接创建了，

创建代码为:
`getAndroidTasks().create(tasks, new AndroidJarTask.JarClassesConfigAction(variantScope));`，

这个 Task 执行结果放在`build/intermediates/packaged/${buildType}/classes.jar`。

不过，这里只是包含当前 lib 生成的 class 文件，不包含依赖的 class 文件。
</br>

了解到这里，我们就可以继续往下看了。

### createPostCompilationTasks

```java
/**
 * Creates the post-compilation tasks for the given Variant.
 *
 * These tasks create the dex file from the .class files, plus optional intermediary steps like
 * proguard and jacoco
 *
 */
public void createPostCompilationTasks(
        @NonNull TaskFactory tasks,
        @NonNull final VariantScope variantScope) {

    checkNotNull(variantScope.getJavacTask());

    final BaseVariantData<? extends BaseVariantOutputData> variantData = variantScope.getVariantData();
    final GradleVariantConfiguration config = variantData.getVariantConfiguration();

    TransformManager transformManager = variantScope.getTransformManager();

    // ---- Code Coverage first -----
    // 如果有单元测试代码，且testCoverageEnabled，则执行JacocoTansform
    boolean isTestCoverageEnabled = config.getBuildType().isTestCoverageEnabled() &&
            !config.getType().isForTesting() &&
            getIncrementalMode(variantScope.getVariantConfiguration()) == IncrementalMode.NONE;
    if (isTestCoverageEnabled) {
        createJacocoTransform(tasks, variantScope);
    }

    boolean isMinifyEnabled = isMinifyEnabled(variantScope);
    boolean isMultiDexEnabled = config.isMultiDexEnabled();
    // Switch to native multidex if possible when using instant run.
    boolean isLegacyMultiDexMode = isLegacyMultidexMode(variantScope);

    AndroidConfig extension = variantScope.getGlobalScope().getExtension();

    // ----- External Transforms -----
    // apply all the external transforms.
    List<Transform> customTransforms = extension.getTransforms();
    List<List<Object>> customTransformsDependencies = extension.getTransformsDependencies();

    for (int i = 0, count = customTransforms.size() ; i < count ; i++) {
        Transform transform = customTransforms.get(i);
        AndroidTask<TransformTask> task = transformManager
                .addTransform(tasks, variantScope, transform);
        // task could be null if the transform is invalid.
        if (task != null) {
            List<Object> deps = customTransformsDependencies.get(i);
            if (!deps.isEmpty()) {
                task.dependsOn(tasks, deps);
            }

            // if the task is a no-op then we make assemble task depend on it.
            if (transform.getScopes().isEmpty()) {
                variantScope.getAssembleTask().dependsOn(tasks, task);
            }

        }
    }

    // ----- Minify next -----

    if (isMinifyEnabled) {
        boolean outputToJarFile = isMultiDexEnabled && isLegacyMultiDexMode;
        createMinifyTransform(tasks, variantScope, outputToJarFile);
    }


    // ----- Multi-Dex support

    AndroidTask<TransformTask> multiDexClassListTask = null;
    // non Library test are running as native multi-dex
    if (isMultiDexEnabled && isLegacyMultiDexMode) {
        if (!variantData.getVariantConfiguration().getBuildType().isUseProguard()) {
            throw new IllegalStateException(
                    "Build-in class shrinker and multidex are not supported yet.");
        }

        // ----------
        // create a transform to jar the inputs into a single jar.
        if (!isMinifyEnabled) {
            // merge the classes only, no need to package the resources since they are
            // not used during the computation.
            JarMergingTransform jarMergingTransform = new JarMergingTransform(
                    TransformManager.SCOPE_FULL_PROJECT);
            variantScope.addColdSwapBuildTask(
                    transformManager.addTransform(tasks, variantScope, jarMergingTransform));
        }

        // ---------
        // create the transform that's going to take the code and the proguard keep list
        // from above and compute the main class list.
        MultiDexTransform multiDexTransform = new MultiDexTransform(
                variantScope,
                extension.getDexOptions(),
                null);
        multiDexClassListTask = transformManager.addTransform(
                tasks, variantScope, multiDexTransform);
        multiDexClassListTask.optionalDependsOn(tasks, manifestKeepListTask);
        variantScope.addColdSwapBuildTask(multiDexClassListTask);
    }
    // create dex transform
    DefaultDexOptions dexOptions = DefaultDexOptions.copyOf(extension.getDexOptions());

    DexTransform dexTransform = new DexTransform(
            dexOptions,
            config.getBuildType().isDebuggable(),
            isMultiDexEnabled,
            isMultiDexEnabled && isLegacyMultiDexMode ? variantScope.getMainDexListFile() : null,
            variantScope.getPreDexOutputDir(),
            variantScope.getGlobalScope().getAndroidBuilder(),
            getLogger(),
            variantScope.getInstantRunBuildContext(),
            AndroidGradleOptions.getBuildCache(variantScope.getGlobalScope().getProject()));
    AndroidTask<TransformTask> dexTask = transformManager.addTransform(
            tasks, variantScope, dexTransform);
    // need to manually make dex task depend on MultiDexTransform since there's no stream
    // consumption making this automatic
    dexTask.optionalDependsOn(tasks, multiDexClassListTask);
}
```

看到最外层的注释，这里就是把 class 文件转化为 dex 文件的地方。那我们来详细跟踪一下代码。

这里应该有很多熟悉的参数名称，比如`minifyEnabled`,`multiDexEnabled`等，这些参数之后接下来就获取`customTransforms`这样一个`Transform`对象列表，这样一个列表是怎么来的呢？

就是通过`Android Plugin`的`extension`的`registerTransform`注册的 transfrom。

这个 Transform 是个什么东东呢？如果你写过 Gradle 插件，对这个应该不陌生，

这个就是`Android Plugin`暴露的 Hook Api，让你可以自定义操作参与打包过程之中。

从这里也可以看出，gradle plugin 引入的顺序，影响 plugin 的`Transform`被调用的顺序。

这些`Transform`就会被加入到`TransformManager`中，按顺序执行。

跳过这个将`Transfrom`加入`TransformManager`的过程，下面就轮到了`// ----- Minify next -----`，

看这个方法名`createMinifyTransform`，应该是创建`Minify`这个`Transform`，并加入`TransformManager`中。接下来我们跟进去看看细节。

简单几层跟进之后，如果`useProguard`属性设置为 true,则执行如下两个方法:

```java
// createJarFile是bool值，Application中是true，Library中是false
createProguardTransform(taskFactory, variantScope, mappingConfiguration, createJarFile);
createShrinkResourcesTransform(taskFactory, variantScope);
```

我们来一一查看，先看

#### createProguardTransform

```java
private void createProguardTransform(
        @NonNull TaskFactory taskFactory,
        @NonNull VariantScope variantScope,
        @Nullable Configuration mappingConfiguration,
        boolean createJarFile) {

    final BaseVariantData<? extends BaseVariantOutputData> variantData = variantScope
            .getVariantData();
    final GradleVariantConfiguration variantConfig = variantData.getVariantConfiguration();
    final BaseVariantData testedVariantData = variantScope.getTestedVariantData();

    ProGuardTransform transform = new ProGuardTransform(variantScope, createJarFile);

    applyProguardConfig(transform, variantData);

    AndroidTask<?> task = variantScope.getTransformManager().addTransform(taskFactory,
            variantScope, transform,
            (proGuardTransform, proGuardTask) ->
                    variantData.mappingFileProviderTask = new FileSupplier() {
                        @NonNull
                        @Override
                        public Task getTask() {
                    return proGuardTask;
                }

                        @Override
                        public File get() {
                    return proGuardTransform.getMappingFile();
                }
                    });

}
```

这里最重要的就是两个语句，

一个是初始化`ProGuardTransform`并将其加入`TransformManager`中，

第二个是给`ProGuardTransform`设置配置，

在这里我们关注的是流程，所以`ProGuardTransform`是如何执行的就不在这里细讲了，

如果你熟悉`Transform` Api，那么就会知道这里执行的是`ProGuardTransform`里的`transform`方法，如果你有兴趣，看这个方法了解 ProGuard 执行的细节。

好了，那我们来看一看`applyProguardConfig`方法

##### applyProguardConfig

```java
final GradleVariantConfiguration variantConfig = variantData.getVariantConfiguration();
transform.setConfigurationFiles(() -> {
    Set<File> proguardFiles = variantConfig.getProguardFiles(
            true,
            Collections.singletonList(ProguardFiles.getDefaultProguardFile(
                    TaskManager.DEFAULT_PROGUARD_CONFIG_FILE, project)));

    // use the first output when looking for the proguard rule output of
    // the aapt task. The different outputs are not different in a way that
    // makes this rule file different per output.
    BaseVariantOutputData outputData = variantData.getOutputs().get(0);
    proguardFiles.add(outputData.processResourcesTask.getProguardOutputFile());
    return proguardFiles;
});

if (variantData.getType() == LIBRARY) {
    transform.keep("class **.R");
    transform.keep("class **.R$*");
}
```

不知道你们有没有过好奇，我们在新建项目的时候，build.gradle 文件中会有这么一行配置`proguardFiles getDefaultProguardFile('proguard-android.txt')`，这个文件在哪里呢，会不会可以改名字呢？这个的答案就在这里就可以找到。

就是下面这行代码

```java
ProguardFiles.getDefaultProguardFile(TaskManager.DEFAULT_PROGUARD_CONFIG_FILE, project))
```

首先，它会检测参数值是否为"proguard-android.txt"或"proguard-android-optimize.txt"，也就说明了，`getDefaultProguardFile`只能填这两个参数。

接下来就是返回这个文件的具体路径在根目录的 build 文件夹下，在`build/intermediates/proguard-files/`这个目录下，文件名最后为当前 gradle plugin 的版本，这两个文件最初的位置在哪里呢？原来是`${sdkPath}/tools/proguard/`这个目录下的那两个文件.

还有，是不是好奇为什么出现在 layout 和 Manifest 里的东东都不会混淆，就是因为这行代码

```java
proguardFiles.add(outputData.processResourcesTask.getProguardOutputFile());
```

这个就是给 proguardFiles 加入`build.intermediates/proguard-rules/${buildType}/aapt_rules.txt`，这个文件就是在前面的过程中生成的 proguard 的规则。

最后就是为 Library 单独配置的规则了，如果是 Library，不混淆它的 R 文件。

到这里`createProguardTransform`方法就结束了。继续看下一个方法`createShrinkResourcesTransform`

#### createShrinkResourcesTransform

```java
private void createShrinkResourcesTransform(
        @NonNull TaskFactory taskFactory,
        @NonNull VariantScope scope) {
    CoreBuildType buildType = scope.getVariantConfiguration().getBuildType();

    if (!buildType.isShrinkResources()) {
        // The user didn't enable resource shrinking, silently move on.
        return;
    }

    // if resources are shrink, insert a no-op transform per variant output
    // to transform the res package into a stripped res package
    for (final BaseVariantOutputData variantOutputData : scope.getVariantData().getOutputs()) {
        VariantOutputScope variantOutputScope = variantOutputData.getScope();

        ShrinkResourcesTransform shrinkResTransform = new ShrinkResourcesTransform(
                variantOutputData,
                variantOutputScope.getProcessResourcePackageOutputFile(),
                variantOutputScope.getShrinkedResourcesFile(),
                androidBuilder,
                logger);
        AndroidTask<TransformTask> shrinkTask = scope.getTransformManager()
                .addTransform(taskFactory, variantOutputScope, shrinkResTransform);
        // need to record this task since the package task will not depend
        // on it through the transform manager.
        variantOutputScope.setShrinkResourcesTask(shrinkTask);
    }
}
```

这个方法很简短，老规矩，不分析这个`Transform`到底是怎么做的，这里将两点:

1. shrinkResources 默认是 false，如果没有在 buildType 中主动声明为 true，则此方法会直接返回。
2. 这个`shrinkResTransform`是在`ProGuardTransform`之后加入的，也就意味着在最后处理代码的时候，先执行 ProGuard,后执行 ShrinkResources。

这个方法要讲的就这么多，我们再返回`createPostCompilationTasks`方法继续看下去。

接下里分析`// ----- Multi-Dex support`这一段代码

```java
AndroidTask<TransformTask> multiDexClassListTask = null;
// non Library test are running as native multi-dex
if (isMultiDexEnabled && isLegacyMultiDexMode) {

    // ----------
    // create a transform to jar the inputs into a single jar.
    if (!isMinifyEnabled) {
        // merge the classes only, no need to package the resources since they are
        // not used during the computation.
        // 把所有的.class打包成一个combined.jar
        JarMergingTransform jarMergingTransform = new JarMergingTransform(
                TransformManager.SCOPE_FULL_PROJECT);
        variantScope.addColdSwapBuildTask(
                transformManager.addTransform(tasks, variantScope, jarMergingTransform));
    }

    // ---------
    // create the transform that's going to take the code and the proguard keep list
    // from above and compute the main class list.
    MultiDexTransform multiDexTransform = new MultiDexTransform(
            variantScope,
            extension.getDexOptions(),
            null);
    multiDexClassListTask = transformManager.addTransform(
            tasks, variantScope, multiDexTransform);
    multiDexClassListTask.optionalDependsOn(tasks, manifestKeepListTask);
    variantScope.addColdSwapBuildTask(multiDexClassListTask);
}
```

可以看到，这里的重点就是两个`Transform`:

- JarMergingTransform
- MultiDexTransform

虽然如此，但是`JarMergingTransform`本身的任务很简单，就是将 class 文件打成 jar 包，这里就不仔细分析了，

熟悉`Transform Api`的话很容易就能看懂，
这个的最终结果是:
`build/intermediates/transform/${outputTypes}/${scope}/combined.jar`这样一个 jar 包，
对着结果来理解应该会更容易了。

好了，这样我们来仔细关注`MultiDexTransform`，我们来看看它的`transform`方法

#### MultiDexTransform transform

```java
//校验referencedInputs只能有一个，要么是jar，要么是directory
File input = verifyInputs(invocation.getReferencedInputs());
shrinkWithProguard(input);
computeList(input);
```

看起来就三个方法，第一个方法就是做一个校验；但是后面两个比较复杂，我们仔细研究一下。

##### MultiDexTransform shrinkWithProguard

```java
private void shrinkWithProguard(@NonNull File input) throws IOException, ParseException {
    dontobfuscate();  // 设置obfuscate = false
    dontoptimize(); // 设置 optimize = false
    dontpreverify(); // 设置 preverify = false
    dontwarn(); // 设置 warn = Lists.newArrayList("**")
    dontnote(); // 设置 note = Lists.newArrayList("**")
    forceprocessing(); // 设置 lastModified = Long.MAX_VALUE

    applyConfigurationFile(manifestKeepListProguardFile);

    if (userMainDexKeepProguard != null) {
        applyConfigurationFile(userMainDexKeepProguard);
    }

    // add a couple of rules that cannot be easily parsed from the manifest.
    keep("public class * extends android.app.Instrumentation { <init>(); }");
    keep("public class * extends android.app.Application { "
            + "  <init>(); "
            + "  void attachBaseContext(android.content.Context);"
            + "}");
    keep("public class * extends android.app.backup.BackupAgent { <init>(); }");
    keep("public class * extends java.lang.annotation.Annotation { *;}");
    keep("class com.android.tools.fd.** {*;}"); // Instant run.

    // handle inputs

    libraryJar(findShrinkedAndroidJar());
    inJar(input);

    // outputs.
    outJar(variantScope.getProguardComponentsJarFile());
    printconfiguration(configFileOut);

    // run proguard
    runProguard();
}
```

看到最后一个方法`runProguard()`，先猜测一下这个方法前面的过程是给`Proguard`设置参数的过程。经过分析，确实如此。

这里就点几点，`userMainDexKeepProguard != null`，

这个在什么情况下不为 null 呢？在`bulidType`中设置过这个参数`multiDexKeepProguard`的情况下不为`null`。

作为`librayJar`的参数到底是哪个 Jar 呢？
是 `build-tools/${buildToolsVersion}/lib/shrinkedAndroid.jar`这个 Jar。

`outJar`的最终位置在哪里呢？
在`build/intermediates/multi-dex/${buildType}/componentClasses.jar`，

好了，最后一个这个配置的输出是在
`build/intermediates/multi-dex/${buildType}/components.flags`，
看这个文件的内容，就是这里的配置。

配置完成了，接下来就是跑`Proguard`输出`componentClasses.jar`的过程，这个过程是`Proguard`的任务，就不在我们流程之中了。详细了解这个过程，请参考`Proguard`的源码。

然后我们就可以看`transform`方法内的第三个方法了。

##### MultiDexTransform computeList

```java
private void computeList(File _allClassesJarFile) throws ProcessException, IOException {
    // manifest components plus immediate dependencies must be in the main dex.
    Set<String> mainDexClasses = callDx(
            _allClassesJarFile,
            variantScope.getProguardComponentsJarFile());

    if (userMainDexKeepFile != null) {
        mainDexClasses = ImmutableSet.<String>builder()
                .addAll(mainDexClasses)
                .addAll(Files.readLines(userMainDexKeepFile, Charsets.UTF_8))
                .build();
    }

    String fileContent = Joiner.on(System.getProperty("line.separator")).join(mainDexClasses);

    Files.write(fileContent, mainDexListFile, Charsets.UTF_8);

}
```

这个最难的方法是`callDx`方法，大概讲一下这个方法的流程：

这个方法最终会调用`AndroidBuilder`的`createMainDexList`方法，此方法是调用`build-tools/${buildToolsVersion}/lib/dx.jar`的`com.android.multidex.ClassReferenceListBuilder`这个类的`main`方法，生成需要放入主 dex 的 class 集合；

接下来判断`userMainDexKeepFile`这个值，这个值的设置在哪里呢？是在`bulidType`中的参数`multiDexKeepFile`决定的。如果存在这样一个文件，就把这个文件里的内容也作为一个集合加入其中。

最后，将前面那个集合的内容写入`maindexlist.txt`这个文件中，这个文件位于`build/intermediates/multi-dex/${buildType}/`目录下。

到这里，`MultiDexTransform`的处理逻辑就分析完了，也就意味着，我们可以再返回`createPostCompilationTasks`方法继续看下去了。

接下来就是这个方法的最后一部分，`dex`处理部分。

```java
DefaultDexOptions dexOptions = DefaultDexOptions.copyOf(extension.getDexOptions());

DexTransform dexTransform = new DexTransform(
        dexOptions,
        config.getBuildType().isDebuggable(),
        isMultiDexEnabled,
        isMultiDexEnabled && isLegacyMultiDexMode ? variantScope.getMainDexListFile() : null,
        variantScope.getPreDexOutputDir(),
        variantScope.getGlobalScope().getAndroidBuilder(),
        getLogger(),
        variantScope.getInstantRunBuildContext(),
        AndroidGradleOptions.getBuildCache(variantScope.getGlobalScope().getProject()));
AndroidTask<TransformTask> dexTask = transformManager.addTransform(
        tasks, variantScope, dexTransform);
// need to manually make dex task depend on MultiDexTransform since there's no stream
// consumption making this automatic
dexTask.optionalDependsOn(tasks, multiDexClassListTask);
variantScope.addColdSwapBuildTask(dexTask);
```

浏览了这部分源码过后，重点也是一个`Transform`的执行过程， 那么我们就来看看这个`DexTransform`

#### DexTransform transform

```java
public void transform(@NonNull TransformInvocation transformInvocation)
        throws TransformException, IOException, InterruptedException {
    TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();

    // Gather a full list of all inputs.
    List<JarInput> jarInputs = Lists.newArrayList();
    List<DirectoryInput> directoryInputs = Lists.newArrayList();
    for (TransformInput input : transformInvocation.getInputs()) {
        jarInputs.addAll(input.getJarInputs());
        directoryInputs.addAll(input.getDirectoryInputs());
    }

    ProcessOutputHandler outputHandler = new ParsingProcessOutputHandler(
            new ToolOutputParser(new DexParser(), Message.Kind.ERROR, logger),
            new ToolOutputParser(new DexParser(), logger),
            androidBuilder.getErrorReporter());

    outputProvider.deleteAll();

    try {
        // if only one scope or no per-scope dexing, just do a single pass that
        // runs dx on everything.
        if ((jarInputs.size() + directoryInputs.size()) == 1
                || !dexOptions.getPreDexLibraries()) {
            File outputDir = outputProvider.getContentLocation("main",
                    getOutputTypes(), getScopes(),
                    Format.DIRECTORY);
            FileUtils.mkdirs(outputDir);

            // first delete the output folder where the final dex file(s) will be.
            FileUtils.cleanOutputDir(outputDir);

            // gather the inputs. This mode is always non incremental, so just
            // gather the top level folders/jars
            final List<File> inputFiles =
                    Stream.concat(
                            jarInputs.stream().map(JarInput::getFile),
                            directoryInputs.stream().map(DirectoryInput::getFile))
                    .collect(Collectors.toList());

            androidBuilder.convertByteCode(
                    inputFiles,
                    outputDir,
                    multiDex,
                    mainDexListFile,
                    dexOptions,
                    getOptimize(),
                    outputHandler);

        } else {
            // Figure out if we need to do a dx merge.
            // The ony case we don't need it is in native multi-dex mode when doing debug
            // builds. This saves build time at the expense of too many dex files which is fine.
            // FIXME dx cannot receive dex files to merge inside a folder. They have to be in a
            // jar. Need to fix in dx.
            boolean needMerge = !multiDex || mainDexListFile != null;// || !debugMode;

            // where we write the pre-dex depends on whether we do the merge after.
            // If needMerge changed from one build to another, we'll be in non incremental
            // mode, so we don't have to deal with changing folder in incremental mode.
            File perStreamDexFolder = null;
            if (needMerge) {
                perStreamDexFolder = intermediateFolder;
                FileUtils.deletePath(perStreamDexFolder);
            }

            // dex all the different streams separately, then merge later (maybe)
            // hash to detect duplicate jars (due to isse with library and tests)
            final Set<String> hashs = Sets.newHashSet();
            // input files to output file map
            final Map<File, File> inputFiles = Maps.newHashMap();
            // set of input files that are external libraries
            final Set<File> externalLibs = Sets.newHashSet();
            // stuff to delete. Might be folders.
            final List<File> deletedFiles = Lists.newArrayList();

            // first gather the different inputs to be dexed separately.
            for (DirectoryInput directoryInput : directoryInputs) {
                File rootFolder = directoryInput.getFile();
                // The incremental mode only detect file level changes.
                // It does not handle removed root folders. However the transform
                // task will add the TransformInput right after it's removed so that it
                // can be detected by the transform.
                if (!rootFolder.exists()) {
                    // if the root folder is gone we need to remove the previous
                    // output
                    File preDexedFile = getPreDexFile(
                            outputProvider, needMerge, perStreamDexFolder, directoryInput);
                    if (preDexedFile.exists()) {
                        deletedFiles.add(preDexedFile);
                    }
                } else if (!isIncremental || !directoryInput.getChangedFiles().isEmpty()) {
                    // add the folder for re-dexing only if we're not in incremental
                    // mode or if it contains changed files.
                    logger.info("Changed file for %s are %s",
                            directoryInput.getFile().getAbsolutePath(),
                            Joiner.on(",").join(directoryInput.getChangedFiles().entrySet()));
                    File preDexFile = getPreDexFile(
                            outputProvider, needMerge, perStreamDexFolder, directoryInput);
                    inputFiles.put(rootFolder, preDexFile);
                    if (isExternalLibrary(directoryInput)) {
                        externalLibs.add(rootFolder);
                    }
                }
            }

            for (JarInput jarInput : jarInputs) {
                switch (jarInput.getStatus()) {
                    case NOTCHANGED:
                        // intended fall-through
                    case CHANGED:
                    case ADDED: {
                        File preDexFile = getPreDexFile(
                                outputProvider, needMerge, perStreamDexFolder, jarInput);
                        inputFiles.put(jarInput.getFile(), preDexFile);
                        if (isExternalLibrary(jarInput)) {
                            externalLibs.add(jarInput.getFile());
                        }
                        break;
                    }
                    case REMOVED: {
                        File preDexedFile = getPreDexFile(
                                outputProvider, needMerge, perStreamDexFolder, jarInput);
                        if (preDexedFile.exists()) {
                            deletedFiles.add(preDexedFile);
                        }
                        break;
                    }
                }
            }

            WaitableExecutor<Void> executor = WaitableExecutor.useGlobalSharedThreadPool();

            for (Map.Entry<File, File> entry : inputFiles.entrySet()) {
                Callable<Void> action = new PreDexTask(
                        entry.getKey(),
                        entry.getValue(),
                        hashs,
                        outputHandler,
                        externalLibs.contains(entry.getKey()) ? buildCache : FileCache.NO_CACHE);
                logger.info("Adding PreDexTask for %s : %s", entry.getKey(), action);
                executor.execute(action);
            }

            for (final File file : deletedFiles) {
                executor.execute(() -> {
                    FileUtils.deletePath(file);
                    return null;
                });
            }

            executor.waitForTasksWithQuickFail(false);
            logger.info("Done with all dexing");

            if (needMerge) {
                File outputDir = outputProvider.getContentLocation("main",
                        TransformManager.CONTENT_DEX, getScopes(),
                        Format.DIRECTORY);
                FileUtils.mkdirs(outputDir);

                // first delete the output folder where the final dex file(s) will be.
                FileUtils.cleanOutputDir(outputDir);
                FileUtils.mkdirs(outputDir);

                // find the inputs of the dex merge.
                // they are the content of the intermediate folder.
                List<File> outputs = null;
                if (!multiDex) {
                    // content of the folder is jar files.
                    File[] files = intermediateFolder.listFiles((file, name) -> {
                        return name.endsWith(SdkConstants.DOT_JAR);
                    });
                    if (files != null) {
                        outputs = Arrays.asList(files);
                    }
                } else {
                    File[] directories = intermediateFolder.listFiles(File::isDirectory);
                    if (directories != null) {
                        outputs = Arrays.asList(directories);
                    }
                }

                if (outputs == null) {
                    throw new RuntimeException("No dex files to merge!");
                }

                androidBuilder.convertByteCode(
                        outputs,
                        outputDir,
                        multiDex,
                        mainDexListFile,
                        dexOptions,
                        getOptimize(),
                        outputHandler);
            }
        }
    } catch (Exception e) {
        throw new TransformException(e);
    }
}
```

这个方法看起来很长， 因为要处理 multidex 和非 multidex 的情况。我们慢慢分析。

首先看第一个 if 里的代码`if ((jarInputs.size() + directoryInputs.size()) == 1|| !dexOptions.getPreDexLibraries())`,

这个代码中的`outputDir`路径为: `build/intermediates/transforms/dex/folders/${outputTypes}/${scope}/main`，

这也就是我们看到的最终的`classes.dex`这个文件的地方，这里面调用了一个很重要的方法，但是在这篇文章我们不会去跟进里面的具体实现，可以等待后续文章分析它的具体实现，就是下面这个方法。

```
androidBuilder.convertByteCode(
    inputFiles,
    outputDir,
    multiDex,
    mainDexListFile,
    dexOptions,
    getOptimize(),
    outputHandler);
```

这个方法的作用说起来很简单，但是里面的过程很复杂，就是将字节码转化为 Dalvik 格式的字节码。

这样，这个 if 里的内容就分析完了，接下来我们看对应的 else 里面的内容。

else 里面的代码什么时候触发呢？最简单的方式就是符合开启`multidex`的条件，并开启`multidex`。

这里的代码和 if 里的一样，收集最终需要转化的 inputs 文件，最终输出文件地址为`build/intermediates/transforms/dex/folders/${outputTypes}/${scope}/main`，

通过`androidBuilder.convertByteCode`转化这些字节码。
</br>

在这里讲一段代码，也是 AS 为 gradle 打包提速的一个方法: 生成 cache

```java
WaitableExecutor<Void> executor = WaitableExecutor.useGlobalSharedThreadPool();

for (Map.Entry<File, File> entry : inputFiles.entrySet()) {
    Callable<Void> action = new PreDexTask(
            entry.getKey(),
            entry.getValue(),
            hashs,
            outputHandler,
            externalLibs.contains(entry.getKey()) ? buildCache : FileCache.NO_CACHE);
    logger.info("Adding PreDexTask for %s : %s", entry.getKey(), action);
    executor.execute(action);
}
```

这个地方的代码为参与最终转化的代码生成了 cache，cache 的具体位置在`$Users/.android/build-cache/`这个目录下，用包的 sha1 值作为目录名称，如果对应的包没改过，则 sha1 值不会变，就使用 cache 中的包。

这样我们就把`DexTransform`讲完了。

回到`createPostCompilationTasks`这个方法，讲完了`DexTransform`也就意味着这个方法结束了，也就是说`PostCompilationTasks`到这里也结束了。
</br>

那么我们就需要去理解整个打包过程最后一个方法了。
