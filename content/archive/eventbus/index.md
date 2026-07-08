---
title: "构建安全、实用、高性能、GC友好的事件系统"
date: 2026-07-02
draft: false
---

## 一、为什么要有事件中心

**事件中心**（Event Center / Event Bus）本质上是一个**全局的消息中转站**。它的核心存在意义只有两个字：**解耦（Decoupling）**

这篇文章记录我逐步构建一个完整事件系统的设计思路，最终版本具有以下特点：

- **零 GC** —— 热路径上不产生堆分配
- **类型安全** —— 编译期就能发现错误
- **线程安全** —— 网络回调、异步加载不会把底层数据结构撕烂
- **迭代安全** —— 回调里再订阅/退订不会崩溃
- **职责清晰** —— 数据从哪来、回收到哪去，泾渭分明
- **使用方便** —— 外部只需继承事件基类即可发布-订阅，无需碰事件中心

注：为了应对事件频繁的情况使用了对象池

## 二、事件的存储

事件中心存储事件监听者最简单的想法是每个事件对应一个枚举，定义一个EventID枚举，和IEventInfo接口统筹不同参数数量的事件，再在内部用一个`Dictionary<int, IEventInfo>`存储事件，但是这样存在一些让人难以接受的问题

1.手动维护一个枚举

2.编译期类型不安全，难调试

3.代码膨胀，每个参数数量的函数都要写一遍

于是有了存储事件监听者的另一个朴素想法：用一个 `Dictionary<Type, List<Action<T>>>` 存起来,同时让每个事件的类继承一个抽象类GameEvent，在发布时约束泛型为GameEvent保证类型安全。

```csharp
public abstract class GameEvent:IPooleable
{
       public abstract void OnRecycled();
}
```

但问题来了——每个 `T` 不同，`List<Action<JumpEvent>>` 和 `List<Action<DeadEvent>>` 是不同类型，没法塞进同一个 Dictionary 的值里。

于是有了第一版：**Dictionary + object**。

```csharp
Dictionary<Type, object> listenerMap= new();
//订阅
((List<Action<T>>)listenerMap[typeof(T)]).Add(listener);
//取消订阅
((List<Action<T>>)listenerMap[typeof(T)]).Remove(listener);
//发布事件
if (listenerMap.TryGetValue(typeof(T), out var obj))
{
     var actions = (List<Action<T>>)obj;
     foreach (var listener in actions)
     {
         listener?.Invoke(eventData);
     }
}
```

能跑，但每次订阅取消订阅操作都有一层 object 强转，发布时通过字典哈希查找。更关键的是——既然事件用类型本身标识，为什么不直接用类型系统定位？

于是有了新的方案，**“静态泛型嵌套类”**

**静态嵌套泛型类**利用了 CLR 的一条底层规则：当代码第一次访问 `SomeClass<int>` 时，CLR 会为 `int` 这门特定泛型参数编译出一份独立的静态类副本。`Registry<JumpEvent>` 和 `Registry<DeadEvent>` 在运行时是两套完全独立的内存空间，各带各的字段，互不干涉。

```csharp
// 每种事件独立一份静态空间，直接字段访问
private static class Registry<T> where T : GameEvent
{
    public static readonly List<Action<T>> ParamListeners = new List<Action<T>>();
    public static readonly List<Action> SignalListeners = new List<Action>();
    public static readonly object Lock = new object();
}

// 使用
Registry<T>.ParamListeners.Add(listener);  // 零查找，零强转
```

对比：

|        | Dictionary                     | 嵌套泛型        |
| ------ | ------------------------------ | ----------- |
| 定位方式   | O(1) 哈希 + 类型强转                 | 静态字段直接寻址    |
| 存储方式   | `object` 引用转换`List<Action<T>>` | 强类型，零转换     |
| 遍历所有事件 | 可 foreach                      | 不需要         |
| 内存     | 字典桶数组                          | 每个 T 一个静态类头 |

## 三、事件的发布与订阅

有了存储，先看看 EventBus 直接暴露的 API：

```csharp
EventBus.Subscribe<PlayerDeadEvent>(OnPlayerDead);
EventBus.Publish(evt);
EventBus.Unsubscribe<PlayerDeadEvent>(OnPlayerDead);
```

订阅时有一个容易踩的坑。为了让无参回调也能订阅，最开始的写法是：

```csharp
// Bug：每次生成新 Lambda → GC；退订时找不到 → 内存泄漏
public static void Subscribe<T>(Action listener) where T : GameEvent
{
    Subscribe<T>(_ => listener());
}
```

`_ => listener()` 每次调用都在堆上 new 一个新委托，`Unsubscribe` 时传入原始 `listener` 根本匹配不到。解决方案：**双通道分离**。

```csharp
// 有参委托和纯 Action 分两个 List 存，互不转化，各自能正确退订
public static readonly List<Action<T>> ParamListeners = new();
public static readonly List<Action> SignalListeners = new();
```

存储搞定了，但每次订阅都直接操作 `EventBus.Subscribe<T>(...)` 还是太麻烦，至少我本人讨厌这种写法，事件创建、填充、发布、回收这些步骤散落在调用方，容易出错。于是往上封装一层事件基类，分为有参数和无参数两类，将订阅，发布，取消订阅封装为静态函数，外部直接调用函数而不用去管事件中心

**SignalEvent（无参事件）** 

无参事件本身只是一个信号，这个信号完全可以重复使用，不被GC回收，因此给每个无参事件都做成单例，外部只需要Signal.Trigger()就可以发送事件

```csharp
public abstract class SignalEvent<T> : GameEvent where T : SignalEvent<T>, new()
{
    private static readonly T Instance = new T();

    public override void OnRecycled() { }

    public static void Trigger() => EventBus.PublishSignal(Instance);

    public static void Subscribe(Action listener) => EventBus.Subscribe<T>(listener);
    public static void Unsubscribe(Action listener) => EventBus.Unsubscribe<T>(listener);
}

// 使用：
PlayerJumpEvent.Subscribe(OnJump);
PlayerJumpEvent.Trigger();
PlayerJumpEvent.Unsubscribe(OnJump);
```

**ParameterizedEvent（参数化事件）** 

有参事件与无参的处理不同，因为数据填入后这个类的数据就变成了脏数据，应该进行回收，但如果高频事件场景频繁new()导致GC肯定是我们不希望看到的，因此用对象池进行回收

```csharp
public abstract class ParameterizedEvent<T> : GameEvent where T : ParameterizedEvent<T>, new()
{
    public abstract override void OnRecycled();

    public static void Trigger(Action<T> initializer)
    {
        var evt = ClassPool<T>.Get();
#if UNITY_EDITOR
        evt._recycled = false;
#endif
        initializer?.Invoke(evt);
        try { EventBus.Publish((T)evt); }
        finally
        {
#if UNITY_EDITOR
            evt._recycled = true;
#endif
            ClassPool<T>.Recycle(evt);
        }
    }

    public static void Subscribe(Action<T> listener) => EventBus.Subscribe<T>(listener);
    public static void Unsubscribe(Action<T> listener) => EventBus.Unsubscribe<T>(listener);
    public static void Subscribe(Action listener) => EventBus.Subscribe<T>(listener);
    public static void Unsubscribe(Action listener) => EventBus.Unsubscribe<T>(listener);
}

// 使用：
PlayerDeadEvent.Subscribe(OnPlayerDead);
PlayerDeadEvent.Trigger(e => { e.PlayerId = 1; e.Cause = "Lava"; });
PlayerDeadEvent.Unsubscribe(OnPlayerDead);
```

把 Subscribe/Unsubscribe/Trigger 收进事件类后，调用方跟 EventBus 完全解耦，只需要跟事件类型打交道：

```csharp
// 发布核心：锁外拷贝快照，锁外执行回调
var snapshot = ListPool<Action<T>>.Get();
snapshot.AddRange(actions);
foreach (var listener in snapshot)
    listener?.Invoke(eventData);
ListPool<Action<T>>.Recycle(snapshot);
```

## 四、事件生命周期的管理

参数化事件走对象池：`ClassPool.Get()` 取出 → 填数据 → `Publish` 分发 → `finally` 里 `ClassPool.Recycle()` 回收。正常情况下这套流程没问题。

隐患出在**异步持有**。如果某个监听者拿到事件数据后开了协程或异步调用：

```csharp
EventBus.Subscribe<PlayerDeadEvent>(evt =>
{
    StartCoroutine(DelayedLog(evt));  // evt 引用被协程持有
});

IEnumerator DelayedLog(PlayerDeadEvent evt)
{
    yield return new WaitForSeconds(2);
    Debug.Log(evt.PlayerId);  // 数据可能已经被覆盖了！
}
```

`finally` 在 Publish 返回时就回收了。2 秒后协程访问 evt，池里这个对象可能已被另一个事件复用，字段全是别人的数据。

**解决方案：两条路径。**

正常路径走池：

```csharp
PlayerDeadEvent.Trigger(e => { e.PlayerId = 1; });  // 自动回收
```

需要异步持有走 new，GC 兜底：

```csharp
// ParameterizedEvent 额外提供
public static void TriggerAsync(Action<T> initializer)
{
    var evt = new T();
    initializer?.Invoke(evt);
    EventBus.Publish((T)evt);
    // 不回收，等 GC
}
```

又或者提供另一个函数,让外部自行回收，这样异步高频事件场景的GC也得到了解决

```csharp
public static T TriggerAsync(Action<T> initializer)
{
    var evt =  ClassPool<T>.Get();
    initializer?.Invoke(evt);
    EventBus.Publish((T)evt);
    return evt;
}
```

加上 Editor 下回收标记辅助定位：

```csharp
// GameEvent 内
#if UNITY_EDITOR
internal bool _recycled;

[Conditional("UNITY_EDITOR")]
public void AssertSafe()
{
    if (_recycled) throw new InvalidOperationException("已被回收，不要异步持有");
}
#endif
```

属性里调 `AssertSafe()`，误用立刻崩在 Editor 里，不带到真机上。

## 五、多线程安全

游戏里多线程场景比想象中常见：网络层在后台线程收到服务器推送、寻路计算在 worker 线程返回结果——这些都可能触发事件。如果不做任何保护，多个线程同时读写 `List<Action<T>>` 会导致底层数组损坏、索引越界，甚至静默丢数据。

**锁分离（Lock Striping）**

直觉做法是给整个 EventBus 加一把全局锁：

```csharp
private static readonly object globalLock = new();
lock (globalLock) { Publish(evt); }
```

这能保证安全，但灾难性的性能瓶颈随之而来：假设 `NetworkPacketEvent` 的某个监听者回调耗时 100ms，在这期间 `JumpEvent`、`DeadEvent` 等所有其他事件的订阅和发布全被阻塞，整个游戏的事件系统彻底卡死。

锁分离的核心思想很简单：**每种事件类型给一把独立的锁**。

```csharp
private static class Registry<T> where T : GameEvent
{
    public static readonly object Lock = new(); // JumpEvent 和 DeadEvent 各有一把
}
```

`JumpEvent` 的订阅只锁 `Registry<JumpEvent>.Lock`，不影响 `DeadEvent` 的发布。不同类型之间完全并行。

**锁外回调（Lock-Outside Callback）**

就算是同一把锁，如果锁的持有时间太长（比如在锁内执行回调），同类型事件的多个线程还是会排长队。设计上把锁的持有压缩到极限——只在拷贝快照时加锁，Invoke 全在锁外：

```csharp
lock (Registry<T>.Lock)
{
    if (Registry<T>.ParamListeners.Count > 0)
    {
        snapshot = ListPool<Action<T>>.Get();
        snapshot.AddRange(Registry<T>.ParamListeners); // 只拷贝引用，极短
    }
}
// 锁已释放

foreach (var listener in snapshot)
    listener?.Invoke(eventData);  // 回调在锁外，多长时间都不阻塞其他线程
```

这还解决了一个隐藏问题：**死锁防御**。如果回调里有人调用 `Subscribe<T>` 或 `Unsubscribe<T>`，而锁内执行了回调，就会同一个线程对同一把锁尝试重入——`lock` 不是可重入锁，死锁。锁外回调彻底避免了这种场景。

总结一下这条设计链：

| 设计决策    | 解决的问题             |
| ------- | ----------------- |
| 每种事件独立锁 | 不同事件类型互不阻塞        |
| 锁只包快照拷贝 | 锁持有时间极短           |
| 回调在锁外执行 | 防死锁，不阻塞同事件类型的其他线程 |

## 六、总结

这套事件系统最终用到了以下几个核心技术点：

- **静态泛型 Registry** — CLR 为每个 T 独立编译，零字典零查找
- **双通道分离存储** — 有参/无参委托各自独立 List，退订直接匹配
- **ListPool 快照** — 池化复用内部数组，泛型迭代器是 struct，热路径零分配
- **ParameterizedEvent 自主回收** — 创建者负责回收，EventBus 只分发不碰生命周期
- **锁分离 + 锁外回调** — 不同事件类型互不阻塞，锁只维持快照拷贝的瞬间

Publish 一次事件（20 个监听者）：一次 lock，一次 ListPool Get/Recycle，20 个指针 AddRange 拷贝，20 次 delegate invoke。全部在 CPU cache 内完成，不触发一次 GC。

这篇文章讨论的是单机 Unity 环境下的实际权衡。在一个每帧几十个事件、每个事件几十个监听者的场景中，这套组合已经足够应对。把复杂度留给编译器，把简单留给使用者。

## 七、完整代码

### EventBus.cs

```csharp
using System;
using System.Collections.Generic;

namespace ZGameFramework.Core
{
    public static class EventBus
    {
        private static class EventRegistry<T> where T : GameEvent
        {
            public static readonly List<Action<T>> ParamListeners = new List<Action<T>>();
            public static readonly List<Action> SignalListeners = new List<Action>();
            public static readonly object Lock = new object();
        }

        public static void Subscribe<T>(Action<T> listener) where T : GameEvent
        {
            lock (EventRegistry<T>.Lock)
            {
                EventRegistry<T>.ParamListeners.Add(listener);
            }
        }

        public static void Subscribe<T>(Action listener) where T : GameEvent
        {
            lock (EventRegistry<T>.Lock)
            {
                EventRegistry<T>.SignalListeners.Add(listener);
            }
        }

        public static void Unsubscribe<T>(Action listener) where T : GameEvent
        {
            lock (EventRegistry<T>.Lock)
            {
                EventRegistry<T>.SignalListeners.Remove(listener);
            }
        }

        public static void Unsubscribe<T>(Action<T> listener) where T : GameEvent
        {
            lock (EventRegistry<T>.Lock)
            {
                EventRegistry<T>.ParamListeners.Remove(listener);
            }
        }

        public static void Publish<T>(T eventData) where T : GameEvent
        {
            List<Action<T>> paramSnapshot = null;
            List<Action> signalSnapshot = null;

            lock (EventRegistry<T>.Lock)
            {
                if (EventRegistry<T>.ParamListeners.Count > 0)
                {
                    paramSnapshot = ListPool<Action<T>>.Get();
                    paramSnapshot.AddRange(EventRegistry<T>.ParamListeners);
                }

                if (EventRegistry<T>.SignalListeners.Count > 0)
                {
                    signalSnapshot = ListPool<Action>.Get();
                    signalSnapshot.AddRange(EventRegistry<T>.SignalListeners);
                }
            }

            if (paramSnapshot != null)
            {
                foreach (var listener in paramSnapshot)
                    listener?.Invoke(eventData);
                ListPool<Action<T>>.Recycle(paramSnapshot);
            }

            if (signalSnapshot != null)
            {
                foreach (var listener in signalSnapshot)
                    listener?.Invoke();
                ListPool<Action>.Recycle(signalSnapshot);
            }
        }

        public static void PublishSignal<T>(T signal) where T : GameEvent
        {
            Publish(signal);
        }
    }
}
```

### GameEvent.cs

```csharp
using System;
using System.Diagnostics;

namespace ZGameFramework.Core
{
    public abstract class GameEvent : IPoolable
    {
        public abstract void OnRecycled();

#if UNITY_EDITOR
        internal bool _recycled;
        [Conditional("UNITY_EDITOR")]
        public void AssertSafe()
        {
            if (_recycled) throw new InvalidOperationException("已被回收，不要异步持有");
        }
#endif
    }
}
```

### SignalEvent.cs

```csharp
using System;

namespace ZGameFramework.Core
{
    public abstract class SignalEvent<T> : GameEvent where T : SignalEvent<T>, new()
    {
        private static readonly T Instance = new T();

        public override void OnRecycled() { }

        public static void Trigger()
        {
            EventBus.PublishSignal(Instance);
        }

        public static void Subscribe(Action listener) => EventBus.Subscribe<T>(listener);
        public static void Unsubscribe(Action listener) => EventBus.Unsubscribe<T>(listener);
    }
}
```

### ParameterizedEvent.cs

```csharp
using System;

namespace ZGameFramework.Core
{
    public abstract class ParameterizedEvent<T> : GameEvent where T : ParameterizedEvent<T>, new()
    {
        public abstract override void OnRecycled();

        public static void Trigger(Action<T> initializer)
        {
            var evt = ClassPool<T>.Get();
#if UNITY_EDITOR
            evt._recycled = false;
#endif
            initializer?.Invoke(evt);
            try
            {
                EventBus.Publish((T)evt);
            }
            finally
            {
#if UNITY_EDITOR
                evt._recycled = true;
#endif
                ClassPool<T>.Recycle(evt);
            }
        }

        public static void TriggerAsync(Action<T> initializer)
        {
            var evt = new T();
#if UNITY_EDITOR
            evt._recycled = false;
#endif
            initializer?.Invoke(evt);
            EventBus.Publish((T)evt);
        }

        /// <summary>
        /// 异步发布，外部必须手动回收,用于异步高频事件避免GC
        /// </summary>
        /// <param name="initializer"></param>
        public static T TriggerAsyncRecycle(Action<T> initializer)
        {
            var evt = ClassPool<T>.Get();
#if UNITY_EDITOR
            evt._recycled = false;
#endif
            initializer?.Invoke(evt);
            EventBus.Publish((T)evt);
            return evt;
        }

        public static void Subscribe(Action<T> listener) => EventBus.Subscribe<T>(listener);
        public static void Unsubscribe(Action<T> listener) => EventBus.Unsubscribe<T>(listener);
        public static void Subscribe(Action listener) => EventBus.Subscribe<T>(listener);
        public static void Unsubscribe(Action listener) => EventBus.Unsubscribe<T>(listener);
    }
}
```
