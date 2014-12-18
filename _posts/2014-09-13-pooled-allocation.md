---
layout: post
title: "C++内存管理：自己实现内存池"
categories:
- C++
tags:
- memory management  
- pooled allocation
---
##目录
[TOC]
### 1. 说明
　　C++内存管理：自己实现内存池是C++内存管理系列第二篇,前一篇博客[原语篇](http://haoson.github.io/memory-primitives/)提到可以重载operator new和operator delete，如果class提供自己的member operator new和operator delete，便可以自己承担内存管理的责任，这也称为Per-class Allocation。

　　那么什么时候class需要提供自己的member operator new和operator delete呢？一般来说，如果某个类有大量小型对象的分配和释放，那么通常就要进行内存控制，定制自己的operator new和operator delete，定制版的性能通常这时候都会胜过缺省版本的性能。因为编译器自带的operator new和operator delete的实现主要是用于处理各种需求，如大块内存分配，小块内存分配，大小混合型内存分配等等，而且缺省版的实现还必须接纳各种分配形态，范围从程序活动期间的少量区块动态分配到大量短命对象的持续分配和归还。所以对于某些应用程序，某些class，定制自己的operator new和operator delete有助于获得程序的性能的提升。

　　这里主要focus在内存池这块，暂不考虑多线程问题。如果要做到线程安全，可以使用互斥量(mutex)，做法是用RAII手法封装mutex的创建、销毁、加锁、解锁这四个操作，不用手动调用lock()和unlock()函数，一切交给栈上封装互斥量的那个对象的构造和析构函数负责，这个对象的生命期正好等于临界区。


### 2. 代码
　　//to do
### 3. 分析
　　//to do
### 4. 改进
　　//to do
### 5. 其他
　　//to do
