---
title: 字符串匹配--RK算法
date: 2017-04-26 09:32:04
tags: 算法
mathjax: true
---

> Rabin-Karp采用了把字符进行预处理，也就是对每个字符进行对应进制数并取模运算，类似于通过某种哈希函数计算其函数值，比较的是每个字符的函数值。预处理时间O(m)，理论匹配时间是O((n-m+1)m)。这里要乘以m是考虑到对hash值相同的进行一次校验;
>
> 由于hash冲突的存在，当hash值相同的时候，还是需要朴素算法来进行必要的比较,所以时间复杂性为O（m*n）。但是**现实中**hash冲突出现的可能性不是很大，所以相比较而言，复杂性还是比较小的，仅仅为O(m+n)



# 初始化变量

- 待匹配的字符串`s`
- 待匹配的子串`m`
- `m`的长度为`M`
- `R`进制:如果字符串中所有的字符都是小写英文字母,那么就是26进制,如果都是数字,那么就是10进制,如果没有说明,那么就以`ASCII表`256进制来计算
- `Q`随机的大素数,在不溢出的情况下选择一个尽可能大的素数
- `RM`这里并不是`R*M`而是 $R^{M-1}$%Q
- `hash()`函数,对`m`进行运算,得到目标hash值
- `targetHash = hash(m)`




# 基本思想

这里以下面参数为例,讲解RK算法的基本思想;

- `s=3141592653589793`
- `m=26535`
- `M=5`
- `Q=997`
- `hash(x) = x % Q`
- `targetHash = 26535 % 997 = 613`



那么在接下来的匹配中可以得出

|  i   |  0   |  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |  9   |    10     |  11  |  12  |  13  |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :-------: | :--: | :--: | :--: |
|  0   |      |      |      |      | 508  |      |      |      |      |      |           |      |      |      |
|  1   |      |      |      |      |      | 201  |      |      |      |      |           |      |      |      |
|  2   |      |      |      |      |      |      | 715  |      |      |      |           |      |      |      |
|  3   |      |      |      |      |      |      |      | 971  |      |      |           |      |      |      |
|  4   |      |      |      |      |      |      |      |      | 442  |      |           |      |      |      |
|  5   |      |      |      |      |      |      |      |      |      | 929  |           |      |      |      |
|  6   |      |      |      |      |      |      |      |      |      |      | 613  (匹配) |      |      |      |

1. 从`s`第0位开始,取前`M`位,用`hash()`方法计算出其值与`targetHash`进行对比
2. 如果相等,那么这`M`位可能会匹配上,
3. 如果不相等,那么`s`后移一位,继续计算相应`M`为的`hash()`,进行对比



以上过程只是对RK算法的一个简单描述,注意到每次也是后移一位,计算`hash()`值,因为我们的s中都是数字,可以很方便的来计算出`hash`值,如果是字母的话,计算hash值可能就不是那么容易了,这也是RK算法要解决的核心问题,因为我们大部分情况下面对的都是字符串,我们需要将字符串转换为对应的数值,如何转换呢?那就是接下来用到的散列函数

#  计算散列函数

```java
private long hash(String key, int M) {
    long h = 0;
    for (int i = 0; i < M; i++) {
      	h = (R * h + key.charAt(i)) % Q;
    }
    return h;
}
```

该函数的目标是计算一次`m`的`targetHash`,计算一次`s`的前`M`位的`hash`,仅此两次调用,那么我们不是每次都要将`s`的游标后移一位,在计算其hash值吗?这就是RK算法巧妙的地方,如果当前不匹配,那么后移一位,其hash值

**注意: **再次强调`Q`一定要是一个尽可能大的素数,以减少冲突;

# 关键思想

RK算法的基础是对于所有位置`i`,不调用hash()方法,高效计算文本中`i+1,`

我们用 $t_i$表示`s.charAt(i)`,那么`s`其实于`i`,长度为`M`的数值(注意此处并不是hash值)为:

$$x_i = t_iR^{M-1}+t_{i+1}R^{M-2}+...+t_{i+M-1}R^0$$  			`(1)`

$h(x_i)=x_i mod Q$									`(2)`

$$x_{i+1}=(x_i-t_iR^{M-1})R+t_{i+M}$$					`(3)`

`(a + b) % c = ((a % c)+(b % c)) % c`				`(4)`

`(a - b) % c = ((a % c)-(b % c)) % c`				`(5)`

`(a * b) % c = ((a % c)*(b % c)) % c`				`(6)`

基于以上公式,可以方便的得到`search()`方法

```java
    //返回匹配成功的索引;
    private int search(String txt) {
        int N = txt.length();
        long txtHash = hash(txt, M); //计算txt文本的前M位的hash值
        if (targetHash == txtHash && check(0)) {
            return 0; //在开始位置处就匹配成功;
        }

        for (int i = M; i < N; i++) {
            //+Q并不影响对其本质的区别;因为 Q % Q = 0
            txtHash = (txtHash + Q - RM * txt.charAt(i - M) % Q) % Q;  //是为了防止出现负数吧...
            txtHash = (txtHash * R + txt.charAt(i)) % Q;
            if (targetHash == txtHash) {
                if (check(i - M + 1)) {
                    return i - M + 1;
                }
            }
        }

        return -1;
    }
```




# 源码

```java
/**
 * Author: zhangxin
 * Time: 2017/4/12 0012.
 * Desc: 基于指纹的字符匹配算法
 */
public class RabinKarp {
    private long targetHash;  //模拟字符串的散列值
    private int M;          //模拟字符串的长度;
    private int Q;          //大素数
    private int R = 256;    //字母表大小;这里最好设置为可以定制的吧;
    private long RM;        //R^(M-1)%Q;

    public RabinKarp(String pat) {
        this.M = pat.length();
        Q = 997; //或者你自己写一个随机的素数表;
        RM = 1;

        //注意这里是从1开始,到M-1,[1,M-1],一共M-1个数;
        for (int i = 1; i < M; i++) {
            RM = (R * RM) % Q;
        }

        targetHash = hash(pat, M);
    }


    private long hash(String key, int M) {
        long res = 0;
        for (int i = 0; i < M; i++) {
            res = (R * res + key.charAt(i)) % Q;
        }
        return res;
    }

    //这个方法一般不用,从概率上来说,不用再check,如果你不放心,可以添加上字符逐一对比的代码
    public boolean check(int i) {
        return true;
    }

    //返回匹配成功的索引;
    private int search(String txt) {
        int N = txt.length();
        long txtHash = hash(txt, M); //计算txt文本的前M位的hash值
        if (targetHash == txtHash && check(0)) {
            return 0; //在开始位置处就匹配成功;
        }

        for (int i = M; i < N; i++) {
            //+Q并不影响对其本质的区别; Q%Q=0
            txtHash = (txtHash + Q - RM * txt.charAt(i - M) % Q) % Q;  //是为了防止出现负数吧...
            txtHash = (txtHash * R + txt.charAt(i)) % Q;
            if (targetHash == txtHash) {
                if (check(i - M + 1)) {
                    return i - M + 1;
                }
            }
        }

        return -1;
    }
}
```





# 优缺点

优点:

1. 它可以用来检测抄袭，因为它能够处理多模式匹配(多个不同长度的`m`)；
2. 虽然在理论上并不比暴力匹配法更优，但在实际应用中它的复杂度仅为O(n+m);
3. 如果能够选择一个好的哈希函数，它的效率将会很高，而且也易于实现。

缺点:

1. 有许多字符串匹配算法的复杂度小于O(n+m)；
2. 有时候它和暴力匹配法一样慢，并且它需要额外空间。