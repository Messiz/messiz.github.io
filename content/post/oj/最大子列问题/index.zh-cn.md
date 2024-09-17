---
title: "最大子列问题"
description: 
date: 2019-09-22
image: 
categories:
    - OJ
musicid:
---

从开始准备写这一篇到实际开始中间鸽了好多天。。

主要是因为懒以及不断的wrong answer，最后在网友博客的帮助下最终ac。

问题描述如下：

**Problem Description**

```
Given a sequence a[1],a[2],a[3]......a[n], your job is to calculate the max sum of a sub-sequence. For example, given (6,-1,5,4,-7), the max sum in this sequence is 6 + (-1) + 5 + 4 = 14.
```

**Input**

```
The first line of the input contains an integer T(1&lt;=T&lt;=20) which means the number of test cases. Then T lines follow, each line starts with a number N(1&lt;=N&lt;=100000), then N integers followed(all the integers are between -1000 and 1000).
```

**Output**

```
For each test case, you should output two lines. The first line is &quot;Case #:&quot;, # means the number of the test case. The second line contains three integers, the Max Sum in the sequence, the start position of the sub-sequence, the end position of the sub-sequence. If there are more than one result, output the first one. Output a blank line between two cases.
```

**Sample Input**

```
2
5 6 -1 5 4 -7
7 0 6 -1 1 -6 7 -5
```

**Sample Output**

```
Case 1:
14 1 4

Case 2:
7 1 6
```

下面直接放上完整代码：

```c++
#include&lt;iostream&gt;
using namespace std;

int main()
{
  //变量定义
  int caseNum;  //输入组数
    int num;    //每组数量
  int arry[100005];   //存放输入
  int currMax, max;   //当前最大和 以及 最大和
  int left, right, temp;    //左右，temp用于计算出left
    cin &gt;&gt; caseNum;
   for (int i = 1; i &lt;= caseNum; ++i)
   {
        
      cin &gt;&gt; num;
       
      //接受输入
       for (int j = 0; j &lt; num; ++j)
        {
            cin &gt;&gt; arry[j];
       }
        //变量初始化
      currMax = 0;
      max = -1001;
      left = 1;
     right = 1;
        temp = 1;

       for (int k = 0; k &lt; num; ++k)
        {
            currMax += arry[k];
           if (max &lt; currMax)
            {
                max = currMax;
                left = temp;
              right = k + 1;  
          }
            
          if (currMax &lt; 0)
          {
                currMax = 0;//此处不能写成currMax = arry[k];
               temp = k + 2;
         }

      }
       //输出
      cout &lt;&lt; &quot;Case &quot; &lt;&lt; i  &lt;&lt; &quot;:&quot; &lt;&lt; endl;
     cout &lt;&lt; max &lt;&lt; &quot; &quot; &lt;&lt; left &lt;&lt; &quot; &quot; &lt;&lt; right &lt;&lt; endl;
       if (i != caseNum)
        {
            cout &lt;&lt; endl;
       }
    }
    return 0;
}
```

之前就曾写过这一道题，但一直超时------因为当时用的是最暴力的嵌套循环遍历

时间复杂度为O(\$n\^2\$)

这种算法的优点在于只需要循环一次，时间复杂度为O(n)。

算法主体部分代码如下：


```c++
for (int k = 0; k &lt; num; ++k)
{
    currMax += arry[k];
    if (max &lt; currMax)
    {
        max = currMax;
        left = temp;
        right = k + 1;  
    }
    
    if (currMax &lt; 0)
    {
        currMax = 0;//此处不能写成currMax = arry[k];
        temp = k + 2;
    }

}
```

从数组头开始，每次将该元素累加至currMax上，作为当前子列的最大值，再与max比较，如果大于max则更新其值。

最关键的一步在于，判断currMax是否小于0，若小于0，则将该子列舍去，从下一位置开始。这里蕴含的原理在于：一旦currMax小于0了，这个子列一定会使后面的子列变小，所以从下一位置开始一定比加上前面的子列大。

我开始还有这样的疑问：它并没有考虑所有的子列，万一从a\[0\]开始，至a\[3\]突然小于0了，那么就直接从a\[4\]开始，而a\[2\]或a\[3\]开始的子列就没有得到验证。

然而如果a\[3\]加上去之后小于0了，那么说明至少前一项（a\[2\]）减去此项（a\[3\]）小于0。因为如果大于0，要满足子列加到a\[3\]突然变负，则需要a\[2\]之前的项小于0，而这是不可能的，因为如果小于0，则在之前的步骤中已经被舍去了。

因而我的疑问只是没有根据的一厢情愿罢了。

（拖沓了近3周，效率啊！！！）