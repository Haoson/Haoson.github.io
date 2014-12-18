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
            Foo* next; //[2]
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

　　对于代码有一点简单说明：<br>
　　1. [1]处判断size是否与sizeof(Foo)大小是否相同，operator new的参数是由编译器传过来的，为什么这里会出现不一致的情况呢？要注意的是member operator new/delete会被继承,对子类使用new运算符时，可能会调用父类中定义的operator new()来获取内存。但是，在这里，内存分配的大小，不应该是sizeof(父类)，而是sizeof(子类)。所以为了防止重载operator new/delete给子类带来的副作用，对于子类的内存分配，这里还是交给global operator new处理。<br>

　　做了一个简单测试，性能上(在动态分配大量小对象的时候)确实有了一定提升

`````C++
    Begin clock=30000 End clock=70000 1000000 OriginalFoo,take 40000clocks.
    Begin clock=70000 End clock=100000 1000000 Foo,take 30000clocks.
`````
　　这里性能上的提升不仅仅是分配和回收的速度变快，也节省了一定的内存空间，前一篇博客原语篇提到new分配内存时，最终还是由malloc来分配，malloc分配空间的时候一般会在原始对象大小上再加一个"cookie"来记录对象大小，per-class allocation实质上是挖一大块空间来自己切割分配，这样当动态分配大量小对象的时候，小对象的"cookie"自然也就节省掉了。但是上述代码为了使用自由链表，引入了一个next指针，空间又浪费了，下面改进章节将会这个指针进行优化。

　　这段代码最大的缺点是永远不会释放空间，而且如果不断创建对象又不释放的话，自由链表会越长越大。如果引入变量记录相关信息，也许可以解决，不过也要考虑花费的代价值不值，毕竟重载operator new/delete就是为了应对特殊场合的，如何取舍还得看具体场景，也许某些场合下完全不释放空间不是问题。
　　
### 4. 改进
　　1. 前面提到为了使用自由链表，引入了一个next指针，这个指针就是为了将对象链起来，既然对象已经归还，那么是否可以借用对象的前4个bytes(32位系统上)来作为链接下一个对象的next指针呢？当然是可以的。所以上述代码可以稍加改造。

`````C++
    #include<cstddef>
    class Foo{
        private:
            union{//匿名union [1]
                int i;
                Foo* next;
            };
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
    };
`````

　　使用匿名union即完成上述改造，匿名union不用于定义对象,它的成员的名字出现在外围作用域中，也就是声明变量的作用，匿名union仅仅通知编译器它的成员变量共同享一个地址,而变量本身是直接引用的。通过引入匿名union，也就实现了上面说的借用对象本身的内存作为分配器的自由链表的分配指针的功能。

　　2. 如果有很多类都需要重载member operator new的话，那么实现逻辑同上应该是基本一致，从代码的可重用和模块化的角度来说，我们应该将内存分配的功能封装起来，这样每个类就可以重用他们而不用管到底如何实现，这也体现了单一职责原则。改进如下：

`````C++
    
`````



### 5. 其他
　　//to do
