---
title: 有向图的最短路径
date: 2016-05-12 11:21:31
tags: 编程
---

课上想的方法，没有用书上的Dijkstra算法和Floyd算法。

<!-- more -->

```
#include<iostream>
#include<cstdio>
using namespace std;
#define DEBUG
#define MAX 999999999
#define N 1010

int nodenum, edgenum, original; //点，边，起点

typedef struct Edge //边
{
	int v1, v2;
	int cost;
}Edge;

Edge edge[N];
int dis[N], pre[N];

bool MinRoad()
{
	for (int i = 1; i <= nodenum; ++i) //初始化
		dis[i] = (i == original ? 0 : MAX);

	for (int i = 1; i <= nodenum - 1; ++i)
	{
		for (int j = 1; j <= edgenum; ++j)
			//逐步逼近最短路径
			if (dis[edge[j].v2] > dis[edge[j].v1] + edge[j].cost)
			{
				dis[edge[j].v2] = dis[edge[j].v1] + edge[j].cost;
				pre[edge[j].v2] = edge[j].v1;
			}
	}

	bool flag = 1;					//判断是否有负值

	for (int i = 1; i <= edgenum; ++i)
		if (dis[edge[i].v2] > dis[edge[i].v1] + edge[i].cost)
		{
			flag = 0;
			break;
		}
	return flag;
}

void print_path(int root) //反向输出最短路径
{
	while (root != pre[root])
	{
		printf("%d<--", root);
		root = pre[root];
	}
	if (root == pre[root])
		printf("%d\n", root);
}

int main()
{

	#ifdef DEBUG
		freopen("input.txt", "r", stdin);
	#endif

	printf("请输入节点数与边数\n");
	scanf("%d %d", &nodenum, &edgenum);

	original = 0;
	pre[original] = original;

	printf("请输入边：（v1，v2，权值）\n");
	printf("\n");

	for (int i = 1; i <= edgenum; ++i)
	{
		scanf("%d %d %d", &edge[i].v1, &edge[i].v2, &edge[i].cost);
	}

	//计算最短路径
	if (MinRoad())
		for (int i = 1; i <= nodenum - 1; ++i)
		{
			printf("到顶点%d最短距离：\n%d\n", i, dis[i]);
			printf("路径为:\n");
			print_path(i);
			printf("\n");
		}
	else
		printf("有负数，找不到最短路径\n");
	return 0;
}
```
