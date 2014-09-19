---
layout: post
title: "kids reread"
category: 
tags: []
---
{% include JB/setup %}

store是有topic的，如果message的topic和store不匹配，message会被抛弃。

拷贝构造函数声明为private并且不定义，来防止拷贝。

bufferstore并不是指把消息存在内存里，而是一个缓冲（采用deque实现），转发到primary或secondary。

filesystem.h里的File类提供了什么功能？有必要么？

##风格

* 数据成员名字后加下划线，下划线分隔：`buffer_type_`
* 类名大写开头，大写分隔：`class Store`
* 成员函数名与类命名一致：`bool PreAddMessage(const Message *msg)`
* struct成员末尾不加下划线：`struct StoreConfig`
* if等控制结构，函数定义，花括号开口在同一行

		bool Store::AddMessage(const Message *msg) {
  			if (PreAddMessage(msg)) {
    			LogDebug("pre check ok");
    			return DoAddMessage(msg);
  			}
  			return false;
		}
