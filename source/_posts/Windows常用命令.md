---
title: Windows常用命令
date: 2020-01-03 12:10:29
tags:
- Windows
categories:
- 后端
- 系统
- Windows
---

# 拷贝文件并排除某些文件

语法：xcopy 源目录 目标目录 参数

```cmd
xcopy .\my-shop D:\Temp\my-shop /Y /E /H /EXCLUDE:EXCLUDE.txt
```

参数解释

/Y： 禁止提示以确认改写一个现存目标文件。 

/E： 复制目录和子目录，包括空的。 

 /H： 复制隐藏和系统文件。 

/EXCLUDE：排除文件配置文件

举例，EXCLUDE.txt 配置内容如下：

```
.git
.idea
target
.iml
EXCLUDE.txt
```

表示文件名或者目录名包含以上字符的文件或者目录都不拷贝。
