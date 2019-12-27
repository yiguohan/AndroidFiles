[TOC]

# 深入理解 Android 之 Gradle

Gradle是当前非常“劲爆”的构建工具。本篇文章就是专为讲解Gradle而来。介绍Gradle之前，先说点题外话。

## 1.题外话

说实话，我在大法工作的时候，就见过Gradle。但是当时我一直不知道这是什么东西。而且打法工具组的工程师还将其和Android Studio大法版一起推送，偶尔一看就更没兴趣了。为什么那个时候如此不待见Gradle呢？因为我此前一直是做ROM开发。在这个层面上，我们用make/mm或者mmm就可以了。而且，编译耗时对我们来说也不啥通电，因为用组内吊炸天的神级服务器完整编译大法的image也要耗费一个小时左右。所以那个时候Gradle完全不是我们的菜。

现在，搞App开发居多，编译/打包等问题立即就成痛点了。比如：

- 一个App有多个版本，Release版、Debug版、Test版。甚至针对不同App Store都有不同的版本。在以前ROM的环境下，虽然可以配置Android.mk，但是需要依赖整个Android源码，而且还不能完全做到满足条件，很多事情需要手动搞。一个app如果涉及到多个开发者，手动操作必然会带来混乱。
- Library工程我们需要编译成jar包，然后发布给其他开发者用。以前是用Eclipse的Export，做一堆选择。要是能自动编译成jar包就爽了。

上述问题对绝大部分App开发者而言都不陌生，而Gradle作为一种很方便的构建工具，可以非常轻松地解决构建过程的各种问题。

## 2.闲言构建

构建，叫`build`也好，叫`make`也行。反正就是根据输入信息然后干一堆事情，最后得到几个产出物（Artifact）。

最最简单的构建工具就是`make`了。`make`就是根据`Makefile`文件中写的规则，执行对应的命令，然后得到目标产物。

> 日常生活中，和构建最类似的一个场景就是做菜。输入各种食材，然后按固定的工序，最后得到一盘菜。当然，做同样一道菜，由于需求不同，做出来的东西也不尽相同。比如，宫保鸡丁这道菜，回民要求不能放大油，口淡的要求少放盐和各种油，辣不怕的男女汉子们可以要求多方辣子...总之，做菜包含固定的工序，但是对于不同条件或需求，需要做不同的处理。

在`Gradle`爆红之前，常用的构建工具是`Ant`，然后又进化到`Maven`。`Ant`和`Maven`这两个工具其实也还算方便，现在还有很多地方在使用。但是二者都有一些缺点，所以让更懒的人觉得不是那么方便。比如，`Maven`编译规则使用`XML`来编写的。`XML`虽然通俗易懂，但是很难在`XML`中描述`if(某条件成立){编译某文件}/else{编译其他文件}`这样有不同条件的任务。

怎么解决？怎么解决好？对程序员而言，自然是编程解决，但是又几个小要求：

- 这种“编程”不要搞得和程序员理解的编程那样复杂。寥寥几笔，轻轻松松把要做的事情描述出来就最好不过。所以，`Gradle`选择了`Groovy`。`Groovy`基于`Java`并拓展了`Java`。`Java`程序员可以无缝切换到使用`Groovy`开发程序。**`Groovy`说白了就是把写`Java`程序变得像写脚本一样简单。写完就可以执行，`Groovy`内部会将其编译成`Java class`然后启动虚拟机来执行。当然，这些底层的渣活不需要你管。**

- 除了可以用很灵活的语言来写构建规则外，`Gradle`另外一个特点就是它是一种`DSL`，即`Domain Specific Language`，领域相关语言。什么是`DSL`，说白了它是某个行业中的行话。还是不明白？徐克导演的《智取威虎山》中就有很典型的`DSL`使用描述，比如：

  > 土匪：蘑菇，你哪路？什么价？（什么人？到哪里去？）
  >
  > 杨子荣：哈！想啥来啥，想吃奶来了妈妈，想娘家的人，孩子他舅舅来了。（找同行）
  >
  > 杨子荣：拜见三爷！
  >
  > 土匪：天王盖地虎！（你好大的胆！敢来来气你的祖宗？）
  >
  > 杨子荣：宝塔镇河妖！（要是那样，叫我从山上摔死，掉河里淹死。）
  >
  > 土匪：野鸡闷头钻，哪能上天王山！（你不是正牌的。）
  >
  > 杨子荣：地上有的是米，喂呀，有根底！（老子是正牌的，老牌的。）

`Gradle`中也又类似的行话，比如`sourceSets`代表源文件的集合等......**太多了，记不住**。以后我们都会接触到这些行话。那么，对使用者而言，这些行话的好处是什么呢？这就是：

**一句行话可以包含很多意思，而且在这个行当里的人一听就懂，不用解释。另外，基于行话，我们甚至可以建立一个模板，使用者只要往这个模板里填必须要填的内容，`Gradle`就可以非常漂亮地完成工作，得到想要的东西。**

> 这就和现在的智能炒菜机器似的，只要选择菜谱，把食材准备好，剩下的事情就不用你操心了。吃货们对这种做菜方式肯定是以反感为主，太没有特色了。但是程序员对`Gradle`类似做法却热烈拥抱。

到此，大家应该明白要真正学会Gradle恐怕是离不开下面两个基础知识：

- `Groovy`，由于它基于`Java`，所以我们仅介绍`Java`之外的东西。了解`Groovy`语言是掌握`Gradle`的基础。
- `Gradle`作为一个工具，它的行话和它“为人处世”的原则。

## 3.Groovy介绍

`Groovy`是一种动态语言。这种语言比较有特点，它和`Java`一样，也运行于`Java`虚拟机中。嗯？？对头，简单粗暴点儿看，你可以认为`Groovy`扩展了`Java`语言。比如，`Groovy`对自己的定义就是：**`Groovy`是在`Java`平台上的、具有像`Python`，`Ruby`和`Smalltalk`语言特性的灵活动态语言，`Groovy`保证了这些特征像`Java`语法一样被`Java`开发者使用。**

除了语言和`Java`相同外，Groovy有时候又像一种脚本语言。前文也提到过，当我执行Groovy脚本时，Groovy会先将其编译成Java类字节码，然后通过Jvm来执行这个Java类。图1展示了Java/Groovy和Jvm之间的关系。

![img](https://img-blog.csdn.net/20150905192824392?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

> *实际上，由于Groovy Code在真正执行的时候已经变成了Java字节码，所以JVM根本不知道自己运行的是Groovy代码*。

下面我们将介绍Groovy。由于此文的主要目的是Gradle，所以我们不会过多讨论Groovy中细枝末节的东西，而是把知识点集中在以后和Gradle打交道时一些常用的地方上。

### 3.1 Groovy开发环境

在学习本节的时候，最好部署一下Groovy开发环境。根据Groovy官网的介绍（*http://www.groovy-lang.org/download.html#gvm*），部署Groovy开发环境非常简单，在Ubuntu或者cygwin之类的地方

- curl -s get.gvmtool.net | bash
- source"$HOME/.gvm/bin/gvm-init.sh"
- gvm install groovy

执行完最后一步，Groovy就下载并安装了。图1是安装时候的示意图

![img](https://img-blog.csdn.net/20150905192939249?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

然后，创建一个test.groovy文件，里边只有一行代码：

``` groovy
println "hello groovy"
```

执行groovy test.groovy，输出结果如图2所示：

![img](https://img-blog.csdn.net/20150905192952248?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

亲们，必须要完成上面的操作啊。做完后，有什么感觉和体会？

*最大的感觉可能就是groovy和shell脚本，或者python好类似。*

另外，除了可以直接使用JDK之外，Groovy还有一套GDK，网址是***\*http://www.groovy-lang.org/api.html\****。

说实话，看了这么多家API文档，还是Google的Android API文档做得好。其页面中右上角有一个搜索栏，在里边输入一些关键字，瞬间就能列出候选类，相关文档，方便得不得了啊.....

### 3.2 一些前提知识

为了后面讲述方面，这里先介绍一些前提知识。初期接触可能有些别扭，看习惯就好了。

- Groovy注释标记和Java一样，支持//或者/**/

- Groovy语句可以不用分号结尾。Groovy为了尽量减少代码的输入，确实煞费苦心

- Groovy中支持动态类型，即定义变量的时候可以不指定其类型。Groovy中，变量定义可以使用关键字def。注意，虽然def不是必须的，但是为了代码清晰，建议还是使用def关键字

  ``` groovy
  def variable1 = 1//可以不使用分号结尾
  def variable2 = "I am a person"
  def int x = 1//变量定义时，也可以直接指定类型
  ```

- 函数定义时，参数的类型也可以不指定。比如

  ```groovy
  String testFunction(arg1,arg2){//无需指定参数类型
    ...
  }
  ```

- 除了变量定义可以不指定类型外，Groovy中函数的返回值也可以是无类型的。比如：

  ```groovy
  //无类型的函数定义，必须使用def关键字
  def  nonReturnTypeFunc(){
      last_line   //最后一行代码的执行结果就是本函数的返回值
  }
  
  //如果指定了函数返回类型，则可不必加def关键字来定义函数
  String getString(){
     return"I am a string"
  }
  ```

其实，所谓的无返回类型的函数，我估计内部都是按返回Object类型来处理的。毕竟，Groovy是基于Java的，而且最终会转成Java Code运行在JVM上

- 函数返回值：Groovy的函数里，可以不使用returnxxx来设置xxx为函数返回值。如果不使用return语句的话，则函数里最后一句代码的执行结果被设置成返回值。比如：

  ```groovy
  //下面这个函数的返回值是字符串"getSomething return value"
  def getSomething(){
       "getSomething return value" //如果这是最后一行代码，则返回类型为String
        1000//如果这是最后一行代码，则返回类型为Integer
  }
  ```

> 注意，如果函数定义时候指明了返回值类型的话，函数中则必须返回正确的数据类型，否则运行时报错。如果使用了动态类型的话，你就可以返回任何类型了。

- Groovy对字符串支持相当强大，充分吸收了一些脚本语言的优点：

  - 单引号''中的内容严格对应Java中的String，不对$符号进行转义

    ```groovy
    def singleQuote='I am $ dolloar'  //输出就是I am $ dolloar
    ```

  - 双引号""的内容则和脚本语言的处理有点像，如果字符中有$号的话，则它会$表达式先求值。

    ```groovy
    def doubleQuoteWithoutDollar = "I am one dollar" //输出 I am one dollar
    def x = 1
    def doubleQuoteWithDollar = "I am $x dolloar" //输出I am 1 dolloar
    ```

  - 三个引号'''xxx'''中的字符串支持随意换行 比如

    ```groovy
     def multieLines = ''' begin
         line  1
         line  2
         end '''
    ```

- 最后，除了每行代码不用加分号外，Groovy中函数调用的时候还可以不加括号。比如：

  ```groovy
  println("test") ---> println"test"
  ```

- 注意，虽然写代码的时候，对于函数调用可以不带括号，但是Groovy经常把属性和函数调用混淆。比如

  ```groovy
  def getSomething(){
    "hello"
  }
  getSomething()  //如果不加括号的话，Groovy会误认为getSomething是一个变量。
  ```

  ![img](https://img-blog.csdn.net/20150905193011828?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

所以，调用函数要不要带括号，我个人意见是如果这个函数是Groovy API或者Gradle API中比较常用的，比如println，就可以不带括号。否则还是带括号。Groovy自己也没有太好的办法解决这个问题，只能兵来将挡水来土掩了。

好了，了解上面一些基础知识后，我们再介绍点深入的内容。

### 3.3 Groovy中的数据类型

Groovy中的数据类型我们就介绍几种和Java不太一样的：

- Java中的基本数据类型
- Groovy中的容器类
- 闭包

#### 3.3.1 基本数据类型

作为动态语言，Groovy世界中的所有事物都是对象。所以，int，boolean这些Java中的基本数据类型，在Groovy代码中其实对应的是它们的包装数据类型。比如int对应为Integer，boolean对应为Boolean。比如下图中的代码执行结果：

![img](https://img-blog.csdn.net/20150905193120964?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 3.3.2 容器类

Groovy中的容器类很简单，就三种：

- List：链表，其底层对应Java中的List接口，一般用ArrayList作为真正的实现类。
- Map：键-值表，其底层对应Java中的LinkedHashMap。
- Range：范围，它其实是List的一种拓展。

对容器而言，我们最重要的是了解它们的用法。下面是一些简单的例子：

- List类

  变量定义：List变量由[]定义，比如

  ```groovy
  def aList = [5,'string',true] //List由[]定义，其元素可以是任何对象
  
  ```

  变量存取：可以直接通过索引存取，而且不用担心索引越界。如果索引超过当前链表长度，List会自动往该索引添加元素

  ```groovy
  assert aList[1] == 'string'
  assert aList[5] == null //第6个元素为空
  aList[100] = 100 //设置第101个元素的值为10
  assert aList[100] == 100
  ```

  那么，aList到现在为止有多少个元素呢？

  println aList.size  ===>结果是101

- Map类

  容器变量定义

  变量定义：Map变量由[:]定义，比如

  ```groovy
  def aMap = ['key1':'value1','key2':true]
  ```

  Map由[:]定义，注意其中的冒号。冒号左边是key，右边是Value。key必须是字符串，value可以是任何对象。另外，key可以用单引号或双引号包起来，也可以不用引号包起来。比如

  ```groovy
  def aNewMap = [key1:"value",key2:true]//其中的key1和key2默认被处理成字符串"key1"和"key2"
  ```

  不过Key要是不使用引号包起来的话，也会带来一定混淆，比如

  ```groovy
  def key1="wowo"
  def aConfusedMap=[key1:"who am i?"]
  ```

  aConfuseMap中的key1到底是"key1"还是变量key1的值“wowo”？显然，答案是字符串"key1"。如果要是"wowo"的话，则aConfusedMap的定义必须设置成：

  ```groovy
  def aConfusedMap=[(key1):"who am i?"]
  ```

  Map中元素的存取更加方便，它支持多种方法：

  ```groovy
  println aMap.keyName    //这种表达方法好像key就是aMap的一个成员变量一样
  println aMap['keyName'] //这种表达方法更传统一点
  aMap.anotherkey = "i am map" //为map添加新元素
  ```

- Range类

  Range是Groovy对List的一种拓展，变量定义和大体的使用方法如下：

  ```groovy
  def aRange = 1..5 //Range类型的变量 由begin值+两个点+end值表示左边这个aRange包含1,2,3,4,5这5个值。
  ```

  如果不想包含最后一个元素，则

  ```groovy
  def aRangeWithoutEnd = 1..<5 //包含1,2,3,4这4个元素
  println aRange.from
  println aRange.to
  ```

#### 3.3.4 Groovy API的一些秘笈

前面讲这些东西，主要是让大家了解Groovy的语法。实际上在coding的时候，是离不开SDK的。由于Groovy是动态语言，所以要使用它的SDK也需要掌握一些小诀窍。

Groovy的API文档位于http://www.groovy-lang.org/api.html

以上文介绍的Range为例，我们该如何更好得使用它呢？

先定位到Range类。它位于groovy.lang包中：

![img](https://img-blog.csdn.net/20150905193143290?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

有了API文档，你就可以放心调用其中的函数了。不过，不过，不过：我们刚才代码中用到了Range.from/to属性值，但翻看Range API文档的时候，其实并没有这两个成员变量。图6是Range的方法

![img](https://img-blog.csdn.net/20150905193206100?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

文档中并没有说明Range有from和to这两个属性，但是却有getFrom和getTo这两个函数。*What happened？*原来：

根据Groovy的原则，如果一个类中有名为xxyyzz这样的属性（其实就是成员变量），Groovy会自动为它添加getXxyyzz和setXxyyzz两个函数，用于获取和设置xxyyzz属性值。

注意，get和set后第一个字母是大写的

所以，当你看到Range中有getFrom和getTo这两个函数时候，就得知道潜规则下，Range有from和to这两个属性。当然，由于它们不可以被外界设置，所以没有公开setFrom和setTo函数。

### 3.4 闭包

#### 3.4.1 闭包的样子

闭包，英文叫Closure，是Groovy中非常重要的一个数据类型或者说一种概念了。闭包的历史来源，种种好处我就不说了。我们直接看怎么使用它！

闭包，是一种数据类型，它代表了一段可执行的代码。其外形如下：

```groovy
def aClosure = {//闭包是一段代码，所以需要用花括号括起来..
    String param1, int param2 ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
    println"this is code" //这是代码，最后一句是返回值，也可以使用return，和Groovy中普通函数一样
}
```

简而言之，Closure的定义格式是：

```groovy
def xxx = {paramters -> code} //或者 def xxx = {无参数，纯code} 这种case不需要->符号
```

说实话，从C/C++语言的角度看，闭包和函数指针很像。闭包定义好后，要调用它的方法就是：

`闭包对象.call(参数)` 或者更像函数指针调用的方法：

`闭包对象(参数) `

比如：

```groovy
aClosure.call("this is string",100)
```

或者

``` groovy
aClosure("this is string", 100)
```

上面就是一个闭包的定义和使用。在闭包中，还需要注意一点：

如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫it，和this的作用类似。it代表闭包的参数。

比如：

```groovy
def greeting = { "Hello, $it!" }
assert greeting('Patrick') == 'Hello, Patrick!'
```

等同于：

```groovy
def greeting = { it -> "Hello, $it!"}
assert greeting('Patrick') == 'Hello, Patrick!'
```

但是，如果在闭包定义时，采用下面这种写法，则表示闭包没有参数！

```groovy
def noParamClosure = { -> true }
```

这个时候，我们就不能给noParamClosure传参数了！

```groovy
noParamClosure ("test")  //报错喔！
```

#### 3.4.2 Closure使用中的注意点

1. 省略圆括号

   闭包在Groovy中大量使用，比如很多类都定义了一些函数，这些函数最后一个参数都是一个闭包。比如：

   ```groovy
   public static <T> List<T>each(List<T> self, Closure closure)
   ```

   上面这个函数表示针对List的每一个元素都会调用closure做一些处理。这里的closure，就有点回调函数的感觉。但是，在使用这个each函数的时候，我们传递一个怎样的Closure进去呢？比如：

   ```groovy
   def iamList = [1,2,3,4,5]  //定义一个List
   iamList.each{ //调用它的each，这段代码的格式看不懂了吧？each是个函数，圆括号去哪了？
         println it
   }
   ```

   上面代码有两个知识点：

   each函数调用的圆括号不见了！原来，Groovy中，当函数的最后一个参数是闭包的话，可以省略圆括号。比如

   ```groovy
   def testClosure(int a1,String b1, Closure closure){
         //dosomething
        closure() //调用闭包
   }
   ```

   那么调用的时候，就可以免括号！

   ```groovy
   testClosure (4, "test", {
      println"i am in closure"
   } ) 
   ```

   注意，这个特点非常关键，因为以后在Gradle中经常会出现图7这样的代码

   ```groovy
   task hello{
       dolast{
           println 'Hello world!'
       }
   }
   ```

   经常碰见这样的没有圆括号的代码。省略圆括号虽然使得代码简洁，看起来更像脚本语言，但是它这经常会让我confuse（不知道其他人是否有同感），以doLast为例，完整的代码应该按下面这种写法

   ```groovy
   doLast({
      println'Hello world!'
   })
   ```

   有了圆括号，你会知道 doLast只是把一个Closure对象传了进去。很明显，它不代表这段脚本解析到doLast的时候就会调用println 'Hello world!' 。

   但是把圆括号去掉后，就感觉好像println 'Hello world!'立即就会被调用一样！

2. 如何确定Closure的参数

   另外一个比较让人头疼的地方是，Closure的参数该怎么搞？还是刚才的each函数：

   ```groovy
   public static <T> List<T> each(List<T>self, Closure closure)
   ```

   如何使用它呢？比如：

   ```groovy
   def iamList = [1,2,3,4,5]  //定义一个List变量
   iamList.each{ //调用它的each函数，只要传入一个Closure就可以了。
     println it
   }
   ```

   看起来很轻松，其实：

   对于each所需要的Closure，它的参数是什么？有多少个参数？返回值是什么？

   我们能写成下面这样吗？

   ```groovy
   iamList.each{String name,int x ->
     return x
   }  //运行的时候肯定报错！
   ```

   所以，Closure虽然很方便，但是它一定会和使用它的上下文有极强的关联。要不，作为类似回调这样的东西，我如何知道调用者传递什么参数给Closure呢？

   此问题如何破解？只能通过查询API文档才能了解上下文语义。比如下图：

   ![img](https://img-blog.csdn.net/20150905193242319?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   图中：

   - each函数说明中，将给指定的closure传递Set中的每一个item。所以，closure的参数只有一个。
   - findAll中，**绝对抓瞎**了。一个是没说明往Closure里传什么。另外没说明Closure的返回值是什么.....。

   对Map的findAll而言，Closure可以有两个参数。findAll会将Key和Value分别传进去。并且，Closure返回true，表示该元素是自己想要的。返回false表示该元素不是自己要找的。示意代码如下所示：

   ```groovy
   def aMap = [k1:'value1',k2:true]
   def results = aMap.findAll{
       key,value->
       println "key=$key, value=$value"
       if(key == 'k1'){
           return true
       }
       return false
   }
   ```

   > Closure 的使用有点坑，很大程度上依赖于你对API的熟悉程度，所以最初阶段，SDK查询时少不了的。

### 3.5  脚本类、文件I/O和XML操作

最后，我们来看一下Groovy中比较高级的用法。

#### 3.5.1 脚本类

1. 脚本中的import其他类

   Groovy中可以像Java那样写package，然后写类。比如在文件夹com/cmbc/groovy目录中放一个文件，叫Test.groovy，如下所示

   ```groovy
   package com.cmbc.groovy
   
   class Test{
       String name
       String title
       
       Test(name,title){
           this.name = name
           this.title = title
       }
       def print(){
           println name + " " + title
       }
   }
   ```

   你看，以上代码中的Test.groovy和Java类就很类似了。当然，如果不声明public/private等访问权限的话，Groovy中类及其变量默认都是public的。

   现在我们在测试的根目录下建立一个test.groovy文件。其代码如下所示：

   ```groovy
   import com.cmbc.groovy.Test
   
   def test = new Test("cmbc","is wonderful")
   test.print()
   ```

   你看，test.groovy先import了com.cmbc.groovy.Test类，然后创建了一个Test类型的对象，接着调用它的print函数。

   这两个groovy文件的目录结构如图所示：

   ![img](https://img-blog.csdn.net/20150905193800825?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   在groovy中，系统自带会加载当前目录/子目录下的xxx.groovy文件。所以，当执行groovy test.groovy的时候，test.groovy import的Test类能被自动搜索并加载到。

2. 脚本到底是什么？

   Java中，我们最熟悉的是类。但是我们在Java的一个源码文件中，不能不写class（interface或者其他....），而Groovy可以像写脚本一样，把要做的事情都写在xxx.groovy中，而且可以通过groovy xxx.groovy直接执行这个脚本。这到底是怎么搞的？

   既然是基于Java的，Groovy会先把xxx.groovy中的内容转换成一个Java类。比如test.groovy的代码是：

   ```groovy
   println 'Groovy world!'
   ```

   Groovy把它转换成这样的Java类：

   执行`groovyc-d classes test.groovy`

   `groovyc`是groovy的编译命令，-d classes用于将编译得到的class文件拷贝到classes文件夹下

   下图是test.groovy脚本转换得到的java class。用jd-gui反编译它的代码：

   ![img](https://img-blog.csdn.net/20150905193818456?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   图中：

   - test.groovy被转换成了一个test类，它从script派生。
   - **每一个脚本都会生成一个static main函数**。这样，当我们groovytest.groovy的时候，其实就是用java去执行这个main函数
   - **脚本中的所有代码都会放到run函数中**。比如，println 'Groovy world'，这句代码实际上是包含在run函数里的。
   - 如果脚本中定义了函数，则函数会被定义在test类中。

   `groovyc`是一个比较好的命令，读者要掌握它的用法。然后利用jd-gui来查看对应class的Java源码

3. 脚本的变量和作用域

   前面说了，xxx.groovy只要不是和Java那样的class，那么它就是一个脚本。而且脚本的代码其实都会被放到run函数中去执行。那么，在Groovy的脚本中，很重要的一点就是脚本中定义的*变量和它的作用域*。举例：

   ```groovy
   def x = 1 //注意，这个x有def（或者指明类型，比如 int x = 1）
   def printx(){
      println x
   }
   ```

   printx() <==报错，说x找不到

   为什么？继续来看反编译后的class文件。

   ![img](https://img-blog.csdn.net/20150905193919401?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   图中：

   - **printx被定义成test类的成员函数**
   - `def x = 1`，这句话是在run中创建的。所以，x=1从代码上看好像是在整个脚本中定义的，但实际上printx访问不了它。printx是test成员函数，除非x也被定义成test的成员函数，否则printx不能访问它。

   那么，如何使得printx能访问x呢？很简单，定义的时候不要加类型和def。即：

   ```groovy
   x = 1 //注意，去掉def或者类型
   def printx(){
      println x
   }
   ```

   printx() <==OK

   这次Java源码又变成什么样了呢？

   ![img](https://img-blog.csdn.net/20150905193938259?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   上图中，x也没有被定义成test的成员函数，而是在run的执行过程中，将x作为一个属性添加到test实例对象中了。然后在printx中，先获取这个属性。

   注意，Groovy的文档说 x = 1这种定义将使得x变成test的成员变量，但从反编译情况看，这是不对得.....

   虽然printx可以访问x变量了，但是假如有其他脚本却无法访问x变量。因为它不是test的成员变量。

   比如，我在测试目录下创建一个新的名为test1.groovy。这个test1将访问test.groovy中定义的printx函数：

   ```groovy
   def atest = new Test()
   atest.printx()
   ```

   这种方法使得我们可以将代码分成模块来编写，比如将公共的功能放到test.groovy中，然后使用公共功能的代码放到test1.groovy中。

   执行`groovy test1.groovy`，报错。说x找不到。这是因为x是在test的run函数动态加进去的。怎么办？

   ```groovy
   import groovy.transform.Field;   //必须要先import
   @Field x = 1  //在x前面加上@Field标注，这样，x就彻彻底底是test的成员变量了。
   ```

   查看编译后的test.class文件，得到：

   ![img](https://img-blog.csdn.net/20150905194014968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   这个时候，test.groovy中的x就成了test类的成员函数了。如此，我们可以在script中定义那些需要输出给外部脚本或类使用的变量了！

#### 3.5.2 文件I/O操作

本节介绍下Groovy的文件I/O操作。直接来看例子吧，虽然比Java看起来简单，但要理解起来其实比较难。尤其是当你要自己查SDK并编写代码的时候。

整体说来，Groovy的I/O操作是在原有Java I/O操作上进行了更为简单方便的封装，并且使用Closure来简化代码编写。主要封装了如下一些了类：

![img](https://img-blog.csdn.net/20150905194032351?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1. 读文件

   Groovy中，文件读操作简单到令人发指：

   ```groovy
   def targetFile = new File(文件名) //File对象还是要创建的。
   ```

   然后打开http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/File.html

   看看Groovy定义的API：

   - 读该文件中的每一行：eachLine的唯一参数是一个Closure。Closure的参数是文件每一行的内容，其内部实现肯定是Groovy打开这个文件，然后读取文件的一行，然后调用Closure...

     ```groovy
     targetFile.eachLine{
         String oneLine ->
         println oneLine
     }//是不是令人发指？？！
     ```

   - 直接得到文件内容

     ```groovy
     targetFile.getBytes()  //文件内容一次性读出，返回类型为byte[]
     ```

     注意前面提到的getter和setter函数，这里可以直接使用targetFile.bytes

   - 使用InputStream。InputStream的SDK在http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/InputStream.html

     ```groovy
     def ism =  targetFile.newInputStream()
     //操作ism，最后记得关掉
     ism.close
     ```

   - 使用闭包操作inputStream，以后在Gradle里会常看到这种搞法

     ```groovy
     targetFile.withInputStream{ ism ->
        //操作ism. 不用close。Groovy会自动替你close
     }
     ```

     确实够简单，令人发指。我当年死活也没找到withInputStream是个啥意思。所以，请各位开发者牢记Groovy I/O操作相关类的SDK地址：

     - java.io.File: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/File.html
     - java.io.InputStream: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/InputStream.html     
     - java.io.OutputStream: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/OutputStream.html
     - java.io.Reader: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/Reader.html
     - java.io.Writer: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/Writer.html
     - java.nio.file.Path: http://docs.groovy-lang.org/latest/html/groovy-jdk/java/nio/file/Path.html

2. 写文件

   和读文件差不多。不再啰嗦。这里给个例子，告诉大家如何copy文件。

   ```groovy
   def srcFile = new File(源文件名)
   def targetFile = new File(目标文件名)
   targetFile.withOutputStream{ os->
     srcFile.withInputStream{ ins->
         os << ins   //利用OutputStream的<<操作符重载，完成从inputstream到OutputStream
          //的输出
      }
   }
   ```

   尼玛....关于OutputStream的<<操作符重载，查看SDK文档后可知：

   ![img](https://img-blog.csdn.net/20150905194055193?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   再一次向极致简单致敬。但是，SDK恐怕是离不开手了...

#### 3.5.3 XML操作

除了I/O异常简单之外，Groovy中的XML操作也极致得很。Groovy中，XML的解析提供了和XPath类似的方法，名为GPath。这是一个类，提供相应API。关于XPath，请脑补https://en.wikipedia.org/wiki/XPath。

GPath功能包括：给个例子好了，来自Groovy官方文档。

```xml
test.xml文件：
<response version-api="2.0">
       <value>
           <books>
               <book available="20" id="1">
                   <title>Don Xijote</title>
                   <author id="1">Manuel De Cervantes</author>
               </book>
               <book available="14" id="2">
                   <title>Catcher in the Rye</title>
                  <author id="2">JD Salinger</author>
              </book>
              <book available="13" id="3">
                  <title>Alice in Wonderland</title>
                  <author id="3">Lewis Carroll</author>
              </book>
              <book available="5" id="4">
                  <title>Don Xijote</title>
                  <author id="4">Manuel De Cervantes</author>
              </book>
           </books>
      </value>
   </response>
```

现在来看怎么玩转GPath：

```groovy
//第一步，创建XmlSlurper类
def xparser = new XmlSlurper()
def targetFile = new File("test.xml")
//轰轰的GPath出场
GPathResult gpathResult =xparser.parse(targetFile)
 
//开始玩test.xml。现在我要访问id=4的book元素。
//下面这种搞法，gpathResult代表根元素response。通过e1.e2.e3这种
//格式就能访问到各级子元素....
def book4 = gpathResult.value.books.book[3]
//得到book4的author元素
def author = book4.author
//再来获取元素的属性和textvalue
assert author.text() == ' Manuel De Cervantes '
获取属性更直观
author.@id == '4' 或者 author['@id'] == '4'
属性一般是字符串，可通过toInteger转换成整数
author.@id.toInteger() == 4
好了。GPath就说到这。再看个例子。我在使用Gradle的时候有个需求，就是获取AndroidManifest.xml版本号（versionName）。有了GPath，一行代码搞定，请看：
def androidManifest = newXmlSlurper().parse("AndroidManifest.xml")
println androidManifest['@android:versionName']
或者
println androidManifest.@'android:versionName'
```

### 3.6 更多

作为一门语言，Groovy是复杂的，是需要深入学习和钻研的。一本厚书甚至都无法描述Groovy的方方面面。

Anyway，从使用角度看，尤其是又限定在Gradle这个领域内，能用到的都是Groovy中一些简单的知识

## 4. Gradle介绍

现在正式进入Gradle。Gradle是一个工具，同时它也是一个编程框架。前面也提到过，使用这个工具可以完成app的编译打包等工作。当然你也可以用它干其他的事情。

Gradle是什么？学习它到什么地步就可以了？

> 看待问题的时候，所站的角度非常重要。
>
> 当你把Gradle当工具看的时候，我们只想着如何用好它。会写、写好配置脚本就OK
>
> 当你把它当做编程框架看的时候，你可能需要学习很多更深入的内容。另外，今天我们把它当工具看，明天因为需求发生变化，我们可能又得把它当编程框架看。

### 4.1 Gradle开发环境部署

Gradle的官网：http://gradle.org/

文档位置：https://docs.gradle.org/current/release-notes。其中的*UserGuide*和*DSL Reference*很关键。User Guide就是介绍Gradle的一本书，而DSL Reference是Gradle API的说明。

以Ubuntu为例，下载Gradle：*http://gradle.org/gradle-download/* 选择Completedistribution和Binary only distribution都行。然后解压到指定目录。

最后，设置~/.bashrc，把Gradle加到PATH里，如图20所示：

![img](https://img-blog.csdn.net/20150905194121511?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

执行source ~/.bashrc，初始化环境。

执行gradle --version，如果成功运行就OK了。

注意，为什么说Gradle是一个编程框架？来看它提供的API文档：

https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html

![img](https://img-blog.csdn.net/20150905194136650?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

原来，我们编写所谓的编译脚本，其实就是玩Gradle的API....所以它从更底层意义上看，是一个编程框架！

既然是编程框架，我在讲解Gradle的时候，尽量会从API的角度来介绍。有些读者肯定会不耐烦，为嘛这么费事？

从我个人的经历来看：因为我从网上学习到的资料来看，几乎全是从脚本的角度来介绍Gradle，结果学习一通下来，只记住参数怎么配置，却不知道它们都是函数调用，都是严格对应相关API的。

而从API角度来看待Gradle的话，有了SDK文档，你就可以编程。编程是靠记住一行行代码来实现的吗？不是，是在你掌握大体流程，然后根据SDK+API来完成的！

其实，Gradle自己的User Guide也明确说了：

Buildscripts are code

### 4.2 基本组件

Gradle是一个框架，它定义一套自己的游戏规则。我们要玩转Gradle，必须要遵守它设计的规则。下面我们来讲讲Gradle的基本组件：

Gradle中，每一个待编译的工程都叫一个Project。每一个Project在构建的时候都包含一系列的Task。比如一个Android APK的编译可能包含：Java源码编译Task、资源编译Task、JNI编译Task、lint检查Task、打包生成APK的Task、签名Task等。

一个Project到底包含多少个Task，其实是由编译脚本指定的插件决定。插件是什么呢？插件就是用来定义Task，并具体执行这些Task的东西。

刚才说了，Gradle是一个框架，作为框架，它负责定义流程和规则。而具体的编译工作则是通过插件的方式来完成的。比如编译Java有Java插件，编译Groovy有Groovy插件，编译Android APP有Android APP插件，编译Android Library有Android Library插件

好了。到现在为止，你知道Gradle中每一个待编译的工程都是一个Project，一个具体的编译过程是由一个一个的Task来定义和执行的。

#### 4.2.1 一个重要的例子

下面我们来看一个实际的例子。这个例子非常有代表意义。下图是一个名为posdevice的目录。这个目录里包含3个Android Library工程，2个Android APP工程。

![img](https://img-blog.csdn.net/20150905194157628?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在上图的例子中：

-  CPosDeviceSdk、CPosSystemSdk、CPosSystemSdkxxxImpl是Android Library。其中，CPosSystemSdkxxxImpl依赖CPosSystemSdk
- CPosDeviceServerApk和CPosSdkDemo是Android APP。这些App和SDK有依赖关系。CPosDeviceServerApk依赖CPosDeviceSdk，而CPosSdkDemo依赖所有的Sdk Library。

> 请回答问题，在上面这个例子中，有多少个Project？
>
> 请回答问题，在上面这个例子中，有多少个Project？
>
> 请回答问题，在上面这个例子中，有多少个Project？

答案是：每一个Library和每一个App都是单独的Project。根据Gradle的要求，每一个Project在其根目录下都需要有一个build.gradle。build.gradle文件就是该Project的编译脚本，类似于Makefile。

看起来好像很简单，但是请注意：posdevice虽然包含5个独立的Project，但是要独立编译他们的话，得：

- cd 某个Project的目录。比如 cd CPosDeviceSdk
- 然后执行 gradle xxxx（xxx是任务的名字。对Android来说，assemble这个Task会生成最终的产物，所以 gradle assemble）

这很麻烦啊，有10个独立Project，就得重复执行10次这样的命令。更有甚者，所谓的独立Project其实有依赖关系的。比如我们这个例子。

那么，我想在posdevice目录下，直接执行gradle assemble，是否能把这5个Project的东西都编译出来呢？

答案自然是可以。在Gradle中，这叫*Multi-Projects Build*。把posdevice改造成支持Gradle的Multi-Projects Build很容易，需要：

- 在posdevice下也添加一个build.gradle。这个build.gradle一般干得活是：配置其他子Project的。比如为子Project添加一些属性。这个build.gradle有没有都无所属。
- 在posdevice下添加一个名为settings.gradle。这个文件很重要，名字必须是settings.gradle。它里边用来告诉Gradle，这个multiprojects包含多少个子Project。

来看settings.gradle的内容，最关键的内容就是告诉Gradle这个multiprojects包含哪些子projects:

```groovy
include 'CPosSystemSdk','CPosDeviceSdk','CPosSdkDemo','CPosDeviceServerApk','CPosSystemSdkxxxPosImpl'
```

> 强烈建议：
>
> 如果你确实只有一个Project需要编译，我也建议你在目录下添加一个settings.gradle。我们团队内部的所有单个Project都已经改成支持Multiple-Project Build了。改得方法就是添加settings.gradle，然后include对应的project名字。

另外，settings.gradle除了可以include外，还可以设置一些函数。这些函数会在gradle构建整个工程任务的时候执行，所以，可以在settings做一些初始化的工作。比如：我的settings.gradle的内容：

```groovy
//定义一个名为initMinshengGradleEnvironment的函数。该函数内部完成一些初始化操作
//比如创建特定的目录，设置特定的参数等
def initMinshengGradleEnvironment(){
    println"initialize Minsheng Gradle Environment ....."
    ......//干一些special的私活....
    println"initialize Minsheng Gradle Environment completes..."
}
//settings.gradle加载的时候，会执行initMinshengGradleEnvironment
initMinshengGradleEnvironment()
//include也是一个函数：
include 'CPosSystemSdk','CPosDeviceSdk','CPosSdkDemo','CPosDeviceServerApk','CPosSystemSdkxxxPosImpl'

```

#### 4.2.2 Gradle命令介绍

1. gradle projects 查看工程信息

   到目前为止，我们了解Gradle什么呢？

   - 每一个Project都必须设置一个build.gradle文件。至于其内容，我们留到后面再说。
   - 对于multi-projects build，需要在根目录下也放一个build.gradle，和一个settings.gradle。
   - 一个Project是由若干taskes来组成的，当`gradle xxx`的时候，实际上是要求gradle执行xxx任务。这个任务就能完成具体的工作。
   - 当然，具体的工作和不同的插件有关系。编译Java要使用Java插件，编译Android App需要使用Android App插件。这些我们都留待后续讨论

   Gradle 提供一些方便命令来查看和Project/Task相关的信息。比如在posdevice中，我想看这个multi-projects到底包含多少个子Project：

   执行`gradle projects`，得到下图：

   ![img](https://img-blog.csdn.net/20150905194216394?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   你看，multi-projects的情况下，posdevice这个目录对应的build.gradle叫Root Project，它包含5个子Project。

   如果你修改settings.gradle，使得include只有一个参数，则gradle projects的子project也会变少，如下图：

   ![img](https://img-blog.csdn.net/20150905194231875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

2. gradle tasks 查看任务信息

   查看了Project信息，这个还比较简单，直接看settings.gradle也知道。那么Project包含哪些Task信息，怎么看呢？上面两张图最后的输出也告诉你了，想看某个Project包含哪些Task信息，只要执行：`gradle project-path:tasks`就行。注意，`project-path`是目录名，后面必须跟冒号。

   对于 Multi-Project，在根目录中，需要指定你想看哪个project的任务。不过你要是已经cd到某个Project的目录了，则不需要指定Project-path。

   来看下图：

   ![img](https://img-blog.csdn.net/20150905194251964?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   上图是 `gradle CPosSystemSdk:tasks`的结果。

   > cd CPosSystemSdk
   >
   > gradle tasks 得到同样的结果

   CPosSystemSdk是一个Android Library工程，Android Library对应的插件定义了好多Task。每种插件定义的Task都不尽相同，这就是所谓的Domain Specific，需要我们对相关领域有比较多的了解。

   这些都是后话，我们以后会详细介绍。

3. gradle task-name 执行任务

   上图中列出了好多任务，这时候就可以通过 gradle 任务名来执行某个任务。这和make xxx很像。比如：

   - gradle clean是执行清理任务，和make clean类似。
   - gradle properites用来查看所有属性信息。

   `gradle tasks`会列出每个任务的描述，通过描述，我们大概能知道这些任务是干什么的，然后gradle task-name 执行它就好。

   这里要强调一点：task和task之间往往是有关系的，这就是所谓的**依赖关系**。比如，assemble task就依赖其他task先执行，assemble才能完成最终的输出。

   依赖关系对我们使用gradle有什么意义呢？

   如果知道Task之间的依赖关系，那么开发者就可以添加一些定制化的Task。比如我为assemble添加一个SpecialTest任务，并指定assemble依赖于SpecialTest。当assemble执行的时候，就会先处理完它依赖的task。自然，SpecialTest就会得到执行了...

   大家先了解这么多，等后面介绍如何写gradle脚本的时候，这就是调用几个函数的事情，Nothing Special!

### 4.3 Gradle的工作流

Gradle的工作流程其实蛮简单，用一个图来表达：

![img](https://img-blog.csdn.net/20150905194317170?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

上图告诉我们，Gradle工作包含三个阶段：

- 首先是初始化阶段。对我们前面的multi-project build而言，就是执行settings.gradle
- Initiliazation phase的下一个阶段是Configration阶段。
- Configration阶段的目标是解析每个project中的build.gradle。比如multi-project build例子中，解析每个子目录中的build.gradle。在这两个阶段之间，我们可以加一些定制化的Hook。这当然是通过API来添加的。
- Configuration阶段完了后，整个build的project以及内部的Task关系就确定了。嗯？前面说过，一个Project包含很多Task，每个Task之间有依赖关系。Configuration会建立一个**有向图**来描述Task之间的依赖关系。所以，我们可以添加一个HOOK，即当Task关系图建立好后，执行一些操作。
- 最后一个阶段就是执行任务了。当然，任务执行完后，我们还可以加Hook。

下面展示一下我按图26为posdevice项目添加的Hook，它的执行结果：

![img](https://img-blog.csdn.net/20150905194338602?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

我在：

- settings.gradle加了一个输出。
- 在posdevice的build.gradle加了beforeProject函数。
- 在CPosSystemSdk加了taskGraph whenReady函数和buidFinished函数。

好了，Hook的代码怎么写，估计你很好奇，而且肯定会埋汰，搞毛这么就还没告诉我怎么写Gradle。马上了！

最后，关于Gradle的工作流程，你只要记住：

- Gradle有一个初始化流程，这个时候settings.gradle会执行。
- 在配置阶段，每个Project都会被解析，其内部的任务也会被添加到一个有向图里，用于解决执行过程中的依赖关系。
- 然后才是执行阶段。你在gradle xxx中指定什么任务，gradle就会将这个xxx任务链上的所有任务全部按依赖顺序执行一遍！

下面来告诉你怎么写代码！

### 4.4 Gradle编程模型及API实例详解

希望你在进入此节之前，一定花时间把前面内容看一遍。

*https://docs.gradle.org/current/dsl/* <==这个文档很重要

Gradle基于Groovy，Groovy又基于Java。所以，Gradle执行的时候和Groovy一样，会把脚本转换成Java对象。Gradle主要有三种对象，这三种对象和三种不同的脚本文件对应，在gradle执行的时候，会将脚本转换成对应的对象：

- Gradle对象：当我们执行gradle xxx或者什么的时候，gradle会从默认的配置脚本中构造出一个Gradle对象。在整个执行过程中，只有这么一个对象。Gradle对象的数据类型就是Gradle。我们一般很少去定制这个默认的配置脚本。
- Project对象：每一个build.gradle会转换成一个Project对象。
- Settings对象：显然，每一个settings.gradle都会转换成一个Settings对象。

注意，对于其他gradle文件，除非定义了class，否则会转换成一个实现了Script接口的对象。这一点和3.5节中Groovy的脚本类相似

当我们执行gradle的时候，gradle首先是按顺序解析各个gradle文件。这里边就有所所谓的生命周期的问题，即先解析谁，后解析谁。下图是Gradle文档中对生命周期的介绍：结合上一节的内容，相信大家都能看明白了。现在只需要看红框里的内容：

![img](https://img-blog.csdn.net/20150905194405209?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 4.4.1 Gradle对象

我们先来看Gradle对象，它有哪些属性呢？如下图所示：

![img](https://img-blog.csdn.net/20150905194421970?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

我在posdevice build.gradle中和settings.gradle中分别加了如下输出：

```groovy
//在settings.gradle中，则输出"In settings,gradle id is"
println "In posdevice, gradle id is " +gradle.hashCode()
println "Home Dir:" + gradle.gradleHomeDir
println "User Home Dir:" + gradle.gradleUserHomeDir
println "Parent: " + gradle.parent
```

得到结果如图所示：

![img](https://img-blog.csdn.net/20150905194440047?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 你看，在settings.gradle和posdevice build.gradle中，我们得到的gradle实例对象的hashCode是一样的（都是791279786）。
- HomeDir是我在哪个目录存储的gradle可执行程序。
- User Home Dir：是gradle自己设置的目录，里边存储了一些配置文件，以及编译过程中的缓存文件，生成的类文件，编译中依赖的插件等等。

Gradle的函数接口在文档中也有。

#### 4.4.2 Project对象

每一个`build.gradle`文件都会转换成一个Project对象。在Gradle术语中，Project对象对应的是`BuildScript`。

Project包含若干Tasks。另外，由于Project对应具体的工程，所以需要为Project加载所需要的插件，比如为Java工程加载Java插件。其实，一个Project包含多少Task往往是插件决定的。

所以，在Project中，我们要：

- 加载插件
- 不同插件有不同的行话，即不同的配置。我们要在Project中配置好，这样插件就知道从哪里读取源文件等
- 设置属性

1. 加载插件

   Project的API位于https://docs.gradle.org/current/javadoc/org/gradle/api/Project.html。加载插件时调用它的`apply`函数。`apply`其实是Project实现`PluginAware`接口定义的：

   ![img](https://img-blog.csdn.net/20150905194503216?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   

   来看代码：

   ```groovy
   //apply函数的用法
   //apply 是一个函数，此处调用的是上图中的最后一个apply重载。注意，Groovy支持函数调用的时候通过：
   //参数名1：参数值1，参数名2：参数值2
   //的方式来传递参数
   
   apply plugin:'com.android.library'//如果是编译Library，则加载此插件
   apply plugin:'com.android.application'//如果是编译Android App，则加载此插件
   ```

   除了加载二进制的插件（上面的插件其实都是下载了对应的jar包，这也是通常意义上我们所理解的插件），还可以加载一个gradle文件。为什么要加载gradle文件呢？

   其实这和代码的模块划分有关。一般而言，我会把一些通用的函数放到一个名叫utils.gradle文件里。然后在其他工程的build.gradle来加载这个utils.gradle。这样，通过一些处理，我就可以调用utils.gradle中定义的函数了。

   加载utils.gradle插件的代码如下：

   ```groovy
   //utils.gradle是我封装的一个gradle脚本，里边定义了一些方便函数，比如读取AndroidManifest.xml中的versionName，或者是copy jar包/APK包到指定的目录
   apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"
   ```

   也是使用apply的最后一个重载。那么，apply最后一个函数到底支持哪些参数呢？还是得看下图的API说明：

   ![img](https://img-blog.csdn.net/20150905194525120?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   我这里不遗余力的列出API图片，就是希望大家在写脚本的时候，碰到不会的，一定要去查看API文档！

2. 设置属性

   如果是单个脚本，则不需要考虑属性的跨脚本传播，但是Gradle往往包含不止一个build.gradle文件，比如我设置的utils.gradle，settings.gradle。如何在多个脚本中设置属性呢？

   Gradle提供了一种名为extra property的方法。extra property是额外属性的意思，在第一次定义该属性的时候需要通过ext前缀来标示它是一个额外的属性。定义好之后，后面的存取就不需要ext前缀了。ext属性支持Project和Gradle对象。即Project和Gradle对象都可以设置ext属性

   举个例子：

   我在settings.gradle中想为Gradle对象设置一些外置属性，所以在initMinshengGradleEnvironment函数中

   ```groovy
   def initMinshengGradleEnvironment(){
       //属性从local.properties中读取
       Properties properties = new Properties()
       File propertyFile = new File(rootDir.getAbsolutePath() + "/local.properties")
       properties.load(propertyFile.newDataInputStream())
       //gradle就是Gradle对象。它默认是Settings和Project的成员变量。可直接获取ext前缀，表明操作的是外置属性。
       //api是一个新的属性名。前面说过，只在第一次定义或者设置它的时候需要ext前缀
       gradle.ext.api = properties.getProperty('sdk.api')
       println gradle.api//再次存取api的时候，就不需要ext前缀了
       ......
   }
   ```

   再来一个例子强化一下：

   我在utils.gradle中定义了一些函数，然后想在其他build.gradle中调用这些函数。那该怎么做呢？

   ```groovy
   //utils.gradle
   //utils.gradle中定义了一个获取AndroidManifest.xml versionName 的函数
   def getVersionNameAdvanced(){
       //下面这行代码中的project是谁？
       def xmlFile = project.file("AndroidManifest.xml")
       def rootManifest = new XmlSlurper().parse(xmlFile)
       return rootManifest('@android:versionName')
   }
   //现在，想把这个API输出到各个Project。由于这个utils.gradle会被每一个Project Apply，所以我可以把getVersionNameAdvanced定义成一个closure，然后赋值到一个外部属性
   //下面的ext是谁的ext?
   ext{//此段花括号中代码是闭包
       //除了ext.xxx=value这种定义方法外，还可以使用ext{}这种书写方法。
       //ext{}不是ext(Closure)对应的函数调用。但是ext{}中的{}确实是闭包
       getVersionNameAdvanced = this.&getVersionNameAdvanced
   }
   ```

   上面代码中有两个问题：

   - project是谁？
   - ext是谁的ext?

   上面两个问题比较关键，我也是花了很长时间才搞清楚。这两个问题归结到一起，其实就是：

   加载utils.gradle的Project对象和utils.gradle本身所代表的Script对象到底有什么关系？

   我们在Groovy中也讲过怎么在一个Script中import另外一个Script中定义的类或者函数（见3.5 脚本类、文件I/O和XML操作一节）。在Gradle中，这一块的处理比Groovy要复杂，具体怎么搞我还没完全弄清楚，但是Project和utils.gradle对于的Script的对象的关系是：

   - 当一个Project apply一个gradle文件的时候，这个gradle文件会转换成一个Script对象。这个，相信大家都已经知道了。
   - Script中有一个delegate对象，这个delegate默认是加载（即调用apply）它的Project对象。但是，在apply函数中，有一个from参数，还有一个to参数(参考apply的API)。通过to参数，你可以把delegate对象指定为别的东西。
   - delegate对象是什么意思？当你在Script中操作一些不是Script自己定义的变量，或者函数时候，gradle会到Script的delegate对象去找，看看有没有定义这些变量或函数。

   现在你知道问题1,2和答案了：

   - 问题1：project就是加载utils.gradle的project。由于posdevice有5个project，所以utils.gradle会分别加载到5个project中。所以，getVersionNameAdvanced才不用区分到底是哪个project。反正一个project有一个utils.gradle对应的Script。
   - 问题2：ext：自然就是Project对应的ext了。此处为Project添加了一些closure。那么，在Project中就可以调用getVersionNameAdvanced函数了

   比如：我在posdevice每个build.gradle中都有如下的代码：

   ```groovy
   tasks.getByName("assemble"){
      it.doLast{
          println "$project.name: After assemble, jar libs are copied tolocal repository"
           copyOutput(true)  //copyOutput是utils.gradle输出的closure
        }
   }
   ```

   通过这种方式，我将一些常用的函数放到utils.gradle中，然后为加载它的Project设置ext属性。最后，Project中就可以调用这种赋值函数了！

   注意：此处我研究的还不是很深，而且我个人感觉：

   - 在Java和Groovy中：我们会把常用的函数放到一个辅助类和公共类中，然后在别的地方import并调用它们。

   - 但是在Gradle，更正规的方法是在xxx.gradle中定义插件。然后通过添加Task的方式来完成工作。gradle的user guide有详细介绍如何实现自己的插件。

3. Task介绍

   Task是Gradle中的一种数据类型，它代表了一些要执行或者要干的工作。不同的插件可以添加不同的Task。每一个Task都需要和一个Project关联。

   Task的API文档位于*https://docs.gradle.org/current/dsl/org.gradle.api.Task.html*。关于Task，我这里简单介绍下build.gradle中怎么写它，以及Task中一些常见的类型

   关于Task。来看下面的例子：

   ```groovy
   //Task是和Project关联的，所以，我们要利用Project的task函数来创建一个Task
   task myTask  //myTask是新建Task的名字
   task myTask { configure closure }
   task myType << { task action } //注意，<<符号是doLast的缩写
   task myTask(type: SomeType)
   task myTask(type: SomeType) { configure closure }
   ```

   上述代码中都用了Project的一个函数，名为task，注意：

   - 一个Task包含若干Action。所以，Task有doFirst和doLast两个函数，用于添加需要最先执行的Action和需要和需要最后执行的Action。Action就是一个闭包。
   - Task创建的时候可以指定Type，通过 `type:name` 表达。这是什么意思呢？其实就是告诉Gradle，这个新建的Task对象会从哪个基类Task派生。比如，Gradle本身提供了一些通用的Task，最常见的有Copy 任务。Copy是Gradle中的一个类。当我们：task myTask(type:Copy)的时候，创建的Task就是一个Copy Task。
   - 当我们使用 taskmyTask{ xxx}的时候。花括号是一个closure。这会导致gradle在创建这个Task之后，返回给用户之前，会先执行closure的内容。
   - 当我们使用task myTask << {xxx}的时候，我们创建了一个Task对象，同时把closure做为一个action加到这个Task的action队列中，并且告诉它“最后才执行这个closure”（*注意，<<符号是doLast的代表*）

   下图是Project中关于task函数说明：

   ![img](https://img-blog.csdn.net/20150905194548323?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

   

陆陆续续讲了这么些内容，我自己感觉都有点烦了。是得，Gradle用一整本书来讲都嫌不够呢。

anyway，到目前为止，我介绍的都是一些比较基础的东西，还不是特别多。但是后续例子该涉及到的知识点都有了。下面我们直接上例子。这里有两个例子：

- posdevice的例子
- 单个project的例子

#### 4.4.3 posdevice实例

现在正是开始通过例子来介绍怎么玩gradle。这里要特别强调一点，根据Gradle的哲学。gradle文件中包含一些所谓的Script Block（姑且这么称它）。ScriptBlock作用是让我们来配置相关的信息。不同的SB有不同的需要配置的东西。这也是我最早说的行话。比如，源码对应的SB，就需要我们配置源码在哪个文件夹里。关于SB，我们后面将见识到！

posdevice是一个multi project。下面包含5个Project。对于这种Project，请大家回想下我们该创建哪些文件？

- setting.gradle是必不可少的
- 根目录下的build.gradle。这个我们没讲过，因为posdevice的根目录本身不包含代码，而是包含其他5个子project。
- 每个project目录下包含对于的build.gradle
- 另外，我把常用的函数封装到一个名为utils.gradle的脚本里了。

马上一个一个来看它们。

1. utils.gradle

   utils.gradle是我自己加的，为我们团队特意加了一些常见函数。主要代码如下：

   ```groovy
   import groovy.util.XmlSlurper  //解析XML时候要引入这个groovy的package
    
   def copyFile(String srcFile,dstFile){
        ......//拷贝文件函数，用于将最后的生成物拷贝到指定的目录
   }
   def rmFile(String targetFile){
       .....//删除指定目录中的文件
   }
    
   def cleanOutput(boolean bJar = true){
       ....//clean的时候清理
   }
    
   def copyOutput(boolean bJar = true){
       ....//copyOutput内部会调用copyFile完成一次build的产出物拷贝
   }
    
   def getVersionNameAdvanced(){//老朋友
      defxmlFile = project.file("AndroidManifest.xml")
      defrootManifest = new XmlSlurper().parse(xmlFile)
      returnrootManifest['@android:versionName']  
   }
    
   //对于android library编译，我会disable所有的debug编译任务
   def disableDebugBuild(){
     //project.tasks包含了所有的tasks，下面的findAll是寻找那些名字中带debug的Task。
     //返回值保存到targetTasks容器中
     def targetTasks = project.tasks.findAll{task ->
        task.name.contains("Debug")
     }
     //对满足条件的task，设置它为disable。如此这般，这个Task就不会被执行
    targetTasks.each{
        println"disable debug task  :${it.name}"
       it.setEnabled false
     }
   }
   //将函数设置为extra属性中去，这样，加载utils.gradle的Project就能调用此文件中定义的函数了
   ext{
       copyFile= this.&copyFile
       rmFile =this.&rmFile
      cleanOutput = this.&cleanOutput
      copyOutput = this.&copyOutput
      getVersionNameAdvanced = this.&getVersionNameAdvanced
      disableDebugBuild = this.&disableDebugBuild
   }
   ```

   

2. settings.gradle

3. posdevice build.gradle

4. CPosDeviceSdk build.gradle

5. CPosDeviceServerApk build.gradle

6. 结果展示



#### 4.4.4 单个project的例子

## 5. 总结







