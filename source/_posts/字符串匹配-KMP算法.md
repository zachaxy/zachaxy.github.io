---
title: 字符串匹配--KMP算法
date: 2017-04-26 09:21:26
tags: 算法
---


> **Knuth-Morris-Pratt 字符串查找算法**（常简称为“**KMP算法**”）可在一个主“文本字符串”`s`内查找一个“词”`m`的出现位置。此算法通过运用对这个词在不匹配时本身就包含足够的信息来确定下一个匹配将在哪里开始的发现，从而避免重新检查先前匹配的字符。

# 初始化变量

待查找的字符串:s    
要匹配的字符串:m    

# 普通的做法

字符串匹配算法最常规的思路是在s中一个字符一个字符的与m中的第一个字符比较,匹配了说明找到了源头(i)这个i接下来很可能会与m完全匹配的,再比较各自接下来一个字符,如果匹配不上,说明上次找的i不是我们想要的源头,接着怎么办?从i+1开始,看i+1是否有可能是这个源头.

## 普通做法的缺点

时间复杂度高,造成这个的原因是因为在匹配过程中,虽然没有完全匹配上.但是很可能已经匹配了一部分,已经匹配上了的这一小部分并没有被充分利用起来;


# KMP算法

>充分利用在匹配过程中,没有完全匹配上但是已经匹配上一部分的资源.

## 几个概念:   

下面的概念都是针对一个字符串而言的,eg:一个字符串为abc

前缀:a,ab,包含首字符,但不包含末字符的字符串;       
后缀:c,bc,包含末字符,但不包含首字符的字符串;       

## 具体步骤

### 构造辅助数组

1. 拿到m字符串,生成一个与m等长的整型数组next[]
2. 初始化next[0] = -1,next[1] = 0
3. 从2开始遍历m,next[i]的值就是 字符串 m[0~i-1] 的相同的最长前缀和最长后缀的长度



### 匹配过程
- 开始匹配,定义两个游标,si和mi,初始均为0
- 如果能匹配上,si++,mi++
- 如果匹配不上,注意此时的隐含条件是mi之前的已经匹配上了,我们想下次移动的时候不是把m移动到上次匹配s的起止位置之后的一个字符处,而是看m[0~i-1]中的前缀和后缀是不是有一样的地方,这样就可以重复利用,这不就是next数组的作用吗?因此查看此时的next数组,将mi = next[mi];接下来当然从si和mi继续匹配,看能否匹配了
- 如果next[mi] == -1,表明连m的第一个字符都匹配不上,那么只能si++;


### 源代码

```java
/**
 * Author: zhangxin
 * Time: 2016/12/16 0016.
 * Desc:KMP算法;
 * 核心是next[]数组的计算,以及在匹配过程中如何使用next,感性上的理解是需要移动数组的,但在实际的使用中时只需要修改si与mi即可;
 */
public class KMP {
    public static int getIndexOf(String s, String m) {
        if (s == null || m == null || m.length() < 1 || s.length() < m.length()) {
            return -1;
        }
        char[] ss = s.toCharArray();
        char[] ms = m.toCharArray();
        int si = 0; //str中当前i的位置;其实真正的游标是si,你想匹配字符串的时候,主标是s掌控的;
        int mi = 0;//match中当前i的位置;
        int[] next = getNextArray(ms);
        while (si < ss.length && mi < ms.length) {
            if (ss[si] == ms[mi]) {
                //当前字符能匹配上,si,mi都前移一位;
                si++;
                mi++;
            } else if (next[mi] == -1) {
                //匹配不上,mi=0,第一个字符都匹配不上,si前移一位;
                si++;
            } else {
                //前面还是有部分能匹配上的,mi=next[mi];si不变;
                mi = next[mi];
            }
        }
        return mi == ms.length ? si - mi : -1;
    }

    /***
     * 获取nextArr数组
     * nextArr[0] = -1;因为如果第一个字符都匹配不上,那么match整体后移1位;
     * nextArr[1] = 0;因为第一个字符没有前缀也没有后缀,肯定是0
     * @param ms match字符串对应的字符数组
     * @return
     */
    public static int[] getNextArray(char[] ms) {
        if (ms.length == 1) {
            return new int[]{-1};
        }
        int[] next = new int[ms.length];
        next[0] = -1;
        next[1] = 0;
        int pos = 2; //当前位置;
        int cn = 0; //前缀开始匹配的位置;
        while (pos < next.length) {
            if (ms[pos - 1] == ms[cn]) {
                next[pos++] = ++cn;
            } else if (cn > 0) {
                //遇到某个字符匹配不上了,那么直接下次不能再用之前的了,而是将cn置0
                // next先不设置,cn==0后,再去进入循环,从头匹配;也许能匹配的上;
                //cn = next[cn]; 不看这一句,删掉;
                cn = 0;
            } else {
                next[pos++] = 0;
            }
        }
        return next;
    }

    public static void main(String[] args) {
        String str = "abcabcababaccc";
        String match = "ababa";
        /*str = "abxxxabwwab";
        match = "xab";*/
        System.out.println(getIndexOf(str, match));
        System.out.println(str.indexOf(match));
    }

}
```


## next[]数组深入分析

```
分析 match = "ababa"  => next[5]
next[i]的含义:match[0~i-1]组成的字符串,最长前缀(不包含最后一个字符)和最长后缀(不包含第一个字符)匹配上的长度;
初始化:
next[0] = -1;显然match[0~0-1]无意义,令其等于-1,代表第一个字符都匹配不上,match直接右移一位
next[1] = 0;显然match[0~1-1]就是match[0],只有一个字符,没有前缀也没有后缀;所以next[1]=0;
接下来开始匹配了,match[2]这个位置前面已经有两个字符了,可能前两个字符相等,那么next[2]=1,不相等,next[2]=0
可以发现的是,第一次匹配一定是match[i-1]和match[0]匹配;接下来如果再能匹配,因为match[i-1]和match[0]已匹配,match[i]和match[1]也能匹配了

ababa的next数组;
    0 1 2 3 4
    a b a b a
   -1 0 0 1 2

接下来拿ababcccc与ababe来匹配,ababe的next数组为{-1,0,0,1,2},和上面的ababa是一样的;
一开始是可以匹配的,当si = mi = 4 时,匹配不上了,这个时候的隐含条件就是m中的[0~mi-1]是都能匹配上的
这时候找next[mi],next[4]=2,说明mi之前的字符串中前两个和最后两个可以匹配上,所以下次比的时候,si = 4,mi = 2,从m[3]字符处开始匹配;
```