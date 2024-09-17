---
title: "hdu2203_亲和串"
description: 刷算法题的时光
date: 2020-08-02
image: 
categories:
    - OJ
musicid:
---

今天又ac一条水题。\
\
不多说直接看题-----\>

*Problem Description*

> 人随着岁数的增长是越大越聪明还是越大越笨，这是一个值得全世界科学家思考的问题,同样的问题Eddy也一直在思考，因为他在很小的时候就知道亲和串如何判断了，但是发现，现在长大了却不知道怎么去判断亲和串了，于是他只好又再一次来请教聪明且乐于助人的你来解决这个问题。\
> 亲和串的定义是这样的：给定两个字符串s1和s2,如果能通过s1循环移位，使s2包含在s1中，那么我们就说s2
> 是s1的亲和串。

*Input*

> 本题有多组测试数据，每组数据的第一行包含输入字符串s1,第二行包含输入字符串s2，s1与s2的长度均小于100000。

*Output*

> 如果s2是s1的亲和串，则输出"yes"，反之，输出"no"。每组测试的输出占一行。

*Sample Input*

> AABCD\
> \
> CDAA\
> \
> ASD\
> \
> ASDF

*Sample Output*

> yes\
> \
> no

思路：之前在《算法》第四版中做过类似的题目，想要将s1看作一个循环的字符串，然后判断s2是否包含在其中。其实就可以将两个s1首位拼接，这样就解决了首尾分开的问题，然后直接判断s2是否在新串中即可。

> if (tmp.find(s2) != tmp.npos)

其中这一行代码是用来判断是否是子串的，如果是子串，则find方法返回子串的位置，否则返回一个特定数（也就是npos）。

```c++
#include&lt;iostream&gt;
#include&lt;string&gt;
using namespace std;

int main()
{
    string s1, s2;
  while (cin &gt;&gt; s1)
    {
        cin &gt;&gt; s2;
        string tmp = s1 + s1;
       if (tmp.find(s2) != tmp.npos)
            cout &lt;&lt; &quot;yes&quot; &lt;&lt; endl;
        else
         cout &lt;&lt; &quot;no&quot; &lt;&lt; endl;
 }

  return 0;
}
```