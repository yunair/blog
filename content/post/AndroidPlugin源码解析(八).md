---
title: AndroidPlugin源码解析-(八)
date: 2017-09-30
tags: ["AndroidPlugin"]
---

这篇文章我们就来看看打包过程中最后一个方法`createPackagingTask`

### createPackagingTask

```java
public void createPackagingTask(@NonNull TaskFactory tasks,
                                @NonNull VariantScope variantScope,
                                boolean publishApk,
                                @Nullable AndroidTask<InstantRunWrapperTask> fullBuildInfoGeneratorTask) {
    GlobalScope globalScope = variantScope.getGlobalScope();
    ApkVariantData variantData = (ApkVariantData) variantScope.getVariantData();

    boolean signedApk = variantData.isSigned(); // signingConfig配置不为null，且正确配置
    boolean multiOutput = variantData.getOutputs().size() > 1; // 这里由于忽视split，所以暂时认为这个值为false

    GradleVariantConfiguration variantConfiguration = variantScope.getVariantConfiguration();
    /**
     * PrePackaging step class that will look if the packaging of the main APK split is
     * necessary when running in InstantRun mode. In InstantRun mode targeting an api 23 or
     * above device, resources are packaged in the main split APK. However when a warm swap is
     * possible, it is not necessary to produce immediately the new main SPLIT since the runtime
     * use the resources.ap_ file directly. However, as soon as an incompatible change forcing a
     * cold swap is triggered, the main APK must be rebuilt (even if the resources were changed
     * in a previous build).
     */
    IncrementalMode incrementalMode = getIncrementalMode(variantConfiguration);

    List<ApkVariantOutputData> outputDataList = variantData.getOutputs();

    // loop on all outputs. The only difference will be the name of the task, and location
    // of the generated data.
    for (final ApkVariantOutputData variantOutputData : outputDataList) {
        final VariantOutputScope variantOutputScope = variantOutputData.getScope();

        final String outputName = variantOutputData.getFullName();
        InstantRunPatchingPolicy patchingPolicy =
                variantScope.getInstantRunBuildContext().getPatchingPolicy();

        DefaultGradlePackagingScope packagingScope =
                new DefaultGradlePackagingScope(variantOutputScope);

        AndroidTask<PackageApplication> packageApp =
                androidTasks.create(
                        tasks,
                        new PackageApplication.StandardConfigAction(
                                packagingScope, patchingPolicy));

        packageApp.configure(
                tasks,
                task -> variantOutputData.packageAndroidArtifactTask = task);

        TransformManager transformManager = variantScope.getTransformManager();

        // Common code for both packaging tasks.
        Consumer<AndroidTask<PackageApplication>> configureResourcesAndAssetsDependencies =
                task -> {
                    task.dependsOn(tasks, variantScope.getMergeAssetsTask());
                    task.dependsOn(tasks, variantOutputScope.getProcessResourcesTask());
                    transformManager
                            .getStreams(StreamFilter.RESOURCES)
                            .forEach(stream -> task.dependsOn(tasks, stream.getDependencies()));
                };

        configureResourcesAndAssetsDependencies.accept(packageApp);

        CoreSigningConfig signingConfig = packagingScope.getSigningConfig();

        //noinspection VariableNotUsedInsideIf - we use the whole packaging scope below.
        if (signingConfig != null) {
            ValidateSigningTask.ConfigAction configAction =
                    new ValidateSigningTask.ConfigAction(packagingScope);

            AndroidTask<?> validateSigningTask = androidTasks.get(configAction.getName());
            if (validateSigningTask == null) {
                validateSigningTask = androidTasks.create(tasks, configAction);
            }

            packageApp.dependsOn(tasks, validateSigningTask);
        }


        packageApp.optionalDependsOn(
                tasks,
                variantOutputScope.getShrinkResourcesTask(),
                // TODO: When Jack is converted, add activeDexTask to VariantScope.
                variantOutputScope.getVariantScope().getJavaCompilerTask(),
                // TODO: Remove when Jack is converted to AndroidTask.
                variantData.javaCompilerTask,
                variantOutputData.packageSplitResourcesTask,
                variantOutputData.packageSplitAbiTask);


        for (TransformStream stream : transformManager.getStreams(StreamFilter.DEX)) {
            // TODO Optimize to avoid creating too many actions
            packageApp.dependsOn(tasks, stream.getDependencies());
        }

        for (TransformStream stream : transformManager.getStreams(StreamFilter.NATIVE_LIBS)) {
            // TODO Optimize to avoid creating too many actions
            packageApp.dependsOn(tasks, stream.getDependencies());
        }

        variantScope.setPackageApplicationTask(packageApp);

        AndroidTask<?> appTask = packageApp;

        if (signedApk) {
            /*
             * There may be a zip align task in the variant output scope, even if we don't
             * need one for this because we're using new packaging.
             */
            if (variantData.getZipAlignEnabled()
                    && variantOutputScope.getSplitZipAlignTask() != null) {
                appTask.dependsOn(tasks, variantOutputScope.getSplitZipAlignTask());
            }
        }

        // single output
        variantOutputScope.setAssembleTask(variantScope.getAssembleTask());
        variantOutputData.assembleTask = variantData.assembleVariantTask;

        variantOutputScope.getAssembleTask().dependsOn(tasks, appTask);

        if (publishApk) {
            final String projectBaseName = globalScope.getProjectBaseName();

            // if this variant is the default publish config or we also should publish non
            // defaults, proceed with declaring our artifacts.
            if (getExtension().getDefaultPublishConfig().equals(outputName)) {
                appTask.configure(tasks, packageTask -> project.getArtifacts().add("default",
                        new ApkPublishArtifact(
                                projectBaseName,
                                null,
                                (FileSupplier) packageTask)));

                for (FileSupplier outputFileProvider :
                        variantOutputData.getSplitOutputFileSuppliers()) {
                    project.getArtifacts().add("default",
                            new ApkPublishArtifact(projectBaseName, null, outputFileProvider));
                }

                try {
                    if (variantOutputData.getMetadataFile() != null) {
                        project.getArtifacts().add(
                                "default" + VariantDependencies.CONFIGURATION_METADATA,
                                new MetadataPublishArtifact(projectBaseName, null,
                                        variantOutputData.getMetadataFile()));
                    }
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }

                if (variantData.getMappingFileProvider() != null) {
                    project.getArtifacts().add(
                            "default" + VariantDependencies.CONFIGURATION_MAPPING,
                            new MappingPublishArtifact(projectBaseName, null,
                                    variantData.getMappingFileProvider()));
                }
            }

            if (getExtension().getPublishNonDefault()) {
                appTask.configure(tasks, packageTask -> project.getArtifacts().add(
                        variantData.getVariantDependency().getPublishConfiguration().getName(),
                        new ApkPublishArtifact(
                                projectBaseName,
                                null,
                                (FileSupplier) packageTask)));

                for (FileSupplier outputFileProvider :
                        variantOutputData.getSplitOutputFileSuppliers()) {
                    project.getArtifacts().add(
                            variantData.getVariantDependency().getPublishConfiguration().getName(),
                            new ApkPublishArtifact(
                                    projectBaseName,
                                    null,
                                    outputFileProvider));
                }

                try {
                    if (variantOutputData.getMetadataFile() != null) {
                        project.getArtifacts().add(
                                variantData.getVariantDependency().getMetadataConfiguration().getName(),
                                new MetadataPublishArtifact(
                                        projectBaseName,
                                        null,
                                        variantOutputData.getMetadataFile()));
                    }
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }

                if (variantData.getMappingFileProvider() != null) {
                    project.getArtifacts().add(
                            variantData.getVariantDependency().getMappingConfiguration().getName(),
                            new MappingPublishArtifact(
                                    projectBaseName,
                                    null,
                                    variantData.getMappingFileProvider()));
                }

                if (variantData.classesJarTask != null) {
                    project.getArtifacts().add(
                            variantData.getVariantDependency().getClassesConfiguration().getName(),
                            variantData.classesJarTask);
                }
            }
        }
    }


    // create install task for the variant Data. This will deal with finding the
    // right output if there are more than one.
    // Add a task to install the application package
    if (signedApk) {
        AndroidTask<InstallVariantTask> installTask = androidTasks.create(
                tasks, new InstallVariantTask.ConfigAction(variantScope));
        installTask.dependsOn(tasks, variantScope.getAssembleTask());
    }

    if (getExtension().getLintOptions().isCheckReleaseBuilds()
            && (incrementalMode == IncrementalMode.NONE)) {
        createLintVitalTask(tasks, variantData);
    }

    // add an uninstall task
    final AndroidTask<UninstallTask> uninstallTask = androidTasks.create(
            tasks, new UninstallTask.ConfigAction(variantScope));

    tasks.named(UNINSTALL_ALL, uninstallAll -> uninstallAll.dependsOn(uninstallTask.getName()));
}
```

这个方法很长，我们挑重点来看。

先看`AndroidTask<PackageApplication>`这个 Task，这个 Task 具体做哪些事情呢？

```java
@Override
protected void doFullTaskAction() throws IOException {
    /*
     * Clear the cache to make sure we have do not do an incremental build.
     */
    cacheByPath.clear();

    /*
     * Also clear the intermediate build directory. We don't know if anything is in there and
     * since this is a full build, we don't want to get any interference from previous state.
     */
    // build/intermediates/incremental/
    FileUtils.deleteDirectoryContents(getIncrementalFolder());

    Set<File> androidResources = new HashSet<>();
    // build/intermediates/res/resources-${buildType}.ap_
    // 开启shrinkResources build/intermediates/res/resources-${buildType}-stripped.ap_
    File androidResourceFile = getResourceFile();
    if (androidResourceFile != null) {
        androidResources.add(androidResourceFile);
    }

    /*
     * Additionally, make sure we have no previous package, if it exists.
     */
    // build/outpus/apk/
    getOutputFile().delete();

    ImmutableMap<RelativeFile, FileStatus> updatedDex =
            IncrementalRelativeFileSets.fromZipsAndDirectories(getDexFolders());
    ImmutableMap<RelativeFile, FileStatus> updatedJavaResources =
                IncrementalRelativeFileSets.fromZipsAndDirectories(getJavaResourceFiles());
    ImmutableMap<RelativeFile, FileStatus> updatedAssets =
                IncrementalRelativeFileSets.fromZipsAndDirectories(
                        Collections.singleton(getAssets()));
    ImmutableMap<RelativeFile, FileStatus> updatedAndroidResources =
            IncrementalRelativeFileSets.fromZipsAndDirectories(androidResources);
    ImmutableMap<RelativeFile, FileStatus> updatedJniResources=
            IncrementalRelativeFileSets.fromZipsAndDirectories(getJniFolders());

    doTask(
            updatedDex,
            updatedJavaResources,
            updatedAssets,
            updatedAndroidResources,
            updatedJniResources);


}
```

这段代码里关键代码执行路径都写在了注释里，发现这段代码调用了`doTask`方法，我们看看这个方法做了什么:

```java
private void doTask(
        @NonNull ImmutableMap<RelativeFile, FileStatus> changedDex,
        @NonNull ImmutableMap<RelativeFile, FileStatus> changedJavaResources,
        @NonNull ImmutableMap<RelativeFile, FileStatus> changedAssets,
        @NonNull ImmutableMap<RelativeFile, FileStatus> changedAndroidResources,
        @NonNull ImmutableMap<RelativeFile, FileStatus> changedNLibs)
        throws IOException {
    PrivateKey key;
    X509Certificate certificate;
    boolean v1SigningEnabled;
    boolean v2SigningEnabled;

    try {
        if (signingConfig != null && signingConfig.isSigningReady()) {
            CertificateInfo certificateInfo =
                    KeystoreHelper.getCertificateInfo(
                            signingConfig.getStoreType(),
                            checkNotNull(signingConfig.getStoreFile()),
                            checkNotNull(signingConfig.getStorePassword()),
                            checkNotNull(signingConfig.getKeyPassword()),
                            checkNotNull(signingConfig.getKeyAlias()));
            key = certificateInfo.getKey();
            certificate = certificateInfo.getCertificate();
            v1SigningEnabled = signingConfig.isV1SigningEnabled();
            v2SigningEnabled = signingConfig.isV2SigningEnabled();
        } else {
            key = null;
            certificate = null;
            v1SigningEnabled = false;
            v2SigningEnabled = false;
        }

        ApkCreatorFactory.CreationData creationData =
                new ApkCreatorFactory.CreationData(
                        getOutputFile(),
                        key,
                        certificate,
                        v1SigningEnabled,
                        v2SigningEnabled,
                        null, // BuiltBy
                        getBuilder().getCreatedBy(),
                        getMinSdkVersion(),
                        PackagingUtils.getNativeLibrariesLibrariesPackagingMode(manifest),
                        getNoCompressPredicate()::apply);

        try (IncrementalPackager packager = createPackager(creationData)) {
            packager.updateDex(changedDex);
            packager.updateJavaResources(changedJavaResources);
            packager.updateAssets(changedAssets);
            packager.updateAndroidResources(changedAndroidResources);
            packager.updateNativeLibraries(changedNLibs);
        }
    } catch (PackagerException | KeytoolException e) {
        throw new RuntimeException(e);
    }

    /*
     * Save all used zips in the cache.
     */
    // cache相关代码，忽视掉
}
```

看到这些参数`v1SigningEnabled, v2SigningEnabled`不知道你会联想到什么，没错，就是和 apk 签名相关。

这里的核心就是这段代码:

```java
IncrementalPackager packager = createPackager(creationData))
packager.updateDex(changedDex);
packager.updateJavaResources(changedJavaResources);
packager.updateAssets(changedAssets);
packager.updateAndroidResources(changedAndroidResources);
packager.updateNativeLibraries(changedNLibs);
```

创建`Packager`对象这里就不看了，在下面的方法中用到这里面的参数的时候再来讲讲对应参数的含义。
也就是看一下接下来的 5 个方法:

```java
packager.updateDex(changedDex);
packager.updateJavaResources(changedJavaResources);
packager.updateAssets(changedAssets);
packager.updateAndroidResources(changedAndroidResources);
```

后面这 5 个方法最终全部是调用`updateFiles`方法，这个方法的主要执行的都是`mApkCreator`对象的方法，所以先讲一下`ApkCreator`这个对象，讲完之后再来看一看这个方法:

对于 ApkZFileCreator 的构造函数来说，主要是构造了一个`ZFile`文件，我们直接看那个方法:

#### ZFiles apk

```java
public static ZFile apk(
        @NonNull File f,
        @NonNull ZFileOptions options,
        @Nullable PrivateKey key,
        @Nullable X509Certificate certificate,
        boolean v1SigningEnabled,
        boolean v2SigningEnabled,
        @Nullable String builtBy,
        @Nullable String createdBy,
        int minSdkVersion)
        throws IOException {
    ZFile zfile = apk(f, options);
    // 根据我们的代码分析过程，这里是null
    if (builtBy == null) {
        builtBy = DEFAULT_BUILD_BY;
    }

    if (createdBy == null) {
        createdBy = DEFAULT_CREATED_BY;
    }

    ManifestGenerationExtension manifestExt = new ManifestGenerationExtension(builtBy,
            createdBy);
    manifestExt.register(zfile);

    if (key != null && certificate != null) {
        if (v1SigningEnabled) {
                String apkSignedHeaderValue =
                        (v2SigningEnabled)
                                ? SignatureExtension
                                        .SIGNATURE_ANDROID_APK_SIGNER_VALUE_WHEN_V2_SIGNED
                                : null;
                SignatureExtension jarSignatureSchemeExt = new SignatureExtension(manifestExt,
                        minSdkVersion, certificate, key,
                        apkSignedHeaderValue);
                jarSignatureSchemeExt.register();
            }
            if (v2SigningEnabled) {
                FullApkSignExtension apkSignatureSchemeV2Ext =
                        new FullApkSignExtension(
                                zfile,
                                minSdkVersion,
                                certificate,
                                key);
                apkSignatureSchemeV2Ext.register();
            }
            if (!v1SigningEnabled) {
                // Remove v1 signature files from the APK
                for (StoredEntry entry : zfile.entries()) {
                    if (SignatureExtension.isIgnoredFile(
                            entry.getCentralDirectoryHeader().getName())) {
                        entry.delete();
                    }
                }
            }
    }

    return zfile;
}
```

首先，这个方法通过`apk(f, options)`生成了`ZFile`文件，这个方法设置了`ZFile`的对齐规则，然后，调用`ZFile`的构造函数生成`ZFile`，也就是空的最终生成的`Apk`文件，生成之后，我们继续往下看，可以看到，这里有几个`Extension`, 分别为`ManifestGenerationExtension`, `SignatureExtension`, `FullApkSignExtension`, 我们来看看这三个类的作用是什么:

##### ManifestGenerationExtension register

```java
public void register(@NonNull ZFile zFile) throws IOException {
    Preconditions.checkState(mExtension == null, "register() has already been invoked.");
    mZFile = zFile;

    rebuildManifest();

    mExtension = new ZFileExtension() {
        @Nullable
        @Override
        public IOExceptionRunnable beforeUpdate() {
            return ManifestGenerationExtension.this::updateManifest;
        }
    };

    mZFile.addZFileExtension(mExtension);
}
```

这个方法的核心在`mZFile.addZFileExtension(mExtension);`，这个`ZFileExtension`的作用是在`ZFile`对应状态的时候通知该`Extensions`执行操作。
在最终的 Apk 中，反编译一下，`META-INF/MANIFEST.MF`这个文件就是这里生成的。

##### SignatureExtension register

```java
public void register() throws IOException {
    Preconditions.checkState(mExtension == null, "register() already invoked");

    mExtension = new ZFileExtension() {
        @Nullable
        @Override
        public IOExceptionRunnable beforeUpdate() {
            return SignatureExtension.this::updateSignatureIfNeeded;
        }

        @Nullable
        @Override
        public IOExceptionRunnable added(@NonNull final StoredEntry entry,
                @Nullable final StoredEntry replaced) {
            if (replaced != null) {
                Preconditions.checkArgument(entry.getCentralDirectoryHeader().getName().equals(
                        replaced.getCentralDirectoryHeader().getName()));
            }

            if (isIgnoredFile(entry.getCentralDirectoryHeader().getName())) {
                return null;
            }

            return () -> {
                if (replaced != null) {
                    SignatureExtension.this.removed(replaced);
                }

                SignatureExtension.this.added(entry);
            };
        }

        @Nullable
        @Override
        public IOExceptionRunnable removed(@NonNull final StoredEntry entry) {
            if (isIgnoredFile(entry.getCentralDirectoryHeader().getName())) {
                return null;
            }

            return () -> SignatureExtension.this.removed(entry);
        }
    };

    mManifestExtension.zFile().addZFileExtension(mExtension);
    readSignatureFile();
}
```

这里也是一样，注册了一个`ZFileExtension`，作用就是在后面写入文件的时候进行签名，这个签名文件的位置是`META-INF/CERT.SF`
这里是 V1 签名的实现，如果允许 V2 签名，则在该文件的前几行的存在`X-Android-APK-Signed: 2`这个声明，否则的话说明没开启 V2 签名。
V2 签名的`register`实现我们就不看了，和这个类似，如果想研究 V2 签名的细节，这里很重要，但是你要了解 Zip 文件格式再来看会很有帮助。

好了，这些讲完之后我们就可以看`updateFiles`方法了，记得每次`ZFile`文件改变状态的时候，都会回调上面的这些`Extension`。

#### IncrementalPackager updateFiles

```java
private void updateFiles(@NonNull Set<PackagedFileUpdate> updates) throws IOException {
    Preconditions.checkNotNull(mApkCreator, "mApkCreator == null");

    // 这个的意思是拿到状态为`FileStatus.REMOVED`的文件，接下来拿到文件名
    Iterable<String> deletedPaths =
            Iterables.transform(
                    Iterables.filter(
                            updates,
                            Predicates.compose(
                                    Predicates.equalTo(FileStatus.REMOVED),
                                    PackagedFileUpdate.EXTRACT_STATUS)),
                    PackagedFileUpdate.EXTRACT_NAME);

    for (String deletedPath : deletedPaths) {
        mApkCreator.deleteFile(deletedPath);
    }
    // 状态为FileStatus.NEW或者FileStatus.CHANGED的PackagedFileUpdate断言
    Predicate<PackagedFileUpdate> isNewOrChanged =
            Predicates.compose(
                    Predicates.or(
                            Predicates.equalTo(FileStatus.NEW),
                            Predicates.equalTo(FileStatus.CHANGED)),
                    PackagedFileUpdate.EXTRACT_STATUS);
    // Source为Base的文件，source的类型为`build/intermediates/incremental/package${buildType}/dex-renamer-state.txt`
    Function<PackagedFileUpdate, File> extractBaseFile =
            Functions.compose(
                    RelativeFile.EXTRACT_BASE,
                    PackagedFileUpdate.EXTRACT_SOURCE);

    Iterable<PackagedFileUpdate> newOrChangedNonArchiveFiles =
            Iterables.filter(
                    updates,
                    Predicates.and(
                            isNewOrChanged,
                            Predicates.compose(
                                    Files.isDirectory(),
                                    extractBaseFile)));

    for (PackagedFileUpdate rf : newOrChangedNonArchiveFiles) {
        mApkCreator.writeFile(rf.getSource().getFile(), rf.getName());
    }

    Iterable<PackagedFileUpdate> newOrChangedArchiveFiles =
            Iterables.filter(
                    updates,
                    Predicates.and(
                            isNewOrChanged,
                            Predicates.compose(
                                    Files.isFile(),
                                    extractBaseFile)));

    Iterable<File> archives = Iterables.transform(newOrChangedArchiveFiles, extractBaseFile);
    Set<String> names = Sets.newHashSet(
            Iterables.transform(
                    newOrChangedArchiveFiles,
                    PackagedFileUpdate.EXTRACT_NAME));

    /*
     * Build the name map. The name of the file in the filesystem (or zip file) may not
     * match the name we want to package it as. See PackagedFileUpdate for more information.
     */
    Map<String, String> pathNameMap = Maps.newHashMap();
    for (PackagedFileUpdate archiveUpdate : newOrChangedArchiveFiles) {
        pathNameMap.put(archiveUpdate.getSource().getOsIndependentRelativePath(),
                archiveUpdate.getName());
    }

    for (File arch : Sets.newHashSet(archives)) {
        mApkCreator.writeZip(arch, pathNameMap::get, name -> !names.contains(name));
    }
}
```

首先通过各种条件过滤出符合条件的文件列表，然后通过`mApkCreator.writeFile` `mApkCreator.writeZip`这两个方法将文件写入 apk 中。

这里就不详细讲了，因为这些方法最终还是调用`ZFile`对象的方法，想了解这些方法的具体实现，请先了解 Zip 文件格式，或者等我之后的文章。

这样，就把所有的`Dex`, `JavaResources`, `Assets`, `AndroidResources`, `JniResources`，写入 Apk 文件中，并进行了签名。
</br>

回到`createPackagingTask`方法继续往下看，可以看到有`ValidateSigningTask`这个 Task。

这个 Task 我们就不详细解释了，因为代码很容易理解，就是检测`keystore`文件是否存在，不存在的话检测默认的`debug.keystore`。

因为`packageApp.dependsOn(tasks, validateSigningTask);`这个方法，我们知道，先验证签名后打包 Apk。
</br>

好了，继续往下看，可以看到一个`ZipAlign`的 Task，这个 Task 的任务就是执行`zipalign`，也就是将文件进行 4 位对齐。

这个 Task 构造了`zipalign`的命令行参数，执行的命令为`${zipalignPath}/zipalign -f -p 4 ${inputFile} ${outputFile}`
</br>

下面就是`if (publishApk)`这个 if 条件了，里面的内容就是增加最终输出的文件的个数，每个`Artifact`代表一个最终的文件。
我们这个系列的文章主要是 apk 的流程，所以这里就不详细分析了。
</br>

前面完成了打包 Task，后面就可以安装到手机里了，安装到手机由`InstallVariantTask`这个任务完成。

这个任务使用`adb`，将文件传入手机中，并使用`pm`命令安装 apk。原先我还以为直接使用`adb install -r ${apkFile}`来实现的，原来不是我理解的啊。

最后，添加`Lint`任务，加入卸载 Apk 的任务，这样，整个 Apk 打包流程就结束了。
