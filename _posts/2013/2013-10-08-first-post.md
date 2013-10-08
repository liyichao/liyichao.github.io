---
layout: post
title: "stat"
category: 
tags: []
---
{% include JB/setup %}

在类成员函数里写`int dp[len]`没有用`memset`初始化，以为它会自动初始化为0，结果看到它初始化为1，不是很大或很小的数，坑爹啊！

判断文件是否是符号链接，应该用

    (buf.st_mode & S_IFLNK) == S_IFLNK

因为`S_IFLNK`和`S_IFREG`有一位是重的，如果用
<pre>buf.st_mode & S_IFLNK</pre>
的话，普通文件也会被认为是符号链接文件。

APUE上判断文件类型是

    swicth(buf->st_mode & S_IFMT) {
      case S_IFREG: nreg++; break;
      case S_IFLNK: nslink++; break;
    }
    
用`stat`在符号链接文件上，一直想，为什么判断不出是符号链接，原来`stat`是跟随符号链接的，要用`lstat`才好。

