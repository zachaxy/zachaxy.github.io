---
title: Dijkstra算法
date: 2017-04-13 15:12:38
tags: 算法
---

> Dijkstra算法,给出一个邻接矩阵和一个起始点,返回这个其实点到邻接矩阵中各个点的最短距离
>
> 使用了广度优先搜索解决非负权有向图的单源最短路径问题，算法最终得到一个最短路径树（一个节点到其他所有节点的最短路径）。
> 该算法常用于路由算法或者作为其他图算法的一个子模块。主要特点是以起始点为中心向外层层扩展，直到扩展到终点为止。

---

基本用法:

```java
public class Dijkstra {
    private static int M = Integer.MAX_VALUE; //此路不通

    public static void main(String[] args) {
        //邻接矩阵,单向连接;
        int[][] weight = {
                {0, 10, M, 30, 100},
                {M, 0, 50, M, M},
                {M, M, 0, M, 10},
                {M, M, 20, 0, 60},
                {M, M, M, M, 0}
        };

        //设置起始点,这里将0设置为起始点,接下来要做的动作是寻找整个图中所有点距离0最近的距离;
        int start = 0;
        //最短路径的数组,长度为0,保存了0到n个点的最短距离,当然第一个值是0到0的距离,距离为0;
        int[] shortPath = dijkstra(weight, start);

        //打印0到各点最近的距离;
        for (int i = 0; i < shortPath.length; i++)
            System.out.println("从" + start + "出发到" + i + "的最短距离为：" + shortPath[i]);
    }

    /***
     * @param weight 有向图的权重矩阵
     * @param start  一个起点编号start（从0编号，顶点存在数组中）
     * @return 返回一个int[] 数组，表示从start到它的最短路径长度
     */
    public static int[] dijkstra(int[][] weight, int start) {
        int n = weight.length;         //顶点个数
        int[] shortPath = new int[n];  //保存start到其他各点的最短路径,最后作为返回值被返回;
        String[] path = new String[n];  //保存start到其他各点最短路径的字符串描述;

        //初始化从start到各个节点的最优路径图
        for (int i = 0; i < n; i++) {
            path[i] = start + "-->" + i;
        }

        //或者用一个boolean类型的数组表示,是否已经访问过;
        int[] visited = new int[n];   //标记当前该顶点的最短路径是否已经求出,1表示已求出

        //初始化，第一个顶点已经求出
        shortPath[start] = 0;
        visited[start] = 1;


        //最外层循环,一共n个节点,要找到start节点到其他n-1个节点的最短距离,因此这里需要循环n-1次;
        for (int count = 1; count < n; count++) {   //要加入n-1个顶点
            int k = -1;        //选出一个距离初始顶点start最近的未标记顶点,暂时记为k;
            int dmin = Integer.MAX_VALUE;  //临时距离;

            //内循环找到一个距离start最近的点;
            for (int i = 0; i < n; i++) {
                if (visited[i] == 0 && weight[start][i] < dmin) {
                    dmin = weight[start][i];
                    k = i;
                }
            }
            //至此已经找到了一个距离start的最短点;
            //将新选出的顶点标记为已求出最短路径，且到start的最短路径就是dmin
            shortPath[k] = dmin;
            visited[k] = 1;

            //以k为中间点，修正从start到未访问各点的距离
            for (int i = 0; i < n; i++) {
                //start不能到i;但是start可以到k,k可以到i,
                if (visited[i] == 0 && weight[k][i] != M && weight[start][k] + weight[k][i] < weight[start][i]) {
                    weight[start][i] = weight[start][k] + weight[k][i];
                    path[i] = path[k] + "-->" + i;
                }
            }
        }

        //打印最短路径们;
        for (int i = 0; i < n; i++) {
            System.out.println("从" + start + "出发到" + i + "的最短路径为：" + path[i]);
        }
        System.out.println("=====================================");
        return shortPath;
    }
}
```

改进:基本用法中为了更好的展示,使用了邻接矩阵,这需要更大的内存空间,一个改进方法是使用邻接链表;

```java
    static void buildGraph() {
        HashMap<String, Vertex> nodes = new HashMap<String, Vertex>();
        nodes.put("a", new Vertex("a"));
        nodes.put("b", new Vertex("b"));
        nodes.put("c", new Vertex("c"));
        nodes.put("d", new Vertex("d"));
        nodes.put("e", new Vertex("e"));

        nodes.get("a").map.put(nodes.get("b"), new Edge(nodes.get("a"), nodes.get("b"), 10));
        nodes.get("a").map.put(nodes.get("d"), new Edge(nodes.get("a"), nodes.get("d"), 30));
        nodes.get("a").map.put(nodes.get("e"), new Edge(nodes.get("a"), nodes.get("e"), 100));

        nodes.get("b").map.put(nodes.get("c"), new Edge(nodes.get("b"), nodes.get("c"), 50));

        nodes.get("c").map.put(nodes.get("e"), new Edge(nodes.get("c"), nodes.get("e"), 10));

        nodes.get("d").map.put(nodes.get("c"), new Edge(nodes.get("d"), nodes.get("c"), 20));
        nodes.get("d").map.put(nodes.get("e"), new Edge(nodes.get("d"), nodes.get("e"), 60));
        
    }

    //节点,没有什么特别之处,和一个String的类型一样;
    static class Vertex {
        String key;
        HashMap<Vertex, Edge> map = new HashMap<Vertex, Edge>();  //key:以Vertex为end边的节点,当前节点到key的距离;

        Vertex(String key) {
            this.key = key;
        }
    }

    //边,包含了一个起始和一个终止节点,已经一个边的长度;
    static class Edge {
        Vertex start;
        Vertex end;
        int Len;
        boolean used;

        Edge(Vertex start, Vertex end, int key) {
            this.start = start;
            this.end = end;
            this.Len = key;
        }
    }
```

