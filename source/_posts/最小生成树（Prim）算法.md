---
title: 最小生成树（Prim）算法
tags: prim
categories: Algorithm && Data Structure
abbrlink: 2d931a86
date: 2018-03-28 01:32:23
---

#### 算法思想

- 假设`G=<V,E>`是连通图，TE是G上最小生成树中边的集合。

- 算法从U={u0}(u0∈V)，TE={ }开始，任取一个顶点u0作为开始点。

- 重复执行下述操作：在所有u∈U, v∈V-U的边(u,v)∈E中找一条代价最小的边(u0,v0)并入集合TE，同时v0并入U，直至U=V为止。

**注意：** 选择最小边时，可能有多条同样权值的边可选，此时任选其一。

![Prim](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201709/20170924022104006.png)


#### 代码实现

```java
public class Prim{  
    /** 
     * 最小生成树的PRIM算法 
     * @param  graph  图 
     * @param start 开始节点 
     * @param n     图中节点数 
     */  
    public static void PRIM(double [][] graph,int start,int n){  
        /*用于保存集合U到V-U之间的最小边和它的值
        mins[i][0]值表示到该节点i边的起始节点,值为-1表示没有到它的起始点
        mins[i][1]值表示到该边的最小值
        mins[i][1]=0表示该节点已将在集合U中  */
    	double [][] mins=new double [n][2];
    	
    	//初始化mins  
        for(int i=0;i<n;i++){
            if(i==start){  
                mins[i][0]=-1;  
                mins[i][1]=0;  
            }else if( graph[start][i]!=-1){//说明存在（start，i）的边  
                mins[i][0]=start;  
                mins[i][1]= graph[start][i];  
            }else{  
                mins[i][0]=-1;  
                mins[i][1]=Double.MAX_VALUE;  
            }  
        }
        
        for(int i=0;i<n-1;i++){
        	int minV = -1;
            double minW=Double.MAX_VALUE;  
            
            for(int j=0;j<n;j++){ //找到mins中最小值
                if(mins[j][1]!=0&&minW>mins[j][1]){  
                    minW=mins[j][1];  
                    minV=j;  
                }  
            }
            
            mins[minV][1]=0;  
            System.out.println("Prim的第"+i+"条最小边=<"+(int)(mins[minV][0]+1)+","+(minV+1)+">，权重="+minW);  
            
            for(int j=0;j<n;j++){//更新mins数组  
                if(mins[j][1]!=0){  
                    if( graph[minV][j]!=-1&& graph[minV][j]<mins[j][1]){  
                        mins[j][0]=minV;  
                        mins[j][1]= graph[minV][j];  
                    }  
                }  
            }  
        }
        
    } 
    
    public static void main(String [] args){  
        double [][] tree={  
                {-1, 2.0, 4.2, 6.7},  
                {2.0 ,-1,-1, 10.0},  
                {4.2,-1, -1, 4.0},    
                {6.7,10.0,4.0,-1}            
        };  
        Prim.PRIM(tree, 0, 4);  
    }   
}
```
