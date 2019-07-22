[TOC]

## Android 系统原理

### 1. 打包原理

Android的包文件apk分为两个部分：代码和资源，所以打包方面也分为资源打包和代码打包两个方面。

具体来说，共分为如下几个步骤：

1. 通过AAPT工具进行资源文件（包括AndroidManifest.xml、布局文件、各种xml资源）的打包，生成R.java文件
2. 通过AIDL工具处理AIDL文件，生成相应的Java文件
3. 通过Javac工具编译项目源码，生成Class文件
4. 通过DX工具将所有Class文件转换成Dex文件，该过程主要完成Java字节码转换成Dalvik字节码，压缩常量池以及清除冗余信息等工作
5. 通过ApkBuilder工具将资源文件、Dex文件打包生成Apk文件
6. 利用KeyStore对生成的Apk文件进行签名
7. 如果是正式版Apk，还会利用ZipAlign工具进行对齐处理，对齐的过程就是将Apk文件中所有的资源文件距离文件的距离都偏移四字节的整数倍，这样通过内存映射访问APK文件速度会更快

![打包流程](https://user-gold-cdn.xitu.io/2019/7/17/16bff4358a12ae4c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 2 安装流程

Android Apk的安装过程主要分为以下几步：

1. 复制Apk到/data/app目录下，解压并扫描安装包
2. 资源管理器解析Apk里的资源文件
3. 解析AndroidManifest.xml文件，并在/data/data/目录下创建对应的应用数据目录
4. 对Dex文件进行优化，并保存在dalvik-cache目录下
5. 将AndroidManifest文件解析出的四大组件信息注册到PackageManagerService中
6. 安装完成，发送广播

![App安装过程示意图](https://user-gold-cdn.xitu.io/2019/7/17/16bff4358d8bdc85?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 3. App启动过程

