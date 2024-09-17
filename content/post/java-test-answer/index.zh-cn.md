---
title: "Java考试题参考"
description: 
date: 2020-09-07
image: 
categories:
    - java
musicid:
---
写在前面\
\
其实不想写的，因为晚上有课，但还是稍微说一点。\
\
前两题加了注释，后两题没加，也许晚上会加。总之我要先上课了。

------------------------------------------------------------------------

*修改*

-   简化第二题delete方法为1行代码
-   更改第二题（13）和第三题（5），对不起各位！

------------------------------------------------------------------------

**第一题**

直接上代码了，前两题有注释：

*NumUtil*

```java
/**
 * 1.C语⾔标准库函数int atoi (const char *)和long int atol ( const char *)，可以分别将给定的字符串转
 * 换为整数（int）和⻓整数（long）。具体转换规则如下：丢弃前⾯的空⽩字符（空格、⽔平以及垂
 * 直制表、换⻚、回⻋、换⾏，其相应的ASCII码值附后），直到第⼀个⾮空⽩字符为⽌；从该字符开
 * 始，接收可选的+或-号，后跟任意的⼗进制数字字符并将其解释为数字值；字符串在构成整数的数
 * 字字符之后可以包含额外的其它字符，⼀旦遇到这样的字符，转换⽴即停⽌；如果字符串的第⼀个
 * ⾮空⽩字符并不是⼀个有效的数字字符，或者它为空或仅仅包含空⽩字符，那么不会执⾏转换，仅
 * 仅返回0。以下是对外提供这两个功能的NumUtil类，请根据该转换规则以及中的注释：（1）如果类
 * 或有关⽅法的声明中存在错误，说明如何修改。 main()⽅法体中，除了编号部分，不允许被修改。
 * （2）完成相关⽅法，要求不得使⽤标准Java⾃身的有关解析数值的类及其⽅法；（3）在main()⽅
 * 法中，根据下表中所给的测试⽤例执⾏单元测试（并⾮实际产品开发中的测试⽅式）。
 */

// NumUtil.java
public class NumUtil {
    //将提供的功能函数均改为static，否则在main函数中无法调用
    //由于是提供函数的Util类，需改为public
    /**
     * Determine if the given character is a whitespace.
     *
     * @param c the character to check.
     * @return true, if the given character is a whitespace;
     * false, otherwise.
     */
    static private boolean isWhitespace(char c) {
        // (1)
        switch (c) {
            case &#39; &#39;:       //空格
            case &#39;\t&#39;:      //制表符
            case &#39;\f&#39;:      //换页
            case &#39;\r&#39;:      //回车
            case &#39;\n&#39;:      //换行
            case &#39;\u000b&#39;:  // \v，垂直制表符，java中没有，故用Unicode编码
                return true;
            default:
                return false;
        }
    }

    /**
     * Determine the position of the first non-whitespace character
     * in the given string.
     *
     * @param source the string to examine.
     * @return the position of the first non-whitespace character
     * in the given string.
     */
    static private int skipWhitespace(String source) {
        // (2)
        int i = 0;
        while (i &lt; source.length()) {
            if (isWhitespace(source.charAt(i)))
                i++;
            else
                break;
        }
        return i;       //i表示第一个非空白字符位置，若i等于字符串长度则表示全空白
    }

    /**
     * Try to convert the given string to a long integer.* @param source the string to convert.
     *
     * @return the converted long integer.
     */
    public static long atol(String source) {
        int sign = 1, i; // sign of the result long integer, i current position in source
        long res = 0; // the final result long integer
// skip the whitespaces
        // (3)
        i = skipWhitespace(source);     //获取第一个非空白字符位置
        if (i == source.length())
            return res;                 //若全空，则返回0
// process the sign symbol
        // (4)
        switch (source.charAt(i)) {
            case &#39;+&#39;:   //sign初始值为1
                i++;
                break;
            case &#39;-&#39;:
                sign = -1;  //修改sign为-1（符号）
                i++;
                break;
            default:
                break;
        }
// process any digit characters
        // (5)
        for (; i &lt; source.length(); i++) {
            char c = source.charAt(i);
            if (c &gt;= &#39;0&#39; &amp;&amp; c &lt;= &#39;9&#39;)
                res = 10 * res + c - &#39;0&#39;;       //由于从前向后遍历，故每次扩大10倍
            else
                break;
        }
// return the converted result long integer
        return sign * res;
    }

    /**
     * Try to convert the given string to an integer.
     *
     * @param source the string to convert.
     * @return the converted integer.
     */
    public static int atoi(String source) {
        // (7)
        return (int) atol(source);   //调用atol函数，将long型结果变量强转为int
    }

    /**
     * Determine if the given test had passed and output the result
     * according to the following format:
     * &lt;pre&gt;
     * Test case: n -&gt; passed|failed
     * where n is the number of the test case.
     * &lt;/pre&gt;
     *
     * @param test     the number of the test case.
     * @param factual  the converted integer.
     * @param expected the expected integer.
     */
    public static void check(int test, int factual, int expected) {
        // (8)
        System.out.print(&quot;Test case: &quot; + test + &quot; -&gt; &quot;);
        if(factual == expected)
            System.out.println(&quot;passed&quot;);
        else
            System.out.println(&quot;failed&quot;);
    }

    // Just for test!
    public static void main(String[] args) {// test case 1
        String source = &quot;&quot;;
        int res = atoi(source);
        check(1, res, 0);
// test case 2
        // (9)
        source = &quot; \t\u000b\f\r\n&quot;;         //根据pdf中&quot;空格|⽔平制表|垂直制表|换⻚|回⻋|换⾏&quot;
        res = atoi(source);
        check(2, res, 12345);
// test case 3
//        …
        // test case 4
//        …
        // test case 5
//        …
    }
}
```

------------------------------------------------------------------------

**第二题**

*CacheItem*

```java
/**
 * 2.在Web应⽤程序中，为了提⾼系统的响应性，可以将⽤户请求的结果缓存在内存中。当⽤户后续
 * 请求相同的资源时，不必从数据存储中提取，⽽可以直接从内存中提供；当某个⽤户对相同的资源
 * 进⾏更新时，除了更新数据存储，还应该更新缓存，但统⼀资源标识符（URI-Uniform Resource
 * Identifier）不能更改；当请求删除某个资源被时，除了从数据存储中将其删除之外，如果该资源已
 * 经被缓存，还应将其从缓存中删除；为了降低系统内存的消耗，每隔⼀定的时间，应该清除缓存中
 * 超过⼀定期限的项⽬。缓存的项⽬中包括请求的URI（字符串形式）、应答体（字符串形式）以及进
 * ⼊缓存的时间（UNIX纪元即⾃从1970年1⽉1⽇0时0分0秒以来的毫秒数）。为了简单起⻅，不考虑
 * 多服务器的分布式缓存⽽仅仅考虑单台服务器，但需要考虑多线程同步问题。以下表示HTTP应答缓
 * 存的类HttpCache在整个JVM中只能有⼀个实例，要求采⽤惰性（lazy）⽅式实例化，并且在多线程
 * 环境下，对该类的实例化不得采⽤同步锁。所需要的有关类的⽅法在后续的表中给出。请编写以下
 * 代码⻣架中编号为(1)、 (2)等部分的代码（答题时标注相应的序号）并说明HttpCache在多线程环境
 * 下哪些些⽅法需要同步以及在当前条件下如何同步。测试数据在后⾯的表格中给出。
 */


package com.njfu.cache;

public class CacheItem {
    private String uri;
    private long since;
    private String body;

    // constructor
    // (1)
    public CacheItem(String uri, String body) { //构造方法
        this.uri = uri;
        this.body = body;
        this.since = System.currentTimeMillis();
    }

    // accessor for uri
    // (2)
    public String getUri() {
        return this.uri;
    }

    public void setUri(String uri) {
        this.uri = uri;
    }

    // accessor for since
    // (3)
    public long getSince() {
        return this.since;
    }

    public void setSince(long since) {
        this.since = since;
    }

    // accessor for body
    // (4)
    public String getBody() {
        return this.body;
    }

    public void setBody(String body) {
        this.body = body;
    }
// When call System.out.println(aCacheItem), output “URI: uri, Since: since, Body: body”
    // (5)

    @Override
    public String toString() {  //覆盖toString方法
        return new String(&quot;URI: &quot; + this.uri + &quot;, Since: &quot; + this.since + &quot;, Body: &quot; + this.body);
    }
}
```

*HttpCache*

```java
// (6)
package com.njfu.cache;

import java.util.HashMap;
import java.util.Iterator; // for Java 8-
import java.util.Map;
import java.util.Set; // Optional

// Don’t inherit from HahMap etc.
public class HttpCache {
    // Cache all the HTTP responses
    private Map&lt;String, CacheItem&gt; cache;

    // Make the globall instance of the HTTP response cache
    // (7)
    private static class CacheMaker {
        private static final HttpCache instance = new HttpCache();
    }

    // constructor
    // (8)
    private HttpCache() {   //构造器private，禁止直接实例化
        this.cache = new HashMap&lt;&gt;();
    }

    /**
     * Retrieve the globally unique instance of the HTTP response cache.
     *
     * @return the globally unique instance of the HTTP response cache.
     */
    public static HttpCache getInstance() {
        // (9)
        /**
         * 当第一次调用getInstance()时，
         * 虚拟机发现访问了CacheMaker.instance，
         * 但这是CacheMaker这个静态内部类的一个静态成员，
         * 但它并没有被初始化，事实上JVM这个时候加载该静态内部类，
         * 并初始化其instance成员
         * 但JVM在加载类的时候，会确保多线程同步，
         * 所以不会存在将HttpCache创建两个或更多的实例的情况
         */
        return CacheMaker.instance;
    }

    /**
     * Cache the HTTP response to the given resource URI.
     *
     * @param uri  the URI of the requested resource.
     * @param body the HTTP response body for the requested resource.
     * @return true, if the cache item for the given URI has already
     * existed, it was just updated; false, this is a new resource
     * request.
     */
    public boolean cache(String uri, String body) {
        // (10)
        CacheItem item = this.cache.get(uri);   //用uri从map中尝试寻找
        if (item == null) {  //找不到则创建新CacheItem，并加入map中
            this.cache.put(uri, new CacheItem(uri, body));
            return false;
        }
        //找到则更新body和since
        item.setBody(body);
        item.setSince(System.currentTimeMillis());
        return true;
    }

    /**
     * Try to retrieve the cached response body by the given request URI.
     *
     * @param uri the request URI whose response body would be retrieved.
     * @return the response body corresponding to the given request
     * URI or null if the cache does not contain it.
     */
    public String get(String uri) { //上面注释中说的比较清楚了，通过uri查找对应cache中body内容
        // (11)
        CacheItem item = this.cache.get(uri);
        return item == null ? null : item.getBody();
    }

    /**
     * Remove the cached HTTP response by the given resource URI.
     *
     * @param uri the URI of the resource to deleted.
     * @return true, if the cache do contain the HTTP response for
     * the given resource URI and it was deleted; false, otherwise.
     */
    public boolean delete(String uri) {
        // (12)
//        CacheItem item = this.cache.get(uri);
//        if(item == null) {  //未找到，返回false
//            return false;
//        }
        //否则删除map中的项，返回true
        //简化为1行代码
        return cache.remove(uri) != null;
//        return true;
    }

    /**
     * Delete all the cached HTTP responses earlier than the given
     * threshold.
     *
     * @param delta the period in milliseconds that determine all
     *              the responses ahead of the current time delta milliseconds
     *              would be deleted.
     */
    public void purge(long delta) {
        // (13) //包括Java 8前后不同的实现
        long now = System.currentTimeMillis();
        //java 8+
        this.cache.entrySet().removeIf(e -&gt; e.getValue().getSince() &lt;= now - delta);
        //java 8-
        for (Iterator&lt;Map.Entry&lt;String, CacheItem&gt;&gt; iter = cache.entrySet().iterator(); iter.hasNext(); ) {
            if(iter.next().getValue().getSince() &lt;= now - delta) {
                iter.remove();
            }
        }
    }

    // Just for test
    @Override
    public String toString() {
        return this.cache.toString();
    }

    // Just for test, multi-threads not concerned!
    public static void main(String[] args) {
// In this case, ‘the HttpCache.’ part is optional
        HttpCache.getInstance().cache(&quot;185080101&quot;, /*(14)*/ &quot;no: \&quot;185080101\&quot;, name: \&quot;zhangsan\&quot;, gender: \&quot;M\&quot;&quot;);
//…
        HttpCache.getInstance().cache(&quot;185080102&quot;,  &quot;no: \&quot;185080102\&quot;, name: \&quot;lisi\&quot;, gender: \&quot;F\&quot;&quot;);
        HttpCache.getInstance().cache(&quot;185080103&quot;,  &quot;no: \&quot;185080103\&quot;, name: \&quot;wangwu\&quot;, gender: \&quot;M\&quot;&quot;);
    }
}
```

------------------------------------------------------------------------

**第三题**

*DatagramServer*

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;
import java.util.function.Consumer;

/**
 * 3.以下是客户端通过数据报同服务器单向通信的代码框架。客户端从控制台上接收⽤户输⼊，按回
 * ⻋键后将所输⼊的内容发送给服务器，当⽤户输⼊EOF时，客户端退出；服务器端在本地环回接⼝
 * 上接收来⾃客户端的数据报并将其内容显示到终端窗⼝的，请根据其中的注释完成编号为(1)、 (2)等
 * 的代码。
 */

public class DatagramServer {
    // Port number
    private static final int SERVER_PORT = 8888;
    // Receiving buffer size, maximum 1KB. For UDP, this is enough
    private static final int BUF_SIZE = 1024;
    public static void main(String args[]) throws Exception {
// Used to log the received packet to console
        Consumer&lt;String&gt; logger = /*(1)*/System.out::println;
// Create a datagram socket
        DatagramSocket server = /*(2)*/new DatagramSocket(SERVER_PORT, InetAddress.getLoopbackAddress());
// Receiving buffer for datagram packet
        byte[] buf = new byte[BUF_SIZE];
// Keep running forever
        for(;;) {
// Create a new packet
            DatagramPacket packet = /*(3)*/new DatagramPacket(buf, BUF_SIZE);
// Try to receive the datagram packet
            /*(4)*/server.receive(packet);
// Use logger to write the content of the received packet to the console
            /*(5)*/logger.accept(new String(packet.getData(), 0, packet.getLength()));
        }
    }
}
```

```java
import java.net.DatagramPacket;
import java.net.DatagramSocket;
import java.net.InetAddress;

public class DatagramClient {
    // Server port number
    private static final int SERVER_PORT = 8888;
    // Client port
    private static final int CLIENT_PORT = 8889;
    // Size of the message buffer
    private static final int BUF_SIZE = 1024;
    public static void main(String args[]) throws Exception {
// Create a datagram socket
        DatagramSocket client = /*(6)*/new DatagramSocket(CLIENT_PORT, InetAddress.getLoopbackAddress());
// Data buffer for datagram packet
        byte[] buf = new byte[BUF_SIZE];
// Read user input from the console. When hit EOF, exist
// pos - the next position where to write the character.
        for(int pos = 0, c; /*(7)*/(c = System.in.read()) != -1;) {
            switch(c) {
                case &#39;\r&#39;: // Windows
                    break;
                case &#39;\n&#39;: // Finished a line
// Send the message contained in the buffer
                    /*(8)*/client.send(new DatagramPacket(buf, pos, InetAddress.getLoopbackAddress(), SERVER_PORT));
// Reset the buffer
                    /*(9)*/pos = 0;
                    break;
                default: // Cache the current character
// assume no more than BUF_SIZE
                    buf[pos++] = (byte)c;
            }
        }
        System.out.println(&quot;Goodbye!&quot;);
        // Close the datagram socket
        /*(10)*/client.close();
    }
}
```
------------------------------------------------------------------------

**第四题**

*MailPanel*

```java
import javax.swing.*;
import java.awt.*;

/**
 * 根据以下所给定的简单电⼦邮件客户端的截图，以及以下代码⻣架中的注释，完成编号为(1)、 (2)
 * 等部分的代码。当点击按钮“退出”时，终⽌该客户端的运⾏，要求采⽤Lambda表达式。
 */

public class MailPanel extends JPanel {
    public MailPanel() {
// create main panel and set grid bag layout
        super( /*(1)*/new GridBagLayout() );
// create grid bag contraints
        GridBagConstraints c = /*(2)*/new GridBagConstraints();
// row 0, column 0 -&gt; 主题:
//        (3)
        c.gridx = 0;
        c.gridy = 0;
        this.add(new JLabel(&quot;主题：&quot;), c);
// row 0, column 1 -&gt; text field for subject, expand horizontally
//        (4)
        c.gridx = 1;
        c.fill = GridBagConstraints.HORIZONTAL;
        c.weightx = 1;
        this.add(new JTextField(), c);
// row 0, column 2 -&gt; 收件⼈：
//        (5)
        c.gridx = 2;
        c.fill = GridBagConstraints.NONE;
        c.weightx = 0;
        this.add(new JLabel(&quot;收件人：&quot;), c);
// row 0, column 3 -&gt; text field for recipient, expand horizontally
//        (6)
        c.gridx = 3;
        c.fill = GridBagConstraints.HORIZONTAL;
        c.weightx = 1;
        this.add(new JTextField(), c);

// row 1, colulmn 0 -&gt; email content
//        (7)
        c.gridx = 0;
        c.gridy = 1;
        c.gridwidth = 4;
        c.weightx = 0.0;
        c.weighty = 1.0;
        c.fill = GridBagConstraints.BOTH;
        this.add(new JTextArea(), c);
// row 2, column 1 -&gt; 发送
//        (8)
        c.gridx = 1;
        c.gridy = 3;
        c.weighty = 0;
        c.fill = GridBagConstraints.NONE;
        c.anchor = GridBagConstraints.CENTER;
        c.gridwidth = 1;
        c.ipady = 0;
        this.add(new JButton(&quot;发送&quot;), c);
// row 2, column 3 -&gt; 退出; exit with success code
//        (9)
        c.gridx = 3;
        c.gridy = 3;
        c.gridwidth = 1;
        JButton bttn = new JButton(&quot;退出&quot;);
        bttn.addActionListener(e -&gt; System.exit(0));
        this.add(bttn, c);
    }
}
```

*MailClient*

```java
import javax.swing.*;

public class MailClient {
    public MailClient() {
// create the main frame
        JFrame frame = /*(10)*/new JFrame(&quot;NJFU Emailer&quot;);
// terminate the JVM when the close icon was clicked
//        (11)
        frame.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);
// create and add the mail panel
//        (12)
        frame.add(new MailPanel());
// set the initial size of the frame, width: 600, height: 500
//        (13)
        frame.setSize(600, 500);
// display the frame
//        (14)
        frame.setVisible(true);
    }

    public static void main(String[] args) {
// create the frame on the event dispatching thread instead of the
// main thread. You should use Lambada
//        (15)
        SwingUtilities.invokeLater(MailClient::new);
    }
}
```