---
title: AndFix解析-(上)
date: 2015-09-25
tags: ["AndFix"]
---

阿里巴巴前一段时间开源了他们用来解决线上紧急 bug 的一款 Android 库——`AndFix`

对 Android 开发者来说真是一个很好的消息。

**基于自己的经验，太长的文字很少有人可以一口气看下来的，所以我打算分成多篇来分析**
这是这个库解析的第一篇，

我们先看一下其中的 Demo 代码，其中调用加载库的代码如下所示:

```java
/**
 * sample application
 *
 * @author sanping.li@alipay.com
 *
 */
public class MainApplication extends Application {
	private static final String TAG = "euler";

	private static final String APATCH_PATH = "/out.apatch";
	/**
	 * patch manager
	 */
	private PatchManager mPatchManager;

	@Override
	public void onCreate() {
		super.onCreate();
		// initialize
		mPatchManager = new PatchManager(this);
		mPatchManager.init("1.0");
		Log.d(TAG, "inited.");

		// load patch
		mPatchManager.loadPatch();
		Log.d(TAG, "apatch loaded.");

		// add patch at runtime
		try {
			// .apatch file path
			String patchFileString = Environment.getExternalStorageDirectory()
					.getAbsolutePath() + APATCH_PATH;
			mPatchManager.addPatch(patchFileString);
			Log.d(TAG, "apatch:" + patchFileString + " added.");
		} catch (IOException e) {
			Log.e(TAG, "", e);
		}

	}
}
```

可以看到代码中首先通过 PatchManager 的构造函数初始化了`PatchManager`对象，

### PatchManager 构造函数

那么`PatchManager`对象里面都有什么呢，我们深入其中了解一下。

```java
public PatchManager(Context context) {
    this.mContext = context;
	this.mAndFixManager = new AndFixManager(this.mContext);
	this.mPatchDir = new File(this.mContext.getFilesDir(), "apatch");
	this.mPatchs = new ConcurrentSkipListSet();
	this.mLoaders = new ConcurrentHashMap();
}
```

原来维持了一个 Context 对象的引用，初始化了 AndFixManager 对象，mPatchDir 对象为存放 Patch 文件的文件夹，初始化了 Patch 的集合，还有持有 ClassLoader 的 Map

其中一些内容的声明如下:

```java
private final Context mContext;
private final AndFixManager mAndFixManager;
private final File mPatchDir;
private final SortedSet<Patch> mPatchs;
private final Map<String, ClassLoader> mLoaders;
```

我们对构造函数的分析在这里就结束了，我们先不深入的跟进 Patch 和 AndFixManager 这两个类了。

### PatchManager init(String version)

接下来分析`PatchManager`类中的`init(String version)`函数，
先来看代码

```java
public void init(String appVersion) {
	//如果mPatchDir不存在，则创建文件夹，如果创建失败，则打印Log
	if(!this.mPatchDir.exists() && !this.mPatchDir.mkdirs()) {
    	Log.e("PatchManager", "patch dir create error.");
    } else if(!this.mPatchDir.isDirectory()) {
	//如果遇到同名的文件，则将该同名文件删除
    	this.mPatchDir.delete();
    } else {
	//在该文件下放入一个名为_andfix_的SharedPreferences文件，
    	SharedPreferences sp = this.mContext.getSharedPreferences("_andfix_", 0);
    	String ver = sp.getString("version", (String)null);
    	if(ver != null && ver.equalsIgnoreCase(appVersion)) {
    		this.initPatchs();
    	} else {
    		this.cleanPatch();
    		sp.edit().putString("version", appVersion).commit();
    	}
	}
}
```

接下来我们分析上面代码中的如下代码

```java
//如果从_andfix_这个文件获取的ver不是null，而且这个ver和外部初始化时传进来的版本号一致
if(ver != null && ver.equalsIgnoreCase(appVersion)) {
	this.initPatchs();
} else {
	this.cleanPatch();
	sp.edit().putString("version", appVersion).commit();
}
```

先看 else 内的内容，else 里执行力`cleanPatch()`和把外部初始化的时候传进来的版本号放入 SharedPreferences 里。
`cleanPatch()`做了什么操作呢，我们跟进去看一看

```java
private void cleanPatch() {
	//获取mPatchDir目录下所有文件
    File[] files = this.mPatchDir.listFiles();
    File[] arr$ = files;
    int len$ = files.length;

    for(int i$ = 0; i$ < len$; ++i$) {
        File file = arr$[i$];
        //将此文件从OptFile文件夹删除
        this.mAndFixManager.removeOptFile(file);
        //这个方法的作用就是如果file是文件，则删除它，如果file是文件夹，则将它和它里面的文件都删除
        if(!FileUtil.deleteFile(file)) {
            Log.e("PatchManager", file.getName() + " delete error.");
        }
    }
}
```

如源码中的注释，就是删除了之前在那两个文件夹下的所有的补丁文件。

现在来分析一下`this.initPatchs()`做了什么事

```java
private void initPatchs() {
    File[] files = this.mPatchDir.listFiles();
    File[] arr$ = files;
    int len$ = files.length;

    for(int i$ = 0; i$ < len$; ++i$) {
        File file = arr$[i$];
        this.addPatch(file);
    }
}
```

代码很简单，就是把 mPatchDir 文件夹下的文件作为参数传给了`addPatch(File)`方法
那`this.addPatch(file)`做了什么呢

```java
//把扩展名为.apatch的文件传给Patch做参数，初始化对应的Patch，
//并把刚初始化的Patch加入到我们之前看到的Patch集合mPatchs中
private Patch addPatch(File file) {
    Patch patch = null;
    //扩展名是否为".apatch"
    if(file.getName().endsWith(".apatch")) {
        try {
            patch = new Patch(file);
            this.mPatchs.add(patch);
        } catch (IOException var4) {
            Log.e("PatchManager", "addPatch", var4);
        }
    }
    return patch;
}
```

上面的代码很好理解，此时，我们已经完整的走下来了`init(String version)`这个方法。
再次出现了 Patch 这个类，但是我们依然要把它放在一边，因为由于篇幅限制，第一篇不会分析这个类。

接下来，我们继续跟着 demo 走，会执行两个方法，一个`mPatchManager.loadPatch()`，一个`mPatchManager.addPatch(patchFileString)`，
`loadPatch()`方法是这个库执行替换的核心方法，我会在以后单独写一篇文章来分析，所以，在这篇文章的最后，我们跟进`addPatch(String)`这个方法一看究竟

```java
public void addPatch(String path) throws IOException {
    File src = new File(path);
    File dest = new File(this.mPatchDir, src.getName());
    if(dest.exists()) {
    	//在mPatchDir文件夹下存在该文件，则AndFixManager移除该文件
        this.mAndFixManager.removeOptFile(dest);
    }
	//将文件从src复制到dest，只不过阿里用了NIO来复制文件
    FileUtil.copyFile(src, dest);
    //调用了另外一个addPatch方法，以文件作为参数
    Patch patch = this.addPatch(dest);
    if(patch != null) {
    	//同样调用了loadPatch(Patch)方法
        this.loadPatch(patch);
    }

}
```

简单来说，上面的方法就是将补丁文件复制到/data/data/{包名}/apatch 目录内，如果在 OptFile 文件夹中存在，则删除。
然后调用另外一个`addPatch(File)`方法，然后`loadPatch()`
下面，我们来看一下重载的`addPatch(File)`方法

```java
private Patch addPatch(File file) {
    Patch patch = null;
    if(file.getName().endsWith(".apatch")) {
        try {
            patch = new Patch(file);
            //把从file文件生成的patch加入到mPatchs这个Set中
            this.mPatchs.add(patch);
        } catch (IOException var4) {
            Log.e("PatchManager", "addPatch", var4);
        }
    }

    return patch;
}
```

该方法主要做的事情在注释中即可了解，到这里，第一篇分析就结束了。
