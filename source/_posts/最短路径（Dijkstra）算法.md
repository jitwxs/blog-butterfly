---
title: 最短路径（Dijkstra）算法
tags: Dijkstra
categories:
  - 算法与数据结构
  - 算法
abbrlink: 40493b95
date: 2018-03-28 01:30:00
copyright_author: Jitwxs
---

### 一、算法功能

给定一个出发点(单源点)和一个有向网`G=(V, E)`, 求出源点到其它各顶点之间的最短路径。

### 二、算法思想

（1）把图中顶点集合分成两组，第一组为集合S，存放已求出其最短路径的顶点，第二组为尚未确定最短路径的顶点集合是`V-S`(令W=V-S)，其中V为网中所有顶点集合。

（2）按**最短路径长度递增**的顺序逐个把W中的顶点加到S中，直到S中包含全部顶点，而W为空。

（3）在加入的过程中，总保持从源点v到S中各顶点的最短路径长度不大于从源点v到W中任何顶点的最短路径长度。

（4）此外，每个顶点对应一个距离，S中的顶点的距离就是从v到此顶点的最短路径长度，W中的顶点的距离从v到此顶点只包括S中的顶点为中间顶点的当前最短路径长度。

### 三、实现步骤

（1）初始时，**S只包含源点**，S={v}，v的距离为0。**U包含除v外的其他顶点**，U中顶点的距离为顶点的权或∞ 。

（2）从U中选取一个**距离最小**的顶点k，把k加入到S中。

（3）以k 作为新考虑的中间点，**修改**U中各顶点的距离。

（4）重复步骤（1）、（2）直到所有顶点都包含在S中。

### 四、举例

求v0到其它各点的最短路径：

![Dijkstra题目](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170924022726419.png)

![Dijkstra答案](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170924022801653.png)

> 答：最短路径为（V0,V2,V4,V3,V5）

### 五、算法实现

```c
/*======================================================
*   Copyright (C) 2017
*   
*   文件名称：dijkstra.c
*   创 建 者：ZhouYang
*   创建日期：2017年06月01日
*   描    述：dijkstra算法实现
*
====================================================*/

#include<stdio.h>
#include<string.h>
#include<stdlib.h>

#define MAX_INT 32767

int main(void)
{
	int i,j,k;
	int vertexLen, edgeLen;
	int len_S,len_U;
	int min,weight,sum=0;
	char initial_node,final_node;
	char node,mid;
	char startVertex, endVertex;

	printf("please input vertex's number:");
	scanf("%d",&vertexLen);
	printf("please input edge's number:");
	scanf("%d",&edgeLen);
	
	int graph[vertexLen][vertexLen];
	char route[vertexLen];
	char S[vertexLen],U[vertexLen];
	int node_min[vertexLen];
	
	for(i=0;i<vertexLen;i++)
		for(j=0;j<vertexLen;j++){
			if(i==j)
				graph[i][j] = 0;
			else
				graph[i][j] = MAX_INT;
		}
	
	for(i=0; i<edgeLen; i++){
		printf("please input the %d edge's start vertex name (vertex name from a to %c):",i+1,'a'+vertexLen-1);
		scanf(" %c",&startVertex);
		printf("please input the %d edge's end vertex name (vertex name from a to %c):",i+1,'a'+vertexLen-1);
		scanf(" %c",&endVertex);
		printf("please input this edge's weight:");
		scanf("%d",&weight);
		
		graph[startVertex-'a'][endVertex-'a'] = weight;	
	}

	for(i=0;i<vertexLen;i++)
		node_min[i]=MAX_INT;
	
	printf("Input initial node:");
	scanf(" %c",&initial_node);
	printf("Input final node:");
	scanf(" %c",&final_node);

	S[0]=initial_node;
	j=0;
	for(i=0;i<vertexLen;i++)
	{
		if(initial_node != 'a'+i)
			U[j++]='a'+i;
	}
	len_S=1;
	len_U=4;

	node_min[initial_node-'a']=0;
	node=initial_node;

	for(i=0;i<vertexLen-1;i++)
	{
		for(j=0;j<vertexLen;j++)
		{
			if(node_min[j]>graph[node-'a'][j]+node_min[node-'a'])
			{
				node_min[j]=graph[node-'a'][j]+node_min[node-'a'];
			}
		}
		
		min=MAX_INT;
		for(j=0;j<len_U;j++)
		{
			if(min>node_min[U[j]-'a'])
			{
				min=node_min[U[j]-'a'];
				mid=U[j];
				k=j;
			}
		}
		while(k<len_U)
		{
			U[k]=U[k+1];
			k++;
		}
		len_U--;
		len_S++;
		S[len_S-1]=mid;
		node=mid;
	}
	
	mid=final_node;
	k=0;
	route[k++]=mid;
	while(1)
	{	
		for(i=0;i<vertexLen;i++)
		{
			if(mid!='a'+i)
			{
				if(node_min[mid-'a']-node_min[i]==graph[i][mid-'a'])
				{
					sum += node_min[mid-'a']-node_min[i];
					mid='a'+i;
					route[k++]=mid;
					break;
				}
			}
		}
		if(mid == initial_node)
			break;
	}
	
	for(i = k-1;i>0;i--)
		printf("%c-->",route[i]);
	printf("%c",final_node);
	printf("\nweight sum is %d",sum);
	return 0;
}

```

**图如下**

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201706/20170606212248641.png)

**运行结果**
![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201706/20170606220125628.png)
