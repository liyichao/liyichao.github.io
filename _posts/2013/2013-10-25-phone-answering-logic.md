---
layout: post
title: "应答逻辑"
category: 
tags: []
---
{% include JB/setup %}
拿到题目，粗看了下，第二题用Ruby或其他脚本语言的就可以了，输入用shell通配*.log就行，然后脚本内按行处理，可能要用正则表达式。先做第一题。
###应答逻辑
实现起来如果直接一点的话，直接一个`bool worker[20]`表示20个客服的忙闲状态就够了，客服告诉系统他离开就置他为忙，但是不计时，如果他本身就是忙的，就拒绝他的离开请求；告诉系统他回来了就置他为闲；接待客户的话就直接从头开始遍历找第一个空闲的，置为忙，同时这个客服的接待时间加1，如果没有找到空闲的，就告诉客户以后再打。

这样的话找空闲的客服时间就是`O(n)`的，这个操作应该很频繁，可能很重要，所以考虑别的实现。自然想到的是优先队列，可以用堆，想到Unix的进程调度，用了个链表维护就绪进程，链表实现比较简单，因此用链表按总接待时间从小到大的顺序存储空闲的客服。

原想`int id`直接表示客服就够了，但是要记录客服接待时间，与其再设一个用`id`索引的 `vector<int>
worked_time`，还不如把`id`和`worked_time`都整到一个结构体里面去，这样又有了一个数据类型`Worker`，同时把`next`指针也放进去，用来实现空闲客服链表，应答客服的请求时
需要根据`id`判断他是否空闲，为了不从头扫一遍空闲链表，`Worker`再放一个域`bool is_free`。

类名就叫`PhoneAnswer`吧，原想把所有结构都放到这个类里，但是又觉得，如果优先队列的实现想换一种的话就很麻烦，耦合的太严重了，还有编译依存性问题，为减小编译依存性，采用`Pimpl Idiom`，这样`PhoneAnswer`就只有一个`PhoneAnswerImpl`指针了，所有实现都delegate给类
`PhoneAnswerImpl`，只要`PhoneAnswerImpl`接口不变，`PhoneAnswer`不需要重新编译，用`PhoneAnswer`的用户也不需要重新编译，只要链接新的`PhoneAnswerImpl`就行，如果用动态库，链接都省了。除了这个，更相关的是《SICP》讲的让程序与数据表示无关的内容，于是把优先队列又从`PhoneAnswerImpl`里分出来，单
独成为`PriorityQueue`类，同时提供一个约定的接口，如取队首，把某个客服加入队列，把某个客服移除队列（应答客服的离开请求）等，这样下层的队列表示就可以变，可以用链表，也可以用堆。（为了实现方便把链表实现相关的`next`指针整进`Worker`结构了，有点不理想）

接下来发现一个大麻烦：`PhoneAnswerImpl`的构造函数。`N`个`Worker`肯定得先分配好，直接声明为`vector`搞定。但是`PriorityQueue`怎么构造？为实现方面，用了一个头结点，不存任何有效内容，这个要先分配好，但是在构造函数里面`new`东西的话，失败怎么办，查了下，`C++`可不是返回`NULL`，是抛出异常，还有资源泄漏这个讨厌的问题，以后还要记得`delete`，好烦啊，索性声明为`Worker m_head`，反正`Worker`是在`PriorityQueue`声明的，就不考虑什么编译依存性问题了，能不用指针就不用。但是它指向的内容呢？所有客服一开始都是空闲的，它指向第一个客服就行了，同时必须初始化空闲链表为所有客服，也就是把`m_all_worker`全连起来，同时把`m_all_worker[0]`的地址赋值给空闲链表头结点的`next`。有两种做法：

- 在`PhoneAnswerImpl`这样声明`PriorityQueue m_free_worker`，那`PriorityQueue`的初始化要在`PhoneAnswerImpl`构造函数里做，为了让`PhoneAnswerImpl`与`PriorityQueue`的下层数据表示无关，可以在`PriorityQueue`里定义`void init(vector<Worker>&all_worker)`，然后在`PhoneAnswerImpl`里调用它，同时把`m_all_worker`传进去，在`init`里要把所有客服链接起来，要修改`Worker`的`next`指针，所以传引用过去。这样的坏处就是在`PhoneAnswerImpl`必须`#incldue "PriorityQueue.hpp"`，好处是不用动态分配了。

- 在`PhoneAnswerImpl`这样声明`std::shared_ptr<PriorityQueue> m_free_worker`，这样只要有个前向声明`class PriorityQueue`就行了。并且定义`PriorityQueue(vector<Worker> &all_worker)`，我采用这种方法。

还有一个问题就是工作时间的上溢出问题，这个可以在服务器代码里用一个长度为24小时的计时器`alarm(60*60*24)`，然后在信号处理函数里把所有客服的已接待时间都清零。这应答系统有个不好的地方就是万一一个客服请求离开，然后再也不回来了话，系统没相应的处理。可以考虑把一天的总接待时间记录下来，然后跟薪水相关什么的，这个扩展还是在信号处理函数里做就行，在清零前`serialize`所有客服的接待时间就好了。系统对象`PhoneAnswer`声明为全局的，并且提供`clear_time()`函数，这样信号处理函数直接调用这个函数就好了。用信号的话，注意不要调用`malloc`之类的不可重入的函数，同时服务器的系统调用可能被中断，中断的系统调用是不是重启跟操作系统相关，暂时先不管了。另外，也可以用cron，定时，比如说每天0点，给服务器发送一个信号`SIGUSR1`，服务器在`SIGUSR1`的处理函数做清零操作。