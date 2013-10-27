---
layout: post
title: "Second Day and Third Day"
category: 
tags: []
---
{% include JB/setup %}

第二天上午写了一上午的博客，把10月25号下午及晚上想的以及已经写的代码都记录下来，下午睡觉然后踢了个球，晚上把应答系统`PhoneAnswer`类完全写好了，并且生成了`PriorityQueue`的测试程序`test_PriorityQueue`和`PhoneAnswer`的测试程序`test_PhoneAnswer`。

由于用到了`shared_ptr`，所以需要`-std=c++11`这个编译器选项，每次输很麻烦，就想到写个`Makefile`，