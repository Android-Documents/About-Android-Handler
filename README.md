# 关于Handler的分析
关于handler机制的解析

##写在前面


大家久等了，前阵花了一周的时间去毕业旅行，所以更新就拖延了一阵，话不多说，我们来回顾一下本系列的前两篇文章的思路和知识点：

### Android消息机制字典型探究（一）

在第一篇文章中，我们总结了Android系统不允许在子线程更新UI的原因，本质上是线程安全问题，从而引出了Handler。

### Android消息机制字典型探究（二）

在第二篇文章中，我们又分析了三种在子线程更新UI的方法，分别是：**View.post(param); Activity.runOnUIThread(param); Handler**，当我们对这三种方法的源码进一步分析发现，其实都是对**Handler**做了一些封装，所以本文我们就来正式全面探究有关Handler的知识点。

当时我去面试的四家公司，都问到了**Handler**的相关知识，有深有浅，所以重要程度不言而喻。面试官拿起你的简历，让你谈谈**Handler**，你仅仅在表象上回答了Android线程通信的机理，然后面试官紧接着问了你如下的几个问题：

- Handler是属于哪个类的？
- Handler、Looper、MessageQueue何时建立的相互关系？
- 主线程的Looper和MessageQueue是何时创建的？
- 在同一线程中，Looper和MessageQueue是怎样的数量对应关系，与Handler又是怎样的数量对应关系？
- MessageQueue中消息为空，线程阻塞挂起等待，为什么不会造成ANR？
- 有关Handler的内存泄漏是怎么一回事？


一脸萌比

so...光知道表象很可能是不够的，而且还给自己挖了一个坑，所以我们对于一个知识点的探寻要全面充分一点。下面正式开始本文。


## Windows和Android消息机制的区别


现在的操作系统普遍采用消息驱动模式。Windows操作系统就是典型的消息驱动模型。但是，Android的消息处理机制和Windows的消息处理机制又不太相同。我给大家画了图，看看二者的区别。

![image](/windows进程消息模型.png)

Windows进程消息模型

![image](/android进程消息模型.png)

Android进程消息模型

通过消息机制图的对比，Windows消息处理模型中，存在一个系统的消息队列，这个队列是整个进程的核心，几乎所有的动作都要转换成消息，然后放到这个队列中，由主线程统一处理。

而Android没有全局的消息队列，消息队列是和某个线程相关联在一起的。每个线程最多有一个消息队列，消息的取出和处理，也在这个线程本身中完成。

也就是说，Android中，如果你想在当前线程使用消息模型，则必须构建一个消息队列，而消息机制的相关主要类是：**Looper、Handler、MessageQueue、Message。
**

我们并不着急去翻看这些类的源码，理清楚底层实现的逻辑，而且先在宏观表象上看看，Android消息机制是如何运行的？


## Android消息机制的宏观原理


先来看一张Android消息处理类之间的关系图

![image](/Android消息处理机制.png)

Android消息处理机制

我们从表象上解释一下原理，Handler负责将Message发送至当前线程的MessageQueue中，Looper时时刻刻监视着MessageQueue，将符合时间要求的Message取出，再带给发送消息的那个Handler通过HandleMessage处理。

对于消息机制的理解不能仅仅停留在这一步，下面我们从源码的角度分析一下具体的逻辑细节。


## Android消息机制相关类的源码分析


### Message

人在边境X（子线程）服役的士兵Message慵懒得躺在一个人数为50（池中最大数量）的军营（Message池）中。不料这时突然接到了上司的obtain()命令（据说obtain命令更加节省军费），让他去首都（主线程）告诉中央领导一些神秘代码。小Message慌乱地整理了下衣角和帽子，带上信封，准备出发。

上司让士兵Message收拾完毕之后等待一个神秘人的电话，并且嘱咐他：到了首都之后，0是这次任务的暗号。

![image](/Message的创建和携带信息.png)

Message的创建和携带信息

Message是消息的载体，Message设计成为**Parcelable**类的派生类，这表明Message可以通过**binder**来跨进程发送。

通常我们都会用**obtain()**方法去创建Message，如果消息池中有Message有，则取出，没有，再重新创建。这样可以防止对象的重复创建，节省资源。

![image](/Message的创建和携带信息.png)

obtain方法源码

"铃铃铃..."小Message接到了一个陌生男子的电话。
“我叫handler，来自activity大本营，是你这次任务的接受者，一会我带你去首都的消息中心去报道。”

### Handler

来自Activity大本营Handler部门是整个消息机制系统的核心部门，当然部门下有很多个 Handler，这次协助小Message任务的叫mHandler。Handler部门下的员工都有一个特点，就是只关心自己的message。

Handler属于Activity，创建任何一个Handler都属于重写了Activity中的Handler。

![image](/Activity中定义了Handler.png)

Activity中定义了Handler

在Handler的构造中，默认完成了对当前线程Looper的绑定，至于Looper是谁，一会再谈。

![image](/Handler的构造方法.png)

Handler的构造方法

通过Looper.myLooper()获取了当前线程保存的Looper实例，又通过mLooper.mQueue获取了Looper中的MessageQueue实例。在此时，mhandler实例与looper和messageQueue实例，关联上了。

mHandler神情骄傲得对小Message说：我已经跟首都的消息中心打好了招呼，准备接收你了，现在有两种车，一种车名叫“send”，一种叫“post”，你想坐哪辆去首都都可以，不过要根据你上司的命令，选择车种类下对应的型号哦~

- send

    ![image](/send.png)

- post

    ![image](/psot.png)

从代码的实现上来看，post方法也是在使用send类的方法在发送消息，只是他们的参数要求是Runnable对象。

通过对Handler源码的分析，发现除了sendMessageAtFrontOfQueue方法之外，其余任何send的相关方法，都经过层层包装走到了sendMessageAtTime方法中，我们来看看源码：

![image](/sendMessageAtTime源码.png)

sendMessageAtTime源码

这时小Message和mHandler一同上了车牌号为“sendMessage”的车，行驶在一条叫“enqueueMessage”的高速公路上，mHandler向一无所知的小Message介绍说，每个像他一样的Message都是通过enqueueMessage路进入MessageQueue的。我们是要去首都的MessageQueue中心，其实你的消息到时候也是我处理的，不过现在还不是时候哦，因为我很忙。

![image](/enqueueMessage源码.png)

enqueueMessage源码

enqueueMessage是MessageQueue的方法，用来将Message根据时间排序，放入到MessageQueue中。其中msg.target = this，是保证每个发送Message的Handler也能处理这个Message。

### Looper

路上的时间不短不长，mHandler依然为小Message热心介绍着MessageQueue和Looper

“在每个驻扎地（线程）中，只有一个MessageQueue和一个Looper，他们两个是相杀相爱，同生共死的好基友，Looper是个跑不死的邮差，一直负责取出MessageQueue中的Message”

"不过通常只有首都（主线程）的Looper和MessageQueue是创建好的，其他地方需要我们人为地创建哦~"

![image](/prepare方法.png)

prepare方法

Looper类提供了prepare方法来创建Looper。可以看到，当重复创建Looper时，会抛出异常，也就是说，每个线程只有一个Looper。

![image](/Looper构造.png)

Looper构造

紧接着在Looper的构造方法中，又创建了与它一一对应的MessageQueue，既然Looper在一个线程中是唯一的，所以MessageQueue也是唯一的。

在Android中，ActivityThread的main方法是程序的入口，主线程的Looper和MessageQueue就是在此时创建的。

![image](/ActivityThread的main方法.png)

ActivityThread的main方法

可以看到，在main方法中，既创建了Looper，也调用了Looper.loop()方法。

mHandler和小Message通过enqueueMessage路来到了MessageQueue中，进入之前，门卫仔仔细细地给小Message贴上了以下标签：
“mHandler负责带入”
“处理时间为0ms”
并且告诉小Message，一定要按照时间顺序排队。
进入队伍中，Looper大哥正在不辞辛劳的将一个又一个跟小Message一样的士兵带走。
![image](/loop方法.png)

loop方法

分析一下loop方法，有一个for的死循环，不断地调用queue.next方法，在消息队列中取Message。并且在Message中取出target，这个target其实就是发送消息的handler，调用它的dispatchMessage方法。

首都的MessageQueue中心虽然人很多，但是大家都井井有条的排着队伍，Looper老哥看了一眼手里的名单，叫到了小Message的名字，看了一眼小Message身上的标签，对他说：“喔，又是mHandler带来的人啊，那把你交给他处理了”

忐忑不安的小Message看到了一个熟悉的身影，mHandler就在面前，显然mHandler有些健忘，可能是接触了太多跟小Message一样的人，为了让mHandler想起自己，小Message说出了上司交给他的暗号0.

![image](/dispatchMessage方法.png)

dispatchMessage方法

可以看见dispatchMessage方法中的逻辑比较简单，具体就是如果mCallback不为空，则调用mCallback的handleMessage()方法，否则直接调用Handler的handleMessage()方法，并将消息对象作为参数传递过去。
在handlerMessage()方法中，小Message出色的完成了自己的任务。

转载：[http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661863&idx=1&sn=ca3012bfe27cd0799174a015aa60c4cf&scene=23&srcid=0607ugAnbl2ZbnmSpT7CAlxk#rd](http://mp.weixin.qq.com/s?__biz=MzA4NTQwNDcyMA==&mid=2650661863&idx=1&sn=ca3012bfe27cd0799174a015aa60c4cf&scene=23&srcid=0607ugAnbl2ZbnmSpT7CAlxk#rd)
