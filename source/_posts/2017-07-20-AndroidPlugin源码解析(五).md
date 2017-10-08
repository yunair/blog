---
title: AndroidPlugin源码解析(五)
date: 2017-07-20
tags: AndroidPlugin
---


上篇提到，我们这篇研究的是接下来两个过程, 分别是:

- createGenerateResValuesTask
- createMergeResourcesTask
- createMergeAssetsTask

接下来，我们一个一个来看

### createGenerateResValuesTask

这个task实际执行下面的方法：

~~~ java
// 该folder名字为resValues
File folder = getResOutputDir();
List<Object> resolvedItems = getItems();

if (resolvedItems.isEmpty()) {
    FileUtils.cleanOutputDir(folder);
} else {
    ResValueGenerator generator = new ResValueGenerator(folder);
    generator.addItems(getItems());

    generator.generate();
}
~~~

这个任务的代码看起来很简单，主要就是else内的内容:

我们来看下`generate`方法:

#### ResValueGenerator generate

~~~ java
public void generate() throws IOException, ParserConfigurationException {
    File pkgFolder = getFolderPath();
    if (!pkgFolder.isDirectory()) {
        if (!pkgFolder.mkdirs()) {
            throw new RuntimeException("Failed to create " + pkgFolder.getAbsolutePath());
        }
    }

    File resFile = new File(pkgFolder, "generated.xml");

    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(true);
    factory.setValidating(false);
    factory.setIgnoringComments(true);
    DocumentBuilder builder;

    builder = factory.newDocumentBuilder();
    Document document = builder.newDocument();

    Node rootNode = document.createElement(TAG_RESOURCES);
    document.appendChild(rootNode);

    rootNode.appendChild(document.createTextNode("\n"));
    rootNode.appendChild(document.createComment("Automatically generated file. DO NOT MODIFY"));
    rootNode.appendChild(document.createTextNode("\n\n"));

    for (Object item : mItems) {
        if (item instanceof ClassField) {
            ClassField field = (ClassField)item;

            ResourceType type = ResourceType.getEnum(field.getType());
            boolean hasResourceTag = (type != null && RESOURCES_WITH_TAGS.contains(type));

            Node itemNode = document.createElement(hasResourceTag ? field.getType() : TAG_ITEM);
            Attr nameAttr = document.createAttribute(ATTR_NAME);

            nameAttr.setValue(field.getName());
            itemNode.getAttributes().setNamedItem(nameAttr);

            if (!hasResourceTag) {
                Attr typeAttr = document.createAttribute(ATTR_TYPE);
                typeAttr.setValue(field.getType());
                itemNode.getAttributes().setNamedItem(typeAttr);
            }
            // 加入 translatable="false"
            if (type == ResourceType.STRING) {
                Attr translatable = document.createAttribute(ATTR_TRANSLATABLE);
                translatable.setValue(VALUE_FALSE);
                itemNode.getAttributes().setNamedItem(translatable);
            }

            if (!field.getValue().isEmpty()) {
                itemNode.appendChild(document.createTextNode(field.getValue()));
            }

            rootNode.appendChild(itemNode);
        } else if (item instanceof String) {
            rootNode.appendChild(document.createTextNode("\n"));
            rootNode.appendChild(document.createComment((String) item));
            rootNode.appendChild(document.createTextNode("\n"));
        }
    }

    String content;
    try {
        content = XmlPrettyPrinter.prettyPrint(document, true);
    } catch (Throwable t) {
        content = XmlUtils.toXml(document);
    }

    Files.write(content, resFile, Charsets.UTF_8);
}
~~~

这里就是在gradle的buildType中定义的resValue，最终会在`build/generated/res/resValues/${buildTypes}/values`目录下生成`generated.xml`文件，

这样，在buildType中定义的resValue就可以在代码中使用了。我在这里通常就是定义一些string，没用过其他resValue类型，所以这里就不详细分析了。

通过`generated.xml`文件来对应这个代码，理解起来很容易。

### createMergeResourcesTask

这个task实际执行下面的方法：

~~~ java
// 预处理VectorDrawable，如果没有，则忽略
ResourcePreprocessor preprocessor = getPreprocessor();
// this is full run, clean the previous output
File destinationDir = getOutputDir();
FileUtils.cleanOutputDir(destinationDir);

List<ResourceSet> resourceSets = getConfiguredResourceSets(preprocessor);

// create a new merger and populate it with the sets.
ResourceMerger merger = new ResourceMerger(minSdk);

// 这个resourceSets的来源是VariantConfiguration的getResourceSets方法
// 这个方法添加资源的顺序，决定了最后merge的时候保留的资源
for (ResourceSet resourceSet : resourceSets) {
    resourceSet.loadFromFiles(getILogger());
    merger.addDataSet(resourceSet);
}

// get the merged set and write it down.
// 当你使用的build tools版本是24.0.0 rc2的之后，可以在gradle.properties中设置android.enableAapt2=true属性使用aapt2
// 但是，不建议你在gradle-plugin:3.0.0-beta1之前版本使用，会有各种问题，那么我们这里关注Aapt1，而不是2版本。
// 当前版本的Aapt2的实现中存在各种Fix的注释
Aapt aapt =
        AaptGradleFactory.make(
                getBuilder(),
                getCrunchPng(),
                getProcess9Patch(),
                variantScope,
                getAaptTempDir());
MergedResourceWriter writer = new MergedResourceWriter(
        destinationDir,
        getPublicFile(),
        getBlameLogFolder(),
        preprocessor,
        aapt::compile,  // 处理.9png
        getIncrementalFolder());

merger.mergeData(writer, false /*doCleanUp*/);

// No exception? Write the known state.
merger.writeBlobTo(getIncrementalFolder(), writer, false);
~~~

这里主要解释两个方法，即`mergedResourceWriter`的两个方法`mergeDta`和`writeBlobTo`

剩下的看代码中的注释即可。

#### MergedResourceWriter mergeDta

~~~ java
public void mergeData(@NonNull MergeConsumer<I> consumer, boolean doCleanUp)
        throws MergingException {
    // 对于MergeResources这个consumer来说，这就是一个赋值操作
    consumer.start(mFactory);

    try {
        // get all the items keys.
        Set<String> dataItemKeys = Sets.newHashSet();

        for (S dataSet : mDataSets) {
            // quick check on duplicates in the resource set.
            dataSet.checkItems();
            ListMultimap<String, I> map = dataSet.getDataMap();
            dataItemKeys.addAll(map.keySet());
        }

        // loop on all the data items.
        for (String dataItemKey : dataItemKeys) {
            // 对于Resources来说，如果存在declare-styleable则需要merge
            if (requiresMerge(dataItemKey)) {
                // get all the available items, from the lower priority, to the higher
                // priority
                List<I> items = Lists.newArrayListWithExpectedSize(mDataSets.size());
                for (S dataSet : mDataSets) {

                    // look for the resource key in the set
                    ListMultimap<String, I> itemMap = dataSet.getDataMap();

                    List<I> setItems = itemMap.get(dataItemKey);
                    items.addAll(setItems);
                }

                mergeItems(dataItemKey, items, consumer);
                continue;
            }

            // for each items, look in the data sets, starting from the end of the list.

            I previouslyWritten = null;
            I toWrite = null;

            /*
             * We are looking for what to write/delete: the last non deleted item, and the
             * previously written one.
             */

            boolean foundIgnoredItem = false;

            setLoop: for (int i = mDataSets.size() - 1 ; i >= 0 ; i--) {
                S dataSet = mDataSets.get(i);

                // look for the resource key in the set
                ListMultimap<String, I> itemMap = dataSet.getDataMap();

                List<I> items = itemMap.get(dataItemKey);
                if (items.isEmpty()) {
                    continue;
                }

                // The list can contain at max 2 items. One touched and one deleted.
                // More than one deleted means there was more than one which isn't possible
                // More than one touched means there is more than one and this isn't possible.
                for (int ii = items.size() - 1 ; ii >= 0 ; ii--) {
                    I item = items.get(ii);

                    if (consumer.ignoreItemInMerge(item)) {
                        foundIgnoredItem = true;
                        continue;
                    }

                    if (item.isWritten()) {
                        assert previouslyWritten == null;
                        previouslyWritten = item;
                    }

                    if (toWrite == null && !item.isRemoved()) {
                        toWrite = item;
                    }

                    if (toWrite != null && previouslyWritten != null) {
                        break setLoop;
                    }
                }
            }

            // done searching, we should at least have something, unless we only
            // found items that are not meant to be written (attr inside declare styleable)
            assert foundIgnoredItem || previouslyWritten != null || toWrite != null;

            if (toWrite != null && !filterAccept(toWrite)) {
                toWrite = null;
            }


            //noinspection ConstantConditions
            if (previouslyWritten == null && toWrite == null) {
                continue;
            }

            // now need to handle, the type of each (single res file, multi res file), whether
            // they are the same object or not, whether the previously written object was
            // deleted.

            if (toWrite == null) {
                // nothing to write? delete only then.
                assert previouslyWritten.isRemoved();

                consumer.removeItem(previouslyWritten, null /*replacedBy*/);

            } else if (previouslyWritten == null || previouslyWritten == toWrite) {
                // easy one: new or updated res
                consumer.addItem(toWrite);
            } else {
                // replacement of a resource by another.

                // force write the new value
                toWrite.setTouched();
                consumer.addItem(toWrite);
                // and remove the old one
                consumer.removeItem(previouslyWritten, toWrite);
            }
        }
    } finally {
        consumer.end();
    }

    if (doCleanUp) {
        // reset all states. We can't just reset the toWrite and previouslyWritten objects
        // since overlayed items might have been touched as well.
        // Should also clean (remove) objects that are removed.
        postMergeCleanUp();
    }
}
~~~


虽然看起来代码有点复杂，但是完成的任务挺清晰的，就是完成`ResourceItem`对象的add和merge操作，然后将`ResourceItem`对象添加到consumer内。

在`consumer.end()`的时候把这些ResourceItem对象写入`/build/intermediates/merge${BuildType}Resources/`下面的所有文件，也就是我们在工程中定义的values文件夹之内的那些文件。然后，将cache和真正的value一一对应，并写入compile-file-map.properties文件中。
</br>

接下来看这个任务中最后一个方法

#### MergedResourceWriter writeBlobTo

~~~ java
public void writeBlobTo(@NonNull File blobRootFolder, @NonNull MergeConsumer<I> consumer, boolean includeTimestamps)
        throws MergingException {
    // write "compact" blob
    DocumentBuilder builder;

    builder = mFactory.newDocumentBuilder();
    Document document = builder.newDocument();

    Node rootNode = document.createElement(NODE_MERGER);
    // add the version code.
    NodeUtils.addAttribute(document, rootNode, null, ATTR_VERSION, MERGE_BLOB_VERSION);

    document.appendChild(rootNode);

    for (S dataSet : mDataSets) {
        Node dataSetNode = document.createElement(NODE_DATA_SET);
        rootNode.appendChild(dataSetNode);
        dataSet.appendToXml(dataSetNode, document, consumer, includeTimestamps);
    }

    // write merged items
    // 创建<mergedItems><configuration><xxx></xxx></configuration></mergedItems>
    // 这里是说明merge之后使用哪个资源文件的地方，如果发现merge之后图片不对，来看一下
    writeAdditionalData(document, rootNode);

    String content = XmlUtils.toXml(document);

    try {
        createDir(blobRootFolder);
    } catch (IOException ioe) {
        throw MergingException.wrapException(ioe).withFile(blobRootFolder).build();
    }
    File file = new File(blobRootFolder, FN_MERGER_XML);
    try {
        Files.write(content, file, Charsets.UTF_8);
    } catch (IOException ioe) {
        throw MergingException.wrapException(ioe).withFile(file).build();
    }
}
~~~

这个方法就是将dataSet转化成一个xml文件，这个xml文件路径是`build/intermediates/incremental/merge${buildTypes}Resources/merger.xml`

最重要的两个方法是：

`dataSet.appendToXml(dataSetNode, document, consumer, includeTimestamps);`和`writeAdditionalData(document, rootNode);`

这里就不继续看内部的代码了，简单说一下这两个方法的作用，

`dataSet.appendToXml(dataSetNode, document, consumer, includeTimestamps);`方法的作用是:

按顺序创建`<dataSet><source><file><xxx></xxx></file></source></dataSet>`这样的xml，可以在`merger.xml`中观察到。
</br>

`writeAdditionalData(document, rootNode);`方法的作用是:

创建`<mergedItems><configuration><xxx></xxx></configuration></mergedItems>`这样的xml

这里是说明merge之后使用哪个资源文件的地方，可以在`merger.xml`中观察到。
</br>

这样，整MergeResourcesTask的过程就讲完了

### createMergeAssetsTask

这个task实际执行下面的方法：

~~~ java
// this is full run, clean the previous output
File destinationDir = getOutputDir();
FileUtils.cleanOutputDir(destinationDir);

List<AssetSet> assetSets = getInputDirectorySets();

// create a new merger and populate it with the sets.
AssetMerger merger = new AssetMerger();

// AssetSet 和ResourceSet一样，都是DataSet的子类
// 所以，做的事和上面的createMergeResourcesTask方法中类似，不过一个是resourceSet对象，一个是AssetSet对象
for (AssetSet assetSet : assetSets) {
    // set needs to be loaded.
    assetSet.loadFromFiles(getILogger());
    merger.addDataSet(assetSet);
}

// get the merged set and write it down.
MergedAssetWriter writer = new MergedAssetWriter(destinationDir);

merger.mergeData(writer, false /*doCleanUp*/);

// No exception? Write the known state.
merger.writeBlobTo(getIncrementalFolder(), writer, false);
~~~

大概浏览一下，会不会觉得这个方法和`createMergeResourcesTask`中的方法很相似。那么，理解起来应该就相对容易一些了。
同样的代码，不同子类的实现。
</br>

先讲一下`mergeData`，对于这个方法来说，MergedAssetWriter中`requiresMerge`直接返回false，也就是不能merge。
MergedAssetWriter中`end`方法什么也不做。
</br>

这里的`addItem`方法需要讲解一下，否则，看其他库中出现gzip包就不会太理解，比如okhttp这个库。

这里，会直接把gzip压缩过的文件还原为原始文件，同时会将该文件重命名为去掉.gz的文件名 // e.g. foo.txt.gz to foo.txt
</br>

至于`dataSet.appendToXml(dataSetNode, document, consumer, includeTimestamps);`方法，作用和上面一样，在不同的文件夹下生成而已。

`writeAdditionalData()`方法就是啥也不做。
</br>

这样，整MergeAssetsTask的过程就讲完了。下一篇，我们继续
`createBuildConfigTask`, `createApkProcessResTask`和`createProcessJavaResTasks`的过程。
</br>

如果大家有不懂，欢迎通过留言和邮件进行交流。