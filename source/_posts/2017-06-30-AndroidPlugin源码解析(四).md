---
title: AndroidPlugin源码解析(四)
date: 2017-06-30
tags: AndroidPlugin
---

接下来几篇就来解析上篇提到的代码。

一个一个Task来看

### createAnchorTasks

这个方法创建的Tasks就如名字所示，是Anchor Tasks（即所谓的锚点任务）。在这里不做仔细的解析。

具体就是创建一些preXXXBuild任务，generateXXXResources/Assets/Sources任务以及compileXXXSources任务。

### createCheckManifestTask

主要作用就是检测Manifest是否存在在指定路径。在这里也不做仔细的解析。

### createDependencyStreams

主要作用是将所有依赖的Jar包合成，通过`TransformManager`来完成此任务。

这里提一句，`TransformManager`是新加入的Api，让我们可以开发Gradle插件来Hook Android自身的打包过程。

Tinker之类的库都是通过这个Api来Hook打包过程，插入自己的代码逻辑的。

### createMergeAppManifestsTask

精简代码如下：

~~~ java
public void createMergeAppManifestsTask(
        @NonNull TaskFactory tasks,
        @NonNull VariantScope variantScope) {
    ApplicationVariantData appVariantData =
            (ApplicationVariantData) variantScope.getVariantData();
    Set<String> screenSizes = appVariantData.getCompatibleScreens();
    // loop on all outputs. The only difference will be the name of the task, and location
    // of the generated manifest
    for (final BaseVariantOutputData vod : appVariantData.getOutputs()) {
        VariantOutputScope scope = vod.getScope();
         List<ManifestMerger2.Invoker.Feature> optionalFeatures = getIncrementalMode(
                    variantScope.getVariantConfiguration()) != IncrementalMode.NONE
                    ? ImmutableList.of(ManifestMerger2.Invoker.Feature.INSTANT_RUN_REPLACEMENT)
                    : ImmutableList.of();
        AndroidTask<? extends ManifestProcessorTask> processManifestTask =
                androidTasks.create(tasks, getMergeManifestConfig(scope, optionalFeatures));
        scope.setManifestProcessorTask(processManifestTask);
        // 将Manifest放入对应的output文件夹内
        // library放入build/intermediates/bundles
        // app放入build/intermediates/manifests
        addManifestArtifact(tasks, scope.getVariantScope().getVariantData());
    }
}
~~~

这里执行的主要代码是两种Task，就是`ManifestProcessorTask`的子类，这个我们来仔细跟进。

两个子类分别是`MergeManifests`和`ProcessManifest`，后者是针对Library打包的时候调用的，

而且这两个子类在我们打包过程中，执行的代码都是`AndroidBuilder`里面的`mergeManifestsForApplication`方法，只不过参数不一样。

所以直接解析`mergeManifestsForApplication` 方法了

#### AndroidBuilder mergeManifestsForApplication

~~~ java
public void mergeManifestsForApplication(
        @NonNull File mainManifest,
        @NonNull List<File> manifestOverlays,
        @NonNull List<? extends AndroidLibrary> libraries,
        String packageOverride,
        int versionCode,
        String versionName,
        @Nullable String minSdkVersion,
        @Nullable String targetSdkVersion,
        @Nullable Integer maxSdkVersion,
        @NonNull String outManifestLocation,
        @Nullable String outAaptSafeManifestLocation,
        @Nullable String outInstantRunManifestLocation,
        ManifestMerger2.MergeType mergeType,
        Map<String, Object> placeHolders,
        @NonNull List<Invoker.Feature> optionalFeatures,
        @Nullable File reportFile) {
    Invoker manifestMergerInvoker =
            ManifestMerger2.newMerger(mainManifest, mLogger, mergeType)
            .setPlaceHolderValues(placeHolders)
            .addFlavorAndBuildTypeManifests(
                    manifestOverlays.toArray(new File[manifestOverlays.size()]))
            .addBundleManifests(libraries)
            .withFeatures(optionalFeatures.toArray(
                    new Invoker.Feature[optionalFeatures.size()]))
            .setMergeReportFile(reportFile);
    if (mergeType == ManifestMerger2.MergeType.APPLICATION) {
        // 如果是Application，则加入移除tools声明的Features
        manifestMergerInvoker.withFeatures(Invoker.Feature.REMOVE_TOOLS_DECLARATIONS);
    }
    //noinspection VariableNotUsedInsideIf
    if (outAaptSafeManifestLocation != null) {
        // 这个代表是Library， Application的时候该值为null
        // 这个Feature的含义是Encode unresolved placeholders to be AAPT friendly.
        // 也就是说某些placeHolders可能没有正确的赋值，留给Application赋值
        manifestMergerInvoker.withFeatures(Invoker.Feature.MAKE_AAPT_SAFE);
    }
    // 这个就是将packageName versionCode versionName minSdkVersion targetSdkVersion maxSdkVersion设置进去
    setInjectableValues(manifestMergerInvoker,
            packageOverride, versionCode, versionName,
            minSdkVersion, targetSdkVersion, maxSdkVersion);
    MergingReport mergingReport = manifestMergerInvoker.merge();
    mLogger.info("Merging result:" + mergingReport.getResult());
    switch (mergingReport.getResult()) {
        case WARNING:
            mergingReport.log(mLogger);
            // fall through since these are just warnings.
        case SUCCESS:
            String xmlDocument = mergingReport.getMergedDocument(
                    MergingReport.MergedManifestKind.MERGED);
            String annotatedDocument = mergingReport.getMergedDocument(
                    MergingReport.MergedManifestKind.BLAME);
            if (annotatedDocument != null) {
                mLogger.verbose(annotatedDocument);
            }
            save(xmlDocument, new File(outManifestLocation));
            mLogger.info("Merged manifest saved to " + outManifestLocation);
            if (outAaptSafeManifestLocation != null) {
                save(mergingReport.getMergedDocument(MergingReport.MergedManifestKind.AAPT_SAFE),
                        new File(outAaptSafeManifestLocation));
            }
            break;
        case ERROR:
            mergingReport.log(mLogger);
            throw new RuntimeException(mergingReport.getReportString());
        default:
            throw new RuntimeException("Unhandled result type : "
                    + mergingReport.getResult());
    }
}
~~~

我们可以看出，初始化了manifestMergerInvoker对象,在初始化的过程中，做了如下操作：
1. 将placeHolder的值设置进来
2. 将BuildType和ProductFlavor对应的Manifests存进来了
3. 将非Provided的library加入进来，代码如下。所以Provided的库不会打包到最终apk中。

~~~ java
for (AndroidBundle library : libraries) {
    if (!library.isProvided()) {
        mLibraryFilesBuilder.add(Pair.of(library.getName(), library.getManifest()));
    }
}
~~~

接下来是`manifestMergerInvoker.merge()`，真正执行merge的地方，代码如下:

##### ManifestMerger2.Invoker merge()

~~~ java
public MergingReport merge() throws MergeFailureException {
    // provide some free placeholders values.
    ImmutableMap<ManifestSystemProperty, Object> systemProperties = mSystemProperties.build();
    if (systemProperties.containsKey(ManifestSystemProperty.PACKAGE)) {
        // if the package is provided, make it available for placeholder replacement.
        // 将PackageName放入待替换Map中
        mPlaceholders.put(PACKAGE_NAME, systemProperties.get(ManifestSystemProperty.PACKAGE));
        // as well as applicationId since package system property overrides everything
        // but not when output is a library since only the final (application)
        // application Id should be used to replace libraries "applicationId" placeholders.
        if (mMergeType != MergeType.LIBRARY) {
            // 如果不是Library，则将ApplicationID放入待替换Map中
            mPlaceholders.put(APPLICATION_ID, systemProperties.get(ManifestSystemProperty.PACKAGE));
        }
    }
    FileStreamProvider fileStreamProvider = mFileStreamProvider != null
            ? mFileStreamProvider : new FileStreamProvider();
    ManifestMerger2 manifestMerger =
            new ManifestMerger2(
                    mLogger,
                    mMainManifestFile,
                    mLibraryFilesBuilder.build(),
                    mFlavorsAndBuildTypeFiles.build(),
                    mFeaturesBuilder.build(),
                    mPlaceholders.build(),
                    new MapBasedKeyBasedValueResolver<ManifestSystemProperty>(systemProperties),
                    mMergeType,
                    Optional.fromNullable(mReportFile),
                    fileStreamProvider);
    return manifestMerger.merge();
}
~~~

可以看到，这个`merge`方法主要就是调用`ManifestMerger2`的`merge`方法，剩下的看代码中的注释即可，`ManifestMerger2`的`merge`方法代码如下:

##### ManifestMerger2 merge()

~~~ java
private MergingReport merge() throws MergeFailureException {
    // initiate a new merging report
    MergingReport.Builder mergingReportBuilder = new MergingReport.Builder(mLogger);
    SelectorResolver selectors = new SelectorResolver();
    // load all the libraries xml files up front to have a list of all possible node:selector
    // values.
    List<LoadedManifestInfo> loadedLibraryDocuments =
            loadLibraries(selectors, mergingReportBuilder);
    // load the main manifest file to do some checking along the way.
    LoadedManifestInfo loadedMainManifestInfo = load(
            new ManifestInfo(
                    mManifestFile.getName(),
                    mManifestFile,
                    XmlDocument.Type.MAIN,
                    Optional.<String>absent() /* mainManifestPackageName */),
            selectors,
            mergingReportBuilder);
    // first do we have a package declaration in the main manifest ?
    Optional<XmlAttribute> mainPackageAttribute =
            loadedMainManifestInfo.getXmlDocument().getPackage();
    
    // perform system property injection
    performSystemPropertiesInjection(mergingReportBuilder,
            loadedMainManifestInfo.getXmlDocument());
    // force the re-parsing of the xml as elements may have been added through system
    // property injection.
    // 因为前面将mergingReportBuilder内的元素加入到loadedMainManifestInfo内了，
    // 所以需要重新解析一遍
    loadedMainManifestInfo = new LoadedManifestInfo(loadedMainManifestInfo,
            loadedMainManifestInfo.getOriginalPackageName(),
            loadedMainManifestInfo.getXmlDocument().reparse());
    // invariant : xmlDocumentOptional holds the higher priority document and we try to
    // merge in lower priority documents.
    Optional<XmlDocument> xmlDocumentOptional = Optional.absent();
    for (File inputFile : mFlavorsAndBuildTypeFiles) {
        mLogger.info("Merging flavors and build manifest %s \n", inputFile.getPath());
        LoadedManifestInfo overlayDocument = load(
                new ManifestInfo(null, inputFile, XmlDocument.Type.OVERLAY,
                        Optional.of(mainPackageAttribute.get().getValue())),
                selectors,
                mergingReportBuilder);
        // check package declaration.
        Optional<XmlAttribute> packageAttribute =
                overlayDocument.getXmlDocument().getPackage();
        overlayDocument.getXmlDocument().getRootNode().getXml().setAttribute("package",
                mainPackageAttribute.get().getValue());
        xmlDocumentOptional = merge(xmlDocumentOptional, overlayDocument, mergingReportBuilder);
        if (!xmlDocumentOptional.isPresent()) {
            return mergingReportBuilder.build();
        }
    }
    mLogger.info("Merging main manifest %s\n", mManifestFile.getPath());
    xmlDocumentOptional =
            merge(xmlDocumentOptional, loadedMainManifestInfo, mergingReportBuilder);
    if (!xmlDocumentOptional.isPresent()) {
        return mergingReportBuilder.build();
    }
    // force main manifest package into resulting merged file when creating a library manifest.
    if (mMergeType == MergeType.LIBRARY) {
        // extract the package name...
        String mainManifestPackageName = loadedMainManifestInfo.getXmlDocument().getRootNode()
                .getXml().getAttribute("package");
        // save it in the selector instance.
        if (!Strings.isNullOrEmpty(mainManifestPackageName)) {
            xmlDocumentOptional.get().getRootNode().getXml()
                    .setAttribute("package", mainManifestPackageName);
        }
    }
    for (LoadedManifestInfo libraryDocument : loadedLibraryDocuments) {
        mLogger.info("Merging library manifest " + libraryDocument.getLocation());
        xmlDocumentOptional = merge(
                xmlDocumentOptional, libraryDocument, mergingReportBuilder);
        if (!xmlDocumentOptional.isPresent()) {
            return mergingReportBuilder.build();
        }
    }
    // done with proper merging phase, now we need to trim unwanted elements, placeholder
    // substitution and system properties injection.
    ElementsTrimmer.trim(xmlDocumentOptional.get(), mergingReportBuilder);
    if (mergingReportBuilder.hasErrors()) {
        return mergingReportBuilder.build();
    }
    if (!mOptionalFeatures.contains(Invoker.Feature.NO_PLACEHOLDER_REPLACEMENT)) {
        // do one last placeholder substitution, this is useful as we don't stop the build
        // when a library failed a placeholder substitution, but the element might have
        // been overridden so the problem was transient. However, with the final document
        // ready, all placeholders values must have been provided.
        performPlaceHolderSubstitution(loadedMainManifestInfo, xmlDocumentOptional.get(),
                mergingReportBuilder);
        if (mergingReportBuilder.hasErrors()) {
            return mergingReportBuilder.build();
        }
    }
    // perform system property injection.
    // 进行比如PACKAGE, VERSION_CODE, VERSION_NAME, MIN_SDK_VERSION之类的系统属性注入
    performSystemPropertiesInjection(mergingReportBuilder, xmlDocumentOptional.get());
    XmlDocument finalMergedDocument = xmlDocumentOptional.get();
    // 做一些验证，排序的工作
    PostValidator.validate(finalMergedDocument, mergingReportBuilder);
    if (mergingReportBuilder.hasErrors()) {
        finalMergedDocument.getRootNode().addMessage(mergingReportBuilder,
                MergingReport.Record.Severity.WARNING,
                "Post merge validation failed");
    }
    // finally optional features handling.
    // 清理tools声明等一些收尾工作
    processOptionalFeatures(finalMergedDocument, mergingReportBuilder);
    MergingReport mergingReport = mergingReportBuilder.build();
    if (mReportFile.isPresent()) {
        writeReport(mergingReport);
    }
    return mergingReport;
}
~~~

在这段代码里，我们可以看到三个LoadedManifestInfo对象，分别是:

1. loadedLibraryDocuments
2. loadedMainManifestInfo
3. overlayDocument

loadedLibraryDocuments虽然是`List<LoadedManifestInfo>`,也可以归于同一类。

这就告诉了我们merge过程中，哪些Manifest会参与其中。再继续往下看，可以注意到，有一行代码

`xmlDocumentOptional = merge(xmlDocumentOptional, overlayDocument, mergingReportBuilder);`

又一个`merge`?!

是的，没错，这个`merge`就是决定最终Manifest中元素的地方，我们来看一下代码:

~~~ java
private Optional<XmlDocument> merge(
        @NonNull Optional<XmlDocument> xmlDocument,
        @NonNull LoadedManifestInfo lowerPriorityDocument,
        @NonNull MergingReport.Builder mergingReportBuilder) throws MergeFailureException {
    // 检验xml 元素的合法性...
    // 
    Optional<XmlDocument> result;
    if (xmlDocument.isPresent()) {
        result = xmlDocument.get().merge(
                lowerPriorityDocument.getXmlDocument(), mergingReportBuilder);
    } else {
        mergingReportBuilder.getActionRecorder().recordDefaultNodeAction(
                lowerPriorityDocument.getXmlDocument().getRootNode());
        result = Optional.of(lowerPriorityDocument.getXmlDocument());
    }

    // if requested, dump each intermediary merging stage into the report.
    if (mOptionalFeatures.contains(Invoker.Feature.KEEP_INTERMEDIARY_STAGES)
            && result.isPresent()) {
        mergingReportBuilder.addMergingStage(result.get().prettyPrint());
    }

    return result;
}
~~~

我们可以看到，这里面主要逻辑就在`if else里了`，先看else的内容:

~~~ java
mergingReportBuilder.getActionRecorder().recordDefaultNodeAction(
                lowerPriorityDocument.getXmlDocument().getRootNode());
~~~

这个方法的内容很简单，就是将存在于`lowerPriorityDocument`中且不存在`mergingReportBuilder`中的元素加入到`mergingReportBuilder`中，

所以，越晚调用此方法，越会忽略`lowerPriorityDocument`中的元素。

可以看到， merge的过程中，三个此方法的调用顺序是:

1. `xmlDocumentOptional = merge(xmlDocumentOptional, overlayDocument, mergingReportBuilder);`
2. `xmlDocumentOptional = merge(xmlDocumentOptional, loadedMainManifestInfo, mergingReportBuilder);`
3. `xmlDocumentOptional = merge(xmlDocumentOptional, libraryDocument, mergingReportBuilder);`

这就是官方文档[Merge Multiple Manifest Files][1]所提到的顺序的原因。

接下来，我们再看if的内容:

##### XmlDocument merge

~~~ java
public Optional<XmlDocument> merge(
        @NonNull XmlDocument lowerPriorityDocument,
        @NonNull MergingReport.Builder mergingReportBuilder) {

    if (getFileType() == Type.MAIN) {
        mergingReportBuilder.getActionRecorder().recordDefaultNodeAction(getRootNode());
    }
    // 处理Manifest里面的tools定义的操作,这里根据tools定义的不同操作，进行不同的处理
    getRootNode().mergeWithLowerPriorityNode(
            lowerPriorityDocument.getRootNode(), mergingReportBuilder);
    // 进行一些隐含的检测，比如uses-sdk, targetSdk，library和main的必须相同
    // 某些权限申请了一个必须申请另一个，比如WRITE_EXTERNAL_STORAGE
    addImplicitElements(lowerPriorityDocument, mergingReportBuilder);

    // force re-parsing as new nodes may have appeared.
    return mergingReportBuilder.hasErrors()
            ? Optional.<XmlDocument>absent()
            : Optional.of(reparse());
}
~~~

看代码中的注释，应该大概能了解函数的含义。
</br>

这样，整个Manifest的merge过程就讲完了.下一篇，我们来研究`createGenerateResValuesTask`和`createMergeResourcesTask`的过程.

如果大家有不懂，欢迎通过留言和邮件进行交流。


[1]: [https://developer.android.com/studio/build/manifest-merge.html]