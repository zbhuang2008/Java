<p>在上一课时我们掌握了 JVM 的内存区域划分，以及 .class 文件的加载机制。也了解到很多初始化动作是在不同的阶段发生的。</p>
<p>但你可能仍对以下这些问题有疑问：</p>
<ul>
<li>怎么查看字节码文件？</li>
<li>字节码文件长什么样子？</li>
<li>对象初始化之后，具体的字节码又是怎么执行的？</li>
</ul>
<p>带着这些疑问，我们进入本课时的学习，本课时将带你动手实践，详细分析一个 Java 文件产生的字节码，并从栈帧层面看一下字节码的具体执行过程。</p>
<h3>工具介绍</h3>
<p>工欲善其事，必先利其器。在开始本课时的内容之前，先给你介绍两个分析字节码的小工具。</p>
<h4>javap</h4>
<p>第一个小工具是 javap，javap 是 JDK 自带的反解析工具。它的作用是将 .class 字节码文件解析成可读的文件格式。我们在第一课时，就是用的它输出了 HelloWorld 的内容。</p>
<p>在使用 javap 时我一般会添加 -v 参数，尽量多打印一些信息。同时，我也会使用 -p 参数，打印一些私有的字段和方法。使用起来大概是这样：</p>
<pre><code data-language="java" class="lang-java">javap&nbsp;-p&nbsp;-v&nbsp;HelloWorld
</code></pre>
<p>在 Stack Overflow 上有一个非常有意思的问题：我在某个类中增加一行注释之后，为什么两次生成的 .class 文件，它们的 MD5 是不一样的？</p>
<p>这是因为在 javac 中可以指定一些额外的内容输出到字节码。经常用的有</p>
<ul>
<li><strong>javac -g:lines</strong>&nbsp;强制生成 LineNumberTable。</li>
<li><strong>javac -g:vars</strong>&nbsp; 强制生成 LocalVariableTable。</li>
<li><strong>javac -g</strong>&nbsp;生成所有的 debug 信息。</li>
</ul>
<p>为了观察字节码的流转，我们本课时就会使用到这些参数。</p>
<h4>jclasslib</h4>
<p>如果你不太习惯使用命令行的操作，还可以使用 jclasslib，jclasslib 是一个图形化的工具，能够更加直观的查看字节码中的内容。它还分门别类的对类中的各个部分进行了整理，非常的人性化。同时，它还提供了 Idea 的插件，你可以从 plugins 中搜索到它。</p>
<p>如果你在其中看不到一些诸如 LocalVariableTable 的信息，记得在编译代码的时候加上我们上面提到的这些参数。</p>
<p>jclasslib 的下载地址：<a href="https://github.com/ingokegel/jclasslib">https://github.com/ingokegel/jclasslib</a></p>
<h3>类加载和对象创建的时机</h3>
<p>接下来，我们来看一个稍微复杂的例子，来具体看一下类加载和对象创建的过程。</p>
<p>首先，我们写一个最简单的 Java 程序 A.java。它有一个公共方法 test，还有一个静态成员变量和动态成员变量。</p>
<pre><code data-language="java" class="lang-java"><span class="hljs-class"><span class="hljs-keyword">class</span>&nbsp;<span class="hljs-title">B</span>&nbsp;</span>{
&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">private</span>&nbsp;<span class="hljs-keyword">int</span>&nbsp;a&nbsp;=&nbsp;<span class="hljs-number">1234</span>;

&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">static</span>&nbsp;<span class="hljs-keyword">long</span>&nbsp;C&nbsp;=&nbsp;<span class="hljs-number">1111</span>;

&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-function"><span class="hljs-keyword">public</span>&nbsp;<span class="hljs-keyword">long</span>&nbsp;<span class="hljs-title">test</span><span class="hljs-params">(<span class="hljs-keyword">long</span>&nbsp;num)</span>&nbsp;</span>{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">long</span>&nbsp;ret&nbsp;=&nbsp;<span class="hljs-keyword">this</span>.a&nbsp;+&nbsp;num&nbsp;+&nbsp;C;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">return</span>&nbsp;ret;
&nbsp;&nbsp;&nbsp;&nbsp;}
}

<span class="hljs-keyword">public</span>&nbsp;<span class="hljs-class"><span class="hljs-keyword">class</span>&nbsp;<span class="hljs-title">A</span>&nbsp;</span>{
&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">private</span>&nbsp;B&nbsp;b&nbsp;=&nbsp;<span class="hljs-keyword">new</span>&nbsp;B();

&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-function"><span class="hljs-keyword">public</span>&nbsp;<span class="hljs-keyword">static</span>&nbsp;<span class="hljs-keyword">void</span>&nbsp;<span class="hljs-title">main</span><span class="hljs-params">(String[]&nbsp;args)</span>&nbsp;</span>{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;A&nbsp;a&nbsp;=&nbsp;<span class="hljs-keyword">new</span>&nbsp;A();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">long</span>&nbsp;num&nbsp;=&nbsp;<span class="hljs-number">4321</span>&nbsp;;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">long</span>&nbsp;ret&nbsp;=&nbsp;a.b.test(num);

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;System.out.println(ret);
&nbsp;&nbsp;&nbsp;&nbsp;}
}
</code></pre>
<p>前面我们提到，类的初始化发生在类加载阶段，那对象都有哪些创建方式呢？除了我们常用的 new，还有下面这些方式：</p>
<ul>
<li>使用 Class 的 newInstance 方法。</li>
<li>使用 Constructor 类的 newInstance 方法。</li>
<li>反序列化。</li>
<li>使用 Object 的 clone 方法。</li>
</ul>
<p>其中，后面两种方式没有调用到构造函数。</p>
<p>当虚拟机遇到一条 new 指令时，首先会检查这个指令的参数能否在常量池中定位一个符号引用。然后检查这个符号引用的类字节码是否加载、解析和初始化。如果没有，将执行对应的类加载过程。</p>
<p>拿我们上面的代码来说，执行 A 代码，在调用 private B b = new B() 时，就会触发 B 类的加载。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/61/AD/CgpOIF4ezuOAK_6bAACFY5oeX-Y174.jpg" alt=""></p>
<p>让我们结合上图回顾一下前面章节的内容。A 和 B 会被加载到元空间的方法区，进入 main 方法后，将会交给执行引擎执行。这个执行过程是在栈上完成的，其中有几个重要的区域，包括虚拟机栈、程序计数器等。接下来我们详细看一下虚拟机栈上的执行过程。</p>
<h3>查看字节码</h3>
<h4>命令行查看字节码</h4>
<p>使用下面的命令编译源代码 A.java。如果你用的是 Idea，可以直接将参数追加在 VM options 里面。</p>
<pre><code data-language="java" class="lang-java">javac&nbsp;-g:lines&nbsp;-g:vars&nbsp;A.java
</code></pre>
<p>这将强制生成 LineNumberTable 和 LocalVariableTable。</p>
<p>然后使用 javap 命令查看 A 和 B 的字节码。</p>
<pre><code data-language="java" class="lang-java">javap&nbsp;-p&nbsp;-v&nbsp;A
javap&nbsp;-p&nbsp;-v&nbsp;B
</code></pre>
<p>这个命令，不仅会输出行号、本地变量表信息、反编译汇编代码，还会输出当前类用到的常量池等信息。由于内容很长，这里就不具体展示了，你可以使用上面的命令实际操作一下就可以了。</p>
<p>注意 javap 中的如下字样。</p>
<p><strong>&lt;1&gt;</strong></p>
<pre><code data-language="js" class="lang-js">1:&nbsp;invokespecial&nbsp;#1&nbsp;&nbsp;&nbsp;//&nbsp;Method&nbsp;java/lang/Object."&lt;init&gt;":()V
</code></pre>
<p>可以看到对象的初始化，首先是调用了 Object 类的初始化方法。注意这里是 &lt;init&gt; 而不是 &lt;cinit&gt;。</p>
<p><strong>&lt;2&gt;</strong></p>
<pre><code data-language="java" class="lang-java">#2&nbsp;=&nbsp;Fieldref&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#6.#27&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//&nbsp;B.a:I
</code></pre>
<p>它其实直接拼接了 #13 和 #14 的内容。</p>
<pre><code data-language="java" class="lang-java">#6&nbsp;=&nbsp;Class&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#29&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//&nbsp;B
#27&nbsp;=&nbsp;NameAndType&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#8:#9&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//&nbsp;a:I
...
#8&nbsp;=&nbsp;Utf8&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a
#9&nbsp;=&nbsp;Utf8&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;I
</code></pre>
<p><strong>&lt;3&gt;</strong></p>
<p>你会注意到 :I 这样特殊的字符。它们也是有意义的，如果你经常使用 jmap 这种命令，应该不会陌生。大体包括：</p>
<ul>
<li>B 基本类型 byte</li>
<li>C 基本类型 char</li>
<li>D 基本类型 double</li>
<li>F 基本类型 float</li>
<li>I 基本类型 int</li>
<li>J 基本类型 long</li>
<li>S 基本类型 short</li>
<li>Z 基本类型 boolean</li>
<li>V 特殊类型 void</li>
<li>L 对象类型，以分号结尾，如 Ljava/lang/Object;</li>
<li>[Ljava/lang/String;&nbsp;数组类型，每一位使用一个前置的"["字符来描述</li>
</ul>
<p>我们注意到 code 区域，有非常多的二进制指令。如果你接触过汇编语言，会发现它们之间其实有一定的相似性。但这些二进制指令，并不是操作系统能够认识的，它们是提供给 JVM 运行的源材料。</p>
<h4>可视化查看字节码</h4>
<p>接下来，我们就可以使用更加直观的工具 jclasslib，来查看字节码中的具体内容了。</p>
<p>我们以 B.class 文件为例，来查看它的内容。</p>
<p><strong>&lt;1&gt;</strong></p>
<p>首先，我们能够看到 Constant Pool（常量池），这些内容，就存放于我们的 Metaspace 区域，属于非堆。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/61/AD/Cgq2xl4ezeKAWB30AADZXqT3TjQ870.jpg" alt=""></p>
<p>常量池包含 .class 文件常量池、运行时常量池、String 常量池等部分，大多是一些静态内容。</p>
<p><strong>&lt;2&gt;</strong></p>
<p>接下来，可以看到两个默认的 &lt;init&gt; 和 &lt;cinit&gt; 方法。以下截图是 test 方法的 code 区域，比命令行版的更加直观。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/61/AD/CgpOIF4ezeKAVmSnAACExsXdgtg544.jpg" alt=""></p>
<p><strong>&lt;3&gt;</strong></p>
<p>继续往下看，我们看到了 LocalVariableTable 的三个变量。其中，slot 0 指向的是 this 关键字。该属性的作用是描述帧栈中局部变量与源码中定义的变量之间的关系。如果没有这些信息，那么在 IDE 中引用这个方法时，将无法获取到方法名，取而代之的则是 arg0 这样的变量名。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/61/AD/Cgq2xl4ezeKASWJHAAB5Ptt1JsM137.jpg" alt=""></p>
<p>本地变量表的 slot 是可以复用的。注意一个有意思的地方，index 的最大值为 3，证明了本地变量表同时最多能够存放 4 个变量。</p>
<p>另外，我们观察到还有 LineNumberTable 等选项。该属性的作用是描述源码行号与字节码行号（字节码偏移量）之间的对应关系，有了这些信息，在 debug 时，就能够获取到发生异常的源代码行号。</p>
<h3>test 函数执行过程</h3>
<h4>Code 区域介绍</h4>
<p>test 函数同时使用了成员变量 a、静态变量 C，以及输入参数 num。我们此时说的函数执行，内存其实就是在虚拟机栈上分配的。下面这些内容，就是 test 方法的字节码。</p>
<pre><code data-language="c" class="lang-c"><span class="hljs-function"><span class="hljs-keyword">public</span>&nbsp;<span class="hljs-keyword">long</span>&nbsp;<span class="hljs-title">test</span><span class="hljs-params">(<span class="hljs-keyword">long</span>)</span></span>;
&nbsp;&nbsp;&nbsp;descriptor:&nbsp;(J)J
&nbsp;&nbsp;&nbsp;flags:&nbsp;ACC_PUBLIC
&nbsp;&nbsp;&nbsp;Code:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-built_in">stack</span>=<span class="hljs-number">4</span>,&nbsp;locals=<span class="hljs-number">5</span>,&nbsp;args_size=<span class="hljs-number">2</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">0</span>:&nbsp;aload_0
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">1</span>:&nbsp;getfield&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">2</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;Field&nbsp;a:I</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">4</span>:&nbsp;i2l
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">5</span>:&nbsp;lload_1
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">6</span>:&nbsp;ladd
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">7</span>:&nbsp;getstatic&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">3</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;Field&nbsp;C:J</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">10</span>:&nbsp;ladd
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">11</span>:&nbsp;lstore_3
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">12</span>:&nbsp;lload_3
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">13</span>:&nbsp;lreturn
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LineNumberTable:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-built_in">line</span>&nbsp;<span class="hljs-number">13</span>:&nbsp;<span class="hljs-number">0</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-built_in">line</span>&nbsp;<span class="hljs-number">14</span>:&nbsp;<span class="hljs-number">12</span>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;LocalVariableTable:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Start&nbsp;&nbsp;Length&nbsp;&nbsp;Slot&nbsp;&nbsp;Name&nbsp;&nbsp;&nbsp;Signature
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">0</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">14</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">0</span>&nbsp;&nbsp;<span class="hljs-keyword">this</span>&nbsp;&nbsp;&nbsp;LB;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">0</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">14</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">1</span>&nbsp;&nbsp;&nbsp;num&nbsp;&nbsp;&nbsp;J
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">12</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">2</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-number">3</span>&nbsp;&nbsp;&nbsp;ret&nbsp;&nbsp;&nbsp;J
</code></pre>
<p>我们介绍一下比较重要的 3 三个数值。<br>
<strong>&lt;1&gt;</strong></p>
<p>首先，注意 stack 字样，它此时的数值为 4，表明了 test 方法的最大操作数栈深度为 4。JVM 运行时，会根据这个数值，来分配栈帧中操作栈的深度。</p>
<p><strong>&lt;2&gt;</strong></p>
<p>相对应的，locals 变量存储了局部变量的存储空间。它的单位是 Slot（槽），可以被重用。其中存放的内容，包括：</p>
<ul>
<li>this</li>
<li>方法参数</li>
<li>异常处理器的参数</li>
<li>方法体中定义的局部变量</li>
</ul>
<p><strong>&lt;3&gt;</strong></p>
<p>args_size 就比较好理解。它指的是方法的参数个数，因为每个方法都有一个隐藏参数 this，所以这里的数字是 2。</p>
<h4>字节码执行过程</h4>
<p>我们稍微回顾一下 JVM 运行时的相关内容。main 线程会拥有两个主要的运行时区域：Java 虚拟机栈和程序计数器。其中，虚拟机栈中的每一项内容叫作栈帧，栈帧中包含四项内容：局部变量报表、操作数栈、动态链接和完成出口。</p>
<p>我们的字节码指令，就是靠操作这些数据结构运行的。下面我们看一下具体的字节码指令。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/61/AD/CgpOIF4ezeKAHVCXAABv7rzSgXE896.jpg" alt=""></p>
<p>（1）<strong>0: aload_0</strong></p>
<p>把第 1 个引用型局部变量推到操作数栈，这里的意思是把 this 装载到了操作数栈中。</p>
<p>对于 static 方法，aload_0 表示对方法的第一个参数的操作。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/CgpOIF4w-GGAA6DnAAEtqWkdOnE696.jpg" alt=""></p>
<p>（2）<strong>1: getfield&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; #2</strong></p>
<p>将栈顶的指定的对象的第 2 个实例域（Field）的值，压入栈顶。#2 就是指的我们的成员变量 a。</p>
<pre><code data-language="c" class="lang-c">#<span class="hljs-number">2</span>&nbsp;=&nbsp;Fieldref&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">6.</span>#<span class="hljs-number">27</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;B.a:I</span>
...
#<span class="hljs-number">6</span>&nbsp;=&nbsp;Class&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">29</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;B</span>
#<span class="hljs-number">27</span>&nbsp;=&nbsp;NameAndType&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">8</span>:#<span class="hljs-number">9</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;a:I</span>
</code></pre>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/Cgq2xl4w-HKABrhgAAEvNAmbGWY870.jpg" alt=""></p>
<p>（3）<strong>i2l</strong></p>
<p>将栈顶 int 类型的数据转化为 long 类型，这里就涉及我们的隐式类型转换了。图中的信息没有变动，不再详解介绍。</p>
<p>（4）<strong>lload_1</strong></p>
<p>将第一个局部变量入栈。也就是我们的参数 num。这里的 l 表示 long，同样用于局部变量装载。你会看到这个位置的局部变量，一开始就已经有值了。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/CgpOIF4w-IuAOmp0AAEzFWM0gmc155.jpg" alt=""></p>
<p>（5）<strong>ladd</strong></p>
<p>把栈顶两个 long 型数值出栈后相加，并将结果入栈。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/Cgq2xl4w-KKAGhwcAAEtNkzwpcw021.jpg" alt=""></p>
<p>（6）<strong>getstatic #3</strong></p>
<p>根据偏移获取静态属性的值，并把这个值 push 到操作数栈上。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/Cgq2xl4w-MWAVt_ZAAE2NxokOfU153.jpg" alt=""></p>
<p>（7）<strong>ladd</strong></p>
<p>再次执行 ladd。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/CgpOIF4w-NCAaU4rAAEtel-Iskk153.jpg" alt=""></p>
<p>（8）<strong>lstore_3</strong></p>
<p>把栈顶 long 型数值存入第 4 个局部变量。</p>
<p>还记得我们上面的图么？slot 为 4，索引为 3 的就是 ret 变量。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/CgpOIF4w-OWAPOn9AAE1Y2sXttM659.jpg" alt=""></p>
<p>（9）<strong>lload_3</strong></p>
<p>正好与上面相反。上面是变量存入，我们现在要做的，就是把这个变量 ret，压入虚拟机栈中。</p>
<p><img src="https://s0.lgstatic.com/i/image3/M01/63/24/CgpOIF4w-O6ARdRFAAE62GkvYGo689.jpg" alt=""></p>
<p>（10）<strong>lreturn</strong></p>
<p>从当前方法返回 long。</p>
<p>到此为止，我们的函数就完成了相加动作，执行成功了。JVM 为我们提供了非常丰富的字节码指令。详细的字节码指令列表，可以参考以下网址：</p>
<p><a href="https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html">https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html</a></p>
<h4>注意点</h4>
<p>注意上面的第 8 步，我们首先把变量存放到了变量报表，然后又拿出这个值，把它入栈。为什么会有这种多此一举的操作？原因就在于我们定义了 ret 变量。JVM 不知道后面还会不会用到这个变量，所以只好傻瓜式的顺序执行。</p>
<p>为了看到这些差异。大家可以把我们的程序稍微改动一下，直接返回这个值。</p>
<pre><code data-language="java" class="lang-java"><span class="hljs-function"><span class="hljs-keyword">public</span>&nbsp;<span class="hljs-keyword">long</span>&nbsp;<span class="hljs-title">test</span><span class="hljs-params">(<span class="hljs-keyword">long</span>&nbsp;num)</span>&nbsp;</span>{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-keyword">return</span>&nbsp;<span class="hljs-keyword">this</span>.a&nbsp;+&nbsp;num&nbsp;+&nbsp;C;
}
</code></pre>
<p>再次看下，对应的字节码指令是不是简单了很多？</p>
<pre><code data-language="c" class="lang-c"><span class="hljs-number">0</span>:&nbsp;aload_0
<span class="hljs-number">1</span>:&nbsp;getfield&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">2</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;Field&nbsp;a:I</span>
<span class="hljs-number">4</span>:&nbsp;i2l
<span class="hljs-number">5</span>:&nbsp;lload_1
<span class="hljs-number">6</span>:&nbsp;ladd
<span class="hljs-number">7</span>:&nbsp;getstatic&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;#<span class="hljs-number">3</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="hljs-comment">//&nbsp;Field&nbsp;C:J</span>
<span class="hljs-number">10</span>:&nbsp;ladd
<span class="hljs-number">11</span>:&nbsp;lreturn
</code></pre>
<p>那我们以后编写程序时，是不是要尽量少的定义成员变量？</p>
<p>这是没有必要的。栈的操作复杂度是 O(1)，对我们的程序性能几乎没有影响。平常的代码编写，还是以可读性作为首要任务。</p>
<h3>小结</h3>
<p>本课时，我们学会了使用 javap 和 jclasslib 两个工具。平常工作中，掌握第一个就够了，后者主要为我们提供更加直观的展示。</p>
<p>我们从实际分析一段代码开始，详细介绍了几个字节码指令对程序计数器、局部变量表、操作数栈等内容的影响，初步接触了 Java 的字节码文件格式。</p>
<p>希望你能够建立起一个运行时的脉络，在看到相关的 opcode 时，能够举一反三的思考背后对这些数据结构的操作。这样理解的字节码指令，根本不会忘。</p>
<p>你还可以尝试着对 A 类的代码进行分析，我们这里先留下一个悬念。课程后面会详细介绍 JVM 在方法调用上的一些特点。</p>