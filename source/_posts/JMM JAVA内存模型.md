---
title: JMM JAVA内存模型
---

要想深入了解Java并发编程，就要先理解好Java内存模型，而要理解Java内存模型又不得不从硬件、计算机内存模型说起

## **CPU执行过程**

计算机在执行程序时，每条指令都是在CPU中执行的，而执行的时候，又免不了要和数据打交道，而计算机上面的临时数据，是储存在主存中的。计算机内存包括高速缓存和主存。我们知道CPU执行指令的速度比从主存读取数据和向主存写入数据快很多，所以为了高效利用CPU，CPU增加了高速缓存(cache)来匹配CPU的执行速度，最终程序的执行过程如下

1. 首先会将数据从主存中复制一份到CPU的高速缓存中
2. 当CPU执行计算的时候就可以直接从高速缓存中读取数据和写入数据
3. 当运算结束后，再将高速缓存的数据更新到主存中

## **缓存一致性问题**

上面的执行过程在单线程情况下并没有问题，但是在多线程情况下就会出现问题，因为CPU如果含有多个核心，则每个核心都有自己独占高速缓存，如果出现多个线程同时执行同一个操作，那么结果是无法预知。例如2个线程同时执行i++，假设i的初始值是0，那么我们希望2个线程执行完成之后i的值变为2，但是事实会是这样吗？

![waYNGT.jpg](https://s1.ax1x.com/2020/09/12/waYNGT.jpg)

可能出现的情况有：

1. 线程1先将i=0从主存中读取到线程1的高速缓存中，然后CPU完成运算，再将i=1写入到主存中，然后线程2开始从主存中读取i=1到线程2的高速缓存中，然后CPU完成运算，再将i=2写入到主存中，那么i=2即为我们想要的结果。
2. 线程1将i=0从主存中读取到线程1的高速缓存中的同时线程2也从主存中读取i=0到线程2的高速缓存中，然后线程1和线程2完成运算后，也都将i=1写入到主存中，那么结果i=1，结果就不是我们想要的了。出现这个情况，我们称为缓存不一致问题。

那么如何解决CPU出现的缓存不一致问题呢？通常使用的解决方法有2种：

1. 通过给总线加锁
2. 使用缓存一致性协议

[![watpoq.jpg](https://s1.ax1x.com/2020/09/12/watpoq.jpg)](https://imgchr.com/i/watpoq)

第1种方法虽然也达到了目的，但是在总线被锁住的期间，其他的CPU也无法访问主存，效率很低，所以就出现了缓存一致性协议即第2种方法，其中最出名的就是Intel的MESI协议，MESI协议保证每个CPU高速缓存中的变量都是一致的。它的核心思想是，当CPU写数据时候，如果发现操作的变量是共享变量(即其他CPU上也存在该变量)，就会发出信号通知其他CPU将它高速缓存中缓存这个变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己高速缓存中缓存该变量的缓存行为无效状态，那么它就会从主存中重新读取。

[![watmwR.jpg](https://s1.ax1x.com/2020/09/12/watmwR.jpg)](https://imgchr.com/i/watmwR)

## **处理器重排序问题**

在多线程场景下，CPU除了会出现缓存一致性问题，还会出现因为处理器重排序即处理器(CPU)为了提高效率可能会对输入的代码进行乱序执行，而造成多线程的情况下出现问题。

例如：

```
//线程1:
context = loadContext(); //语句1
inited = true; //语句2
//线程2: 
while(!inited ){
	sleep()
}
doSomethingwithconfig(context);
```

线程1由于处理器重排序，先执行性了语句2，那么此时线程2会认为context已经初始化完成，那么跳出循环，去执行doSomethingwithconfig(context)方法，实际上此时context并未初始化(即线程1的语句1还未执行)，而导致程序出错。

## **什么是计算机内存模型**

上面提到的缓存一致性问题、处理器重排序问题都是在多线程情况下CPU可能出现的问题，那我们应该怎么处理这些问题？

> 可见性即当一个变量修改后，这个变量会马上更新到主存中，其他线程会收到通知这个变量修改过了，使用这个变量的时候重新去主存获取

## **什么是Java内存模型**

从前面的介绍了解到计算机内存模型是一种解决多线程场景下的一个主存操作规范，既然是规范，那么不同的编程语言都可以遵循这种操作规范，在多线程场景下访问主存保证原子性、可见性、有序性。

Java内存模型(Java Memory Model，JMM)即是Java语言对这个操作规范的遵循，JMM规定了所有的变量都存储在主存中，每个线程都有自己的工作区，线程将使用到的变量从主存中复制一份到自己的工作区，线程对变量的所有操作(读取、赋值等)都必须在工作区，不同的线程也无法直接访问对方工作区，线程之间的消息传递都需要通过主存来完成。可以把这里主存类比成计算机内存模型中的主存，工作区类比成计算机内存模型中的高速缓存。

![waNmjg.jpg](https://s1.ax1x.com/2020/09/12/waNmjg.jpg)

而我们知道JMM其实是工作主存中的，Java内存模型中的工作区也是主存中的一部分，所以可以这样说Java内存模型解决的是内存一致性问题(主存和主存)而计算机内存模型解决的是缓存一致性问题(CPU高速缓存和主存)，这两个模型类似，但是作用域不一样，Java内存模型保证的是主存和主存之间的原子性、可见性、有序性，而计算机内存模型保证的是CPU高速缓存和主存之间的原子性、可见性、有序性。

[![waanTs.jpg](https://s1.ax1x.com/2020/09/12/waanTs.jpg)](https://imgchr.com/i/waanTs)

