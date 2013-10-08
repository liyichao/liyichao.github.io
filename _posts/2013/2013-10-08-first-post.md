---
layout: post
title: "first post"
category: 
tags: []
---
{% include JB/setup %}

<p>
  在类成员函数里写
  <pre>int dp[len]</pre>
  没有memset初始化，以为它会自动初始化为0，
  结果看到它初始化为1，不是很大或很小的数，坑爹啊！
</p>

<p>
判断文件是否是符号链接，应该用
<pre>
(buf.st_mode & S_IFLNK) == S_IFLNK
</pre>
因为S_IFLNK和S_IFREG有一位是重的，如果用
<pre>buf.st_mode & S_IFLNK</pre>
的话，普通文件也会被认为是符号链接文件。
</p>

<p>
  APUE上判断文件类型是
  <pre>
    swicth(buf->st_mode & S_IFMT) {
    case S_IFREG: nreg++; break;
    case S_IFLNK: nslink++; break;
    }
    </pre>
</p>

<p>
  用stat在符号链接文件上，一直想，为什么判断不出是符号链接，原来stat是跟随符号链接的，要用lstat才好。
</p>
