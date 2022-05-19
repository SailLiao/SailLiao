---
title: 使用ffmpeg从视频中截取图像帧
date: 2020-12-25 15:38:14
tags: ffmpeg
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-9mprjw.jpg
---

在抽取视频帧方面，ffmpeg比opencv java版速度好太多了

# ffmpeg 安装

centos 6 与 7 安装ffmpeg还是有区别的，不过事先都需要安装拓展源
```bash
yum -y install epel-release
```

## centos 6
```bash
su -c 'yum localinstall --nogpgcheck https://download1.rpmfusion.org/free/el/rpmfusion-free-release-6.noarch.rpm https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-6.noarch.rpm'

yum -y install ffmpeg ffmpeg-devel
```

## centos 7
```bash
su -c 'yum localinstall --nogpgcheck https://download1.rpmfusion.org/free/el/rpmfusion-free-release-7.noarch.rpm https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-7.noarch.rpm'

rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro

rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-1.el7.nux.noarch.rpm

yum -y install ffmpeg ffmpeg-devel
```

windows 下安装直接有现成的 exe 
> 链接: https://pan.baidu.com/s/1n5mJ1grkTFT9rLuJ8XlJZA 
> 提取码: b7du 
记得将 bin 所在的路径加入到环境变量path里面

# 提取帧图片

下面的例子是 每隔多少帧取一帧

```bash
# 老版本例如 centos6.5
/usr/bin/ffmpeg -i /tmp/1.mp4 -vf "select='gt(t,1)*lt(t,100)*not(mod(n,5))'" -vsync 0 /tmp/1/%d.jpg

# 新版本
/usr/bin/ffmpeg -i /tmp/1.mp4  -vf "select=between(n,1,100)*not(mod(n,5))"  -vsync 0 /tmp/1/%d.jpg
```
* /usr/bin/ffmpeg 为ffmpeg的路径
* /tmp/1.mp4 为视频路径
* select=between(n,1,100)*not(mod(n,5)) 中 100 为视频的总帧数 5 为隔多少帧取一帧
* /tmp/1/ 为输出的帧图片的位置 位置一定要先创建好

## 获取视频总帧数

可以通弄过 ffprobe
```bash
ffprobe -v error -count_frames -select_streams v:0 -show_entries stream=nb_read_frames -of default=nokey=1:noprint_wrappers=1 /tmp/1.mp4
```
也可以通过java代码opencv来获取