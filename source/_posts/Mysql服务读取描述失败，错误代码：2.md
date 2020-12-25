---
title: Mysql服务读取描述失败，错误代码：2
date: 2020-12-24 17:36:37
tags: Mysql
cover: https://img9.51tietu.net/pic/2019-091403/pxrxw44qm24pxrxw44qm24.png
---

每次用360清理机器后，我的mysql服务就出问题，在服务里面看mysql的被删了，然后我就把360删了

## 恢复的方式

1. 管理员方式启动 cmd
2. 切换数据库bin目录
3. 输入命令 SC DELETE MYSQL
4. 重启电脑
5. 管理员方式启动 cmd
6. 切换数据库bin目录
7. 输入mysqld.exe -install
8. net start mysql
