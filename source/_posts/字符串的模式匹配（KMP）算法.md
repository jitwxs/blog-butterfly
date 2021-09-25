---
title: 字符串的模式匹配（KMP）算法
tags: kmp
categories:
  - 算法与数据结构
  - 算法
abbrlink: e6a4d507
date: 2018-09-10 09:42:21
references:
  - name: 字符串匹配的KMP算法
    url: http://www.ruanyifeng.com/blog/2013/05/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: KMP 算法
    url: https://subetter.com/articles/kmp-algorithm.html
    rel: nofollow noopener noreferrer
    target: _blank
  - name: KMP字符串匹配算法
    url: https://www.bilibili.com/video/av11866460?zw
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、背景

给定一个`主串`（以 S 代替）和`模式串`（以 P 代替），要求找出 P 在 S 中出现的位置，此即**串的模式匹配问题**。

`Knuth-Morris-Pratt` 算法（简称 KMP）是解决这一问题的常用算法之一，这个算法是由高德纳（Donald Ervin Knuth）和沃恩·普拉特在1974年构思，同年詹姆斯·H·莫里斯也独立地设计出该算法，最终三人于1977年联合发表。

在继续下面的内容之前，有必要在这里介绍下两个概念：**真前缀** 和 **真后缀**。

![真前后缀](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910091526971.png)

由上图所得， "真前缀"指**除了自身以外，一个字符串的全部头部组合**；"真后缀"指**除了自身以外，一个字符串的全部尾部组合**。

## 二、朴素字符串匹配算法

初遇串的模式匹配问题，我们脑海中的第一反应，就是朴素字符串匹配（即所谓的暴力匹配），代码如下：

```java
private static int simple(char[] S, char[] P) {
    // i为完整串S下标，j为模式串P下标
    int i = 0, j = 0;

    while (i < S.length && j < P.length) {
        // 若相等，都前进一步
        if (S[i] == P[j]) {
            i++;
            j++;
        } else {
            i = i - j + 1;
            j = 0;
        }
    }

    // 匹配成功
    if (j == P.length) {
        return i - j;
    }
    return -1;
}
```
暴力匹配的时间复杂度为 $O(nm)$，其中 $n$ 为 S 的长度，$m$ 为 P 的长度。很明显，这样的时间复杂度很难满足我们的需求。

接下来进入正题：时间复杂度为 $O(n+m)$ 的 KMP 算法。

## 三、KMP字符串匹配算法

### 3.1 算法流程

以下摘自阮一峰的字符串匹配的KMP算法，并作稍微修改。

（1）首先，主串"BBC ABCDAB ABCDABCDABDE"的第一个字符与模式串"ABCDABD"的第一个字符，进行比较。因为B与A不匹配，所以模式串后移一位。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/201809100920441.png)

（2）因为B与A又不匹配，模式串再往后移。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092109282.png)

（3）就这样，直到主串有一个字符，与模式串的第一个字符相同为止。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092127830.png)

（4）接着比较主串和模式串的下一个字符，还是相同。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/201809100921481.png)

（5）直到主串有一个字符，与模式串对应的字符不相同为止。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092205146.png)

（6）这时，最自然的反应是，将模式串整个后移一位，再从头逐个比较。这样做虽然可行，但是效率很差，因为你要把"搜索位置"移到已经比较过的位置，重比一遍。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092221315.png)

（7）一个基本事实是，当空格与D不匹配时，你其实是已经知道前面六个字符是"ABCDAB"。KMP算法的想法是，设法利用这个已知信息，不要把"搜索位置"移回已经比较过的位置，而是继续把它向后移，这样就提高了效率。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092238573.png)

（8）怎么做到这一点呢？可以针对模式串，设置一个跳转数组`int next[]`，这个数组是怎么计算出来的，后面再介绍，这里只要会用就可以了。

|    i    |  0   |  1   |  2   |  3   |  4   |  5   |  6   |
| :-----: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|   模式串   |  A   |  B   |  C   |  D   |  A   |  B   |  D   |
| next[i] |  -1  |  0   |  0   |  0   |  0   |  1   |  2   |

（9）已知空格与D不匹配时，前面六个字符"ABCDAB"是匹配的。根据跳转数组可知，不匹配处D的next值为2，因此接下来**从模式串下标为2的位置开始匹配**。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092347236.png)

（10）因为空格与Ｃ不匹配，C处的next值为0，因此接下来模式串从下标为0处开始匹配。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092403362.png)

（11）因为空格与A不匹配，此处next值为-1，表示模式串的第一个字符就不匹配，那么直接往后移一位。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092440973.png)

（12）逐位比较，直到发现C与D不匹配。于是，下一步从下标为2的地方开始匹配。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/2018091009245921.png)

（13）逐位比较，直到模式串的最后一位，发现完全匹配，于是搜索完成。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910092514887.png)

如果你还是不太理解，可以看看这个视频，对KMP的流程说的简单明了：

<center class="video-container"><iframe src="//player.bilibili.com/player.html?aid=11866460&cid=19594712&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe></center>

### 3.2 next 数组是如何求出的

next 数组的求解基于“真前缀”和“真后缀”，即 `next[i]` 等于 `P[0]...P[i - 1]` 最长的相同真前后缀的长度（请暂时忽视i等于 0 时的情况，下面会有解释）。我们依旧以上述的表格为例，为了方便阅读，我复制在下方了。

|     i     |  0   |  1   |  2   |  3   |  4   |  5   |  6   |
| :-------: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|    模式串    |  A   |  B   |  C   |  D   |  A   |  B   |  D   |
| next[ i ] |  -1  |  0   |  0   |  0   |  0   |  1   |  2   |

1. $i = 0$，对于模式串的首字符，我们统一为 `next[0] = -1`；
2. $i = 1$，前面的字符串为 `A`，其最长相同真前后缀长度为 0，即 `next[1] = 0`；
3. $i = 2$，前面的字符串为 `AB`，其最长相同真前后缀长度为 0，即 `next[2] = 0`；
4. $i = 3$，前面的字符串为 `ABC`，其最长相同真前后缀长度为 0，即 `next[3] = 0`；
5. $i = 4$，前面的字符串为 `ABCD`，其最长相同真前后缀长度为 0，即 `next[4] = 0`；
6. $i = 5$，前面的字符串为 `ABCDA`，其最长相同真前后缀为 `A`，即 `next[5] = 1`；
7. $i = 6$，前面的字符串为 `ABCDAB`，其最长相同真前后缀为 `AB`，即 `next[6] = 2`；
8. $i = 7$，它是最后一个字符，我们直接忽略掉它。

那么，为什么根据最长相同真前后缀的长度就可以实现在不匹配情况下的跳转呢？举个代表性的例子：假如 `i = 6` 时不匹配，此时我们是知道其位置前的字符串为 `ABCDAB`，仔细观察这个字符串，首尾都有一个 `AB`，既然在`i = 6`处的D不匹配，我们为何不直接把 `i = 2` 处的 C 拿过来继续比较呢，因为都有一个 `AB` 啊，而这个 `AB` 就是 `ABCDAB` 的最长相同真前后缀，其长度2正好是跳转的下标位置。

有的读者可能存在疑问，若在 `i = 5` 时匹配失败，按照我讲解的思路，此时应该把 `i = 1` 处的字符拿过来继续比较，但是这两个位置的字符是一样的啊，都是 `B`，既然一样，拿过来比较不就是无用功了么？其实不是我讲解的有问题，也不是这个算法有问题，而是这个算法还未优化，关于这个问题在下面会详细说明，不过建议读者不要在这里纠结，跳过这个，下面你自然会恍然大悟。

思路如此简单，接下来就是代码实现了，如下：

```java
private static int[] getNext(char[] P) {
    // i为数组下标
    int i = 0, j = -1;
    int[] next = new int[P.length];
    // next数组第0位为-1
    next[0] = -1;

    while (i < next.length - 1) {
        if (j == -1 || P[i] == P[j]) {
            i++;
            j++;
            next[i] = j;
        } else {
            j = next[j];
        }
    }

    return next;
}
```

上述代码就是用来求解模式串中每个位置的`next[]`值，下面具体分析，我把代码分为两部分来讲：

**（1）$i$ 和 $j$ 的作用是什么？**

$i$ 和 $j$ 就像是两个”指针“，一前一后，通过移动它们来找到最长的相同真前后缀。

**（2）if...else...语句里做了什么？**

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910093104457.png)

假设 $i$ 和 $j$ 的位置如上图，由 `next[i] = j` 得，也就是对于位置i来说，**区段[0, i - 1]的最长相同真前后缀分别是[0, j - 1]和[i - j, i - 1]，即这两区段内容相同**。

按照算法流程，`if(P[i] == P[j])`，则 `i++; j++; next[i] = j;`；若不等，则 `j = next[j]`，见下图：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180910093138161.png)

`next[j]` 代表[0, j - 1]区段中最长相同真前后缀的长度。如图，用左侧两个椭圆来表示这个最长相同真前后缀，即这两个椭圆代表的区段内容相同；同理，右侧也有相同的两个椭圆。所以else语句就是利用第一个椭圆和第四个椭圆内容相同来加快得到[0, i - 1]区段的相同真前后缀的长度。

细心的朋友会问if语句中 `j == -1` 存在的意义是何？第一，程序刚运行时，j 是被初始为 -1，直接进行 `P[i] == P[j]` 判断无疑会边界溢出；第二，else 语句中 `j = next[j]`，j 是不断后退的，若j在后退中被赋值为 -1（也就是 `j = next[0]`），在 `P[i] == P[j]` 判断也会边界溢出。综上两点，其意义就是为了特殊边界判断。

## 四、完整代码

```java
public class Main {
    /**
     * 计算最大相同真前后缀数组
     * @param P 模式串
     */
    private static int[] getNext(char[] P) {
        // i为数组下标
        int i = 0, j = -1;
        int[] next = new int[P.length];
        // next数组第0位为-1
        next[0] = -1;

        while (i < next.length - 1) {
            if (j == -1 || P[i] == P[j]) {
                i++;
                j++;
                next[i] = j;
            } else {
                j = next[j];
            }
        }

        return next;
    }

    /**
     * 找到首次匹配下标
     * @param S 完整串
     * @param P 模式串
     */
    private static int kmp(char[] S, char[] P) {
        int[] next = getNext(P);
        // i为完整串下标，j为模式串下标
        int i = 0, j = 0;

        while (i < S.length && j < P.length) {
            // 对模式串第一位不匹配或str[i] == pattern[j]
            if (j == -1 || S[i] == P[j]) {
                i++;
                j++;
            } else {
                // 匹配失败时，跳转下标
                j = next[j];
            }
        }

        // 匹配成功
        if (j == P.length) {
            return i - j;
        }

        return -1;
    }

    public static void main(String[] args) {
        int len = kmp("bbc abcdab abcdabcdabde".toCharArray(), "abcdabd".toCharArray());
        System.out.println(len);
    }
}
```

## 五、KMP优化

|     i     |  0   |  1   |  2   |  3   |  4   |  5   |  6   |
| :-------: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|    模式串    |  A   |  B   |  C   |  D   |  A   |  B   |  D   |
| next[ i ] |  -1  |  0   |  0   |  0   |  0   |  1   |  2   |

以上面的表格为例，若在 `i = 5` 时匹配失败，按照之前的代码，此时应该把 `i = 1` 处的字符拿过来继续比较，但是这两个位置的字符是一样的，都是 `B` ，既然一样，拿过来比较不就是无用功了么？

这在之前已经解释过，之所以会这样是因为 KMP 不够完美。可以进行如下改写生成next数组的方法：

```java
private static int[] getNextVal(char[] P) {
   // i为数组下标
    int i = 0, j = -1;
    int[] next = new int[P.length];
    // next数组第0位为-1
    next[0] = -1;

    while (i < next.length - 1) {
        if (j == -1 || P[i] == P[j]) {
            i++;
            j++;

			// 修改了这里
            if(P[i] != P[j]) {
                next[i] = j;
            } else {
                next[i] = next[j];
            }
        } else {
            j = next[j];
        }
    }

    return next;
}
```
在此也给各位读者提个醒，KMP算法严格来说分为KMP算法（未优化版）和KMP算法（优化版），所以建议读者在表述KMP算法时，最好告知你的版本，因为两者在某些情况下区别很大，这里简单说下。

**KMP 算法（未优化版）：** next 数组表示最长的相同真前后缀的长度，我们不仅可以利用next来解决模式串的匹配问题，也可以用来解决类似字符串重复问题等等，这类问题大家可以在各大 OJ 找到，这里不作过多表述。

**KMP 算法（优化版）：** 根据代码很容易知道（名称也改为了 nextval），优化后的 next 仅仅表示相同真前后缀的长度，但**不一定是最长**（称其为“最优相同真前后缀”更为恰当）。此时我们利用优化后的 next 可以在模式串匹配问题中以更快的速度得到我们的答案（相较于未优化版），但是上述所说的字符串重复问题，优化版本则束手无策。

所以，该采用哪个版本，取决于你在现实中遇到的实际问题。

