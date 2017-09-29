### RID: Finding Reference Count Bugs with Inconsistent Path Pair Checking

### 1.概述

​          本文的核心思想是通过找到不一致路径来找到与引用计数相关的bug。文章逻辑就是，先阐述refcount这个概念，并提出开发者需要自己确保refcount的正确性，很容易出bug；然后提出不一致路径对的概念，并说明为什么不一致路径对会导致refcount的bug；让后提出基于summary的程序间分析可以找到不一致路径对，先举了一个例子说明怎么找IPP，在说明了一般情况下寻找IPP的方法；然后讲了RID的一些实现细节和限制；最后说明了如何进行性能测试和测试的结果。



### 2.论文详解

#### 2.1 refcount

​        引用计数是封装在structure中的整数，用于跟踪对结构的引用数量。通过分析，**refcount 的api主要有以下四个特征：**

1. 一次计数大多数一次递增或减少1个
2. 引用计数的确切值很少被访问。 特别地，在任何分支条件下几乎不使用确切的值。
3. 在任何系统状态，refcount都可能是0 
4. refcount不能是负数。

​        **对于这四条特性是这样处理的**：我们只考虑一般情况下的refcount的使用，也就是满足1、2的情况，然后违反3、4条的就属于bug。



#### 2.2 IPP

​      举了一个pm-count的例子说明什么是不一致路径对。如果不知道pm-count的确切值，我们很难知道究竟是哪个路径被执行。**不一致路径对，需要满足以下4个特点：**

1. 两条路径都在同一个函数，
2. 两条路径都是从函数的入口开始到出口结束
3. 两条路径对refcount的改变不一致
4. 有可能出现这样的情况：给定相同的参数，两个路径都是可行的，并且返回值是一样的。

​       在文章的3.2节中举了例子说明为什么IPP会导致refcount bugs。



#### 2.3 一个具体的检测IPP的例子

​     为了检测程序中的IPP，我们需要比较函数中的两个路径，并确定IPP定义中的第三和第四个条件是否可以满足。定义了**function summary** 来记录不同约束条件下的引用计数更改和返回值，在程序中检测IPP是一个基于summary的程序间分析。summary是由一条条**entry**构成的,entry 主要包括三部分的内容：用change字段表示refcount的改变，用return字段表示返回值，用constraints字段表示参数和返回值。关于entry的例子可以看论文中**Figure 2**的下部分。

​      分析主要分为三步：枚举路径；独立的计算每一条路径的summary；检查每条路径之间的不一致，然后总结出这个函数的summary。

​      以下分析需要参考**Figure 1** 和**Figure 2**

​      **第一步，枚举路径**： foo（）包括两条路径，p1和p2,这个从代码中一眼就能看出来，然后把每条路径的限制条件（v的取值0）和路径联系起来。

​       **第二步，计算每一条路径的summary**，这里有一个点是，一个路径的summary是需要它调用的函数信息，**也就是说，是它调用的函数和他本身的constraints把它分成了许多entry ** ，根据Figure2 可以看出，路径的entry的cons就是对本身的cons和调用的函数的cons的交集的枚举。看这个例子，p2本身的cons是v<=0和[dev]！= null。它调用的函数有两条entry，entry1的cons是 [dev]！=null，[0]>=0 ，这个[0]回到path本身其实就是v.所以这条路径的cons就是上述两条cons的交集，即[dev]≠null∧v=0，再加上返回值[0]=0。

​        计算完成之后，需要把局部变量去掉，因为在函数外边他是不会被看到的。比如本例中的v

​        **第三步，检查合并路径的summary** 算出路径的summary就能找到IPP了。但是还要给出一个function的summary（我认为是为了给调用它的函数看），如果有不一致路径对，就randomly的取一个，这是为了避免这个function的不一致在调用它的函数中再次被检测，那样就重复了。



#### 2.4 一个普遍的检测IPP的方法

​       还是分为如上三步。Figue4是一个抽象的描述图。 **有一个准备工作是做函数抽象**，保留参数、分支条件和返回值。Figure 是抽象函数的指令的语法，如何抽象是参考了参考文献12和13.

​        给出了entry的一个形式划的定义，是一个三元组：**S = (cons; changes; return)**

​        一个形式化的计算路径summary的方法： 是基于符号执行的技术来计算，**给出了一个state的定义，是一个五元组：St = (ip; cons; changes; return; vmap)** ，每执行一条指令状态会发生变化，当执行一个函数调用的时候，每一条被调用函数的entry被实例化，如果当前状态的cons和entry的cons是可以被满足的，就创建一个新的state，并执行一些操作，具体的看**Algorithm 1** 。     **任何时间遇到return语句，就创建一条entry，用当前state的cons，change和return。**

​          路径检测，简单，不详细描述了。。。。。



### 3. 一些实现细节和限制

1. RID要求一些预定义的函数摘要，这样就不用看函数的body了。**假定摘要已经有了？？？？？？？？？？**
2. 在分析大规模程序时如何减少工作量，简化工作，两个方法，一是把函数分为3类，第一类直接产生refcount change，第二类 返回值之类的会导致change，第三类，和change无关，然后针对不同类分析的不一样。第二个方法是限制枚举的路径和每条路径产生的summary。
3. 分析前的一些简化操作，比如一些静态函数在许多头部都有定义，会被分析很多次，就使用weak symbols 表示。
4. 限制：由于RID在做函数抽象的过程中会丢失函数指针、位操作符等信息，会导致丢失部分路径 ；RID中循环只展开一次；RID限制了路径的数量，会导致丢失路径。



### 4. 评估

- 4.1.测试用例：

​      For the Linux kernel, we use the 3.17 release and concentrate on the DPM (Dynamic Power Management)  subsystem

​      We choose three  Python/C programs, namely krbV, pyaudio and ldap, and compares the reports of RID with warnings given by cpychecker [17] (the source code of Pungi[13] is not yet available).

- 4.2 对检测出的bug的分析

​       主要是两部分，一个是开发者对api的 错误理解，一个是不能正确的处理异

- 4.3 对一些api 高比例的错误使用

  This clearly shows that, even in relative mature code bases, how an API is commonly used does not necessarily indicates how the API should be used

- 4.4 误报和没有检测出来的bug

   The major source of false positives from RID is operations not covered by RID’s program abstraction

​         missed bug 举了一个例子。

- 4.5 performance

    Table 1 shows the number of functions in different categories

- 通过检测python程序，和别的工具对比，证明自己的工具好

  Table 2 lists the number of common bugs found by Cpychecker and RID, along with number of bugs detected by only one of the two tools

