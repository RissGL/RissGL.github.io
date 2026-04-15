---
title: "算法"
date: 2026-04-15
draft: false

---

## 前言

本人是算法苦手，写这篇博客本身是为了利用费曼学习法帮助自己巩固算法知识，不过你也可以看看

### A*算法

#### 原理

A*算法是游戏开发中最常用的寻路算法，它的原理也符合人类的直觉  

核心是三个值，分别是g值，h值，f值，以寻路来说  

这里三个值分别代表着，到目前节点要复付出的代价，目前节点预测到终点的代价，最终的代价    这里的h值通常代表着曼哈顿距离，代表着无视障碍物直接到终点的预测花费，而f值为g值+h值     

除了这三个值之外我们还需要一个待探索列表，用于记录有潜力的节点，我们通过每次都取该列表中f值最低的去进行寻路，以提高寻路效率

### 实践

在开始算法之前，为了使我们写出来的东西具有实用性和泛用性，而不是当个科普就过了，请定义以下接口

```c#
//只要实现了这个接口就能被我们后续的寻路脚本处理
public interface IPathGraph<T>
{
    //得到邻居列表
    public List<T> GetNeighbors(T node);

    //得到移动代价
    public int GetMoveCost(T fromNode,T toNode);

    //得到h值
    public int GetHeuristicCost(T fromNode, T toNode);
}
```

接下来我们新建一个静态类命名为AStarPathfinder，再这个类里面我们需要三个方法,你可以根据原理先自己思考一下如何实现函数

```
public static class AStarPathfinder
{
    public static List<T> FindPath(IPathGraph<T>,T start,T end,out int totalCost)
     where T:System.IEquatable<T>// 约束 T 必须实现 IEquatable，以便作为字典的 Key
    {

    }

    //该函数用于回溯路径，在到达终点的时候回溯出最短路径
    private static List<T> RetracePath<T>(Dictionary<T, T> cameFrom, T current)
    {

    }

    //用于得到最小f值的节点
    private static T GetLowestFCostNode<T>(List<T> openList, Dictionary<T, int> fCostDict)
    {

    }
}
```

```c#
    public static List<T> FindPath<T>(IPathGraph<T> graph, T start, T end, out int totalCost)
        where T : System.IEquatable<T>// 约束 T 必须实现 IEquatable，以便作为字典的 Key
    {
        totalCost = 0;

        List<T> openList=new List<T>();//开放列表
        List<T> finalPath=new List<T>();//最终路径

        //由于需要存储一个节点引用和对应的g,f值，这里使用字典再合适不过
        Dictionary<T,int> gCostDict=new Dictionary<T,int>();
        Dictionary<T,int> fCostDict=new Dictionary<T,int>();

        //初始化数据
        gCostDict[start] = 0;
        fCostDict[start] = graph.GetHeuristicCost(start, end);

        //存储路径，以便后续回溯
        Dictionary<T,T> cameFrom=new Dictionary<T,T>();


        openList.Add(start);//加入初始节点

        //只要openList还有东西就继续搜索
        while (openList.Count > 0)
        {
            //拿到预测f值最低的节点
            T current = GetLowestFCostNode(openList, fCostDict);

            //如果到达终点就记录数据并跳出循环
            if (current.Equals(end))
            {
                finalPath = RetracePath<T>(cameFrom, current);
                totalCost = gCostDict[current];
                break;
            }

            //已经到达的节点就能直接从openList移除了
            openList.Remove(current);

            //得到所有临近单位
            List<T> neighbors = graph.GetNeighbors(current);

            foreach (T neighbor in neighbors) 
            {
                //计算暂定g成本
                int tentativeGCost=gCostDict[current]+graph.GetMoveCost(current,neighbor);

                //当没到过这个节点或这个节点算出来比之前小的话纳入统计
                if (!gCostDict.ContainsKey(neighbor) || tentativeGCost < gCostDict[neighbor])
                {
                    //将这个节点的来路记为current
                    cameFrom[neighbor]=current;
                    gCostDict[neighbor]=tentativeGCost;
                    fCostDict[neighbor]=tentativeGCost+graph.GetHeuristicCost(neighbor,end);

                    //加入下一次探索的列表
                    if (!openList.Contains(neighbor))
                    {
                        openList.Add(neighbor);
                    }
                }
            }
        }

        return finalPath;
    }

    //该函数用于回溯路径，在到达终点的时候回溯出最短路径
    private static List<T> RetracePath<T>(Dictionary<T, T> cameFrom, T current)
    {
        List<T> path=new List<T>();
        path.Add(current);

        //这里通过不断的回溯将路径加入列表，有点双向链表的感觉
        while (cameFrom.ContainsKey(current))
        {
            path.Add(cameFrom[current]);
            current = cameFrom[current];
        }

        //上一步得到的路径是终点到起点的，我们要做一步反转才是正确的
        path.Reverse();
        return path;
    }

    //用于得到最小f值的节点
    private static T GetLowestFCostNode<T>(List<T> openList, Dictionary<T, int> fCostDict)
    {
        T lowestNode = openList[0];
        int fNodeCostOne = fCostDict.ContainsKey(lowestNode)?fCostDict[lowestNode]:int.MaxValue;

        for (int i = 1; i < openList.Count; i++)
        {
            T node = openList[i];
            int fNodeCost= fCostDict.ContainsKey(node) ? fCostDict[node] : int.MaxValue;

            if (fNodeCost < fNodeCostOne)
            {
                fNodeCostOne = fNodeCost;
                lowestNode = node;
            }
        }

        return lowestNode;
    }
```
