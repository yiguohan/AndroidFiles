[TOC]

## 预备知识

1. 基本的 Android 开发只是
2. 了解Android Studio 基本使用

## 看完本文可以你可以学到

1. 掌握 Gradle 的基本使用
2. 了解 Gradle 及 Android Gradle Plugin
3. 了解 Gradle 构建阶段及生命周期回调
4. 掌握 Task, Transform 等概念
5. 学会自定义 Task，自定义Gradle插件

如果您已经达到上面的程度，那么可以不用再看下文了，直接看最后的总结即可

本文将从下面几个部分进行讲解

![gradle-summary](https://user-gold-cdn.xitu.io/2019/5/9/16a9d22459b907a3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 1. Gradle是什么

![gradle1](https://user-gold-cdn.xitu.io/2019/5/9/16a9d22bcaca4164?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

官方解释是: Gradle is an open-source build automation tool focused on flexibility and performance. Gradle build scripts are written using a Groovy or Kotlin DSL.
 可以从三个角度来理解

### 1.1 Gradle是一个自动化构建工具

gradle 是通过组织一系列 task 来最终完成自动化构建的，所以 task 是 gradle 里最重要的概念
 我们以生成一个可用的 apk 为例，整个过程要经过 `资源的处理`，`javac 编译`，`dex 打包`，`apk 打包`，`签名`等等步骤，每个步骤就对应到 gradle 里的一个 task

gradle 可以类比做一条流水线，task 可以比作流水线上的机器人，每个机器人负责不同的事情，最终生成完整的构建产物

![gradle-pipelining](https://user-gold-cdn.xitu.io/2019/5/9/16a9d23033ef5cd4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 1.2 Gradle 脚本使用了 Groovy 或者 Kotlin DSL

Gradle 使用 Groovy 或者 Kotlin 编写，不过目前还是 Groovy 居多。

那什么是 DSL 呢？

>  DSL 是 Domain Specific Language 的简称，是为了解决某一类任务专门设计的计算机语言。

DSL 相对应的是 GPL (General-Purpose Language)，比如 Java。

与 GPL 相比起来，DSL 使用简单，定义比较简洁，比起配置文件，DSL 又可以实现语言逻辑。

 对 Gradle 脚本来说，他实现了简洁的定义，又有充分的语言逻辑，以 android {} 为例，这本**身是一个函数调用，参数是一个闭包**，但是这种定义方式明显要简洁很多。

### 1.3  gradle 基于 groovy 编写，而 groovy 是基于 jvm 语言

Gradle 使用 Groovy 编写，Groovy 是基于 jvm 的语言，所以本质上是面向对象的语言，面向对象语言的特点就是一切皆对象，所以，**在 Gradle 里，`.gradle` 脚本的本质就是类的定义，一些配置项的本质都是方法调用，参数是后面的 {} 闭包。**

比如 `build.gradle` 对应` Project` 类，`buildScript` 对应 `Project.buildScript` 方法

## 2. Gradle项目分析

![gradle2](https://user-gold-cdn.xitu.io/2019/5/9/16a9d2384dbb84c8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

关于 gradle 的项目层次，我们新建一个项目看一下，项目地址在 [EasyGradle](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2F5A59%2Fandroid-training%2Ftree%2Fmaster%2Fgradle%2FEasyGradle)

![gradle-projcet](https://user-gold-cdn.xitu.io/2019/5/9/16a9d24ae80474c5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 2.1 settings.gradle

settings.gradle 是负责配置项目的脚本，对应 [Settings](https://github.com/gradle/gradle/blob/v4.1.0/subprojects/core/src/main/java/org/gradle/api/initialization/Settings.java) 类。

gradle 构建过程中，会根据 settings.gradle 生成 Settings 的对象，对应的可调用的方法在[文档](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html)里可以查找。

其中几个主要的方法有:

- include(projectPaths)
- includeFlat(projectNames)
- project(projectDir)

一般在项目里见到的引用子模块的方法，就是使用 include，这样引用，子模块位于根项目的下一级：

``` groovy
include ':app'
```

如果想指定子模块的位置，可以使用 project 方法获取 Project 对象，设置其 projectDir 参数

``` groovy
include ':app'
project(':app').projectDir = new File('./app')
```

### 2.2 rootproject/build.gradle

build.gradle 负责整体项目的一些配置，对应的是 [Project](https://github.com/gradle/gradle/blob/v4.1.0/subprojects/core/src/main/java/org/gradle/api/Project.java) 类。

Gradle 构建的时候，会根据 build.gradle 生成 Project 对象，所以在 build.gradle 里写的 DSLl，其实都是 Project 接口的一些方法。

Project 其实是一个接口，真正的实现类是 [DefaultProject](https://github.com/gradle/gradle/blob/v4.1.0/subprojects/core/src/main/java/org/gradle/api/internal/project/DefaultProject.java)。

build.gradle 里可以调用的方法在 [Project](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html) 可以查到。

其中几个主要方法有：

- buildscript  配置脚本的 classpath
- allprojects  配置项目及其子项目
- respositories  配置仓库地址，后面的依赖都会去这里配置的地址查找
- dependencies  配置项目的依赖

以 EasyGradle 项目来看

```groovy
buildscript { // 配置项目的 classpath
    repositories {  // 项目的仓库地址，会按顺序依次查找
        google()
        jcenter()
        mavenLocal()
    }
    dependencies { // 项目的依赖
        classpath 'com.android.tools.build:gradle:3.0.1'
        classpath 'com.zy.plugin:myplugin:0.0.1'
    }
}

allprojects { // 子项目的配置
    repositories {
        google()
        jcenter()
        mavenLocal()
    }
}
```

### 2.3 module/build.gradle

build.gradle 是子项目的配置，对应的也是 [Project](https://github.com/gradle/gradle/blob/v4.1.0/subprojects/core/src/main/java/org/gradle/api/Project.java) 类。

子项目和根项目的配置是差不多的

子项目和根项目有一个明显的区别，就是**引用了一个插件 `apply plugin "com.android.application"`**

后面的 android dsl 就是 application 插件的 extension，关于 android plugin dsl 可以看 [android-gradle-dsl](http://google.github.io/android-gradle-dsl/current/)

其中几个主要方法有：

- compileSdkVersion  指定编译需要的 sdk 版本
- defaultConfig  指定默认的属性，会运用到所有的 variants 上
- buildTypes  一些编译属性可以在这里配置，可配置的所有属性在 [这里](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.BuildType.html)
- productFlavor  配置项目的 flavor

以 app 模块的 build.gradle 来看

```groovy
apply plugin: 'com.android.application' // 引入 android gradle 插件

android { // 配置 android gradle plugin 需要的内容
    compileSdkVersion 26
    defaultConfig { // 版本，applicationId 等配置
        applicationId "com.zy.easygradle"
        minSdkVersion 19
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
    }
    buildTypes { 
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions { // 指定 java 版本
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }

    // flavor 相关配置
    flavorDimensions "size", "color"
    productFlavors {
        big {
            dimension "size"
        }
        small {
            dimension "size"
        }
        blue {
            dimension "color"
        }
        red {
            dimension "color"
        }
    }
}

// 项目需要的依赖
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar']) // jar 包依赖
    implementation 'com.android.support:appcompat-v7:26.1.0' // 远程仓库依赖
    implementation 'com.android.support.constraint:constraint-layout:1.1.3'
    implementation project(':module1') // 项目依赖
}
```

### 2.4 依赖

在 gradle 3.4 里引入了新的依赖配置，如下：

| 新配置         | 弃用配置 | 行为                                                         | 作用                                                         |
| -------------- | -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| implementation | compile  | 依赖项在编译时对模块可用，并且仅在运行时对模块的消费者可用。对于大型多项目构建，使用`implementation`而不是`api`/`compile`可以显著缩短构建时间，因为它可以减少构建系统需要从新编译的项目量。大多数应用和测试模块都应使用此配置 | `implementation`只会暴露给直接以来的模块，使用此配置，在模块修改后，只会重新编译直接依赖的模块，间接依赖的模块不需要改动。 |
| api            | compile  | 依赖项在编译时对模块可用，并且在编译时和运行时还对模块的消费者可用。此配置的行为类似于`compile`（现已弃用），一般情况下，您应当尽在库模块中使用它。应用模块应使用`implementation`，除非您想要将其API公开给单独的测试模块。 | api会暴露给简介依赖的模块，使用此配置，在模块修改以后，模块的直接依赖和简介以来的模块都需要重新编译。 |
| compileOnly    | provided | 依赖项仅在编译时对模块可用，并且在编译或运行时对其消费者不可用。此配置的行为类似于`provided`(现已弃用)。 | 只在编译期间依赖模块，打包以后运行时不会依赖，可以用来解决一些库冲突的问题。 |
| runtimeOnly    | apk      | 依赖项仅在运行时对模块及其消费者可用。此配置的行为类似于apk（现已弃用）。 | 旨在运行时依赖模块，编译时不依赖。                           |

还是以 EasyGradle 为例，看一下各个依赖的不同：项目里有三个模块：app，module1， module2

模块 app 中有一个类 ModuleApi

模块 module1 中有一个类 Module1Api

模块 module2 中有一个类 Module2Api

其依赖关系如下：

![dep](https://user-gold-cdn.xitu.io/2019/5/9/16a9d25adee88f17?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 2.4.1 implementation

当 module1 使用 implementation 依赖 module2 时，在 app 模块中无法引用到 Module2Api 类

![implementation](https://user-gold-cdn.xitu.io/2019/5/9/16a9d2621a9046a9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 2.4.2 api 依赖

当 module1 使用 api 依赖 module2 时，在 app 模块中可以正常引用到 Module2Api 类，如下图

![api](https://user-gold-cdn.xitu.io/2019/5/9/16a9d265c5985b46?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 2.4.3 compileOnly 依赖

当 module1 使用 compileOnly 依赖 module2 时，在编译阶段 app 模块无法引用到 Module2Api 类，module1 中正常引用，但是在运行时会报错

![compileOnly](https://user-gold-cdn.xitu.io/2019/5/9/16a9d26939eddfa2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

反编译打包好的 apk，可以看到 Module2Api 是没有被打包到 apk 里的

![ompileOnly-apk](https://user-gold-cdn.xitu.io/2019/5/9/16a9d26e14209aa0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 2.4.4 runtimeOnly 依赖
当 module1 使用 runtimeOnly 依赖 module2 时，在编译阶段，module1 也无法引用到 Module2Api

![runtimeOnly](https://user-gold-cdn.xitu.io/2019/5/9/16a9d271fa13e976?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 2.5 flavor

在介绍下面的流程之前，先明确几个概念，`flavor`，`dimension`，`variant`

在 android gradle plugin 3.x 之后，每个 `flavor` 必须对应一个 `dimension`，可以理解为 `flavor` 的分组，然后不同 `dimension` 里的 `flavo`r 两两组合形成一个 `variant`

举个例子 如下配置：

```groovy
flavorDimensions "size", "color"

productFlavors {
    big {
        dimension "size"
    }
    small {
        dimension "size"
    }
    blue {
        dimension "color"
    }
    red {
        dimension "color"
    }
}
```

那么生成的 `variant` 对应的就是 `bigBlue`，`bigRed`，`smallBlue`，`smallRed`

每个 `variant` 可以对应的使用 `variantImplementation` 来引入特定的依赖，比如：`bigBlueImplementation`，只有在 编译 bigBlue variant的时候才会引入

### 3. Gradle Wrapper

![gradle3](https://user-gold-cdn.xitu.io/2019/5/9/16a9d277be4d498c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 3.1 gradlew/gradlew.bat

这个文件用来下载特定版本的 gradle 然后执行的，就不需要开发者在本地再安装 gradle 了。

这样做有什么好处呢？开发者在本地安装 gradle，会碰到的问题是不同项目使用不同版本的 gradle 怎么处理，用 wrapper 就很好的解决了这个问题，可以在不同项目里使用不同的 gradle 版本。

gradle wrapper 一般下载在 GRADLE_CACHE/wrapper/dists 目录下

#### 3.2 gradle->wrapper->gradle-wrapper.properties

是一些 gradlewrapper 的配置，其中用的比较多的就是 distributionUrl，可以执行 gradle 的下载地址和版本。

#### 3.3 gradle->wrapper->gradle-wrapper.jar

gradlewrapper 运行需要的依赖包

### 4. gradle init.gradle

![gradle4](https://user-gold-cdn.xitu.io/2019/5/9/16a9d27ad6531dae?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)









