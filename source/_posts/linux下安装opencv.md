---
title: linux下安装opencv
date: 2020-12-28 10:56:35
tags: opencv
cover: https://img9.51tietu.net/pic/2019-091400/opm22jae3hqopm22jae3hq.jpg
---

本次安装的是 opencv3.4.1，在java里面对应的maven配置是
```xml
<dependency>
    <groupId>org.bytedeco.javacpp-presets</groupId>
    <artifactId>opencv</artifactId>
    <version>3.4.1-1.4.1</version>
</dependency>
```

# 安装依赖包
```bash
yum install cmake gcc gcc-c++ gtk+-devel gimp-devel gimp-devel-tools gimp-help-browser zlib-devel libtiff-devel libjpeg-devel libpng-devel gstreamer-devel libavc1394-devel libraw1394-devel libdc1394-devel jasper-devel jasper-utils swig python libtool nasm build-essential ant
```

# 下载代码
在[github](https://github.com/opencv/opencv/tree/3.4.1)上对应的tag里面下载对应的代码，我们下载3.4.1
可以使用 git clone，也可以使用压缩包下载

# 编译安装

## 创建build文件夹
```bash
cd ~/opencv
mkdir build
cd build
```

## 编译
```
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local ..
make
sudo make install
```

## 生成jar包
```bash
make -j8
sudo make install
```
完成后 build 文件夹下就有对应的 jar 包生成

## 在 IDEA 中使用 jar 包
在 VM options 中添加参数
```bash
-Djava.library.path=D:/opencv/build/java/x86
```
指定jar包位置就行
