---
title: "Stringstream复用中刷新缓冲区的问题"
description: Stringstream复用中刷新缓冲区的问题
date: 2020-02-04
image: 
categories:
    - c++
musicid:
---
今天看书时遇到set的使用，于是将给的例子自己写了一遍，结果意外地发现了一些问题。

题目很简单：

```
问题：
输入文本，找出不同单词，按字典顺序从小到大输出，不区分大小写。

样例输入：
Adventures in Disneyland

Two blondes were going to Disneyland when they came to a fork in the road. The sign read:&quot;Disneyland Left.&quot;

So they went home.
```
下面直接放上代码：

```c++
#include&lt;iostream&gt;
#include&lt;string&gt;
#include&lt;sstream&gt;
#include&lt;set&gt;

using namespace std;

set&lt;string&gt; dict;

int main()
{
   string word, buff;
  stringstream sstream;
   

    while (cin &gt;&gt; word)
  {
        //循环处理一个输入的单词
        for (int i = 0; i &lt; word.size(); i++)
        {
            if (isalpha(word[i]))
              word[i] = tolower(word[i]);
         else
             word[i] = &#39; &#39;;
        }
        
      //此处需清空stringstream的缓冲区，否则无法复用sstream读取多组数据
      //而.clear()方法不能真正做到清除，而会使内存越来越高，因此采用.str()方法
     //clear清空的是标志位，实际占用的内存并未释放！！！
        //先将其赋为空字符串&quot;&quot;，再调用clear()方法
       sstream.str(&quot;&quot;);
      sstream.clear();
      sstream &lt;&lt; word;  //可能出现两个单词被标点连接，从而处理完为两个单词的情况，故使用stringstream
        while (sstream &gt;&gt; buff)
            dict.insert(buff);  //加入集合
   }
    //输出结果（set模板类并没有定义[]，故需要通过迭代器来遍历）
    for (auto it = dict.begin(); it != dict.end(); it++)
        cout &lt;&lt; *it &lt;&lt; endl;

    return 0;
}
```

开始时没有这两行代码，结果输出只有一个单词

```c++
sstream.str(&quot;&quot;);
sstream.clear();
```

经过调试，发现是因为sstream是在前面定义的，第一次读取后，缓冲区一直没有刷新，所以后面的单词无法读入sstream。而书上的代码则在需要读入处每次都重新定义一个stringstream类型的变量，所以没有这个问题。

```c++
...
stringstream ss(word);      //书上该处的代码如是
...
```

经过一番搜索，找到了解决办法，即运用clear方法，但同时得知，clear方法仅仅清除了标志位，而对于实际的缓冲区内存并未做实际的释放处理。所以需要在clear之前手动将sstream赋值为空字符串，即：

```c++
sstream.str(&quot;&quot;);
```