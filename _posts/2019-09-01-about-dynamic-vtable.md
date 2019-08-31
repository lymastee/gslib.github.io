---
layout: post
title: 动态虚表
date: '2015-12-18 21:45:10 +0800'
tags: dynamic vtable
categories: gslib
---
## 原理
用API hook的方式去实现动态虚表。有以下几个话题：

1. 如何确定vtable的大小
2. 如何确定vtable中每一个函数的地址和index
3. 虚表是per class的，但是实际需要的是per instance的
4. 为何需要从jump table跳转到vtable

## 如何确定vtable的大小

比较简单的方法就是写个类(class AUX_Target)，继承自目标类(class Target)，在最后附加一个虚函数(virtual void end_of_vtable())，然后为了保证这个函数不会被优化掉，在里面写上些不可能和已有的虚函数重合的实现，例如__asm nop。

然后获取end_of_vtable的地址，由该地址得到函数在虚表中的index，就是vtable的大小了。

## 如何确定vtable中每一个函数的地址和index

### 确定vtable中函数的地址

直接&TargetClass::func获得的地址一定是某个jump table中的地址，我们需要的是真实的函数体地址，因为跳转到该函数体地址的jump table可能有无数个，我们无法通过它在jump table中的地址确定两个函数究竟是否是同一个。

有一个能更接近函数体真实地址的写法，[this]()->void* { { __asm eax, TargetClass::TargetFunc } } 但是很遗憾，尽管它大部分时候返回的是正确的函数体地址，但仍然可能会返回jump table中的地址，并且更新vs2017最新update后这个写法会报一个错误，并且只能在x86下使用，x64禁止内联汇编。

所以我们需要从jump table中的地址解析起，通过对比机器码的特征，一直找到真实的函数体地址。

具体实现可以参见[dvt.cpp](https://github.com/lymastee/gslib/blob/master/src/gslib/dvt.cpp)里的uint dsm_get_address(uint addr, uint pthis);

### 确定函数的index

做法就是从虚表的第一个函数起，一直到虚表结尾，用每一个函数体的真实地址和该函数的真实地址对比，返回这个index。

具体实现参见[dvt.h](https://github.com/lymastee/gslib/blob/master/include/gslib/dvt.h)里的vtable_ops::get_virtual_method_index方法。

## 如何创建per instance虚表

步骤如下：

1. 分配虚表大小的内存，标记为executive（emmm，这一步骤ios应该做不到）
2. 拷贝原虚表（更好的方式应该是先构建自己的jump table，我们称之为bridge，将bridge中的地址拷贝过来用）

具体实现[dvt.h](https://github.com/lymastee/gslib/blob/master/include/gslib/dvt.h)里的vtable_ops::create_per_instance_vtable。

## 为何需要从jump table跳转到vtable

首先假设：

    class A
    {
    public:
        virtual void func1() {}
    };
    
    template<class _cls>
    class vtable_ops:
        public _cls
    {
    public:
        virtual void end_of_vtable() { __asm { nop } }
    };
    
    class B:
        public A
    {
    public:
        virtual void func1() override
        {
            printf("this is B");
            __super::func1();
        }
    };
    
这种情况，vtable_ops<B>::end_of_vtable()获取的尺寸认为B的虚表大小为1。但是执行B::func1的时候里面的__super::func1是相对跳转过去的，因此如果使用拷贝虚表的方法来创建per instance虚表，需要拷贝大小为2的虚表。实际情况远比这个例子复杂，如果一个类有复杂的继承关系，我们就必须借助一些工具来确定虚表的正确尺寸。这是不可取的。

因此我们需要构建自己的jump table来跳转回原始vtable。具体实现参见[dvt.cpp](https://github.com/lymastee/gslib/blob/master/include/gslib/dvt.cpp)中的dvt_bridge_code::install。

## 用途

做这个初衷是为了实现qt中类似connect slot/signal的效果的。但是又不想加入类似moc这样的过程。

目前的实现还算过得去，比较沙雕的一点就是connect_notify中最后一个参数必须要填写参数表的总大小，不知道怎么才能去掉。