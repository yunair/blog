---
title: AndFix解析-(下)
date: 2015-10-23
tags: ["AndFix"]
---

我们接着分析阿里开源的 AndFix 库，
上一篇分析了`Patch`类，这个类相当于我们提供补丁的容器，容器里有了东西，我们要对容器进行操作了, 于是开始了我们这次的分析。

在第二篇里，我们添了`Patch`类的那个坑，那么这篇文章我们就把最后两个坑填一填，即`loadPatch()`方法和`AndFixManager`类。

在阿里给的 Demo 里，我们还有最后的`loadPatch()`方法没有深入，所以先从`loadPatch()`方法开始。

### PatchManager loadPatch()

```java
public void loadPatch() {
	mLoaders.put("*", mContext.getClassLoader());// wildcard
	Set<String> patchNames;
	List<String> classes;
	for (Patch patch : mPatchs) {
		patchNames = patch.getPatchNames();
		for (String patchName : patchNames) {
			classes = patch.getClasses(patchName);
			mAndFixManager.fix(patch.getFile(),mContext.getClassLoader(), classes);
		}
	}
}
```

不知道大家是否还记得之前提到的`mLoaders`这个成员变量，隔了这么久，说实话我也忘记了，在这里我先带大家一起回忆一下，
`private final Map<String, ClassLoader> mLoaders;`，`mLoaders`原来是储存不同`ClassLoader`的 Map 啊。
好的，我们继续向下进行，在第一篇，通过 private 的`initPatchs()`方法，和 public 的`addPatch()`方法，将`Patch`加入了`mPatchs`这个`List`，
所以，这里只要去遍历这个`List`，来获取不同的`Patch`，并对每个`Patch`做操作即可。
每个`Patch`，代表了一个`.apatch`的文件，`getClasses(patchName)`代表着这个`patch`的`patchName`对应的所有需要修改的类。
`patchName`从两方面而来，一个是你用`apkpatch.jar`的时候使用-n 选项指定或者默认的，另外一个方面是写入`Manifest`的时候以
`-Classes`结尾的 key，这个我暂时还没有遇到过，遇到过的同学可以给我讲讲，抱歉博客暂时没有评论功能，可以发邮件给我,airzhaoyn@gmail.com。

好了，这里讲的差不多了，我们继续深入。
可以看到，获取了一个`patchName`对应的所有的需要修改的类后，就会调用`AndFixManager`类的`fix(File, ClassLoader,List<String>)`方法，
先来看看源码

### AndFixManager fix(File, ClassLoader,List<String>)

```java
/**
 * fix
 *
 * @param file
 *            patch file
 * @param classLoader
 *            classloader of class that will be fixed
 * @param classes
 *            classes will be fixed
 */
public synchronized void fix(File file, ClassLoader classLoader,
		List<String> classes) {
	//是否是支持的Android版本(在AndFixManager类初始化的时候会修改mSupport变量)
	if (!mSupport) {
		return;
	}
    //验证这个文件的签名和此应用是否一致
	if (!mSecurityChecker.verifyApk(file)) {// security check fail
		return;
	}

	//这段代码的源码中的注释很清楚，我就不写了
	File optfile = new File(mOptDir, file.getName());
	boolean saveFingerprint = true;
	if (optfile.exists()) {
		// need to verify fingerprint when the optimize file exist,
		// prevent someone attack on jailbreak device with
		// Vulnerability-Parasyte.
		// btw:exaggerated android Vulnerability-Parasyte
		// http://secauo.com/Exaggerated-Android-Vulnerability-Parasyte.html
		if (mSecurityChecker.verifyOpt(optfile)) {
			saveFingerprint = false;
		} else if (!optfile.delete()) {
			return;
		}
	}

    //start
	final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
			optfile.getAbsolutePath(), Context.MODE_PRIVATE);

	if (saveFingerprint) {
		mSecurityChecker.saveOptSig(optfile);
	}

	ClassLoader patchClassLoader = new ClassLoader(classLoader) {
		@Override
		protected Class<?> findClass(String className)
				throws ClassNotFoundException {
			Class<?> clazz = dexFile.loadClass(className, this);
			if (clazz == null
					&& className.startsWith("com.alipay.euler.andfix")) {
				return Class.forName(className);// annotation’s class
												// not found
			}
			if (clazz == null) {
				throw new ClassNotFoundException(className);
			}
			return clazz;
		}
	};
	//Enumerate the names of the classes in this DEX file.
	Enumeration<String> entrys = dexFile.entries();
	Class<?> clazz = null;
	while (entrys.hasMoreElements()) {
		String entry = entrys.nextElement();
		if (classes != null && !classes.contains(entry)) {
			continue;// skip, not need fix
		}
		clazz = dexFile.loadClass(entry, patchClassLoader);
		if (clazz != null) {
			fixClass(clazz, classLoader);
		}
	}
}
```

对这部分的源码解析，从源码中标注//start 开始。
我们先看一下 DexFile 是什么，官方文档这样说：`Manipulates DEX files. It is used primarily by class loaders.`
就是主要被类加载器使用的操作 Dex 文件的类。
好了，我们可以继续看源码了，先获取一个 DexFile 对象的，然后通过匿名内部类实现了一个 ClassLoaders 的子类，遍历这个 Dex 文件中所有的类，
如果需要修改的类集合(即从`PatchManager loadPatch()`方法中传过来的类集合)中在这个 Dex 文件中找到了一样的类，则使用`loadClass(String, ClassLoader)`加载这个类，
然后调用`fixClass(String, ClassLoader)`修复这个类。

### AndFixManager fixClass(Class<?>, ClassLoader)

```java
private void fixClass(Class<?> clazz, ClassLoader classLoader) {
    //使用反射获取这个类中所有的方法
	Method[] methods = clazz.getDeclaredMethods();
	//MethodReplace是这个库自定义的Annotation，标记哪个方法需要被替换
	MethodReplace methodReplace;
	String clz;
	String meth;
	for (Method method : methods) {
	    //找到被MethodReplace注解的方法
		methodReplace = method.getAnnotation(MethodReplace.class);
		if (methodReplace == null)
			continue;
		clz = methodReplace.clazz();
		meth = methodReplace.method();
		if (!isEmpty(clz) && !isEmpty(meth)) {
			replaceMethod(classLoader, clz, meth, method);
		}
	}
}
```

在源码中基本把这个方法给解读完了，接下来就要看看它调用的`replaceMethod(ClassLoader, String,String, Method)`方法

### AndFixManager replaceMethod(ClassLoader, String,String, Method)

```java
private void replaceMethod(ClassLoader classLoader, String clz,
			String meth, Method method) {
	//对每个类创建一个不会冲突的key
    String key = clz + "@" + classLoader.toString();
    //mFixedClass是一个Map<String, Class<?>>，并被实例化为ConcurrentHashMap<>();
    Class<?> clazz = mFixedClass.get(key);
    if (clazz == null) {// class not load
    	Class<?> clzz = classLoader.loadClass(clz);
    	// initialize target class and modify access flag of class’ fields to public
    	clazz = AndFix.initTargetClass(clzz);
    }
    if (clazz != null) {// initialize class OK
    	mFixedClass.put(key, clazz);
    	//获取名为meth的方法
    	Method src = clazz.getDeclaredMethod(meth,
    			method.getParameterTypes());
    	AndFix.addReplaceMethod(src, method);
    }
}
```

这里源代码中的英文注释(作者注释)已经很清楚了，去看看调用的那个静态方法`AndFix.addReplaceMethod(src, method);`

### AndFix addReplaceMethod(Method, Method)

```java
public static void addReplaceMethod(Method src, Method dest) {
	replaceMethod(src, dest);
    initFields(dest.getDeclaringClass());
}
```

replaceMethod 是一个 native 方法，声明如下:
`private static native void replaceMethod(Method dest, Method src);`
由于暂时我对 JNI 不是很熟悉，所以这里就不分析了。
看看另外一个方法`initFields(Class<?>)`

```java
/**
 * modify access flag of class’ fields to public
 *
 * @param clazz
 *            class
 */
private static void initFields(Class<?> clazz) {
	Field[] srcFields = clazz.getDeclaredFields();
	for (Field srcField : srcFields) {
		Log.d(TAG, "modify " + clazz.getName() + "." + srcField.getName()
				+ " flag:");
		setFieldFlag(srcField);
	}
}
```

这里英文注释也很清楚了，只是其中调用了`setFieldFlag(srcField);`这个我们没见过的方法，
声明为`private static native void setFieldFlag(Field field);`
又是个 JNI 方法，暂时不进行分析。

到这里，对阿里 AndFix 库 Java 层面上的代码的分析就结束了，如果还想了解 native 层的代码，我将会放在下一篇，敬请等待。
