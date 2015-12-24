---
layout: post
title: "C++内存管理：自己实现内存池"
categories:
- C++
tags:
- memory management  
- pooled allocation
---
- 目录
{:toc}

### 1. 说明
　　C++内存管理：自己实现内存池是C++内存管理系列第二篇,前一篇博客[原语篇](http://haoson.github.io/memory-primitives/)提到可以重载operator new和operator delete，如果class提供自己的member operator new和operator delete，便可以自己承担内存管理的责任，这也称为Per-class Allocation。
　　那么什么时候class需要提供自己的member operator new和operator delete呢？一般来说，如果某个类有大量小型对象的分配和释放，那么通常就要进行内存控制，定制自己的operator new和operator delete，定制版的性能通常这时候都会胜过缺省版本的性能。因为编译器自带的operator new和operator delete的实现主要是用于处理各种需求，如大块内存分配，小块内存分配，大小混合型内存分配等等，而且缺省版的实现还必须接纳各种分配形态，范围从程序活动期间的少量区块动态分配到大量短命对象的持续分配和归还。所以对于某些应用程序，某些class，定制自己的operator new和operator delete有助于获得程序的性能的提升。
　　这里主要focus在内存池这块，暂不考虑多线程问题。如果要做到线程安全，可以使用互斥量(mutex)，做法是用RAII手法封装mutex的创建、销毁、加锁、解锁这四个操作，不用手动调用lock()和unlock()函数，一切交给栈上封装互斥量的那个对象的构造和析构函数负责，这个对象的生命期正好等于临界区。C++11提供了[lock_guard](http://en.cppreference.com/w/cpp/thread/lock_guard)类就是这个作用。

### 2. 代码
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

### 3. 分析
　　上述代码为Foo类重载了operator new和operator delete，对象new的时候使用自由链表一次性分配多个对象(bulk allocation)，然后从"对象池"中取对象给用户，对象delete的时候不直接归还(Caching)，而是将归还的对象挂到自由链表头。
　　对于代码有一点简单说明：
　　[1]处判断size是否与sizeof(Foo)大小是否相同，operator new的参数是由编译器传过来的，为什么这里会出现不一致的情况呢？要注意的是member operator new/delete会被继承,对子类使用new运算符时，可能会调用父类中定义的operator new()来获取内存。但是，在这里，内存分配的大小，不应该是sizeof(父类)，而是sizeof(子类)。所以为了防止重载operator new/delete给子类带来的副作用，对于子类的内存分配，这里还是交给global operator new处理。
　　做了一个简单测试，性能上(在动态分配大量小对象的时候)确实有了一定提升

        Begin clock=30000 End clock=70000 1000000 OriginalFoo,take 40000clocks.
        Begin clock=70000 End clock=100000 1000000 Foo,take 30000clocks.

　　这里性能上的提升不仅仅是分配和回收的速度变快，也节省了一定的内存空间，前一篇博客原语篇提到new分配内存时，最终还是由malloc来分配，malloc分配空间的时候一般会在原始对象大小上再加一个"cookie"来记录对象大小，per-class allocation实质上是挖一大块空间来自己切割分配，这样当动态分配大量小对象的时候，小对象的"cookie"自然也就节省掉了。但是上述代码为了使用自由链表，引入了一个next指针，空间又浪费了，下面改进章节将会这个指针进行优化。
　　这段代码最大的缺点是永远不会释放空间，而且如果不断创建对象又不释放的话，自由链表会越长越大。如果引入变量记录相关信息，也许可以解决，不过也要考虑花费的代价值不值，毕竟重载operator new/delete就是为了应对特殊场合的，如何取舍还得看具体场景，也许某些场合下完全不释放空间不是问题。

### 4. 改进
　　前面提到为了使用自由链表，引入了一个next指针，这个指针就是为了将对象链起来，既然对象已经归还，那么是否可以借用对象的前4个bytes(32位系统上)来作为链接下一个对象的next指针呢？当然是可以的。所以上述代码可以稍加改造。

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

　　使用匿名union即完成上述改造，匿名union不用于定义对象,它的成员的名字出现在外围作用域中，也就是声明变量的作用，匿名union仅仅通知编译器它的成员变量共同享一个地址,而变量本身是直接引用的。通过引入匿名union，也就实现了上面说的借用对象本身的内存作为分配器的自由链表的分配指针的功能。
　　如果有很多类都需要重载member operator new的话，那么实现逻辑同上应该是基本一致，从代码的可重用和模块化的角度来说，我们应该将内存分配的功能封装起来，这样每个类就可以重用他们而不用管到底如何实现，这也体现了单一职责原则。下面实现一个不太复杂的固定大小的内存分配器,设计思想参考MFC的CFixedAlloc。

### 5. 固定大小的内存分配器

        //memory_pool.h
        struct MemoryPool{
            MemoryPool* next;//每次挖一大块内存，当使用完再挖一块，各大块之间通过next指针链接，组成一个自由链表
            void* data(){//每次挖出的一大块内存的前几个字节放next指针(等价于一个MemoryPool对象大小)，之后的空间才会分配出去
                return this+1;
            }
            static MemoryPool* create(MemoryPool*& head,size_t alloc_size,size_t block_size);
            void freeDataChain();//释放所有申请的空间
        };
        //memory_pool.cpp
        MemoryPool* MemoryPool::create(MemoryPool*& head,size_t alloc_size,size_t block_size){
            assert(alloc_size>0 && block_size>0);
            MemoryPool* p = reinterpret_cast<MemoryPool*>(new char[sizeof(MemoryPool)+alloc_size*block_size]);
            p->next = head;//新挖出的一大块内存入链
            head = p;//head作为自由链表的头
            return p;
        }
        void MemoryPool::freeDataChain(){
            MemoryPool* p = this;
            MemoryPool* temp;
            while(p){
                temp = p->next;
                char* cptr = reinterpret_cast<char*>(p);
                delete[] cptr;
                p = temp;
            }
        }

　　MemoryPool类只负责申请一大块内存并链接起来组成自由链表(把每次申请的一大块内存看做一个节点),create函数具体建立一条自由链表为特定大小(alloc_size)区块服务。

        //fixed_alloc.h
        #include"memory_pool.h"
        class FixedAlloc{
            public:
                FixedAlloc(size_t nAllocSize,size_t nBlockSize=32);
                size_t getAllocSize(){
                    return allocSize;
                }
                void* allocate();//分配allocSize大小的块
                void deallocate(void* p);//回收alloc出去的内存
                void deallocateAll();//释放所有从这个alloc分配出去的内存
                ~FixedAlloc();
            protected:
                union Obj{// [1]
                    union Obj* free_list_link;
                };
                size_t allocSize;//分配块的大小
                size_t blockSize;//每次申请的块的数目
                MemoryPool* blocks_head;//所有区块组成的自由链表头
                Obj* free_block_head;//未分配出去的区块头
        };
        //fixed_alloc.cpp
        FixedAlloc::FixedAlloc(size_t nAllocSize,size_t nBlockSize){
            assert(nAllocSize>=sizeof(Obj));
            assert(nBlockSize>1);
            //alloc的自由链表的设计思想是借用对象的前4个字节(32位系统上)来作为next指针链接block，如果对象本身大小比4个字节还小，那么必须向上调整对象大小
            if(nAllocSize<sizeof(Obj))
                nAllocSize = sizeof(Obj);
            if(nBlockSize<=1)//blockSize小于等于1让内存池失去了存在的意义
                nBlockSize = 32;
            allocSize = nAllocSize;
            blockSize = nBlockSize;
            blocks_head = nullptr;
            free_block_head = nullptr;
        }
        FixedAlloc::~FixedAlloc(){
            deallocateAll();//真正的归还
        }
        void FixedAlloc::deallocateAll(){
            blocks_head->freeDataChain();
            blocks_head = nullptr;
            free_block_head = nullptr;
        }
        void FixedAlloc::deallocate(void* p){
            if(p){//归还的block入链，挂在最前面
                Obj* objptr = static_cast<Obj*>(p);//归还回来的block里面数据已经没有用，直接覆盖，强制转型，作为入链的一个小节点
                objptr->free_list_link = free_block_head;
                free_block_head = objptr;
            }
        }
        void* FixedAlloc::allocate(){
            if(!free_block_head){//内存池中没有未分配的block
                MemoryPool* p = MemoryPool::create(blocks_head,blockSize,allocSize); 
                Obj* objptr = static_cast<Obj*>(p->data());//MemoryPool挖出的一大块的前几个字节是记录信息作用(一个MemoryPool对象)
                free_block_head = objptr;
                Obj* temp;
                for(size_t i=0;i!=blockSize-1;++i){//将所有block链起来
                    temp = reinterpret_cast<Obj*>(reinterpret_cast<char*>(objptr)+allocSize);//每一个入链的小节点大小都是allocSize
                    objptr->free_list_link =temp;
                    objptr = temp;
                }
                objptr->free_list_link = nullptr;//最后一个block的指针指向nullptr
            }
            void* p = free_block_head;//将第一个区块分配出去
            free_block_head = free_block_head->free_list_link;//free_block_head指向剩下的区块头
            return p;
        }

　　FixedAlloc类对外提供接口，FixedAlloc类需要的内存从MemoryPool类拿，MemoryPool类通过create函数申请一大块，FixedAlloc类拿到这一大块之后进行切割，形成一个自由链表，然后从这个自由链表中分配一块给用户。
　　fixed_alloc.h文件中有一处简单说明下，就是标注了[1]的地方。这个union对象的存在就是前面提到的，为了借用对象本身的内存来作为指向下一个小区块的指针，从而形成自由链表来进行内存管理。因为对象归还后，对象内是什么已经不重要了，所以我们直接覆盖对象数据，强制转型为Obj类型的指针指向这个区块。
　　做这么多主要还是为了把内存分配的功能封装起来，通过以上，这部分工作基本完成。那么怎么使用这个分配器呢？拿之前的Foo类继续做改造。

        //advanced_foo.h
        #include"fixed_alloc.h"
        class Foo{
            private:
                int i;
            public:
                Foo(int x):i(x){}
                int get(){
                    return i;
                }
                static void* operator new(size_t size);
                static void operator delete(void*p,size_t size);
            protected:
                static FixedAlloc alloc;
        };
        
        //advanced_foo.cpp
        FixedAlloc Foo::alloc(sizeof(Foo),64);
        void* Foo::operator new(size_t size){
            if(sizeof(Foo)!=size)
                return ::operator new(size);
            return alloc.allocate();
        }
        void Foo::operator delete(void*p,size_t size){
            if(p==nullptr)//C++保证删除null指针永远安全
                return;
            if(size!=sizeof(Foo)){//谁分配，谁归还（如果是global operator new分配的内存就应该由global operator delete归还）
                ::operator delete(p);
                return;
            }
            return alloc.deallocate(p);
        }

　　哇，果然设计上干净了跟多。具体的类只需要关心自己的要做什么就行了，内存管理的事已经交由alloc来做了。

　　做了一个简单测试，结果如下：

        ==4814== Memcheck, a memory error detector
        ==4814== Copyright (C) 2002-2012, and GNU GPL'd, by Julian Seward et al.
        ==4814== Using Valgrind-3.8.1 and LibVEX; rerun with -h for copyright info
        ==4814== Command: ./main
        ==4814== 
        Begin clock=660000 End clock=1370000 每次创建1000个Foo对象，然后删除，循环1000次，使用内存分配器的情况下，使用了 710000clocks.
        Begin clock=1380000 End clock=4370000 每次创建1000个Foo对象，然后删除，循环1000次，不使用内存分配器的情况下，使用了 2990000clocks.
        ==4814== 
        ==4814== HEAP SUMMARY:
        ==4814==     in use at exit: 0 bytes in 0 blocks
        ==4814==   total heap usage: 1,000,016 allocs, 1,000,016 frees, 4,004,160 bytes allocated
        ==4814== 
        ==4814== All heap blocks were freed -- no leaks are possible
        ==4814== 
        ==4814== For counts of detected and suppressed errors, rerun with: -v
        ==4814== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0) 

　　从上面的测试结果来看，首先没有内存泄露，程序总是应该先写对再去考虑性能的。。。其次，选用了一个简单的用例，从输出的两行来看，内存分配器还是起作用了，效果还行，更进一步的性能测试我就不做了。

### 6. 总结
　　这篇博客利用上一篇博客-原语篇讲到的重载member operator new/delete为特定类实现了内存管理，先给出了一个简单的per-class allocation的实现，然后一步步做了一些改进，最后实现了一个简单的内存池：固定大小的内存分配器。
