---
title: C++实现无向图的深度广度搜索
date: 2016-04-24 17:22:43
tags: 编程
---
这次建立了类，所以一定是C++代码！
其实并不需要用指针的引用这么麻烦，最后封装成类的时候没有改，但也足够简洁了。
所以以后要先写类。

<!-- more -->

```
#include <iostream>
#include <queue>
using namespace std;

//全局变量
const int maximum = 100;
int visited[maximum];   //标记已被访问过的顶点

//无向图类
class Graph
{
	char vertex[maximum];			//存顶点
	int arc[maximum][maximum];		//存边（邻接矩阵）
	int vertexnum, arcnum;          //顶点数和边数

public:
	void CreateGraph(Graph *&G);	//图的初始化
	void DFS(Graph *&G, int v);		//深度优先遍历
	void BFS(Graph *&G, int v);		//广度优先遍历
	void Print(Graph *G);			//打印邻接矩阵
};

void Graph::CreateGraph(Graph *&G)
{
	int i, j;
	cout << "请输入顶点数和边数" << endl;
	cin >> G->vertexnum >> G->arcnum;	//输入顶点数和边数
	cout << "请输入每个顶点的值" << endl;
	for (i = 0;i < G->vertexnum;i++)	//输入每个顶点的值
		cin >> G->vertex[i];
	for (i = 0;i < G->vertexnum;i++)	//初始化邻接矩阵
		for (j = 0;j < G->vertexnum;j++)
			G->arc[i][j] = 0;
	cout << "请输入相连的顶点标号" << endl;
	for (i = 0;i < G->arcnum;i++)
	{
		int n, m, w;
		cin >> n >> m;				   //修改邻接矩阵中的值
		G->arc[n][m]=1;
		G->arc[m][n]=1;
	}
}

void Graph::DFS(Graph *&G, int v)
{
	int j;
	if (!visited[v])
	{
		cout << G->vertex[v] << " ";
		visited[v] = 1;						//标记为访问过
	}
	for (j = 0;j < G->vertexnum;j++)
	{
		if (G->arc[v][j] && !visited[j])	//邻接矩阵的第(v,j)元素不为0
		{
			DFS(G, j);
		}
	}
}

void Graph::BFS(Graph *&G, int v)
{
	queue<int> q;
	int x, j;
	if (!visited[v])
	{
		cout << G->vertex[v] << " ";
		visited[v] = 1;
		q.push(v); //被访问的顶点入队
	}
	while (!q.empty())	//队不空进循环
	{
		x = q.front();	//取队头元素
		q.pop();		//队头出队
		for (j = 0;j < G->vertexnum;j++)
		{
			if (G->arc[x][j] && !visited[j])
			{
				cout << G->vertex[j] << " ";
				visited[j] = 1; //标记为访问过
				q.push(j);   //被访问的顶点继续入队
			}
		}
	}
}

//图的邻接矩阵的输出函数
void Graph::Print(Graph *G)
{
	int i, j;
	for (i = 0;i < G->vertexnum;i++)
	{
		for (j = 0;j < G->vertexnum;j++)
			cout << G->arc[i][j] << " ";
		cout << endl;
	}
}

int main()
{
	Graph *G = new Graph;
	G->CreateGraph(G);

	cout << "输出邻接矩阵:" << endl;
	G->Print(G);

	cout << "深度优先搜索:";
	G->DFS(G, 0);
	cout << endl;

	memset(visited, 0, sizeof(visited));	//重置visited数组

	cout << "广度优先搜索:";
	G->BFS(G, 0);
	cout << endl;

	return 0;
}
```
