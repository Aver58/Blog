---
layout:     post
title:      "【Unity游戏框架搭建】三、对象池实现"
subtitle:   "TeddyFrameWork"
date:       2019-08-20 11:00:00
author:     "Zero"
header-img: "img/post-bg-2015.jpg"
tags:
    - Unity
---

### [对象池模式](https://gpp.tkchu.me/object-pool.html)
1. 为啥要对象池？
> 比如我们英雄拿个格林机关枪扫射，动则上百的子弹，这样就要生成上百个对象，
但实际，同时展示在屏幕上的子弹大概就10-20个，其他的都消失了，为了减少内存碎片，
我们可以把消失的对象拿出来复用，这就是我们想要实现的对象池。

2. 在Unity中我们经常会用到对象池，使用对象池无非就是解决两个问题:
>1. 减少new时候寻址造成的消耗，该消耗的原因是内存碎片。
>1. 减少Object.Instantiate时内部进行序列化和反序列化而造成的CPU消耗。

```C#

using System.Collections.Generic;
using Debugger = LuaInterface.Debugger;

// where T:new()指明了创建T的实例时应该具有构造函数。
public class ObjectPool<T> where T:new()
{
    private readonly Stack<T> m_Stack = new Stack<T>();

    public int objectCount { get; private set; }
    # 取出一个对象，池子没有库存没有就创建，有的话就弹出堆栈
    public T Get()
    {
        T element;
        if(m_Stack.Count == 0)
        {
            element = new T();
            objectCount++;
        }
        else
        {
            element = m_Stack.Pop();
        }
        return element;
    }

    # 用完对象，扔回池子，方便复用
    public void Release(T element)
    {
        if(m_Stack.Count > 0 && ReferenceEquals(m_Stack.Peek(), element))
            Debugger.LogError("【ObjectPool.Release】error：Trying to destroy object that is already released to pool.");
        m_Stack.Push(element);
    }
}

```