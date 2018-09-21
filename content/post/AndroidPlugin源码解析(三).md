---
title: AndroidPlugin源码解析-(三)
date: 2017-06-15
tags: ["AndroidPlugin"]
---

上个系列文章留下的`taskManager.createTasksForVariantData(tasks, variantData);`的分析放到这个系列来进行。

还记得上篇的内容吗？这个方法的具体实现主要分为 application 和 library。

本文将会跳过`Instant Run`相关代码

由于代码篇幅原因，此篇文章主要是两个 taskManager 创建 Task 的步骤。并不包含具体的分析过程，分析过程放在后面几篇文章讲解。

对比下面的代码可以看出，Application Task 和 Library Task 大部分都是相同的，只有几个方法不同。

### ApplicationTaskManager createTasksForVariantData

```java
@Override
public void createTasksForVariantData(
        @NonNull final TaskFactory tasks,
        @NonNull final BaseVariantData<? extends BaseVariantOutputData> variantData) {
    assert variantData instanceof ApplicationVariantData;
    final String projectPath = project.getPath();
    final String variantName = variantData.getName();
    final VariantScope variantScope = variantData.getScope();

    createAnchorTasks(tasks, variantScope);

    createCheckManifestTask(tasks, variantScope);
    //...
    //有一些和手表相关的Tasks创建过程
    //...
    // Create all current streams (dependencies mostly at this point)
    createDependencyStreams(tasks, variantScope);
    // Add a task to process the manifest(s)
    createMergeAppManifestsTask(tasks, variantScope);
    // Add a task to create the res values
    createGenerateResValuesTask(tasks, variantScope);
    // Add a task to compile renderscript files.
    createRenderscriptTask(tasks, variantScope);
    // Add a task to merge the resource folders
    createMergeResourcesTask(tasks, variantScope);
    // Add a task to merge the asset folders
    createMergeAssetsTask(tasks, variantScope);
    // Add a task to create the BuildConfig class
    createBuildConfigTask(tasks, variantScope);
    // Add a task to process the Android Resources and generate source files
    createApkProcessResTask(tasks, variantScope);
    // Add a task to process the java resources
    createProcessJavaResTasks(tasks, variantScope);
    // Add a task to process the Aidl
    createAidlTask(tasks, variantScope);
    createShaderTask(tasks, variantScope);
    // Add NDK tasks
    if (!isComponentModelPlugin) {
         createNdkTasks(variantScope);
    } else {
        if (variantData.compileTask != null) {
            variantData.compileTask.dependsOn(getNdkBuildable(variantData));
        } else {
            variantScope.getCompileTask().dependsOn(tasks, getNdkBuildable(variantData));
        }
    }
    variantScope.setNdkBuildable(getNdkBuildable(variantData));
    // Add external native build tasks
    createExternalNativeBuildJsonGenerators(variantScope);
    createExternalNativeBuildTasks(tasks, variantScope);
    // Add a task to merge the jni libs folders
    createMergeJniLibFoldersTasks(tasks, variantScope);

    // Add a compile task
    CoreJackOptions jackOptions =
               variantData.getVariantConfiguration().getJackOptions();
    AndroidTask<? extends JavaCompile> javacTask =
            createJavacTask(tasks, variantScope);
    if (jackOptions.isEnabled()) {
        AndroidTask<TransformTask> jackTask =
                createJackTask(tasks, variantScope, true /*compileJavaSource*/);
        setJavaCompilerTask(jackTask, tasks, variantScope);
    } else {
        // Prevent the use of java 1.8 without jack, which would otherwise cause an
        // internal javac error.
        if(variantScope.getGlobalScope().getExtension().getCompileOptions()
                .getTargetCompatibility().isJava8Compatible()) {
            // Only warn for users of retrolambda and dexguard
            if (project.getPlugins().hasPlugin("me.tatarka.retrolambda")
                    || project.getPlugins().hasPlugin("dexguard")) {
                getLogger().warn("Jack is disabled, but one of the plugins you "
                        + "are using supports Java 8 language features.");
            } else {
                androidBuilder.getErrorReporter().handleSyncError(
                        variantScope.getVariantConfiguration().getFullName(),
                        SyncIssue.TYPE_JACK_REQUIRED_FOR_JAVA_8_LANGUAGE_FEATURES,
                        "Jack is required to support java 8 language features. "
                                + "Either enable Jack or remove "
                                + "sourceCompatibility JavaVersion.VERSION_1_8."
                );
            }
        }
        addJavacClassesStream(variantScope);
        setJavaCompilerTask(javacTask, tasks, variantScope);
        getAndroidTasks().create(tasks,
                new AndroidJarTask.JarClassesConfigAction(variantScope));
        createPostCompilationTasks(tasks, variantScope);
    }

    // Add data binding tasks if enabled
    if (extension.getDataBinding().isEnabled()) {
        createDataBindingTasks(tasks, variantScope);
    }
    createStripNativeLibraryTask(tasks, variantScope);
    if (variantData.getSplitHandlingPolicy().equals(
            SplitHandlingPolicy.RELEASE_21_AND_AFTER_POLICY)) {
        if (getExtension().getBuildToolsRevision().getMajor() < 21) {
            throw new RuntimeException("Pure splits can only be used with buildtools 21 and later");
        }
        createSplitTasks(tasks, variantScope);
    }
    @Nullable AndroidTask<InstantRunWrapperTask> fullBuildInfoGeneratorTask
                            = createInstantRunPackagingTasks(tasks, variantScope);
    createPackagingTask(tasks, variantScope, true /*publishApk*/, fullBuildInfoGeneratorTask);
    // create the lint tasks.
    createLintTasks(tasks, variantScope);
}
```

### LibraryTaskManager createTasksForVariantData

```java
@Override
public void createTasksForVariantData(
        @NonNull final TaskFactory tasks,
        @NonNull final BaseVariantData<? extends BaseVariantOutputData> variantData) {
    final boolean generateSourcesOnly = AndroidGradleOptions.generateSourcesOnly(project);
    final LibraryVariantData libVariantData = (LibraryVariantData) variantData;
    final GradleVariantConfiguration variantConfig = variantData.getVariantConfiguration();
    final CoreBuildType buildType = variantConfig.getBuildType();
    final VariantScope variantScope = variantData.getScope();
    GlobalScope globalScope = variantScope.getGlobalScope();
    final File intermediatesDir = globalScope.getIntermediatesDir();
    final Collection<String> variantDirectorySegments = variantConfig.getDirectorySegments();
    final File variantBundleDir = variantScope.getBaseBundleDir();
    final String projectPath = project.getPath();
    final String variantName = variantData.getName();
    // 创建一些preXXXBuild任务，generateXXXResources/Assets/Sources任务以及
    // compileXXXSources任务
    createAnchorTasks(tasks, variantScope);
    // Create all current streams (dependencies mostly at this point)
    createDependencyStreams(tasks, variantScope);
    // checkXXXManifest
    createCheckManifestTask(tasks, variantScope);
    // Add a task to create the res values
    createGenerateResValuesTask(tasks, variantScope);
    // Add a task to process the manifest(s)
    createMergeLibManifestsTask(tasks, variantScope);
    // Add a task to compile renderscript files.
    createRenderscriptTask(tasks, variantScope);
    AndroidTask<MergeResources> packageRes = basicCreateMergeResourcesTask(
                                    tasks,
                                    variantScope,
                                    "package",
                                    FileUtils.join(variantBundleDir, "res"),
                                    false /*includeDependencies*/,
                                    false /*process9Patch*/);
     if (variantData.getVariantDependency().hasNonOptionalLibraries()) {
         // Add a task to merge the resource folders, including the libraries, in order to
         // generate the R.txt file with all the symbols, including the ones from
         // the dependencies.
         createMergeResourcesTask(
                 tasks,
                 variantScope,
                 false /*process9patch*/);
     }
     packageRes.configure(tasks,
             new Action<Task>() {
                 @Override
                 public void execute(Task task) {
                     MergeResources mergeResourcesTask = (MergeResources) task;
                     mergeResourcesTask.setPublicFile(FileUtils.join(
                             variantBundleDir,
                             SdkConstants.FN_PUBLIC_TXT // public.txt
                     ));
                 }
             });
    // Add a task to merge the assets folders
    createMergeAssetsTask(tasks, variantScope);
    // Add a task to create the BuildConfig class
    createBuildConfigTask(tasks, variantScope);
    // Add a task to process Res
    createProcessResTask(tasks, variantScope, variantBundleDir,
                            false /*generateResourcePackage*/);
    // process java resources
    createProcessJavaResTasks(tasks, variantScope);
    // Add a task to process Aidl
    createAidlTask(tasks, variantScope);
    createShaderTask(tasks, variantScope);
    // Add a compile task
    AndroidTask<? extends JavaCompile> javacTask =
                            createJavacTask(tasks, variantScope);
    addJavacClassesStream(variantScope);
    TaskManager.setJavaCompilerTask(javacTask, tasks, variantScope);
    // Add data binding tasks if enabled
    if (extension.getDataBinding().isEnabled()) {
        createDataBindingTasks(tasks, variantScope);
    }
    // Add dependencies on NDK tasks if NDK plugin is applied.
    if (!isComponentModelPlugin) {
        // Add NDK tasks
        createNdkTasks(variantScope);
    }
    variantScope.setNdkBuildable(getNdkBuildable(variantData));
    // External native build
    createExternalNativeBuildJsonGenerators(variantScope);
    createExternalNativeBuildTasks(tasks, variantScope);
    // merge jni libs.
    createMergeJniLibFoldersTasks(tasks, variantScope);
    Sync packageRenderscript = project.getTasks().create(
             variantScope.getTaskName("package", "Renderscript"), Sync.class);
     // package from 3 sources. the order is important to make sure the override works well.
     packageRenderscript.from(variantConfig.getRenderscriptSourceList())
             .include("**/*.rsh");
     packageRenderscript.into(new File(variantBundleDir, FD_RENDERSCRIPT));
    // merge consumer proguard files from different build types and flavors
    MergeFileTask mergeProGuardFileTask = project.getTasks().create(
            variantScope.getTaskName("merge", "ProguardFiles"),
            MergeFileTask.class);
    mergeProGuardFileTask.setVariantName(variantConfig.getFullName());
    mergeProGuardFileTask.setInputFiles(
            project.files(variantConfig.getConsumerProguardFiles())
                    .getFiles());
    mergeProGuardFileTask.setOutputFile(
            new File(variantBundleDir, FN_PROGUARD_TXT));
    // copy lint.jar into the bundle folder
    Copy lintCopy = project.getTasks().create(
            variantScope.getTaskName("copy", "Lint"), Copy.class);
    lintCopy.dependsOn(LINT_COMPILE);
    lintCopy.from(new File(
            globalScope.getIntermediatesDir(),
            "lint/lint.jar"));
    lintCopy.into(variantBundleDir);
    final Zip bundle = project.getTasks().create(variantScope.getTaskName("bundle"), Zip.class);
    if (variantData.getVariantDependency().isAnnotationsPresent()) {
        libVariantData.generateAnnotationsTask =
                createExtractAnnotations(project, variantData);
    }
    if (libVariantData.generateAnnotationsTask != null && !generateSourcesOnly) {
        bundle.dependsOn(libVariantData.generateAnnotationsTask);
    }
    final boolean instrumented = variantConfig.getBuildType().isTestCoverageEnabled();
    TransformManager transformManager = variantScope.getTransformManager();
    // ----- Code Coverage first -----
    if (instrumented) {
        createJacocoTransform(tasks, variantScope);
    }
    // ----- External Transforms -----
    // apply all the external transforms.
    List<Transform> customTransforms = extension.getTransforms();
    // 插件中Transform任务
    List<List<Object>> customTransformsDependencies =
        extension.getTransformsDependencies();
    for (int i = 0, count = customTransforms.size() ; i < count ; i++) {
        Transform transform = customTransforms.get(i);
        // Check the transform only applies to supported scopes for libraries:
        // We cannot transform scopes that are not packaged in the library
        // itself.
        Sets.SetView<Scope> difference = Sets.difference(transform.getScopes(),
                TransformManager.SCOPE_FULL_LIBRARY);
        if (!difference.isEmpty()) {
            String scopes = difference.toString();
            androidBuilder.getErrorReporter().handleSyncError(
                    "",
                    SyncIssue.TYPE_GENERIC,
                    String.format("Transforms with scopes '%s' cannot be applied to library projects.",
                            scopes));
        }
        AndroidTask<TransformTask> task = transformManager
                .addTransform(tasks, variantScope, transform);
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
    if (buildType.isMinifyEnabled()) {
        createMinifyTransform(tasks, variantScope, false);
    }
    // now add a transform that will take all the class/res and package them
    // into the main and secondary jar files.
    // This transform technically does not use its transform output, butthat's
    // ok. We use the transform mechanism to get incremental data from
    // the streams.
    String packageName = variantConfig.getPackageFromManifest();
    if (packageName == null) {
        throw new BuildException("Failed to read manifest", null);
    }
    LibraryJarTransform transform = new LibraryJarTransform(
            new File(variantBundleDir, FN_CLASSES_JAR),
            new File(variantBundleDir, LIBS_FOLDER),
            variantScope.getTypedefFile(),
            packageName,
            getExtension().getPackageBuildConfig());
    excludeDataBindingClassesIfNecessary(variantScope, transform);
    AndroidTask<TransformTask> jarPackagingTask = transformManager
            .addTransform(tasks, variantScope, transform);
    if (!generateSourcesOnly) {
        bundle.dependsOn(jarPackagingTask.getName());
    }
    // now add a transform that will take all the native libs and package
    // them into the libs folder of the bundle.
    LibraryJniLibsTransform jniTransform = new LibraryJniLibsTransform(
            new File(variantBundleDir, FD_JNI));
    AndroidTask<TransformTask> jniPackagingTask = transformManager
            .addTransform(tasks, variantScope, jniTransform);
    if (!generateSourcesOnly) {
        bundle.dependsOn(jniPackagingTask.getName());
    }

    bundle.dependsOn(
        packageRes.getName(),
        packageRenderscript,
        lintCopy,
        mergeProGuardFileTask,
        // The below dependencies are redundant in a normal build as
        // generateSources depends on them. When generateSourcesOnly iinjected they are
        // needed explicitly, as bundle no longer depends on compileJava
        variantScope.getAidlCompileTask().getName(),
        variantScope.getMergeAssetsTask().getName(),
        variantData.getOutputs().get(0).getScope().getManifestProcessorTask.getName());
    if (!generateSourcesOnly) {
        bundle.dependsOn(variantScope.getNdkBuildable());
    }
    bundle.setDescription("Assembles a bundle containing the library in " +
            variantConfig.getFullName() + ".");
    bundle.setDestinationDir(
            new File(globalScope.getOutputsDir(), BuilderConstants.EXT_LIB_ARCHIVE));
    bundle.setArchiveName(globalScope.getProjectBaseName()
            + "-" + variantConfig.getBaseName()
            + "." + BuilderConstants.EXT_LIB_ARCHIVE);
    bundle.setExtension(BuilderConstants.EXT_LIB_ARCHIVE);
    bundle.from(variantBundleDir);
    bundle.from(FileUtils.join(intermediatesDir,
            StringHelper.toStrings(ANNOTATIONS, variantDirectorySegments)));
    // get the single output for now, though that may always be the case for a library.
    LibVariantOutputData variantOutputData = libVariantData.getOutputs().get(0);
    variantOutputData.packageLibTask = bundle;
    variantScope.getAssembleTask().dependsOn(tasks, bundle);
    variantOutputData.getScope().setAssembleTask(variantScope.getAssembleTask());
    variantOutputData.assembleTask = variantData.assembleVariantTask;
    if (getExtension().getDefaultPublishConfig().equals(variantConfig.getFullName())) {
        VariantHelper.setupDefaultConfig(project,
                variantData.getVariantDependency().getPackageConfiguration());
        // add the artifact that will be published
        project.getArtifacts().add("default", bundle);
        getAssembleDefault().dependsOn(variantScope.getAssembleTask().getName());
    }
    // also publish the artifact with its full config name
    if (getExtension().getPublishNonDefault()) {
        project.getArtifacts().add(
                variantData.getVariantDependency().getPublishConfiguration().getName(), bundle);
        bundle.setClassifier(
                variantData.getVariantDependency().getPublishConfiguration().getName());
    }
    // configure the variant to be testable.
    variantConfig.setOutput(new LocalTestedAarLibrary(
            bundle.getArchivePath(),
            variantBundleDir,
            variantData.getName(),
            project.getPath(),
            variantData.getName(),
            false /*isProvided*/));
   createLintTasks(tasks, variantScope);
}
```
