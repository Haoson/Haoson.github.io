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

　　这里主要focus在内存池这块，暂不考虑多线程问题。如果要做到线程安全，可以使用互斥量(mutex)，做法是用RAII手法封装mutex的创建、销毁、加锁、解锁这四个操作，不用手动调用lock()和unlock()函数，一切交给栈上封装互斥量的那个对象的构造和析构函数负责，这个对象的生命期正好等于临界区。C++11提供了[lock_guard](http://en.cppreference.com/w/cpp/thread/lock_guard)类就是这个作用。


### 2. 代码
`````C++
    #include<cstddef>
    class Foo{
        private:
            int i;
        public:
            Foo(int x):i(x){}
            int get(){
                return i;
            }
            static void* operator new(size_t);
            static void operator delete(void*,size_t);
        private:
            static Foo* free_list;
            static const int foo_chunk=24;
            Foo* next;
    };
`````

`````C++
    Foo* Foo::free_list = nullptr;
    void* Foo::operator new(size_t size){
        if(size!=sizeof(Foo)){//如果大小出错，分配内存动作交给global operator new [1]
            return ::operator new(size);
        }
        Foo* p;
        if(!free_list){//当自由链表为空或者用光的时候，再创建一条自由链表
            size_t chunk = foo_chunk*size;
            free_list = p = reinterpret_cast<Foo*>(new char[chunk]);
            while((p++)!=free_list+chunk-1){//将分配得到的所有对象链起来
                p->next = p+1;
            }
            p->next = nullptr;
        }
        p = free_list;
        free_list = free_list->next;//分配出自由链表上的第一个对象
        return p;
    }

    void Foo::operator delete(void* ptr,size_t size){
        if(ptr==nullptr)//C++保证删除null指针永远安全
            return;
        if(size!=sizeof(Foo)){//谁分配，谁归还（如果是global operator new分配的内存就应该由global operator delete归还）
            ::operator delete(ptr);
            return;
        }
        Foo* p = static_cast<Foo*>(ptr);
        p->next  = free_list;//未实际释放空间，只是重新将归还的对象挂到free list上
        free_list = p;
    }
`````

### 3. 分析
　　上述代码为Foo类重载了operator new和operator delete，对象new的时候使用自由链表一次性分配多个对象(bulk allocation)，然后从"对象池"中取对象给用户，对象delete的时候不直接归还(Caching)，而是将归还的对象挂到自由链表头。

　　对于代码有两点简单说明：<br>
　　1. [1]处判断size是否与sizeof(Foo)大小是否相同，operator new的参数是由编译器传过来的，为什么这里会出现不一致的情况呢？要注意的是member operator new/delete会被继承

　　做了一个简单测试，性能上(在动态分配大量小对象的时候)确实有了一定提升

`````C++
    Begin clock=30000 End clock=70000 1000000 OriginalFoo,take 40000clocks.
    Begin clock=70000 End clock=100000 1000000 Foo,take 30000clocks.
`````

　　
### 4. 改进
　　//to do
### 5. 其他
　　//to do
