<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">本课时我们主要自己模拟一个 JVM 内存溢出的场景。在模拟 JVM 内存溢出之前我们先来看下这样的几个问题。</span><br></p>
<ul>
 <li><p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">老年代溢出为什么那么可怕？</span></p></li>
 <li><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">元空间也有溢出？怎么优化？</span></p></li>
 <li><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">如何配置栈大小？避免栈溢出？</span></p></li>
 <li><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">进程突然死掉，没有留下任何信息时如何进行排查？</span></p></li>
</ul>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">年轻代由于有老年代的担保，一般在内存占满的时候，并没什么问题。但老年代满了就比较严重了，它没有其他的空间用来做担保，只能 OOM 了，也就是发生 Out Of Memery Error。JVM 会在这种情况下直接停止工作，是非常严重的后果。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">OOM 一般是内存泄漏引起的，表现在 GC 日志里，一般情况下就是 GC 的时间变长了，而且每次回收的效果都非常一般。GC 后，堆内存的实际占用呈上升趋势。接下来，我们将模拟三种溢出场景，同时使用我们了解的工具进行观测。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">在开始之前，请你下载并安装一个叫作 VisualVM 的工具，我们使用这个图形化的工具看一下溢出过程。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">虽然 VisualVM 工具非常好用，但一般生产环境都没有这样的条件，所以大概率使用不了。新版本 JDK 把这个工具单独抽离了出去，需要自行下载。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这里需要注意下载安装完成之后请在插件选项中勾选 Visual GC 下载，它将可视化内存布局。</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">堆溢出模拟</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">首先，我们模拟堆溢出的情况，在模拟之前我们需要准备一份测试代码。这份代码开放了一个 HTTP 接口，当你触发它之后，将每秒钟生成 1MB 的数据。由于它和 GC Roots 的强关联性，每次都不能被回收。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">程序通过 JMX，将在每一秒创建数据之后，输出一些内存区域的占用情况。然后通过访问 </span><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(65, 131, 196); font-size: 12pt;">http://localhost:8888 </span><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">触发后，它将一直运行，直到堆溢出。</span></span></p>
<pre>import&nbsp;com.sun.net.httpserver.HttpContext;
import&nbsp;com.sun.net.httpserver.HttpExchange;
import&nbsp;com.sun.net.httpserver.HttpServer;
import&nbsp;java.io.OutputStream;
import&nbsp;java.lang.management.ManagementFactory;
import&nbsp;java.lang.management.MemoryPoolMXBean;
import&nbsp;java.net.InetSocketAddress;
import&nbsp;java.util.ArrayList;
import&nbsp;java.util.List;
public&nbsp;class&nbsp;OOMTest&nbsp;{
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;final&nbsp;int&nbsp;_1MB&nbsp;=&nbsp;1024&nbsp;*&nbsp;1024;
&nbsp;&nbsp;&nbsp;static&nbsp;List&lt;byte[]&gt;&nbsp;byteList&nbsp;=&nbsp;new&nbsp;ArrayList&lt;&gt;();
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;oom(HttpExchange&nbsp;exchange)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String&nbsp;response&nbsp;=&nbsp;"oom&nbsp;begin!";
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exchange.sendResponseHeaders(200,&nbsp;response.getBytes().length);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OutputStream&nbsp;os&nbsp;=&nbsp;exchange.getResponseBody();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.write(response.getBytes());
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.close();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;ex)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for&nbsp;(int&nbsp;i&nbsp;=&nbsp;0;&nbsp;;&nbsp;i++)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;byte[]&nbsp;bytes&nbsp;=&nbsp;new&nbsp;byte[_1MB];
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;byteList.add(bytes);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(i&nbsp;+&nbsp;"MB");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;memPrint();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Thread.sleep(1000);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;e)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;static&nbsp;void&nbsp;memPrint()&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for&nbsp;(MemoryPoolMXBean&nbsp;memoryPoolMXBean&nbsp;:&nbsp;ManagementFactory.getMemoryPoolMXBeans())&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(memoryPoolMXBean.getName()&nbsp;+
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"&nbsp;&nbsp;committed:"&nbsp;+&nbsp;memoryPoolMXBean.getUsage().getCommitted()&nbsp;+
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"&nbsp;&nbsp;used:"&nbsp;+&nbsp;memoryPoolMXBean.getUsage().getUsed());
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;srv()&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpServer&nbsp;server&nbsp;=&nbsp;HttpServer.create(new&nbsp;InetSocketAddress(8888),&nbsp;0);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpContext&nbsp;context&nbsp;=&nbsp;server.createContext("/");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;context.setHandler(OOMTest::oom);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;server.start();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;void&nbsp;main(String[]&nbsp;args)&nbsp;throws&nbsp;Exception{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;srv();
&nbsp;&nbsp;&nbsp;}
}</pre>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">我们使用 CMS 收集器进行垃圾回收，可以看到如下的信息。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px; color: rgb(63, 63, 63);">命令：</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">java -Xmx20m&nbsp; -Xmn4m&nbsp; &nbsp;-XX:+UseConcMarkSweepGC&nbsp; -verbose:gc -Xlog:gc,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">gc+ref=debug,gc+heap=debug,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">gc+age=trace:file=/tmp/logs/gc_%p.log:tags,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">uptime,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">time,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">level -Xlog:safepoint:file=/tmp/logs/safepoint_%p.log:tags,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">uptime,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">time,</span></p>
<p style="text-align: justify; line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">level -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/logs -XX:ErrorFile=/tmp/logs/hs_error_pid%p.log -XX:-OmitStackTraceInFastThrow&nbsp; OOMTest</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">输出：</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[0.025s][info][gc] Using Concurrent Mark Sweep</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">0MB</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'non-nmethods' &nbsp;committed:2555904 &nbsp;used:1120512</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Metaspace &nbsp;committed:4980736 &nbsp;used:854432</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'profiled nmethods' &nbsp;committed:2555904 &nbsp;used:265728</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Compressed Class Space &nbsp;committed:524288 &nbsp;used:96184</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Par Eden Space &nbsp;committed:3407872 &nbsp;used:2490984</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Par Survivor Space &nbsp;committed:393216 &nbsp;used:0</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'non-profiled nmethods' &nbsp;committed:2555904 &nbsp;used:78592</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CMS Old Gen &nbsp;committed:16777216 &nbsp;used:0</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">...省略</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.377s][info][gc] GC(9) Concurrent Mark 1.592ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.377s][info][gc] GC(9) Concurrent Preclean</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Preclean 0.721ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Abortable Preclean</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Abortable Preclean 0.006ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Pause Remark 17M-&gt;17M(19M) 0.344ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Sweep</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Sweep 0.248ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Reset</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[16.378s][info][gc] GC(9) Concurrent Reset 0.013ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">17MB</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'non-nmethods' &nbsp;committed:2555904 &nbsp;used:1120512</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Metaspace &nbsp;committed:4980736 &nbsp;used:883760</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'profiled nmethods' &nbsp;committed:2555904 &nbsp;used:422016</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Compressed Class Space &nbsp;committed:524288 &nbsp;used:92432</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Par Eden Space &nbsp;committed:3407872 &nbsp;used:3213392</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Par Survivor Space &nbsp;committed:393216 &nbsp;used:0</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CodeHeap 'non-profiled nmethods' &nbsp;committed:2555904 &nbsp;used:88064</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">CMS Old Gen &nbsp;committed:16777216 &nbsp;used:16452312</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.380s][info][gc] GC(10) Pause Initial Mark 18M-&gt;18M(19M) 0.187ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.380s][info][gc] GC(10) Concurrent Mark</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.384s][info][gc] GC(11) Pause Young (Allocation Failure) 18M-&gt;18M(19M) 0.186ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.386s][info][gc] GC(10) Concurrent Mark 5.435ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.395s][info][gc] GC(12) Pause Full (Allocation Failure) 18M-&gt;18M(19M) 10.572ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">[18.400s][info][gc] GC(13) Pause Full (Allocation Failure) 18M-&gt;18M(19M) 5.348ms</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Exception in thread "main" java.lang.OutOfMemoryError: Java heap space</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at OldOOM.main(OldOOM.java:20)</span></p>
<p style="margin-bottom: 0pt; margin-top: 0pt; font-size: 11pt; color: rgb(73, 73, 73); line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">最后 JVM 在一阵疯狂的 GC 日志输出后，进程停止了。在现实情况中，JVM 在停止工作之前，很多会垂死挣扎一段时间，这个时候，GC 线程会造成 CPU 飙升，但其实它已经不能工作了。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">VisualVM 的截图展示了这个溢出结果。可以看到 Eden 区刚开始还是运行平稳的，内存泄漏之后就开始疯狂回收（其实是提升），老年代内存一直增长，直到 OOM。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp; &nbsp; &nbsp;<img src="https://s0.lgstatic.com/i/image3/M01/65/E6/Cgq2xl5DytGAc09TAAFdBh0n9eo313.jpg"> &nbsp; &nbsp; &nbsp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">很多参数会影响对象的分配行为，但不是非常必要，我们一般不去调整它们。为了观察这些参数的默认值，我们通常使用 -XX:+PrintFlagsFinal 参数，输出一些设置信息。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">命令：</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"># java -XX:+PrintFlagsFinal 2&gt;&amp;1 | grep SurvivorRatio</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;uintx SurvivorRatio &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(152, 26, 26);">=</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(17, 102, 68);">8</span> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {product} {default}</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Java13 输出了几百个参数和默认值，我们通过修改一些参数来观测一些不同的行为。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><strong style="font-size: 12pt;">NewRatio</strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;"> 默认值为 2，表示年轻代是老年代的 1/2。追加参数 “-XX:NewRatio=1”，可以把年轻代和老年代的空间大小调成一样大。在实践中，我们一般使用 -Xmn 来设置一个固定值。注意，这两个参数不要用在 G1 垃圾回收器中。</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><strong style="font-size: 12pt;">SurvivorRatio</strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;"> 默认值为 8。表示伊甸区和幸存区的比例。在上面的例子中，Eden 的内存大小为：0.8*4MB。S 分区不到 1MB，根本存不下我们的 1MB 数据。</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><strong style="font-size: 12pt;">MaxTenuringThreshold</strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;"> &nbsp;这个值在 CMS 下默认为 6，G1 下默认为 15。这是因为 G1 存在动态阈值计算。这个值和我们前面提到的对象提升有关，如果你想要对象尽量长的时间存在于年轻代，则在 CMS 中，可以把它调整到 15。</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java -XX:+PrintFlagsFinal -XX:+UseConcMarkSweepGC 2&gt;&amp;1 | grep MaxTenuringThreshold</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java -XX:+PrintFlagsFinal -XX:+UseG1GC 2&gt;&amp;1 | grep MaxTenuringThreshold</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">PretenureSizeThreshold</span></strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">&nbsp;这个参数默认值是 0，意味着所有的对象年轻代优先分配。我们把这个值调小一点，再观测 JVM 的行为。追加参数 -XX:PretenureSizeThreshold=1024，可以看到 VisualVm 中老年代的区域增长。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><strong style="font-size: 12pt;">TargetSurvivorRatio</strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;"> 默认值为 50。在动态计算对象提升阈值的时候使用。计算时，会从年龄最小的对象开始累加，如果累加的对象大小大于幸存区的一半，则将当前的对象 age 作为新的阈值，年龄大于此阈值的对象直接进入老年代。工作中不建议调整这个值，如果要调，请调成比 50 大的值。</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">你可以尝试着更改其他参数，比如垃圾回收器的种类，动态看一下效果。尤其注意每一项内存区域的内容变动，你会对垃圾回收器有更好的理解。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><strong style="font-size: 12pt;">UseAdaptiveSizePolicy</strong><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;"> ，因为它和 CMS 不兼容，所以 CMS 下默认为 false，但 G1 下默认为 true。这是一个非常智能的参数，它是用来自适应调整空间大小的参数。它会在每次 GC 之后，重新计算 Eden、From、To 的大小。很多人在 Java 8 的一些配置中会见到这个参数，但其实在 CMS 和 G1 中是不需要显式设置的。</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">值的注意的是，Java 8 默认垃圾回收器是 Parallel Scavenge，它的这个参数是默认开启的，有可能会发生把幸存区自动调小的可能，造成一些问题，显式的设置 SurvivorRatio 可以解决这个问题。 </span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">下面这张截图，是切换到 G1 之后的效果。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java -Xmx20m &nbsp; -XX:+UseG1GC &nbsp;-verbose:gc -Xlog:gc,gc+ref=debug,gc+heap=debug,gc+age=trace:file=/tmp/logs/gc_%p.log:tags,uptime,time,level -Xlog:safepoint:file=/tmp/logs/safepoint_%p.log:tags,uptime,time,level -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/logs -XX:ErrorFile=/tmp/logs/hs_error_pid%p.log -XX:-OmitStackTraceInFastThrow &nbsp;OOMTest</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp; &nbsp; &nbsp;<img src="https://s0.lgstatic.com/i/image3/M01/65/E5/CgpOIF5DytKAIL4RAAE2rXZgveA253.jpg"> &nbsp; &nbsp; &nbsp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">可以通过下面这个命令调整小堆区的大小，来看一下这个过程。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">-XX:G1HeapRegionSize=&lt;N&gt;M</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">元空间溢出</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">堆一般都是指定大小的，但元空间不是。所以如果元空间发生内存溢出会更加严重，会造成操作系统的内存溢出。我们在使用的时候，也会给它设置一个上限 for safe。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">元空间溢出主要是由于加载的类太多，或者动态生成的类太多。下面是一段模拟代码。通过访问 </span><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(65, 131, 196); font-size: 12pt;">http://localhost:8888 </span><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">触发后，它将会发生元空间溢出。</span></span></p>
<pre>import&nbsp;com.sun.net.httpserver.HttpContext;
import&nbsp;com.sun.net.httpserver.HttpExchange;
import&nbsp;com.sun.net.httpserver.HttpServer;
import&nbsp;java.io.OutputStream;
import&nbsp;java.lang.reflect.InvocationHandler;
import&nbsp;java.lang.reflect.Method;
import&nbsp;java.lang.reflect.Proxy;
import&nbsp;java.net.InetSocketAddress;
import&nbsp;java.net.URL;
import&nbsp;java.net.URLClassLoader;
import&nbsp;java.util.HashMap;
import&nbsp;java.util.Map;
public&nbsp;class&nbsp;MetaspaceOOMTest&nbsp;{
&nbsp;&nbsp;&nbsp;public&nbsp;interface&nbsp;Facade&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;void&nbsp;m(String&nbsp;input);
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;class&nbsp;FacadeImpl&nbsp;implements&nbsp;Facade&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@Override
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public&nbsp;void&nbsp;m(String&nbsp;name)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;class&nbsp;MetaspaceFacadeInvocationHandler&nbsp;implements&nbsp;InvocationHandler&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;private&nbsp;Object&nbsp;impl;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public&nbsp;MetaspaceFacadeInvocationHandler(Object&nbsp;impl)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;this.impl&nbsp;=&nbsp;impl;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@Override
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;public&nbsp;Object&nbsp;invoke(Object&nbsp;proxy,&nbsp;Method&nbsp;method,&nbsp;Object[]&nbsp;args)&nbsp;throws&nbsp;Throwable&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;return&nbsp;method.invoke(impl,&nbsp;args);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;Map&lt;String,&nbsp;Facade&gt;&nbsp;classLeakingMap&nbsp;=&nbsp;new&nbsp;HashMap&lt;String,&nbsp;Facade&gt;();
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;oom(HttpExchange&nbsp;exchange)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String&nbsp;response&nbsp;=&nbsp;"oom&nbsp;begin!";
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exchange.sendResponseHeaders(200,&nbsp;response.getBytes().length);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OutputStream&nbsp;os&nbsp;=&nbsp;exchange.getResponseBody();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.write(response.getBytes());
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.close();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;ex)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for&nbsp;(int&nbsp;i&nbsp;=&nbsp;0;&nbsp;;&nbsp;i++)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String&nbsp;jar&nbsp;=&nbsp;"file:"&nbsp;+&nbsp;i&nbsp;+&nbsp;".jar";
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;URL[]&nbsp;urls&nbsp;=&nbsp;new&nbsp;URL[]{new&nbsp;URL(jar)};
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;URLClassLoader&nbsp;newClassLoader&nbsp;=&nbsp;new&nbsp;URLClassLoader(urls);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Facade&nbsp;t&nbsp;=&nbsp;(Facade)&nbsp;Proxy.newProxyInstance(newClassLoader,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;new&nbsp;Class&lt;?&gt;[]{Facade.class},
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;new&nbsp;MetaspaceFacadeInvocationHandler(new&nbsp;FacadeImpl()));
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;classLeakingMap.put(jar,&nbsp;t);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;e)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;srv()&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpServer&nbsp;server&nbsp;=&nbsp;HttpServer.create(new&nbsp;InetSocketAddress(8888),&nbsp;0);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpContext&nbsp;context&nbsp;=&nbsp;server.createContext("/");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;context.setHandler(MetaspaceOOMTest::oom);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;server.start();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;void&nbsp;main(String[]&nbsp;args)&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;srv();
&nbsp;&nbsp;&nbsp;}
}</pre>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这段代码将使用 Java 自带的动态代理类，不断的生成新的 class。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java -Xmx20m&nbsp; -Xmn4m&nbsp; &nbsp;-XX:+UseG1GC&nbsp; -verbose:gc -Xlog:gc,gc+ref=debug,gc+heap=debug,gc+age=trace:file=/tmp/logs/gc_%p.log:tags,uptime,time,level -Xlog:safepoint:file=/tmp/logs/safepoint_%p.log:tags,uptime,time,level -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/logs -XX:ErrorFile=/tmp/logs/hs_error_pid%p.log -XX:-OmitStackTraceInFastThrow -XX:MetaspaceSize=16M -XX:MaxMetaspaceSize=16M&nbsp; MetaspaceOOMTest</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">我们在启动的时候，限制 Metaspace 空间大小为 16MB。可以看到运行一小会之后，Metaspace 会发生内存溢出。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"></span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">[6.509s][info][gc] GC(28) Pause Young (Concurrent Start) (Metadata GC Threshold) 9M-&gt;9M(20M) 1.186ms</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">[6.509s][info][gc] GC(30) Concurrent Cycle</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">[6.534s][info][gc] GC(29) Pause Full (Metadata GC Threshold) 9M-&gt;9M(20M) 25.165ms</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">[6.556s][info][gc] GC(31) Pause Full (Metadata GC Clear Soft References) 9M-&gt;9M(20M) 21.136ms</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">[6.556s][info][gc] GC(30) Concurrent Cycle 46.668ms</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">java.lang.OutOfMemoryError: Metaspace</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">Dumping heap to /tmp/logs/java_pid36723.hprof ...</span></p>
<p style="line-height: 1.75em;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;;">Heap dump file created [17362313 bytes in 0.134 secs]</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp; &nbsp; &nbsp;<img src="https://s0.lgstatic.com/i/image3/M01/65/E6/Cgq2xl5DytKAMGfeAAE5q9rh1rM558.jpg"> &nbsp; &nbsp; &nbsp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">但假如你把堆 Metaspace 的限制给去掉，会更可怕。它占用的内存会一直增长。</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">堆外内存溢出</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">严格来说，上面的 Metaspace 也是属于堆外内存的。但是我们这里的堆外内存指的是 Java 应用程序通过直接方式从操作系统中申请的内存。所以严格来说，这里是指直接内存。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> </span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">程序将通过 ByteBuffer 的 allocateDirect 方法每 1 秒钟申请 1MB 的直接内存。不要忘了通过链接触发这个过程。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">但是，使用 VisualVM 看不到这个过程，使用 JMX 的 API 同样也看不到。关于这部分内容，我们将在堆外内存排查课时进行详细介绍。</span></p>
<pre>import&nbsp;com.sun.net.httpserver.HttpContext;
import&nbsp;com.sun.net.httpserver.HttpExchange;
import&nbsp;com.sun.net.httpserver.HttpServer;
import&nbsp;java.io.OutputStream;
import&nbsp;java.lang.management.ManagementFactory;
import&nbsp;java.lang.management.MemoryPoolMXBean;
import&nbsp;java.net.InetSocketAddress;
import&nbsp;java.nio.ByteBuffer;
import&nbsp;java.util.ArrayList;
import&nbsp;java.util.List;
public&nbsp;class&nbsp;OffHeapOOMTest&nbsp;{
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;final&nbsp;int&nbsp;_1MB&nbsp;=&nbsp;1024&nbsp;*&nbsp;1024;
&nbsp;&nbsp;&nbsp;static&nbsp;List&lt;ByteBuffer&gt;&nbsp;byteList&nbsp;=&nbsp;new&nbsp;ArrayList&lt;&gt;();
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;oom(HttpExchange&nbsp;exchange)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String&nbsp;response&nbsp;=&nbsp;"oom&nbsp;begin!";
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;exchange.sendResponseHeaders(200,&nbsp;response.getBytes().length);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OutputStream&nbsp;os&nbsp;=&nbsp;exchange.getResponseBody();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.write(response.getBytes());
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;os.close();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;ex)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for&nbsp;(int&nbsp;i&nbsp;=&nbsp;0;&nbsp;;&nbsp;i++)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ByteBuffer&nbsp;buffer&nbsp;=&nbsp;ByteBuffer.allocateDirect(_1MB);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;byteList.add(buffer);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(i&nbsp;+&nbsp;"MB");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;memPrint();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;try&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Thread.sleep(1000);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}&nbsp;catch&nbsp;(Exception&nbsp;e)&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;private&nbsp;static&nbsp;void&nbsp;srv()&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpServer&nbsp;server&nbsp;=&nbsp;HttpServer.create(new&nbsp;InetSocketAddress(8888),&nbsp;0);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HttpContext&nbsp;context&nbsp;=&nbsp;server.createContext("/");
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;context.setHandler(OffHeapOOMTest::oom);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;server.start();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;void&nbsp;main(String[]&nbsp;args)&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;srv();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;static&nbsp;void&nbsp;memPrint()&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for&nbsp;(MemoryPoolMXBean&nbsp;memoryPoolMXBean&nbsp;:&nbsp;ManagementFactory.getMemoryPoolMXBeans())&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(memoryPoolMXBean.getName()&nbsp;+
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"&nbsp;&nbsp;committed:"&nbsp;+&nbsp;memoryPoolMXBean.getUsage().getCommitted()&nbsp;+
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"&nbsp;&nbsp;used:"&nbsp;+&nbsp;memoryPoolMXBean.getUsage().getUsed());
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;}
}</pre>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">通过 top 或者操作系统的监控工具，能够看到内存占用的明显增长。为了限制这些危险的内存申请，如果你确定在自己的程序中用到了大量的 JNI 和 JNA 操作，要显式的设置 MaxDirectMemorySize 参数。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">以下是程序运行一段时间抛出的错误。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">Exception in thread "Thread-2" java.lang.OutOfMemoryError: Direct buffer memory</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at java.nio.Bits.reserveMemory(Bits.java:694)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at java.nio.DirectByteBuffer.&lt;init&gt;(DirectByteBuffer.java:123)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at OffHeapOOMTest.oom(OffHeapOOMTest.java:27)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.AuthFilter.doFilter(AuthFilter.java:83)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:82)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.ServerImpl$Exchange$LinkHandler.handle(ServerImpl.java:675)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at com.sun.net.httpserver.Filter$Chain.doFilter(Filter.java:79)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.ServerImpl$Exchange.run(ServerImpl.java:647)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.ServerImpl$DefaultExecutor.execute(ServerImpl.java:158)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.ServerImpl$Dispatcher.handle(ServerImpl.java:431)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at sun.net.httpserver.ServerImpl$Dispatcher.run(ServerImpl.java:396)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;at java.lang.Thread.run(Thread.java:748)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">启动命令。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java -XX:MaxDirectMemorySize=10M -Xmx10M OffHeapOOMTest</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">栈溢出</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">还记得我们的虚拟机栈么？栈溢出指的就是这里的数据太多造成的泄漏。通过 -Xss 参数可以设置它的大小。比如下面的命令就是设置栈大小为 128K。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">-Xss128K</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">从这里我们也能了解到，由于每个线程都有一个虚拟机栈。线程的开销也是要占用内存的。如果系统中的线程数量过多，那么占用内存的大小也是非常可观的。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">栈溢出不会造成 JVM 进程死亡，危害“相对较小”。下面是一个简单的模拟栈溢出的代码，只需要递归调用就可以了。</span></p>
<pre>public&nbsp;class&nbsp;StackOverflowTest&nbsp;{
&nbsp;&nbsp;&nbsp;static&nbsp;int&nbsp;count&nbsp;=&nbsp;0;
&nbsp;&nbsp;&nbsp;static&nbsp;void&nbsp;a()&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(count);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;count++;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;static&nbsp;void&nbsp;b()&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(count);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;count++;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a();
&nbsp;&nbsp;&nbsp;}
&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;void&nbsp;main(String[]&nbsp;args)&nbsp;throws&nbsp;Exception&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a();
&nbsp;&nbsp;&nbsp;}
}</pre>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">运行后，程序直接报错。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">Exception</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">in</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">thread</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(170, 17, 17);">"main"</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">lang</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">StackOverflowError</span></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">at</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">io</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">write</span>(<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>:<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(17, 102, 68);">526</span>)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">at</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">io</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">print</span>(<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>:<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(17, 102, 68);">597</span>)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">at</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">io</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">println</span>(<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">PrintStream</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>:<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(17, 102, 68);">736</span>)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp;<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">at</span> <span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">StackOverflowTest</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">a</span>(<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">StackOverflowTest</span>.<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(0, 0, 0);">java</span>:<span style="font-size: 16px; font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(17, 102, 68);">5</span>)</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">如果你的应用经常发生这种情况，可以试着调大这个值。但一般都是因为程序错误引起的，最好检查一下自己的代码。</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">进程异常退出</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">上面这几种溢出场景，都有明确的原因和报错，排查起来也是非常容易的。但是还有一类应用，死亡的时候，静悄悄的，什么都没留下。 </span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">以下问题已经不止一个同学问了：<strong>我的 Java 进程没了，什么都没留下，直接蒸发不见了</strong></span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">why？是因为对象太多了么？</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这是趣味性和技巧性非常突出的一个问题。让我们执行 dmesg 命令，大概率会看到你的进程崩溃信息躺在那里。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"> &nbsp; &nbsp; &nbsp; &nbsp;<img src="https://s0.lgstatic.com/i/image3/M01/65/E5/CgpOIF5DytKAL-kVAAEITZsHixY196.jpg"> &nbsp; &nbsp; &nbsp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;"><span style="font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; color: rgb(63, 63, 63); font-size: 12pt;">为了能看到发生的时间，我们习惯性加上参数 T</span>（dmesg -T）。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这个现象，其实和 Linux 的内存管理有关。由于 Linux 系统采用的是虚拟内存分配方式，JVM 的代码、库、堆和栈的使用都会消耗内存，但是申请出来的内存，只要没真正 access过，是不算的，因为没有真正为之分配物理页面。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">随着使用内存越用越多。第一层防护墙就是 SWAP；当 SWAP 也用的差不多了，会尝试释放 cache；当这两者资源都耗尽，杀手就出现了。oom-killer 会在系统内存耗尽的情况下跳出来，选择性的干掉一些进程以求释放一点内存。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">所以这时候我们的 Java 进程，是操作系统“主动”终结的，JVM 连发表遗言的机会都没有。这个信息，只能在操作系统日志里查找。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">要解决这种问题，首先不能太贪婪。比如一共 8GB 的机器，你把整整 7.5GB 都分配给了 JVM。当操作系统内存不足时，你的 JVM 就可能成为 oom-killer 的猎物。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">相对于被动终结，还有一种主动求死的方式。有些同学，会在程序里面做一些判断，直接调用 System.exit() 函数。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这个函数危险得很，它将强制终止我们的应用，而且什么都不会留下。你应该扫描你的代码，确保这样的逻辑不会存在。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">再聊一种最初级最常见还经常发生的，会造成应用程序意外死亡的情况，那就是对 Java 程序错误的启动方式。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">很多同学对 Linux 不是很熟悉，使用 XShell 登陆之后，调用下面的命令进行启动。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">java com.cn.AA &amp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">这样调用还算有点意识，在最后使用了“&amp;”号，以期望进程在后台运行。但可惜的是，很多情况下，随着 XShell Tab 页的关闭，或者等待超时，后面的 Java 进程就随着一块停止了，很让人困惑。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">正确的启动方式，就是使用 nohup 关键字，或者阻塞在其他更加长命的进程里（比如docker）。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">nohup java com.cn.AA &amp;</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">进程这种静悄悄的死亡方式，通常会给我们的问题排查带来更多的困难。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">在发生问题时，要确保留下了足够的证据，来支持接下来的分析。不能喊一句“出事啦”，然后就陷入无从下手的尴尬境地。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">通常，我们在关闭服务的时候，会使用“kill -15”，而不是“kill -9”，以便让服务在临死之前喘口气。信号9和15的区别，是面试经常问的一个问题，也是一种非常有效的手段。</span></p>
<h2><p><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">小结</span></p></h2>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">本课时我们简单模拟了堆、元空间、栈的溢出。并使用 VisualVM 观察了这个过程。</span></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><br></p>
<p style="line-height: 1.7;margin-bottom: 0pt;margin-top: 0pt;font-size: 11pt;color: #494949;"><span style="color: rgb(63, 63, 63); font-family: 微软雅黑, &quot;Microsoft YaHei&quot;; font-size: 16px;">接下来，我们了解到进程静悄悄消失的三种情况。如果你的应用也这样消失过，试着这样找找它。这三种情况也是一个故障排查流程中要考虑的环节，属于非常重要的边缘检查点。相信聪明的你，会将这些情况揉进自己的面试体系去，真正成为自己的实战经验。</span></p>