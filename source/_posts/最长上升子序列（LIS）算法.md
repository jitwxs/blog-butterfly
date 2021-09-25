---
title: 最长上升子序列（LIS）算法
tags: LIS
categories:
  - 算法与数据结构
  - 算法
abbrlink: f2158548
date: 2018-03-28 01:34:09
---

**理解：**该子序列中后一项都比前一项大，例如有序列**2 7 1 5 6 4 3 8 9**，则`最长上升子序列`为**2 5 6 8 9**。

**具体应用：**用于确定一个代价最小的调整方案，使一个序列变为升序。只需要固定LIS中的元素，调整其他元素即可。

## 动态规划实现 O(n<sup>2</sup>)

我们将序列存入数组a中，定义一个dp数组，存放最大长度。

>2 7 1 5 6 4 3 8 9 

我们能够得出一个规律：

**dp[i] = 前面value值中小于当前value值的dp最大值+1。**

| i | value | dp |
|:-----:|:-----:|:-----:| 
| 0 | 2 | dp[0] = 1 |
| 1 | 7 | dp[1] = dp[0] + 1 = 2 |
| 2 | 1 | dp[2] = dp[0] = 1 |
| 3 | 5 | dp[3] = dp[0] + 1 = 2 |
| 4 | 6 | dp[4] = dp[3] + 1 = 3 |
| 5 | 4 | dp[5] = dp[0] + 1 = 2 |
| 6 | 3 | dp[6] = dp[0] + 1 = 2 |
| 7 | 8 | dp[7] = dp[4] + 1 = 4 |
| 8 | 9 | dp[8] = dp[7] + 1 = 5 |

根据规律，可以推出上表，最终结果为dp数组中最大值，代码如下：

```java
public class Main {
	public static void main(String[] args) {
		int[] a = { 2, 7, 1, 5, 6, 4, 3, 8, 9 };
		int[] dp = new int[a.length];
		int res = 0;
		
		for (int i = 0; i < a.length; i++) {
			dp[i] = 1;
			for (int j = 0; j <= i; j++) {
				if (a[j] < a[i]) {
					dp[i] = Math.max(dp[i], dp[j] + 1);
				}
			}
			res = Math.max(res, dp[i]);
		}
		
		System.out.println(res);
	}
}
```

### 二分查找实现 $O(nlogn)$

如果我们只关心LIS的长度，可以使用`二分查找`来实现，这种方法时间复杂度更低。

---

### 牛刀小试

HDU 1087 Super Jumping! Jumping! Jumping! 

**Problem Description**
Nowadays, a kind of chess game called “Super Jumping! Jumping! Jumping!” is very popular in HDU. Maybe you are a good boy, and know little about this game, so I introduce it to you now.

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201803/20180327153716571.png)

The game can be played by two or more than two players. It consists of a chessboard（棋盘）and some chessmen（棋子）, and all chessmen are marked by a positive integer or “start” or “end”. The player starts from start-point and must jumps into end-point finally. In the course of jumping, the player will visit the chessmen in the path, but everyone must jumps from one chessman to another absolutely bigger (you can assume start-point is a minimum and end-point is a maximum.). And all players cannot go backwards. One jumping can go from a chessman to next, also can go across many chessmen, and even you can straightly get to end-point from start-point. Of course you get zero point in this situation. A player is a winner if and only if he can get a bigger score according to his jumping solution. Note that your score comes from the sum of value on the chessmen in you jumping path.
Your task is to output the maximum value according to the given chessmen list.
 
**Input**
Input contains multiple test cases. Each test case is described in a line as follow:
N value_1 value_2 …value_N
It is guarantied that N is not more than 1000 and all value_i are in the range of 32-int.
A test case starting with 0 terminates the input and this test case is not to be processed.
 
**Output**
For each case, print the maximum according to rules, and one line one case. 

**Sample Input**

>3 1 3 2
4 1 2 3 4
4 3 3 2 1
0

**Sample Output**

>4
10
3

**Code：**

```java
import java.util.Arrays;
import java.util.Scanner;

public class Main {
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		while (true) {
			int n = sc.nextInt();
			if (n == 0) {
				break;
			}

			int[] a = new int[n];
			int[] dp = new int[n];
			Arrays.fill(dp, 0);
			for (int i = 0; i < n; i++) {
				a[i] = sc.nextInt();
			}

			int score = 0;
			for (int i = 0; i < n; i++) {
				dp[i] = a[i];
				for (int j = 0; j <= i; j++) {
					if (a[j] < a[i]) {
						dp[i] = Math.max(dp[i], dp[j] + a[i]);
					}
				}
				score = Math.max(score, dp[i]);
			}
			System.out.println(score);
		}
	}
}
```
