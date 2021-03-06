---
layout:     post
title:      "【Unity游戏框架搭建】二、消息系统"
subtitle:   "TeddyFrameWork"
date:       2019-08-20 11:00:00
author:     "Zero"
header-img: "img/post-bg-2015.jpg"
tags:
    - Unity
---

### 引言
1. 为什么要有消息系统呢？
> 解耦合
>- 举个例子，比如原始时代，你要告诉隔壁老王，下雨了该收衣服了，你就要跑到隔壁，拿到老王这个对象，才能当面告诉他，让他处理；
>- 现在，有了手机（消息系统），就可以不管老王长啥样？你可以发个消息告诉他，他手机收到这个消息，就可以做对应处理（收衣服）了。 


## C#层使用委托，Lua层实现消息系统

---

#### 基于delegate实现的Event（C# Event）

1. 首先讲一下Delegate,Delegates 是C#的一个语法。
>- Delegate是啥，中文翻译即“委托”
>- 在我看来，委托是一个类，把方法包起来，方便调用

2. 使用Delegate一般分为三步：
    1. 定义一种委托类型 
    1. 声明一个该类型的委托函数
    1. 通过声明的委托调用函数执行相关的操作

```
using UnityEngine;

public class DelegateExample : MonoBehaviour {
    //Step1. 为Delegate定义一种函数原型
    public delegate void eventHandler(int args);
    //Step2. 基于上面的委托定义事件someEvent
    public eventHandler someEvent;

    private void Start()
    {
        //Step3. 引用Delegate变量实现相应的函数
        someEvent = PrintNum;
        someEvent(5); //输入：Print number: 5

        someEvent = PrintDoubleNum;
        someEvent(5);//输入：Print number: 10
        
        someEvent += PrintNum;//输入：Print number: 10
                              //输入：Print number: 5 同时输出2条
    }

    public void PrintNum(int num)
    {
        print("Print number: " + num);
    }

    public void PrintDoubleNum(int num)
    {
        print("Print double number: " + num*2);
    }
	
}

```

通过运行上面的实例，我们发现someEvent每次赋值，就把指针指向不同的函数，可以得到不同的结果。
1. 如果说delegate是个类，那event就是这个委托的实例，
    比如delegate是指做任何事，那event就可以是具体的一件事，比如吃饭、洗澡等真正的事。
2. 上面的代码是为了方便观看，真正应用的时候，我们通常会遇到
>- 比如战斗模块(A)需要角色对象(B)播放大招，
>- 这时候如果直接取到B对象来调用，就会产生严重耦合，A和B模块就被我们黏到了一起，分不开了。
>- 为了解耦合，我们可以在角色模块监听用户的点击事件，然后战斗模块发出用户点击事件，角色模块收到消息，
就可以完成大招的释放，这样就能把A、B模块分开了。

```C#

using UnityEngine;

// 事件定义类
public class GameEventDefine{
    public delegate void TouchHandler();
    public TouchHandler On_Touch;
}

// 英雄举例类(接受者)
public class HeroExample : MonoBehaviour {
    private void Start()
    {
        GameEventDefine.On_Touch = DoSomeThing;
    }
    private void DoSomeThing(){
        print("DoSomeThing");
    }
}

// 战斗举例类(发送者)
public class BattleExample{
    private void Start()
    {
        // 假装用户点击了
        GameEventDefine.On_Touch();
    }
}

```

#### Lua层实现消息系统
直接上代码（md支持不太好，一直页面崩溃）[详情可以看我同事写的文章](https://www.cnblogs.com/wang-jin-fu/p/11255831.html)

1. 用字典维护了一张注册信息列表Messagelisteners，记录注册对象target和触发回调callBack
2. 注册事件时候，需要在GMsgDef定义自己的事件名，然后调用GameMsg.AddMessage(msgName, target, callBack)来注册事件
3. 触发事件调用GameMsg.SendMessage(msgName, ...)，就可以完成事件的跨模块通知。
4. 这里用到了一个对象池概念，我们将在下一章讲解.
