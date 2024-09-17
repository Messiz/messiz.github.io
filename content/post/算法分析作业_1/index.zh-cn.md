---
title: "算法分析作业_1"
description: 
date: 2020-09-23
image: 
categories:
    - 算法
musicid:
---

废话不多说，直接上题目：

有三种帐篷：三人，双人，情侣，给定其价格p1, p2,
p3，以及男女人数，及其中情侣对数，问最佳方案（总价格最低）。\
ps：三人和双人帐篷可以不住满，但情侣帐篷必须住满

------------------------------------------------------------------------

**代码**

```c++
#include&lt;iostream&gt;
using namespace std;


//参数依次为男生人数，女生人数，情侣人数，三人帐篷价格，双人帐篷价格，爱心帐篷价格
void schedual(int boy, int girl, int couple, int p1, int p2, int p3)
{
   int b = boy, g = girl, c = couple;

 int current_tent[3] = { 0 };    //当前方案帐篷数
 int min_tent[3] = { 0 };        //最佳方案帐篷数
 int current_price = 0;          //当前方案价格
    int min_price = INT_MAX;        //最佳方案价格
    //p1三人, p2双人, p3爱心
   //单价（帐篷住满时每人的开销）
 double sp1 = p1 / 3.0;
   double sp2 = p2 / 2.0;
   double sp3 = p3 / 2.0;

 //从爱心帐篷顶数入手，遍历所有可能(0~couple)，剩下的男女按照帐篷价格以及单价来确定各自顶数
  for (int i = 0; i &lt;= couple; i++)
    {
        boy -= i;
     girl -= i;

      current_price += p3 * i;
      current_tent[2] += i;
     if (p1 &lt; p2)    //三人帐篷价格低于双人帐篷
       {
            while (boy &gt; 0)
           {
                current_price += p1;
              current_tent[0]++;
                boy -= 3;
         }
            while (girl &gt; 0)
          {
                current_price += p1;
              current_tent[0]++;
                girl -= 3;
            }
        }
        else if (sp1 &lt; sp2)//三人帐篷价格高于双人帐篷，但单价低于双人帐篷
     {
            while (boy &gt;= 3)
          {
                current_price += p1;
              current_tent[0]++;
                boy -= 3;
         }
            while (girl &gt;= 3)
         {
                current_price += p1;
              current_tent[0]++;
                girl -= 3;
            }
            if (p1 / 2.0 &lt; p2 / 2.0)//两人住三人帐篷低于双人
            {
                while (boy &gt; 0)
               {
                    current_price += p1;
                  current_tent[0]++;
                    boy -= 3;
             }
                while (girl &gt; 0)
              {
                    current_price += p1;
                  current_tent[0]++;
                    girl -= 3;
                }
            }
            else//两人住三人帐篷高于双人
           {
                while (boy &gt; 0)
               {
                    current_price += p2;
                  current_tent[1]++;
                    boy -= 2;
             }
                while (girl &gt; 0)
              {
                    current_price += p2;
                  current_tent[1]++;
                    girl -= 2;
                }
            }
        }
        else//三人价格高于双人，单价同样
     {
            while (boy != 3 &amp;&amp; boy &gt; 0)
           {
                current_price += p2;
              current_tent[1]++;
                boy -= 2;
         }
            while (girl != 3 &amp;&amp; girl &gt; 0)
         {
                current_price += p2;
              current_tent[1]++;
                girl -= 2;
            }
            if (p1 &lt; 2 * p2)//三人价格低于两个双人
         {
                while (boy &gt; 0)
               {
                    current_price += p1;
                  current_tent[0]++;
                    boy -= 3;
             }
                while (girl &gt; 0)
              {
                    current_price += p1;
                  current_tent[0]++;
                    girl -= 3;
                }
            }
            else
         {
                while (boy &gt; 0)
               {
                    current_price += p2;
                  current_tent[1]++;
                    boy -= 2;
             }
                while (girl &gt; 0)
              {
                    current_price += p2;
                  current_tent[1]++;
                    girl -= 2;
                }
            }
        }

      if (current_price &lt; min_price)
        {
            min_price = current_price;
            min_tent[0] = current_tent[0];
            min_tent[1] = current_tent[1];
            min_tent[2] = current_tent[2];
        }
        //恢复数据，便于下一次循环
       boy = b;
      girl = g;
     couple = c;
       current_price = 0;
        current_tent[0] = 0;
      current_tent[1] = 0;
      current_tent[2] = 0;

    }
    cout &lt;&lt; &quot;最低价格：&quot; &lt;&lt; min_price &lt;&lt; endl;
   cout &lt;&lt; &quot;三人帐篷：&quot; &lt;&lt; min_tent[0] &lt;&lt; endl;
 cout &lt;&lt; &quot;双人帐篷：&quot; &lt;&lt; min_tent[1] &lt;&lt; endl;
 cout &lt;&lt; &quot;爱心帐篷：&quot; &lt;&lt; min_tent[2] &lt;&lt; endl;

}


int main()
{
   //测试数据
   schedual(5, 5, 2, 6, 1, 2);
   schedual(5, 5, 4, 3, 2, 1);
   schedual(102, 69, 45, 8, 5, 4);
   schedual(32, 24, 16, 5, 4, 3);
    schedual(82, 56, 23, 8, 7, 5);
    schedual(52, 74, 40, 6, 6, 4);

  return 0;
}
```