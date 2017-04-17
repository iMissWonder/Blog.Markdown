---
title: 三种排序算法
date: 2016-05-22 12:05:56
tags: 编程
---
堆排序、快速排序、冒泡排序的运行时间比较。代码如下。
<!-- more -->

```
#include <iostream>
#include <time.h>

#define DEBUG
#define MAX 20000
using namespace std;

class Heap {
public:
	Heap(int inarray[], int arraylength)
	{
		length = arraylength;
		for (int i = 0; i < length; i++)
		{
			array[i] = inarray[i];
		}
		//最后一个有孩子的节点的位置 i=(length -1) / 2  
		for (int i = (length - 1) / 2; i >= 0; --i)
			HeapAdjust(array, i, length);
	}

	// 堆排序算法
	void HeapSort()
	{
		//从最后一个元素开始对序列进行调整  
		for (int i = length - 1; i > 0; --i)
		{
			//交换堆顶元素H[0]和堆中最后一个元素  
			int temp = array[i]; array[i] = array[0]; array[0] = temp;
			//每次交换堆顶元素和堆中最后一个元素之后，都要对堆进行调整  
			HeapAdjust(array, 0, i);
		}
	}

	// 打印堆
	void printHeap()
	{
		for (int i = 0; i < length; i++)
		{
			cout << array[i] << "  ";
		}
		cout << endl;
	}


private:
	/**
	* 已知H[s…m]除了H[s] 外均满足堆的定义
	* 调整H[s],使其成为大顶堆.即将对第s个结点为根的子树筛选,
	*
	* H是待调整的堆数组
	* s是待调整的数组元素的位置
	* length是数组的长度
	*
	*/
	void HeapAdjust(int H[], int s, int length)
	{
		int tmp = H[s];
		int child = 2 * s + 1; //左孩子结点的位置。(i+1 为当前调整结点的右孩子结点的位置)  
		while (child < length) {
			if (child + 1 <length && H[child]<H[child + 1]) { // 如果右孩子大于左孩子(找到比当前待调整结点大的孩子结点)  
				++child;
			}
			if (H[s]<H[child]) {  // 如果较大的子结点大于父结点  
				H[s] = H[child]; // 那么把较大的子结点往上移动，替换它的父结点  
				s = child;       // 重新设置s ,即待调整的下一个结点的位置  
				child = 2 * s + 1;
			}
			else {            // 如果当前待调整结点大于它的左右孩子，则不需要调整，直接退出  
				break;
			}
			H[s] = tmp;         // 当前待调整的结点放到比其大的孩子结点位置上  
		}

		//print(H, length);

	}

	/*
	void print(int a[], int n)
	{
		for (int j = 0; j < n; j++)
		{
			cout << a[j] << "  ";
		}
		cout << endl;
	}
	*/

	int array[MAX];
	int length;
};

class Quick {
public:
	Quick(int inarray[], int n)
	{
		length = n;
		for (int i = 0; i < length; i++)
		{
			array[i] = inarray[i];
		}
	}
	int partition(int low, int high)
	{
		int privotKey = array[low];			                          //基准元素  
		while (low < high) {										  //从表的两端交替地向中间扫描  
			while (low < high  && array[high] >= privotKey) --high;   //从high 所指位置向前搜索，至多到low+1 位置。将比基准元素小的交换到低端  
			swap(&array[low], &array[high]);
			while (low < high  && array[low] <= privotKey) ++low;
			swap(&array[low], &array[high]);
		}
		return low;
	}
	void QuickSort(int low, int high) {
		if (low < high) {
			int privotLoc = partition(low, high);   //将表一分为二  
			QuickSort(low, privotLoc - 1);          //递归对低子表递归排序  
			QuickSort(privotLoc + 1, high);         //递归对高子表递归排序  
		}
	}
	void print() {
		for (int i = 0; i < length; i++) {
			cout << array[i] << "  ";
		}
		cout << endl;
	}

private:
	void swap(int *a, int *b)
	{
		int tmp = *a;
		*a = *b;
		*b = tmp;
	}

	int array[MAX];
	int length;
};

void BubbleSort(int a[], int length)
{
	for (int i = 0; i < length - 1; ++i) {
		for (int j = 0; j < length - i - 1; ++j) {
			if (a[j] > a[j + 1])
			{
				int temp = a[j];
				a[j] = a[j + 1];
				a[j + 1] = temp;
			}
		}
	}
}

int main() {

	//待排序数组
	int array[MAX];
	srand((unsigned)time(NULL));
	for (int i = 0; i < MAX; i++)
	{
		array[i] = rand() % MAX;
	}

	//计时变量
	clock_t start, finish;
	double duration;

	//堆排序
	Heap H(array, MAX);		    	//创建堆
	if (MAX < 20)
	{
		cout << "初始值：";
		H.printHeap();
	}
	start = clock();				//开始计时
	H.HeapSort();					//排序
	finish = clock();				//结束计时
	duration = finish - start;		//计算时间
	if (MAX < 20)
	{
		cout << "结果：";
		H.printHeap();
	}
	cout << "1.堆排序" << endl;
	cout << "时间：" << duration << "ms" << endl << endl;

	//快速排序
	Quick Q(array, MAX);			//创建堆
	if (MAX < 20)
	{
		cout << "初始值：";
		Q.print();
	}
	start = clock();				//开始计时
	Q.QuickSort(0, MAX - 1);		//排序
	finish = clock();				//结束计时
	duration = finish - start;		//计算时间
	if (MAX < 20)
	{
		cout << "结果：";
		H.printHeap();
	}
	cout << "2.快速排序" << endl;
	cout << "时间：" << duration << "ms" << endl << endl;

	//冒泡排序
	start = clock();				//开始计时
	BubbleSort(array, MAX);			//排序
	finish = clock();				//结束计时
	duration = finish - start;		//计算时间
	cout << "3.冒泡排序" << endl;
	cout << "时间：" << duration << "ms" << endl << endl;
}
```
