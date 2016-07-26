---
layout: post
title: 修改Android Gradle plugin的aapt路径
date: 2016-07-26 15:01:30
categories: [Android]
tags: [Android]
---

最近在做一套Andorid插件化方案，在解决资源id冲突的问题时，采用的是修改aapt的方式来做的，修改aapt的方式遇到的最大的问题就是编译插件的时候得替换sdk中的aapt，或者采用自己写的脚本来编译，这样开发过程肯定是很不方便的。因此本文尝试在Gradle plugin上修改aapt路径。

<!--more-->

### 准备工作
首先下载一份Gradle plugin的源码：
[https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/](https://android.googlesource.com/platform/tools/base/+/gradle_2.0.0/build-system/) 其中`gradle_2.0.0`是分支名，可以按照最新的分支来修改。

经过一段艰苦的grep源码之后，可以看到插件中取aapt的方式都是
![]({{ site.baseurl }}/assets/ico/p1.png)

重点在这：

    BuildToolInfo buildToolInfo = mTargetInfo.getBuildTools();
    String aapt = buildToolInfo.getPath(BuildToolInfo.PathId.AAPT);

而这个path是在`BuildToolInfo`构造的时候通过调用`com.android.sdklib.BuildToolInfo#add(com.android.sdklib.BuildToolInfo.PathId, java.io.File)`来添加到一个map中的，因此我们只要再调用这个方法，替换map中的值即可替换aapt路径了。

### 具体方案
替换的方式很简单，只要理清楚了Gradle plugin中的一些类的关系就可以获得这个BuildToolInfo对象了，因为Groovy本身是完全兼容Java语法的，因此直接使用Java的反射方式来修改。具体代码如下：  

```java  
/**
 * 修改Aapt路径
 */
Task modifyAaptPathTask = task('modifyAaptPath') << {
    android.applicationVariants.all { variant ->
        BuildToolInfo buildToolInfo = variant.androidBuilder.getTargetInfo().getBuildTools()
        Method addMethod = BuildToolInfo.class.getDeclaredMethod("add", BuildToolInfo.PathId.class, File.class)
        addMethod.setAccessible(true)
        String newAaptPath = "youpath/aapt"
        addMethod.invoke(buildToolInfo, BuildToolInfo.PathId.AAPT, new File(rootDir, newAaptPath))
        println "[LOG] new aapt path = " + buildToolInfo.getPath(BuildToolInfo.PathId.AAPT)
    }
}

/**
 * 在preBuild task执行前修改aapt path
 */
preBuild.doFirst {
    modifyAaptPathTask.execute()
}
```
