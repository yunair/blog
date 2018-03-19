---
title: AndroidPlugin源码解析(六)
date: 2017-08-15
tags: AndroidPlugin
---


上篇提到，我们这篇研究的是接下来三个过程, 分别是:

- createBuildConfigTask
- createApkProcessResTask
- createProcessJavaResTasks

接下来，我们一个一个来看

### createBuildConfigTask

这个task实际执行如下方法:

~~~ java
// must clear the folder in case the packagename changed, otherwise,
// there'll be two classes.
File destinationDir = getSourceOutputDir();
FileUtils.cleanOutputDir(destinationDir);

BuildConfigGenerator generator = new BuildConfigGenerator(
       getSourceOutputDir(),
       getBuildConfigPackageName());

// Hack (see IDEA-100046): We want to avoid reporting "condition is always true"
// from the data flow inspection, so use a non-constant value. However, that defeats
// the purpose of this flag (when not in debug mode, if (BuildConfig.DEBUG && ...) will
// be completely removed by the compiler), so as a hack we do it only for the case
// where debug is true, which is the most likely scenario while the user is looking
// at source code.
//map.put(PH_DEBUG, Boolean.toString(mDebug));
generator.addField("boolean", "DEBUG",
       isDebuggable() ? "Boolean.parseBoolean(\"true\")" : "false")
       .addField("String", "APPLICATION_ID", '"' + getAppPackageName() + '"')
       .addField("String", "BUILD_TYPE", '"' + getBuildTypeName() + '"')
       .addField("String", "FLAVOR", '"' + getFlavorName() + '"')
       .addField("int", "VERSION_CODE", Integer.toString(getVersionCode()))
       .addField("String", "VERSION_NAME", '"' + Strings.nullToEmpty(getVersionName()) + '"')
       .addItems(getItems());

List<String> flavors = getFlavorNamesWithDimensionNames();
int count = flavors.size();
if (count > 1) {
   for (int i = 0; i < count; i += 2) {
       generator.addField(
               "String", "FLAVOR_" + flavors.get(i + 1), '"' + flavors.get(i) + '"');
   }
}

generator.generate();
~~~

这个方法的结果是`build/generated/source/${buildTypes}/${packageName}`目录下生成`BuildConfig.java`文件。这个我相信大家都很熟悉了。

我们可以看到, 即使我们在build.gradle中不填任何`buildConfigField`，也会生成这样一个文件，这个文件里的内容一定有
DEBUG, APPLICATION_ID, BUILD_TYPE, FLAVOR, VERSION_CODE, VERSION_NAME字段。

DEBUG这个字段是由isDebuggable这个方法决定的。也就是我们平时会在debug{}或者release{}中写的`debuggable true`。
</br>

`getItems`不用说，就是把我们用`buildConfigField`声明的参数转化为`Field`。

最后的`generate`方法就不看具体实现了，就是按照Java文件格式，将这些内容写入`BuildConfig.java`文件中。
</br>

好了，这个`BuildConfigTask`到这里就分析完成了。
</br>

我们来看下一个方法

### createApkProcessResTask

~~~ java
// we have to clean the source folder output in case the package name changed.
File srcOut = getSourceOutputDir();
if (srcOut != null) {
    FileUtils.cleanOutputDir(srcOut);
}

@Nullable
// 对于application来说，这个文件是`build/intermediates/res/resources-${buildType}.ap_
// 对于library来说，为null
// 这个是aapt执行完成后输出的压缩文件，里面包含了所有资源文件，和最终的`resources.arsc`
File resOutBaseNameFile = getPackageOutputFile();

File manifestFileToPackage = getManifestFile();

AndroidBuilder builder = getBuilder();
MergingLog mergingLog = new MergingLog(getMergeBlameLogFolder());
ProcessOutputHandler processOutputHandler = new ParsingProcessOutputHandler(
        new ToolOutputParser(new AaptOutputParser(), getILogger()), // 处理执行完的log
        new MergingLogRewriter(mergingLog, builder.getErrorReporter()));
        // merge后的log位于 
        // build/intermediates/blame/res/${buildConfig}/

// 生成对应的Aapt对象
Aapt aapt = AaptGradleFactory.make(
        getBuilder(),
        processOutputHandler,
        true,
        true,
        variantScope.getGlobalScope().getProject(),
        variantScope.getVariantConfiguration().getType(),
        FileUtils.mkdirs(new File(getIncrementalFolder(), "aapt-temp")),
        aaptOptions.getCruncherProcesses());
// 给Aapt设置参数
AaptPackageConfig.Builder config = new AaptPackageConfig.Builder()
        .setManifestFile(manifestFileToPackage)
        .setOptions(getAaptOptions())
        .setResourceDir(getResDir())
        .setLibraries(getLibraries())
        .setCustomPackageForR(getPackageForR())
        .setSymbolOutputDir(getTextSymbolOutputDir())
        .setSourceOutputDir(srcOut)
        .setResourceOutputApk(resOutBaseNameFile)
        .setProguardOutputFile(getProguardOutputFile())
        .setMainDexListProguardOutputFile(getMainDexListProguardOutputFile())
        .setVariantType(getType())
        .setDebuggable(getDebuggable())
        .setPseudoLocalize(getPseudoLocalesEnabled())
        .setResourceConfigs(getResourceConfigs())
        .setSplits(getSplits())
        .setPreferredDensity(getPreferredDensity());

builder.processResources(aapt, config, getEnforceUniquePackageName());

if (resOutBaseNameFile != null && LOG.isInfoEnabled()) {
    LOG.info("Aapt output file {}", resOutBaseNameFile.getAbsolutePath());
}
~~~

大概看一下，这里其实就是执行aapt的过程。

大家应该没有手动执行过aapt的命令，那么跟着我一起看一下aapt命令是如何执行的。

这里和前面提到的一样，只分析Aapt1，至于Aapt2，在这个版本还未成熟，还不能使用。
</br>

那么，我们来看`builder.processResources(aapt, config, getEnforceUniquePackageName());`这个方法:

~~~ java
AaptPackageConfig aaptConfig = aaptConfigBuilder.build();
try {
    aapt.link(aaptConfig).get();
} catch (Exception e) {
    throw new ProcessException("Failed to execute aapt", e);
}
~~~

这个方法我没有贴完，先看`aapt.link(aaptConfig)`方法，然后再继续往下看

#### AbstractAapt link

~~~ java
public ListenableFuture<Void> link(@NonNull AaptPackageConfig config)
        throws AaptException {
    validatePackageConfig(config); // 做一些校验
    return makeValidatedPackage(config);
}
~~~

这个方法很简单，就两个方法，`validatePackageConfig(config);`这个方法就是做一些校验，这里就不关心了。

`makeValidatedPackage(config);`其实就是执行aapt命令。

那么如何执行命令呢，就是由其中的`ProcessInfoBuilder builder = makePackageProcessBuilder(config);`方法创建的，这里其实就是构成了执行aapt的命令，这个命令是:

~~~ shell
aapt package -v -f --no-crunch -i ${android.jarPath} -M ${ManifestPath} -S ${resourcesDir} -m -J ${sourceOutputDir} -F ${resourceOutputApkPath} -G ${proguardOutputPath} -D ${mainDexClassListFilePath} --debug-mode -0 apk --output-text-symbols ${symbolOutputDirPath} 
~~~

如果是Android Library,则会添加`--non-constant-id`选项，看到这个选项应该能了解为何Library的R文件不是`public static final`了。

如果buildTools >= 23，则会添加`--no-version-vectors`。

这个时候，apk的压缩等级为0。

我们关注一下aapt命令中的各个路径:

- android.jarPath = `${sdk}/platforms/android-${compileSdk}/android.jar`
- ManifestPath:
  - library中manifestFile路径为:`build/intermediates/manifests/aapt/${buildConfig}/AndroidManifest.xml`
  - app中manifestFile路径为: `build/intermediates/manifests/full/${buildConfig}/AndroidManifest.xml`
- resourcesDir = `build/intermediates/res/merged/${buildConfig}/`
- sourceOutputDir = `build/generated/source/r/${buildTypes}/${packageName}`
- resourceOutputApkPath 
  - 对于application来说，这个文件是`build/intermediates/res/resources-${buildType}.ap_
  - 对于library来说，为null
- proguardOutputPath = `build/intermediates/proguard-rules/${buildType}/aapt_rules.txt`
- mainDexClassListFilePath = `build/intermediates/multi-dex/${buildTypes}/manifest_keep.txt`
- symbolOutputDirPath = `build/intermediates/symbols/${buildTypes}`

这里有两个文件夹的生成是有条件的，首先当然是proguardOutputPath，只有设置了`minifyEnabled true`，才会生成。

第二，是mainDexClassListFilePath, 只有`minSdk < 21 && buildTools > 24.0.0`才会生成。

这样，aapt这个命令操作了哪些参数就很明显了，至于aapt对这些参数是如何处理的，这就不是我这篇博客的内容了，想了解这方面的内容，请到[老罗][1]的博客继续了解。
</br>

好了，`aapt.link`方法分析完了，我们继续看`builder.processResources(aapt, config, getEnforceUniquePackageName());`这个方法剩下的内容，

剩下的代码结果就是`build/generated/source/r/${buildTypes}/${packageName}`目录下的各个`R.java`文件，根据最终结果反推这些代码的含义还是很容易的，所以这里就不仔细分析了。

这样，ProcessResTask就分析完毕了。
</br>

我们来看下一个方法

### createProcessJavaResTasks

~~~ java
public void createProcessJavaResTasks(
        @NonNull TaskFactory tasks,
        @NonNull VariantScope variantScope) {
    TransformManager transformManager = variantScope.getTransformManager();

    // now copy the source folders java resources into the temporary location, mainly to
    // maintain the PluginDsl COPY semantics.
    AndroidTask<Sync> processJavaResourcesTask =
            androidTasks.create(tasks, new ProcessJavaResConfigAction(variantScope));
    variantScope.setProcessJavaResourcesTask(processJavaResourcesTask);

    // create the stream generated from this task
    variantScope.getTransformManager().addStream(OriginalStream.builder()
            .addContentType(DefaultContentType.RESOURCES)
            .addScope(Scope.PROJECT)
            .setFolder(variantScope.getSourceFoldersJavaResDestinationDir())
            .setDependency(processJavaResourcesTask.getName())
            .build());

    // compute the scopes that need to be merged.
    Set<Scope> mergeScopes = getResMergingScopes(variantScope);

    // Create the merge transform
    MergeJavaResourcesTransform mergeTransform = new MergeJavaResourcesTransform(
            variantScope.getGlobalScope().getExtension().getPackagingOptions(),
            mergeScopes, DefaultContentType.RESOURCES, "mergeJavaRes");

    variantScope.setMergeJavaResourcesTask(
            transformManager.addTransform(tasks, variantScope, mergeTransform));
}
~~~

这个方法的作用就像名字说的，处理Java的资源，通常我们不会遇到Java的资源。

但是有几种情况会出现Java资源：

比如，使用RxJava的时候，使用okhttp的时候，这些库都带着Java的资源；还有Java Doc， LICENSE之类的资源。

这里的资源根据你在packageOptions{}中设置的选项，最后merge成一个jar包或者一个文件夹，文件夹的情况我没见到过，这里就不分析了。

Jar包的路径为:
`build/intermediates/transforms/mergeJavaRes/${buildType}/jars/${TypeValue}/${ScopesValue}/main.jar`

至于TypeValue和ScopesValue，这其实是`QualifiedContent`内部的`DefaultContentType`和`Scope`两个枚举类的value值。

这里就是`Transform Api`的一些用法，看这个代码可以学习`Transform Api`，对于写`Gradle plugin`很有用。
</br>

好了，这里的处理过程不是重点，我们了解生成什么就可以了，所以，ProcessJavaResTasks到这里就结束了。


[1]: http://blog.csdn.net/luoshengyang/article/details/8744683
