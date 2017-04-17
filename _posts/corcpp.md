---
title: C还是C++？
date: 2016-04-13 10:00:02
tags: 编程
---
数据结构课查二叉树的作业，老师说我这个没建立二叉树类，所以用的不是C++，是C语言。
我十分不解，没建立类所以就是C语言了么？我使用指针的引用还是C语言么？没有对象就不是C++么？
<!-- more -->


### Binarytree.h
```
#ifndef BINARYTREE_H
#define BINARYTREE_H

#include <iostream>
#include <fstream>

//二叉树结点类的定义
template<class T>
struct TreeNode
{
	T data;									//节点的内容
	TreeNode<T> *Lchild, *Rchild;			//节点的左子树和右子树
};

//二叉树的建立
template <class T>
void createBinaryTree(TreeNode<T> *&root)   //传递指针的引用
{
	TreeNode<T>	*p = root;
	T nodeValue;
	nodeValue = cin.get();
	if (nodeValue == '#'||nodeValue == '\n')
	{
		root = NULL;
	}
	else
	{
		root = new TreeNode<T>();			//构造一个节点
		root->data = nodeValue;
		createBinaryTree(root->Lchild);		//递归构造左子树
		createBinaryTree(root->Rchild);		//递归构造右子树
	}
}

//二叉树的先序遍历
template <class T>
void preOrder(TreeNode<T> *& p)
{
	if (p)
	{
		cout << p->data;
		preOrder(p->Lchild);
		preOrder(p->Rchild);
	}
}

//二叉树的中序遍历
template <class T>
void inOrder(TreeNode<T> *& p)
{

	if (p)
	{
		inOrder(p->Lchild);
		cout << p->data;
		inOrder(p->Rchild);
	}
}

//二叉树的后序遍历
template <class T>
void postOrder(TreeNode<T> *&p)
{
	if (p)
	{
		postOrder(p->Lchild);
		postOrder(p->Rchild);
		cout << p->data;
	}
}

//统计二叉树中结点的个数
template<class T>
int countNode(TreeNode<T> *&p)
{
	if (p == NULL) return 0;
	return 1 + countNode(p->Lchild) + countNode(p->Rchild);
}

//求二叉树的深度
template<class T>
int depth(TreeNode<T> *&p)
{
	if (p == NULL)
		return -1;
	int h1 = depth(p->Lchild);
	int h2 = depth(p->Rchild);
	if (h1>h2)return (h1 + 1);
	return h2 + 1;
}

//搜索
int level = -1;
bool found = false;

template <class T>
int search(TreeNode<T> *&p,char key)
{
	if (p != NULL && !found)		//一个条件不满足时就跳出
	{
		level++;					//层数增加
		if (p->data == key)			//找到后标志
			found = true;
		else						//没有找到继续找当前节点的左孩子或者右孩子
		{
			search(p->Lchild, key);
			search(p->Rchild, key);
			if (!found)
				level--;			//没有发现层数减1
		}
	}
	return level;					//返回层数
}

template <class T>
int printlevel(TreeNode<T> *&p, int depth)
{
	if (!p || depth < 0)
		return 0;
	if (0 == depth) {
		if (p)
		{
			cout << p->data << " ";
		}
		return 1;
	}
	return printlevel(p->Lchild, depth - 1) + printlevel(p->Rchild, depth - 1);
}

template <class T>
void printtree(TreeNode<T> *&p)
{
	int i = 0;
	for (i = 0; ; i++)
	{
		if (!printlevel(p, i))
			break;
		cout << endl;
	}
}

#endif
```

### main.cpp
```
#define DEBUG

#include <iostream>
#include <fstream>
#include "binarytree.h"
using namespace std;

int main(void)
{
#ifdef DEBUG
	ifstream cin("input.txt");
#endif
	TreeNode<char> *treeroot;
	createBinaryTree(treeroot);

	cout << "先序遍历：" << endl;
	preOrder(treeroot);
	cout << endl;

	cout << "中序遍历：" << endl;
	inOrder(treeroot);
	cout << endl;

	cout << "后序遍历：" << endl;
	postOrder(treeroot);
	cout << endl;

	cout << "二叉树深度：";
	cout << depth(treeroot) << endl;

	cout <<"E元素深度为："<< search(treeroot, 'E') << endl;;

	cout << "层序遍历：" << endl;
	printtree(treeroot);

	system("pause");
	return 0;
}

```
