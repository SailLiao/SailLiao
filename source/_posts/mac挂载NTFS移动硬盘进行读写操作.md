---
title: mac挂载NTFS移动硬盘进行读写操作
date: 2020-12-24 17:27:25
tags: 工具
cover: https://img9.51tietu.net/pic/2019-091409/10zihc5v01d10zihc5v01d.jpg
---

在Mac上，默认情况对NTFS磁盘的挂载方式是只读(read-only)的，其实Mac原生是支持NTFS的，但是后来由于微软的限制，苹果把这个功能给屏蔽了，但是我们可以通过命令行方式打开这个选项。

接入移动硬盘后，我们首先查看一下挂载信息

```bash
$ sudo mount
$ /dev/disk2s1 on /Volumes/新加卷 (ntfs, local, nodev, nosuid, read-only, noowners)
```

可以看到默认的挂载方式是把磁盘挂载成了只读(read-only)的。

下面我们通过下面的命令来把磁盘挂载成可写的。

```bash
$ sudo umount /Volumes/新加卷
$ sudo mount -t ntfs -o rw,auto,nobrowse /dev/disk2s1 /data/mydisk/
```

文章来源: [Mac挂载NTFS移动硬盘进行读写操作](https://www.jianshu.com/p/1b25ab5a401f)