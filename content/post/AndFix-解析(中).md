---
title: AndFix解析-(中)
date: 2015-10-10
tags: ["AndFix"]
---

我们接着分析阿里开源的 AndFix 库，上次留下了三个坑，一个方法，两个类，不知道你们是否想急切了解呢？
`loadPatch()`方法和`AndFixManager`和`Patch`类。

分析`loadPatch()`方法的时候离不开`AndFixManager`这个类，所以，我会在分析`loadPatch()`方法的时候分析`AndFixManager`这个类。
`Patch`类相当于一个容器，把修复 bug 所需的信息放在其中，`Patch`类相对来说比较独立，不需要牵扯到另外两个坑，所以，就先把这个坑埋了。

要分析`Patch`类，就不能不分析阿里提供的打包.apatch 的工具`apkpatch-1.0.2.jar`，`Patch`获取的信息其实就是`apkpatch`打包时放入其中的信息。
我下面对`apkpatch.jar`的分析以这个版本的源码为准。

分析源码，那些 try-catch-finally 和逻辑无关，所以，我会把这些代码从源码中删掉的，除非有提示用户的操作

我们先来看看`Patch`类

### class Patch

从前一篇的分析中，我们看到调用了`Patch`的构造函数，那我们就从构造函数开始看。

#### Patch Constructor

```java
public Patch(File file) throws IOException {
    this.mFile = file;
    this.init();
}
```

#### Patch init()

将传入的文件保存在类变量中，并调用`init()`函数，那么`init()`函数干什么呢？
代码里异常和清理的代码我给删掉了，毕竟，这和我们对源码的分析关系不大，那么我们看一下剩下的代码

```java
public void init(){
    JarFile jarFile = new JarFile(this.mFile);
    JarEntry entry = jarFile.getJarEntry(ENTRY_NAME);
    InputStream inputStream = jarFile.getInputStream(entry);
    Manifest manifest = new Manifest(inputStream);
    Attributes main = manifest.getMainAttributes();
    this.mName = main.getValue(PATCH_NAME);
    this.mTime = new Date(main.getValue(CREATED_TIME));
    this.mClassesMap = new HashMap();
    Iterator it = main.keySet().iterator();

    while(it.hasNext()) {
        Name attrName = (Name)it.next();
        String name = attrName.toString();
        if(name.endsWith(CLASSES)) {
            List strings = Arrays.asList(main.getValue(attrName).split(","));
            if(name.equalsIgnoreCase(PATCH_CLASSES)) {
                this.mClassesMap.put(this.mName, strings);
            } else {
                this.mClassesMap.put(name.trim().substring(0, name.length() - 8), strings);
            }
        }
    }
}
```

先分析上面的代码，通过 JarFile, JarEntry, Manifest, Attributes 一层层的获取 jar 文件中值，并放入对应的类变量中
这些值是从哪里来的呢？是从生成这个.apatch 文件的时候写入的，即`apkpatch.jar`文件中写入的。
虽然这个文件后缀名是 apatch，不是 jar，但是它生成的时候是使用 Attributes，Manifest，jarEntry 将数据写入的，
其实依旧是 jar 格式，修改了扩展名而已

上面的那些常量都是什么呢，我们来看看这个类的类变量，就是一些对应的字符串。

```java
private static final String ENTRY_NAME = "META-INF/PATCH.MF";
private static final String CLASSES = "-Classes";
private static final String PATCH_CLASSES = "Patch-Classes";
private static final String CREATED_TIME = "Created-Time";
private static final String PATCH_NAME = "Patch-Name";
private final File mFile;
private String mName;
private Date mTime;
private Map<String, List<String>> mClassesMap;
```

看完了 Patch 类，是不是还一头雾水呢, 其他的好理解， mClassesMap 里面的各种类名是从哪里来，里面的类作用又是什么呢？
这里，我们就需要进入`apkpatch.jar`一探究竟。

### apkpatch.jar

对于 Jar 文件，我们可以使用`jd-gui`文件打开，阿里这里并没有对这个 jar 文件加密，所以我们可以直接看，对于 Jar 文件，我们从`Main()`方法开始看起。
在`package com.euler.patch;`里面，有一个`Main`类，其中有那个我们常见的`main()`方法，那么我们就进入看一看。
代码比较长，我们一段一段来看

#### Main Main()

```java
//CommandLineParser，CommandLine， Option, Options，OptionBuilder,  PosixParser等等类都是
//org.apache.commons.cli这个包中的类，负责解析命令行传给程序的参数，所以下面的很多内容应该就不用我过多解释了
CommandLineParser parser = new PosixParser();
CommandLine commandLine = null;
//option()方法就是将命令行需要解析的参数加入其中，有三种解析方式
//private static final Options allOptions = new Options();
//private static final Options patchOptions = new Options();
//private static final Options mergeOptions = new Options();
option();
try {
  commandLine = parser.parse(allOptions, args);
} catch (ParseException e) {
  System.err.println(e.getMessage());
  //提示用户如何使用这个jar包
  usage(commandLine);
  return;
}

if ((!commandLine.hasOption('k')) && (!commandLine.hasOption("keystore"))) {
  usage(commandLine);
  return;
}
if ((!commandLine.hasOption('p')) && (!commandLine.hasOption("kpassword"))) {
  usage(commandLine);
  return;
}
if ((!commandLine.hasOption('a')) && (!commandLine.hasOption("alias"))) {
  usage(commandLine);
  return;
}
if ((!commandLine.hasOption('e')) && (!commandLine.hasOption("epassword"))) {
  usage(commandLine);
  return;
}
```

这一部分主要是查看命令行是否有那些参数，如果没有要求的参数，则提示用户需要哪些参数。

接下来注意一下你下载`apkpatch`这个文件的时间，根据 github 的历史提交记录，如果你 2015 年 10 月以后下载的，可以忽视掉这个提醒，如果你是 10 月之前下载的那个工具，建议你更新一下这个工具，否则可能会导致`-o`所在的文件夹被清空。

好了，提醒完了，我们继续

```java
String keystore = commandLine.getOptionValue('k');
String password = commandLine.getOptionValue('p');
String alias = commandLine.getOptionValue('a');
String entry = commandLine.getOptionValue('e');
String name = "main";
if ((commandLine.hasOption('n')) || (commandLine.hasOption("name"))){
  name = commandLine.getOptionValue('n');
}
```

对于我们的 release 版应用来说，我们会指定`storeFile， storePassword, keyAlias, keyPassword`,
这些即是上面`keystore, password, alias, entry`对应的值。至于 name，这是最后生成的文件的名称，基本可以不用管。

下面，我们看一下`main()`函数中最后的一个 if-else

```java
if ((commandLine.hasOption('m')) || (commandLine.hasOption("merge"))) {
    String[] merges = commandLine.getOptionValues('m');
    File[] files = new File[merges.length];
    for (int i = 0; i < merges.length; i++) {
      files[i] = new File(merges[i]);
    }
    MergePatch mergePatch = new MergePatch(files, name, out, keystore,
      password, alias, entry);
    mergePatch.doMerge();
} else {
    if ((!commandLine.hasOption('f')) && (!commandLine.hasOption("from"))) {
      usage(commandLine);
      return;
    }
    if ((!commandLine.hasOption('t')) && (!commandLine.hasOption("to"))) {
      usage(commandLine);
      return;
    }

    File from = new File(commandLine.getOptionValue("f"));
    File to = new File(commandLine.getOptionValue('t'));
    if ((!commandLine.hasOption('n')) && (!commandLine.hasOption("name"))) {
      name = from.getName().split("\\.")[0];
}
```

在此，我选择不分析`MergePatch`这个部分，毕竟，你只有先生成了，后面才可能需要做 merge 操作。
from 指修复 bug 后的 apk，to 指之前有 bug 的 apk。所以这部分的分析就结束了。

`main()`函数里最后一部分内容

```java
ApkPatch apkPatch = new ApkPatch(from, to, name, out, keystore,
  password, alias, entry);
apkPatch.doPatch();
```

将那些参数传给 ApkPatch 去初始化，然后调用`doPatch()`方法

#### ApkPatch

##### ApkPatch Constructor

我们一步步来看，首先是调用构造函数

```java
public ApkPatch(File from, File to, String name, File out, String keystore, String password, String alias, String entry){
    super(name, out, keystore, password, alias, entry);
    this.from = from;
    this.to = to;
}
```

这些代码大家都经常写了，我们跟着进入看看父类的构造函数做了什么，

#### Build

##### Build Constructor

`public class ApkPatch extends Build` ApkPatch 的函数头，说明它是`Build`类的子类

```java
public Build(String name, File out, String keystore, String password, String alias, String entry)
{
    this.name = name;
    this.out = out;
    this.keystore = keystore;
    this.password = password;
    this.alias = alias;
    this.entry = entry;
    if (!out.exists())
      out.mkdirs();
    else if (!out.isDirectory())
      throw new RuntimeException("output path must be directory.");
    try{
      FileUtils.cleanDirectory(out);
    } catch (IOException e) {
      throw new RuntimeException(e);
    }
}
```

在这里，看到了我之前提到的`-o`操作的问题，会清空那个文件夹内的内容。

好了，我们接下来就可以看看最后一个方法，`doPatch()`方法了。

##### ApkPatch doPatch()

```java
public void doPatch() {
    File smaliDir = new File(this.out, "smali");
    smaliDir.mkdir();

    File dexFile = new File(this.out, "diff.dex");
    File outFile = new File(this.out, "diff.apatch");
    //DiffInfo是一个容器类，主要保存了新加的和修改的 类，方法和字段。
    DiffInfo info = new DexDiffer().diff(this.from, this.to);

    this.classes = buildCode(smaliDir, dexFile, info);

    build(outFile, dexFile);

    release(this.out, dexFile, outFile);
}
```

在 out 文件夹内生成了一个`smali`文件夹，还有`diff.dex`, `diff.apatch`文件。
看到 diff()方法，应该能想到就是比较两个文件的不同，所以 DiffInfo 就是储存两个文件不同的一个容器类，
由于篇幅原因，这里就不深入其中了，有兴趣的同学可以深入其中看一下。

但是，在这个`diff()`方法中，有一个重要的问题需要大家注意，就是其中只针对`classes.dex`做了 diff，如果你使用了 Google 的 Multidex，那么结果就是你其它 dex 文件中的任何 bug，依旧无法修复，因为这个生成的`DiffInfo`中没有其它 dex 的信息，这个时候就需要大家使用 JavaAssist 之类的工具修改阿里的这个 jar 文件，然后达到你自己的修复非`classes.dex`文件中 bug 的目的，问题出在`DexFileFactory.class`类中。

接下来，我们可以看到调用了三个方法，`buildCode(smaliDir, dexFile, info);`
` build(outFile, dexFile);``release(this.out, dexFile, outFile); `
一个个的跟进去看一看。

##### ApkPatch buildCode(smaliDir, dexFile, info)

```java
private static Set<String> buildCode(File smaliDir, File dexFile, DiffInfo info)
    throws IOException, RecognitionException, FileNotFoundException{
    Set classes = new HashSet();
    Set list = new HashSet();
    //从这里可以看出，list保存了DiffInfo容器中的新添加的classes和被修改过的classes
    list.addAll(info.getAddedClasses());
    list.addAll(info.getModifiedClasses());

    baksmaliOptions options = new baksmaliOptions();

    options.deodex = false;
    options.noParameterRegisters = false;
    options.useLocalsDirective = true;
    options.useSequentialLabels = true;
    options.outputDebugInfo = true;
    options.addCodeOffsets = false;
    options.jobs = -1;
    options.noAccessorComments = false;
    options.registerInfo = 0;
    options.ignoreErrors = false;
    options.inlineResolver = null;
    options.checkPackagePrivateAccess = false;
    if (!options.noAccessorComments) {
      options.syntheticAccessorResolver = new SyntheticAccessorResolver(list);
    }
    ClassFileNameHandler outFileNameHandler = new ClassFileNameHandler(
      smaliDir, ".smali");
    ClassFileNameHandler inFileNameHandler = new ClassFileNameHandler(
      smaliDir, ".smali");
    DexBuilder dexBuilder = DexBuilder.makeDexBuilder();

    for (DexBackedClassDef classDef : list) {
      String className = classDef.getType();
      //将相关的类信息写入outFileNameHandler
      baksmali.disassembleClass(classDef, outFileNameHandler, options);
      File smaliFile = inFileNameHandler.getUniqueFilenameForClass(
        TypeGenUtil.newType(className));
      classes.add(TypeGenUtil.newType(className)
        .substring(1, TypeGenUtil.newType(className).length() - 1)
        .replace('/', '.'));
      SmaliMod.assembleSmaliFile(smaliFile, dexBuilder, true, true);
    }
    dexBuilder.writeTo(new FileDataStore(dexFile));

    return classes;
}
```

看到 baksmali，反编译过 apk 的同学一定不陌生，这就是 dex 的打包工具，还有对应的解包工具 smali，就到这里，这方面不继续深入了。
如果想深入了解 dex 打包解包工具的源码，参见[This Blog][1]

可以看到，这个方法的返回值将 DiffInfo 中新添加的 classes 和修改过的 classes 做了一个重命名，然后保存了起来，同时，将相关内容写入 smali 文件中。
这个重命名是一个怎样的重命名呢，看一下生成的 smali 文件夹里任意一个 smali 文件，我就拿 Demo 里的`MainActivity.smali`来说明
这个类在文件中的类名是`Lcom/euler/andfix/MainActivity;`，看到这个名字，再看下面这个方法就很清晰了

```java
classes.add(TypeGenUtil.newType(className)
.substring(1, TypeGenUtil.newType(className).length() - 1)
.replace('/', '.'));
```

就是把`Lcom/euler/andfix/MainActivity;`替换成`com.euler.andfix.MainActivity_CF;`
那个\_CF 哪里来的呢，就是那个`TypeGenUtil`类做的操作了。
这个类就一个方法，该方法进对 String 进行了操作，我们来看一看源码

```java
public static String newType(String type){
    return type.substring(0, type.length() - 1) + "_CF;";
}
```

可以看到，去掉了类名多余的那个';'，然后在后面加了个\_CF 后缀。重命名应该是为了不合之前安装的 dex 文件的名字冲突。

(new FileDataStore(file) 作用就是清空 file 里的内容)
最后，将 dexFile 文件清空，把 dexBuilder 的内容写入其中。

到这里，`buildCode(smaliDir, dexFile, info)`方法就结束了, 看看下一个方法`build(outFile, dexFile);`

##### ApkPatch build(outFile, dexFile)

上一个方法已经把内容填充到 dexFile 内了，我们来看看它的源码。

```java
protected void build(File outFile, File dexFile)
    throws KeyStoreException, FileNotFoundException, IOException, NoSuchAlgorithmException, CertificateException, UnrecoverableEntryException{
    //获取应用签名相关信息
    KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
    KeyStore.PrivateKeyEntry privateKeyEntry = null;
    InputStream is = new FileInputStream(this.keystore);
    keyStore.load(is, this.password.toCharArray());
    privateKeyEntry = (KeyStore.PrivateKeyEntry)keyStore.getEntry(this.alias,
      new KeyStore.PasswordProtection(this.entry.toCharArray()));

    PatchBuilder builder = new PatchBuilder(outFile, dexFile,
      privateKeyEntry, System.out);

    //将getMeta()中获取的Manifest内容写入"META-INF/PATCH.MF"文件中
    builder.writeMeta(getMeta());

    //单纯调用了close()命令, 将异常处理放在子函数中。
    //这里做了一个异常分层，即不同抽象层次的子函数需要处理的异常不一样
    //具体请参阅《Code Complete》(代码大全)在恰当的抽象层次抛出异常
    builder.sealPatch();
}
```

要打包了，打包完是要签名的，所以需要获取签名的相关信息，这一部分就不详细讲解了。
这里提一下`getMeta()`函数，因为开头我们提到的`Patch` `init()` 函数内
这句`List strings = Arrays.asList(main.getValue(attrName).split(","));`中的那个 strings 就是从这里写入的

```java
 protected Manifest getMeta()
{
    Manifest manifest = new Manifest();
    Attributes main = manifest.getMainAttributes();
    main.putValue("Manifest-Version", "1.0");
    main.putValue("Created-By", "1.0 (ApkPatch)");
    main.putValue("Created-Time",
      new Date(System.currentTimeMillis()).toGMTString());
    main.putValue("From-File", this.from.getName());
    main.putValue("To-File", this.to.getName());
    main.putValue("Patch-Name", this.name);
    main.putValue("Patch-Classes", Formater.dotStringList(this.classes));
    return manifest;
}
```

那个 classes 就是我们那个将类名修改后保存起来的`Set`，
`Formater.dotStringList(this.classes));`这个方法就是遍历`Set`，并将其中用`','`作为分隔符，将整个`Set`中保存的`String`拼接
所以在`Patch` `init()`方法内，就可以反过来组成一个`List`。

那么，我们来看看中间那条没注释的语句，通过`PatchBuilder()`的构造函数生成`PatchBuilder`，我们来看看`PatchBuilder`的构造函数

##### PatchBuilder Constructor

```java
public PatchBuilder(File outFile, File dexFile, KeyStore.PrivateKeyEntry key, PrintStream verboseStream){
    this.mBuilder = new SignedJarBuilder(new FileOutputStream(outFile, false), key.getPrivateKey(),
        (X509Certificate)key.getCertificate());
    this.mBuilder.writeFile(dexFile, "classes.dex");
}
```

这个就是把 dexFile 里的内容，经过修改，加上签名，写入 classes.dex 文件中，但是这里实在看不出来其中的细节，
所以，我们要进入`SignedJarBuilder`类一探究竟

先看它的构造函数。

##### SignedJarBuilder Constructor

```java
public SignedJarBuilder(OutputStream out, PrivateKey key, X509Certificate certificate)
    throws IOException, NoSuchAlgorithmException{
    this.mOutputJar = new JarOutputStream(new BufferedOutputStream(out));
    //设置压缩级别
    this.mOutputJar.setLevel(9);
    this.mKey = key;
    this.mCertificate = certificate;
    if ((this.mKey != null) && (this.mCertificate != null)) {
      this.mManifest = new Manifest();
      Attributes main = this.mManifest.getMainAttributes();
      main.putValue("Manifest-Version", "1.0");
      main.putValue("Created-By", "1.0 (ApkPatch)");
      this.mBase64Encoder = new BASE64Encoder();
      this.mMessageDigest = MessageDigest.getInstance("SHA1");
    }
}
```

这个方法做了一些初始化，做了一些赋值操作，大家经常写类似的代码，就不进行分析了，只是为了让大家看`writeFile(File, String)`的时候容易理解一些。

##### SignedJarBuilder writeFile(File, String)

`writeFile(File, String)`做了什么呢？

```java
public void writeFile(File inputFile, String jarPath){
    FileInputStream fis = new FileInputStream(inputFile);
    JarEntry entry = new JarEntry(jarPath);
    entry.setTime(inputFile.lastModified());
    writeEntry(fis, entry);
}
```

这个方法调用了`writeEntry(InputStream, JarEntry)`方法, 来看一看它的源码

##### SignedJarBuilder writeEntry(InputStream, JarEntry)

```java
private void writeEntry(InputStream input, JarEntry entry)
  throws IOException{
    this.mOutputJar.putNextEntry(entry);
    int count;
    while ((count = input.read(this.mBuffer)) != -1)
    {
      int count;
      this.mOutputJar.write(this.mBuffer, 0, count);

      if (this.mMessageDigest != null) {
        //将mBuffer中的内容放入mMessageDigest内部数组
        this.mMessageDigest.update(this.mBuffer, 0, count);
      }
    }

    this.mOutputJar.closeEntry();
    if (this.mManifest != null)
    {
      Attributes attr = this.mManifest.getAttributes(entry.getName());
      if (attr == null) {
        attr = new Attributes();
        this.mManifest.getEntries().put(entry.getName(), attr);
      }
      attr.putValue("SHA1-Digest", this.mBase64Encoder.encode(this.mMessageDigest.digest()));
    }
}
```

mBuffer 是 4096 bytes 的数组。
这段代码主要从 input 中读取一个 buffer 的数据，然后写入 entry 中，这里应该就可以理解我上面说的 dexFile 里的内容，写入 classes.dex 文件中了吧。

`build(outFile, dexFile);`结束了，看看最后一个方法`release(this.out, dexFile, outFile)`

##### ApkPatch release(this.out, dexFile, outFile)

```java
protected void release(File outDir, File dexFile, File outFile) throws NoSuchAlgorithmException, FileNotFoundException, IOException
{
    MessageDigest messageDigest = MessageDigest.getInstance("md5");
    FileInputStream fileInputStream = new FileInputStream(dexFile);
    byte[] buffer = new byte[8192];
    int len = 0;
    while ((len = fileInputStream.read(buffer)) > 0) {
      messageDigest.update(buffer, 0, len);
    }

    String md5 = HexUtil.hex(messageDigest.digest());
    fileInputStream.close();
    outFile.renameTo(new File(outDir, this.name + "-" + md5 + ".apatch"));
}
```

最后就是把 dexFile 进行了 md5 加密，并把`build(outFile, dexFile);`函数中生成的 outFile 重命名。这样，AndFix 框架所需要的补丁文件就生成了。

还记得上面提到的 Patch 类中读取 Manifest 的那些属性吗？在`build(outFile, dexFile);`函数中，那个`getMeta()`函数把它读取的属性写到了文件中。

到这里，第二篇分析也结束了，下一篇将会进入 Android 代码中一探它做了什么的究竟。

[1]: http://sunzeduo.blog.51cto.com/2758509/1540085
