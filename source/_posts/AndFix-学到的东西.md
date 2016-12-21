---
title: AndFix 学到的东西
date: 2016-06-13 23:08:32
tags: Android
---

AndFix已经使用了一段时间了，但是到[AndFix](https://github.com/alibaba/AndFix)上看了一下，最近2个月都没有更新代码了，有141个issues和3个pull request没人处理，其实AndFix的Contributors就俩个人，一个是[rogerAce](https://github.com/lirenlong)还有个是[supern lee](https://github.com/supern),虽然快要沦为了阿里的KPI项目，但是并不妨碍AndFix在业界的地位-一个低成本快速接入的Bugxiuf第一方案。

第二方案[Nuwa](https://github.com/jasonross)，Nuwa的原理是修改了gradle的编译task流程，替换dex的方式来实现。但是可惜的是gradle plugin在1.5以后取消了predexdebug这个task，而Nuwa恰恰是依赖这个task的，所以导致Nuwa在gradle plugin1.5版本后无法使用，而且Nuwa的作者是贾吉鑫也在一次技术分享的时候也表示，不再维护Nuwa，因为他感觉AndFix已经做的足够好，他不想把AndFix做的事情再做一次。

闲话扯完，网上AndFix的教程和解析已经很多，这里就仅分享一下我在AndFix中学到的东西。

### SortedSet

```
private final SortedSet<Patch> mPatchs;
this.mPatchs = new ConcurrentSkipListSet();

```

 >SortedSet是个接口，它里面的（只有TreeSet这一个实现可用）中的元素一定是有序的。
 保证迭代器按照元素递增顺序遍历的集合，可以按照元素的自然顺序（参见 Comparable）进行排序， 或者按照创建有序集合时提供的 Comparator进行排序。要采用此排序，

>ConcurrentSkipListSet（在JavaSE 6新增的）提供的功能类似于TreeSet，能够并发的访问有序的set。因为ConcurrentSkipListSet是基于“跳跃列表（skip list）”实现的，只要多个线程没有同时修改集合的同一个部分，那么在正常读、写集合的操作中不会出现竞争现象。

mPatchs是一个有序并发Set,Set中的元素都必须实现 Comparable 接口,所以Patch实现了Comparable的compareTo方法,可见Patch是按照时间从小到大顺序
```
    public int compareTo(Patch another) {
        return this.mTime.compareTo(another.getTime());
    }
```

### Patch生成

首先看一下Patch是什么？解压之后([盗图](http://www.cnblogs.com/soaringEveryday/p/5338214.html))

![](data:image/png;base64,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)

meta-inf文件夹为：

![](data:image/png;base64,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)

打开patch.mf文件可以发现两个apk的差异信息：

```
Manifest-Version: 1.0
Patch-Name: app-release-fix
To-File: app-release-online.apk
Created-Time: 30 Mar 2016 06:26:27 GMT
Created-By: 1.0 (ApkPatch)
Patch-Classes: com.qianmi.shine.activity.me.AboutActivity_CF
From-File: app-release-fix.apk

```
AndFix是如何把文件读取出来，并且初始化Patch的尼？Android大部分的文件都是压缩包，所以这里的处理也有一定的代表性。

```
  private void init() throws IOException {
        JarFile jarFile = null;
        InputStream inputStream = null;

        try {
            jarFile = new JarFile(this.mFile);
            JarEntry entry = jarFile.getJarEntry("META-INF/PATCH.MF");
            inputStream = jarFile.getInputStream(entry);
            Manifest manifest = new Manifest(inputStream);
            Attributes main = manifest.getMainAttributes();
            this.mName = main.getValue("Patch-Name");
            this.mTime = new Date(main.getValue("Created-Time"));
            this.mClassesMap = new HashMap();
            Iterator it = main.keySet().iterator();

            while(it.hasNext()) {
                Name attrName = (Name)it.next();
                String name = attrName.toString();
                if(name.endsWith("-Classes")) {
                    List strings = Arrays.asList(main.getValue(attrName).split(","));
                    if(name.equalsIgnoreCase("Patch-Classes")) {
                        this.mClassesMap.put(this.mName, strings);
                    } else {
                        this.mClassesMap.put(name.trim().substring(0, name.length() - 8), strings);
                    }
                }
            }
        } finally {
            if(jarFile != null) {
                jarFile.close();
            }

            if(inputStream != null) {
                inputStream.close();
            }

        }

    }

```
> JarFile 类用于从任何可以使用 java.io.RandomAccessFile 打开的文件中读取 jar 文件的内容。它扩展了 java.util.zip.ZipFile 类，使之支持读取可选的 Manifest 条目。Manifest 可用于指定关于 jar 文件及其条目的元信息。


这里使用了JarFile来解压.patch文件，然后找到META-INF/PATCH.MF 然后找到Patch-Classes，这样就可以取出里面被修复的类，当然这些类都是在原来的包名+CF，当真正修复的时候执行this.mAndFixManager.fix(patch.getFile(), cl, classes)方法，里面patch.getFile()里面是dex，classes里面是指定修复的类。

### fix

下面就是AndFix中最重要的一个方法了，核心的fix逻辑都在这里

```
 public synchronized void fix(File file, ClassLoader classLoader, List<String> classes) {
        if(this.mSupport) {
            if(this.mSecurityChecker.verifyApk(file)) {
                try {
                    File e = new File(this.mOptDir, file.getName());
                    boolean saveFingerprint = true;
                    if(e.exists()) {
                        if(this.mSecurityChecker.verifyOpt(e)) {
                            saveFingerprint = false;
                        } else if(!e.delete()) {
                            return;
                        }
                    }

                    DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(), e.getAbsolutePath(), 0);
                  。。。
                                clazz = dexFile.loadClass(entry, classLoader);
				  。。。
                                    this.fixClass(clazz, classLoader);
                        return;
                    }
                } catch (IOException var12) {
                    Log.e("AndFixManager", "pacth", var12);
                }
            }
        }
    }

```

mSupport- 是否支持热更新修复，简单看了一下，就是不支持yunOS，sdk在8~23都可以。

verifyApk- 对比一下签名信息，看dex是否合法,这里面的方法也很典型，自己写库的时候也可以用。

verifyOpt- 获取file的md5值看是否改变过

DexFile.loadDex
>打开一个DEX文件，并提供一个文件来保存优化过的DEX数据。如果优化过的格式已存在并且是最新的，就直接使用它。如果不是，虚拟机将试图重新创建一个。该方法主要用于应用希望在通常的应用安装机制之外下载和执行DEX文件。不能在应用里直接调用该方法，而应该通过一个类装载器例如dalvik.system.DexClassLoader.
关于动态加载可以看看[扩展](http://blog.csdn.net/jiangwei0910410003/article/details/17679823)

接下来就是找到带MethodReplace注解的方法，这个注解是在代码对比过程自动生成的，
```
  MethodReplace methodReplace = (MethodReplace)method.getAnnotation(MethodReplace.class);
            if(methodReplace != null) {
                String clz = methodReplace.clazz();
                String meth = methodReplace.method();
                if(!isEmpty(clz) && !isEmpty(meth)) {
                    this.replaceMethod(classLoader, clz, meth, method);
                }
            }
```
replaceMethod传入的有class名字，方法的名字，方法本身,这样根据meth就有找到原来app中对应的方法
```
Method src1 = clazz.getDeclaredMethod(meth, method.getParameterTypes());
```
一个有bug的方法，一个修复后的方法，一招乾坤大罗移，
```
AndFix.addReplaceMethod(src1, method);

```
为啥说是乾坤大罗移是因为AndFix在C的层面将源方法（meth）的各个属性被替换成了新的方法（target）的各个属性，就完成了方法的替换，当虚拟机误以为方法还是之前的“方法”。

```
extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
        JNIEnv* env, jobject src, jobject dest) {
    jobject clazz = env->CallObjectMethod(dest, jClassMethod);
    ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(
            dvmThreadSelf_fnPtr(), clazz);
    clz->status = CLASS_INITIALIZED;

    Method* meth = (Method*) env->FromReflectedMethod(src);
    Method* target = (Method*) env->FromReflectedMethod(dest);
    LOGD("dalvikMethod: %s", meth->name);

    meth->clazz = target->clazz;
    meth->accessFlags |= ACC_PUBLIC;
    meth->methodIndex = target->methodIndex;
    meth->jniArgInfo = target->jniArgInfo;
    meth->registersSize = target->registersSize;
    meth->outsSize = target->outsSize;
    meth->insSize = target->insSize;

    meth->prototype = target->prototype;
    meth->insns = target->insns;
    meth->nativeFunc = target->nativeFunc;
}

```


至此，AndFix的整个过程就结束了，可见要完成这样的库，需要会写插件，NDK，还有对类的加载过程非常了解，最关键的是，从问题入手找解决方法的过程，试着想一下，如果你有了这些技术栈，现在需要解决动态修复的问题，你会怎么做？我可能会把整个类都替换掉，因为在我感觉中，一块代码的替换好像比一个类的替换要难，但是阿里的同学做到了。实在佩服佩服，也为阿里在开源届做的贡献点赞!!!!






