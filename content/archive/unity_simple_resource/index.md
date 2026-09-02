---
title: "Unity框架之带有引用计数的Resources模块"
date: 2026-09-02
draft: false
---

## 一、为什么要有资源加载模块

  unity自带的Resources.Load()最简单易上手，但是会遇到以下问题

- **重复加载** 同一个 prefab 被十个系统各自 `Load` 一次，谁也不知道别人加载过没

- **释放失控**：Resources 体系没有"谁还在用"的概念，卸载全靠 `Resources.UnloadUnusedAssets()` 兜底，而它只释放**完全没有任何引用**的资源，只要有脚本忘了置空引用，资源就永远躺在内存里

- **生命周期不明**：没有任何机制回答"这个资源现在还有没有人用"

- **异步回调地狱**：`LoadAsync` 的回调写法层层嵌套，取消加载更是想都别想

  这篇文章记录我构建一个带引用计数的 Resources 管理模块的完整思路，因为该模块用来我自己制作游戏，Resources暂时够用所以没有引入AB包，但是该模块骨架完整，拓展AB包部分也比较容易，可以自行尝试。 完整代码在最后。

- **引用计数** —— 谁加载谁负责，计数归零即可回收

- **并发去重** —— 同一资源的多次异步加载只发一次请求

- **同步异步统一缓存** —— 底层同一份 ResBase，上层爱用哪个用哪个

- **取消安全** —— token 取消不留脏状态，下次加载自动重试

- **Loader 池化** —— 加载器本体走对象池，热路径零 GC

- **一键清场** —— 切场景后一个 `UnloadUnused()` 清干净

 <br>

## 二、缓存条目 ResBase

  整个模块的最小部分是ResBase，用于记录资源的加载信息

  主要记录：资产是什么，资产加载状态，资产引用状态

```csharp
public class ResBase
{
    /// <summary>缓存表唯一 key，避免同路径不同文件</summary>
    public string Key { get; private set; }
    /// <summary>Resources.Load 用的原始路径</summary>
    public string Path { get; private set; }
    /// <summary>加载完成后的资源</summary>
    public Object Asset { get; private set; }

    public int RefCount { get; private set; } = 0;

    public bool IsLoaded => Asset != null;
    public bool IsLoading => loadingRequest != null;
    public bool NeedDel=> RefCount <= 0;

    /// <summary>进行中的异步加载请求，防止同一资源并发重复加载</summary>
    private ResourceRequest loadingRequest;
}
```

**RefCount 的定义**：refcount不是有多少物体在显示这个资源，而是被持有的状态，0则代表没有任何人引用，缓存表可以在合适的时机清理出去节省空间

我们需要为ResBase提供两个方法，使其可以同步/异步加载资源，这里是加载调用链路的尽头

```csharp
        /// <summary>
        /// 同步加载资源，返回资源对象。若资源已加载则直接返回缓存对象。
        /// </summary>
        public T LoadSync<T>() where T : Object
        {
            if (Asset is T loaded) return loaded;

            if (Asset != null)
            {
                Debug.LogWarning($"[Res] 资源类型不匹配！{Key} 已加载为 " +
                    $"{Asset.GetType().Name}，尝试加载为 {typeof(T).Name}");
            }

            if (loadingRequest != null)
            {
                Debug.LogWarning($"[Res] 资源正在异步加载中，请保持加载方式相同{Key}");
            }

            Asset = Resources.Load<T>(Path);
            loadingRequest = null;

            if (Asset == null)
            {
                Debug.LogWarning($"[Res] 资源加载失败！{Key}");
            }
            return Asset as T;
        }

        /// <summary>
        /// 异步加载，并发调用复用同一次加载；token 取消时安全重置状态
        /// </summary>
        public async UniTask<T> LoadAsync<T>(CancellationToken token =default)where T : Object
        {
            if (Asset is T loaded) return loaded;

            if (Asset != null)
            {
                Debug.LogWarning($"[Res] 资源类型不匹配！{Key} 已加载为 " +
                    $"{Asset.GetType().Name}，尝试加载为 {typeof(T).Name}");
            }

            ResourceRequest request = loadingRequest ?? (loadingRequest = Resources.LoadAsync<T>(Path));
            try 
            {
                await request.ToUniTask(cancellationToken: token);
            } 
            finally 
            {
                loadingRequest = null;
            }

            Asset = request.asset;
            if (Asset == null) Debug.LogError($"[Res] 异步加载失败: {Path}");
            return Asset as T;
        }
```

这里对加载的特殊情况处理有三

1. 加载失败时asset为null，抛出报错，下次再加载这份资源由于asset为null会重新加载

2. 重复同时异步加载同一个资源时，通过判断loadingRequest状态使其指向同一个加载请求，避免重复

3. 取消安全：token 取消时 `finally` 把 `loadingRequest` 置空，状态归位，下次照常重试。Unity 的 `ResourceRequest` 本身无法中途取消，这里做的是"逻辑上当作没发生过"——请求自己会跑完，但结果不会被收养进缓存。
   
   <br>

## 三、资源加载登记表

ResTable 就是一张静态字典，负责把"路径 + 类型"映射到缓存条目：

```csharp
//工具方法
public static string BuildKey<T>(string path)
{
    return $"{path}_{typeof(T).name}"
}
```

为什么类型要进 key？`Resources.Load<T>` 是按类型过滤的，同一路径可能以不同类型访问。类型编进 key，各类型各存各的缓存条目，互不覆盖。

ResTable 对外就三个操作：

```csharp
GetOrCreateRes<T>(path) // 取或建
ReleaseRes(res) // 归还一个引用，计数归零则移出登记册
ClearUnused() // 清掉所有 RefCount=0 的条目
```

`ReleaseRes` 里有一个点需要注意：归还时先用 `ReferenceEquals(res, cached)` 确认手里的条目就是登记册里那份,防止一个已经过期的 ResBase 引用错误地操作新条目。

 `ClearUnused` 则用于清理所有没人持有的条目，在切场景这些情况下调用统一释放。清理时机和引擎对齐：切场景时先 `ClearUnused()` 把没人持有的条目从表里移除，再调 `Resources.UnloadUnusedAssets()` 让引擎真正释放无引用资产。表清完了、引擎卸完了，两边才算对齐。

<br>

## 四、持有者 ResLoader

RefCount 由谁加谁减？答案是 ResLoader——每个系统、每个界面从 ResManager 领一个自己的 ResLoader，把整个生命周期内用到的资源都挂在它身上：

同样提供两个方法，同步/异步加载

```csharp
public T LoadSync<T>(string path) where T : UnityEngine.Object
{ 
    var res=ResTable.GetOrCreateRes<T>(path);
    res.Acquire();
    resList.Add(res);
    return res.LoadSync<T>();
}

public async UniTask<T> LoadAsync<T>(string path,CancellationToken token = default) 
    where T : UnityEngine.Object
{
    var res=ResTable.GetOrCreateRes<T>(path);
    res.Acquire();
    resList.Add(res);
    try
    {
        return await res.LoadAsync<T>(token);
    } 
    catch
    {
        ResTable.ReleaseRes(res);//加载失败时释放引用计数
        resList.Remove(res);
        throw;//异常抛给调用方处理
    }
}
```

异步加载多了失败/取消的分支，加载抛异常时要把刚 Acquire 的引用还回去，否则失败一次就泄漏一次。

用完调一次 `ReleaseAll()`：把 resList 里记的账逐个归还，然后把自己扔回 ClassPool。同一个 Loader 在使用周期里可以加载任意多个资源，最后一次性归还，调用方不用一对一路配对 Release。用完调用一次`ReleaseAll`将

```csharp
        /// <summary>
        /// 释放所有资源引用，并回收自身
        /// </summary>
        public void ReleaseAll() 
        {
            if (isRecycled) return;
            foreach (var res in resList)
            {
                ResTable.ReleaseRes(res);
            }
            resList.Clear();
            ClassPool<ResLoader>.Recycle(this);
        }
```

<br>

## 五、ResManager

ResManager 是外部唯一入口，把散装零件收拢成几个顺手的 API，对外提供两种使用模式

**模式一** 直接获取Loader

```csharp
        public static ResLoader GetResLoader()
        {
            return ClassPool<ResLoader>.Get();
        }
```

**模式二** 不走引用计数等链路直接使用同步/异步Load

```csharp
        public static T Load<T>(string path) where T :UnityEngine.Object
        {
            return ResTable.GetOrCreateRes<T>(path).LoadSync<T>();
        }

        public static UniTask<T> LoadAsync<T>(string path,CancellationToken token=default) where T : UnityEngine.Object
        {
            return ResTable.GetOrCreateRes<T>(path).LoadAsync<T>(token);
        }
```

以及另外提供一些方法，例如批量预热加载，和清理缓存表，在后面完整代码给出

<br>

## 六、总结

  经过简单设计，Resouces模块已经简单搭好，虽然没法热更、没法按需下载，但是已经比原版的Resouces好用了一些。

 引用计数带来了资源释放的确定性，并发去重 + 同步异步一个缓存使得多个请求只请求一次，还有一键清场和预热功能。这套模块在游戏不复杂的情况下已经够用，不过后续也会升级拓展，加入更多新的功能。

<br>

## 七、完整代码

#### 1.ResBase

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;
using UnityEngine;
using Object = UnityEngine.Object;

namespace ZGameFramework.Core
{
    /// <summary>
    /// 单个资源的缓存条目。
    /// RefCount 语义 = "当前有几个 ResLoader 在使用它"，0 表示可被清理缓存。
    /// Path 用于 Resources.Load；Key 是缓存表唯一标识（路径 + 类型）。
    /// 加载失败留下空壳（Asset==null），下次加载自动重试，无需特殊清理。
    /// </summary>
    public class ResBase
    {
        /// <summary>缓存表唯一 key，避免同路径不同文件</summary>
        public string Key { get; private set; }

        /// <summary>Resources.Load 用的原始路径</summary>
        public string Path { get; private set; }

        /// <summary>加载完成后的资源</summary>
        public Object Asset { get; private set; }

        public int RefCount { get; private set; } = 0;

        public bool IsLoaded => Asset != null;
        public bool IsLoading => loadingRequest != null;

        public bool NeedDel=> RefCount <= 0;

        private bool negativeReported=false;//防止重复报负引用计数错误

        /// <summary>进行中的异步加载请求，防止同一资源并发重复加载</summary>
        private ResourceRequest loadingRequest;

        public ResBase(string path, string key)
        {
            Path = path;
            Key = key;
            loadingRequest = null;
        }

        public void Acquire() => RefCount++;

        public void Release()
        {
            if (RefCount <= 0)
            {
                if (!negativeReported)
                {
                    Debug.LogError($"[Res] 引用计数为负！{Key} 的 Release 比 Acquire 多，加载/释放不配对");
                    RefCount = 0;
                }
                return;
            }
            RefCount--;
        }

        /// <summary>
        /// 同步加载资源，返回资源对象。若资源已加载则直接返回缓存对象。
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <returns></returns>
        public T LoadSync<T>() where T : Object
        {
            if (Asset is T loaded) return loaded;

            if (Asset != null)
            {
                Debug.LogWarning($"[Res] 资源类型不匹配！{Key} 已加载为 " +
                    $"{Asset.GetType().Name}，尝试加载为 {typeof(T).Name}");
            }

            if (loadingRequest != null)
            {
                Debug.LogWarning($"[Res] 资源正在异步加载中，请保持加载方式相同{Key}");
            }

            Asset = Resources.Load<T>(Path);
            loadingRequest = null;

            if (Asset == null)
            {
                Debug.LogWarning($"[Res] 资源加载失败！{Key}");
            }
            return Asset as T;
        }

        /// <summary>
        /// 异步加载，并发调用复用同一次加载；token 取消时安全重置状态
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <param name="token"></param>
        /// <returns></returns>
        public async UniTask<T> LoadAsync<T>(CancellationToken token =default)where T : Object
        {
            if (Asset is T loaded) return loaded;

            if (Asset != null)
            {
                Debug.LogWarning($"[Res] 资源类型不匹配！{Key} 已加载为 " +
                    $"{Asset.GetType().Name}，尝试加载为 {typeof(T).Name}");
            }

            ResourceRequest request = loadingRequest ?? (loadingRequest = Resources.LoadAsync<T>(Path));

            try 
            {
                await request.ToUniTask(cancellationToken: token);
            } 
            finally 
            {
                loadingRequest = null;
            }

            Asset = request.asset;
            if (Asset == null) Debug.LogError($"[Res] 异步加载失败: {Path}");
            return Asset as T;
        }
    }
}
```

#### 2.ResLoader

```csharp
using Cysharp.Threading.Tasks;
using System.Collections.Generic;
using System.Security.Cryptography;
using System.Threading;

namespace ZGameFramework.Core
{
    public class ResLoader : IPoolable
    {
        private readonly List<ResBase> resList = new List<ResBase>();

        private bool isRecycled=false;
        public ResLoader() { }

        public T LoadSync<T>(string path) where T : UnityEngine.Object
        { 
            var res=ResTable.GetOrCreateRes<T>(path);
            res.Acquire();
            resList.Add(res);
            return res.LoadSync<T>();
        }

        public async UniTask<T> LoadAsync<T>(string path,CancellationToken token = default) 
            where T : UnityEngine.Object
        {
            var res=ResTable.GetOrCreateRes<T>(path);
            res.Acquire();
            resList.Add(res);
            try
            {
                return await res.LoadAsync<T>(token);
            } 
            catch
            {
                ResTable.ReleaseRes(res);//加载失败时释放引用计数
                resList.Remove(res);
                throw;//异常抛给调用方处理
            }
        }

        /// <summary>
        /// 释放所有资源引用，并回收自身
        /// </summary>
        public void ReleaseAll() 
        {
            if (isRecycled) return;
            foreach (var res in resList)
            {
                ResTable.ReleaseRes(res);
            }
            resList.Clear();
            ClassPool<ResLoader>.Recycle(this);
        }

        void IPoolable.OnRecycled()
        {
            isRecycled = true;
            resList.Clear();
        }

        void IPoolable.OnGet()
        {
            isRecycled = false;
        }
    }
}
```

#### 3.ResTable

```csharp
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography;
using System.Text;

namespace ZGameFramework.Core
{
    /// <summary>
    /// 资源缓存表
    /// </summary>
    public class ResTable
    {
        private static readonly Dictionary<string, ResBase> table = new();

        public static int ResCount => table.Count;
        /// <summary>
        /// 有效引用的资源数量
        /// </summary>
        public static int ActiveResCount
        {
            get
            {
                int count = 0;
                foreach (var res in table.Values)
                {
                    if (res.RefCount > 0)
                    {
                        count++;
                    }
                }
                return count;
            }
        }

        public static string BuildKey<T>(string path)
        {
            return $"{path}_{typeof(T).Name}";
        }

        /// <summary>
        /// 得到或者创建一个资源
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <param name="path"></param>
        /// <returns></returns>
        public static ResBase GetOrCreateRes<T>(string path)
        {
            string key = BuildKey<T>(path);

            if (table.TryGetValue(key, out var res))
            {
                return res;
            }

            ResBase newRes = new(path, key);

            table.Add(key, newRes);

            return newRes;
        }

        /// <summary>
        /// 归还一个引用，引用计数归零时从登记册移除
        /// </summary>
        /// <param name="res"></param>
        public static void ReleaseRes(ResBase res)
        {
            if (res == null) return;
            if (table.TryGetValue(res.Key, out var cached))
            {
                res.Release();
                if (res.NeedDel && ReferenceEquals(res, cached))
                {
                    //引用计数归零时从登记册移除
                    if (res.NeedDel) table.Remove(res.Key);
                }
            }
        }

        /// <summary>
        /// 清理所有没有被 loader 持有的缓存条目（切场景时调用）
        /// </summary>
        public static void ClearUnused()
        {
            var keys = ListPool<string>.Get();
            foreach (var k in table)
            {
                if (k.Value.IsLoading)
                {
                    continue;//加载中的资源不清理，避免切场景时正在加载的资源被清理掉
                }
                if (k.Value.NeedDel)
                {
                    keys.Add(k.Value.Key);
                }
            }

            foreach (var key in keys)
            {
                table.Remove(key);
            }
            ListPool<string>.Recycle(keys);
        }

        public static void ClearAll()
        {
            table.Clear();
        }

        public static int GetRefCount<T>(string path)
        {
            string key = BuildKey<T>(path);
            if (table.TryGetValue(key, out var res))
            {
                return res.RefCount;
            }
            return 0;
        }
    }
}
```

#### 4.ResManager

```csharp
using Cysharp.Threading.Tasks;
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;

namespace ZGameFramework.Core 
{
    public static class ResManager
    {
        /// <summary>
        /// 直接获取一个ResLoader对象，
        /// </summary>
        /// <returns></returns>
        public static ResLoader GetResLoader()
        {
            return ClassPool<ResLoader>.Get();
        }

        public static int ResCount => ResTable.ResCount;//调试用，缓存的资源数量

        //不计数加载资源，切场景后统一清理

        public static T Load<T>(string path) where T :UnityEngine.Object
        {
            return ResTable.GetOrCreateRes<T>(path).LoadSync<T>();
        }

        public static UniTask<T> LoadAsync<T>(string path,CancellationToken token=default) where T : UnityEngine.Object
        {
            return ResTable.GetOrCreateRes<T>(path).LoadAsync<T>(token);
        }

        /// <summary>清理缓存表 + 卸载无引用的 Resources 资源。切场景完成后调用。</summary>
        public static async UniTask UnloadUnused()
        {
            ResTable.ClearUnused();
            await Resources.UnloadUnusedAssets().ToUniTask();
        }

        /// <summary>
        /// 预热，并行把资源提前载入 Resources 内部缓存
        /// 之后用任意具体类型 Load/Instantiate 会新建自己的 ResBase，
        /// 但 Resources.Load 命中引擎内部缓存，不会重复 IO。
        /// </summary>
        public static async UniTask PreloadAsync(IReadOnlyList<string> paths,
            Action<float> onProgress = null, CancellationToken token = default)
        {
            if (paths == null || paths.Count == 0) return;

            int total = paths.Count;
            var state = new LoadProgressState(total, onProgress);
            var tasks = new UniTask<UnityEngine.Object>[total];

            for (int i = 0; i < total; i++)
            {
                token.ThrowIfCancellationRequested();
                tasks[i] = PreloadOneAsync(paths[i], token, state);
            }

            await UniTask.WhenAll(tasks);   // 并行，全部完成才算完
        }

        private static async UniTask<UnityEngine.Object> PreloadOneAsync(string path,
            CancellationToken token, LoadProgressState state)
        {
            // 直接走引擎 API，绕开 ResTable，避免留下 path_Object 
            var request = Resources.LoadAsync<UnityEngine.Object>(path);
            await request.ToUniTask(cancellationToken: token);
            state.Step();
            return request.asset;
        }

        /// <summary>
        /// 并行加载的共享进度计数器（主线程安全，UniTask 都在主线程回调）
        /// </summary>
        private class LoadProgressState
        {
            private readonly float total;
            private readonly Action<float> onProgress;
            private int done;

            public LoadProgressState(int total, Action<float> onProgress)
            {
                this.total = total;
                this.onProgress = onProgress;
            }

            public void Step()
            {
                done++;
                onProgress?.Invoke(done / total);
            }
        }


        //池化实例化
        public static GameObject Instantiate(string path, Transform parent = null)
        {
            var prefab = Load<GameObject>(path);
            return prefab == null ? null : PoolManager.Instance.GetGameObject(prefab, parent);
        }

        public static async UniTask<GameObject> InstantiateAsync(string path, Transform parent = null)
        {
            var prefab =await LoadAsync<GameObject>(path);
            return prefab == null ? null : PoolManager.Instance.GetGameObject(prefab, parent);
        }

        public static GameObject Instantiate(GameObject prefab, Transform parent = null)
        {
            return prefab == null ? null : PoolManager.Instance.GetGameObject(prefab, parent);
        }

        public static void Recycle(GameObject gameObject)
        {
            if (gameObject != null) PoolManager.Instance.PushGameObject(gameObject);
        }


        public static int GetRefCount<T>(string path) where T : UnityEngine.Object
        {
            return ResTable.GetRefCount<T>(path);
        }


    }
}
```
