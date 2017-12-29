使用vld查看OPCode 2016-02-04
前几天翻译了一篇关于Zend引擎的文章，这篇文章主要是讲Zend引擎怎么执行PHP代码的，确切地说是怎么执行OPCode的，因为PHP程序会先被编译为中间形式（OPCode），然后才会在引擎上执行。如果要了解Zend引擎怎么执行OPCode，那么就必须得先知道编译器会把PHP代码编译成什么样的OPCode。当然如果你还不知道什么是OPCode，那你可以看下鸟哥的这篇文章，或者 Sara Golemon的这篇文章（需要翻墙才能看，貌似鸟哥的文章是源于Sara的文章），另外我强烈建议你在阅读这篇文章之前先读完我翻译的那篇Zend执行引擎的文章，这篇文章不只有介绍了关于OPCode的东西，还有跟OPCode是怎么执行相关的东西，而且这篇文章中的内容比Sara的那篇文章要更新一些，毕竟那篇文章是在08年写的。有必要吐槽一下PHP的文档工作，像这些跟PHP底层相关的东西基本上没有官方文档，除了这个文档列出了第二版zend引擎支持的所有OPCode外，我在PHP官网上基本没有找到任何跟OPCode相关的东西。不过好消息是有一些工具可以帮助你查看OPCode，其中最有名的就数Derick Rethans开发的vld扩展了。这篇文章就是介绍怎么使用vld来查看OPCode的。

对于如何安装vld扩展在此我就不介绍了，网上有太多这种文章了，另外这篇文章的所有示例都是在PHP5.6上执行的，vld扩展的版本我没办法查到，不过这个扩展的更新也不频繁，版本相差一点输出的东西差别也不大。

一个示例
我们先看一个简单的示例：

<?php
echo “hello world”;
假设上面的代码被保存在test1.php这个文件中，我们在当前目录下运行下面的命令：

php -dvld.active=1  test1.php
这个命令的输出为：

Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/gongyong1/tmp/php/vld/test1.php
function name:  (null)
number of ops:  3
compiled vars:  none
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_STMT
         1        ECHO                                                     'Hello+world'
   4     2      > RETURN                                                   1
branch: #  0; line:     3-    4; sop:     0; eop:     2; out1:  -2
path #1: 0,
Hello world
首先我们从上面的输出的内容的最后一行看到输出了"hello world"这个字符串，这就是test1.php中echo出来的，如果你在命令行参数中加上-dvld.execute=0这个参数就不会输出这个字符串了，它表示不用执行你的PHP脚本（确切地说是不执行中间代码，也就是OPCode），vld.execute为1表示执行，为0表示不执行，默认为1，这里简单说下-dvld.execute=0这个参数是什么意思，-d是PHP所接受的选项参数，你可以在命令中执行php -h查看这个选项的帮助信息：

○ → php -h
Usage: php [options] [-f] <file> [--] [args...]
  ...
  -a               Run interactively
  -c <path>|<file> Look for php.ini file in this directory
  -n               No php.ini file will be used
  -d foo[=bar]     Define INI entry foo with value 'bar'
  -e               Generate extended information for debugger/profiler
  -f <file>        Parse and execute <file>.
  -h               This help
...
它表示在添加一个PHP ini的配置项，这个配置项的名称是vld.execute，值是0，或者是1，这表示一个bool值，0是关闭，非0则是开启，使用这个选项就如同在php.ini文件中添加了一行vld.execute=0|1的配置项。在命令行中执行PHP脚本时使用-d选项设置的配置项只在这次脚本执行的过程中有效。另外还说一点，如果你不想在php.ini中开启vld这个扩展，你也可以在命令行中添加-dextension=vld.so这个参数（linux中），它表示开启vld扩展，当然也只是在这次脚本执行中生效。

对于vld有哪些配置参数，以及每个参数有何含义我们等会再介绍，我们现在回到上面的输出。

我相信当你第一次使用vld来查看OPCode时，首先映入你的视线的肯定是那个表格，我们把它提取出来：

line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_STMT
         1        ECHO                                                     'Hello+world'
   4     2      > RETURN                                                   1
如果你是这样的话，那我不得不对你的眼力表示佩服，这里列出的就是OPCode，也是我们使用vld查看的东西。你需要的关注是line、#*、op、return、operands这几列，其中op这一列列出的就是“真正”的OPCode，你可能会有个问题，为什么不用关注其他几列呢？这个问题我思考了良久，而且在网上也搜索了半天也没完全搞清楚其他几列是什么意思，除了ext，它是extended_value的缩写，它表示OPCode执行后的扩展值，例如算术运行发生溢出（没测试过），vld也没有文档，Derick Rethans也没有写文章介绍，我看到的所有有谈到使用vld查看OPCode的文章都没有说明它们的含义，所以对于这一点我只能说我也不清楚，既然大家都没提到，相信它们也不重要吧。我们还是把注意力集中在line、#*、op、return、operands这些列上吧。

line ：PHP程序中代码的行号，我们的代码只有一行，那就是echo “hello world”，这行代码在第3行，所以OPCode是从第3行开始，此时你也许会问既然只有一行，最后为什么还会出现第4行呢？好眼力！等会再说
#* ：这一行表示OPCode的序号，实际上PHP编译器是把PHP编译为一个OPArray，这个OPArray中就包含了所有的OPCode，Zend引擎就是从这个数组中取出OPCode，一个接一个地执行的，所以这个序号代表了OPCode的执行顺序，更进一步可以说是PHP程序的执行顺序。需要指明的一点是，编译器并不是把PHP程序中的一行转换为1个OPCode，有时候1行PHP代码可能会被编译为多个OPCode，如果你稍微了解一些编译器的基本原理你就可以理解这一点
op ：就是OPCode，整个输出最有价值的东西，基本上也是我们使用vld时最需要关注的地方（当然return和operands也是很有价值的）
return ： OPCode执行后返回的结果，我们的程序中只有一个echo语句，它也没有返回任何东西，所以这一列都为空
operands ：操作数，OPCode执行时会用到的参数，我们看到echo语句会接受一个'Hello+world'参数，这是一个字符串（这个字符串里面为什么有一个'+'？这个问题你应该去问Derick Rethans，或者Zend编译器的开发者）
在这个表格的上面还有一段文字，同样的，我也只提出其中有意义的部分介绍下（实际上也只是我能理解的，哈哈）：

filename：当前执行的脚本的文件路径
function name：当前这个OPArray对应的函数名，这里为null，表示这个OPArray是当前PHP脚本被编译后生成的，等会再详述
number of ops：这个脚本被编译后生成的OPCode的个数，也就是OPArray中包含的OPCode的个数，这里是3个，你数数表格中的OPCode个数
compiled vars：PHP脚本中定义的变量，我们后面介绍OPCode的操作数时会再对它做进一步说明
在这个表格的下面还有两个东西：

branch ： 这个脚本中的分支，确切地说是这个脚本编译成的OPArray中的OPCode的所有分支
path ：这个脚本可能的执行路径，确切地说是这个脚本编译成的OPArray中的OPCode的所有执行路径
ok，基本上就这么些东西，最有价值的就是表格中的东西。

再来一个示例
echo 'Hello world'这个示例太简单了，也看不出什么东西，我们再来看一个稍微复杂点的示例：

1   <?php
2
3   function test($a,$b) {
4       $c = $a + $b;
5       return $c;
6   }
7
8   $i = 1;
9   $j = 2;
10
11  test($i,$j);
为了讲解方便，我把代码的行号标注出来了（如果你要直接拷贝这段代码可能要麻烦你把行号去掉才能运行，sorry！）。这个文件被保存为test2.php，使用下面的命令来查看它的OPCode：

php -dvld.active=1  -dvld.execute=0 test2.php
下面是上面的命令的输出：

Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/gongyong1/tmp/php/vld/test2.php
function name:  (null)
number of ops:  13
compiled vars:  !0 = $i, !1 = $j
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_STMT
         1        NOP
   8     2        EXT_STMT
         3        ASSIGN                                                   !0, 1
   9     4        EXT_STMT
         5        ASSIGN                                                   !1, 2
  11     6        EXT_STMT
         7        EXT_FCALL_BEGIN
         8        SEND_VAR                                                 !0
         9        SEND_VAR                                                 !1
        10        DO_FCALL                                      2          'test'
        11        EXT_FCALL_END
  12    12      > RETURN                                                   1
branch: #  0; line:     3-   12; sop:     0; eop:    12; out1:  -2
path #1: 0,
Function test:
Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/gongyong1/tmp/php/vld/test2.php
function name:  test
number of ops:  10
compiled vars:  !0 = $a, !1 = $b, !2 = $c
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_NOP
         1        RECV                                             !0
         2        RECV                                             !1
   4     3        EXT_STMT
         4        ADD                                              ~0      !0, !1
         5        ASSIGN                                                   !2, ~0
   5     6        EXT_STMT
         7      > RETURN                                                   !2
   6     8*       EXT_STMT
         9*     > RETURN                                                   null
branch: #  0; line:     3-    6; sop:     0; eop:     9
path #1: 0,
End of function test
首先我们可以看到两个表格，第二个表格的第一行文字是：

Function test:
并且第二个表格上面的"function name"的值是test，它表示这个部分是test函数编译出的OPCode，更确切的说是test函数被编译成的OPArray中包含的OPCode。那么第一部分就是除这个test函数外test2.php的其他代码编译生成的OPCode，我们把这段代码叫做PHP脚本（script）。通过这一点你也可以了解到PHP编译器是以什么为单位编译出OPArray的（关于OPArray的定义你可以看我翻译的那篇关于Zend执行引擎的文章），很显然PHP脚本和函数都是基本单位，还有一个基本单位是传递给eval()的以字符串形式表示的PHP代码（这一点需要特别说明下，我测试了过，vld并未把eval中的代码作为一部分单独输出，注意只是没有输出而已，从输出的OPCode来看eval只是有独特的OPCode来处理，所以对eval的调用也只是编译出了几个OPCode而已，当然这并不意味着eval的参数不会被单独编译为一个OPArray，这个工作可能是在运行时进行的，这就是PHP的动态特性的体现）。另外类中的方法也是一种基本单位，不过它们跟函数是一样的，在PHP内部的表示也基本是完全一样的。

我们下面来讲解下第二部分的OPCode，也就是test函数被编译出的OPCode，我先把这个表格单独贴出来：

line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_NOP
         1        RECV                                             !0
         2        RECV                                             !1
   4     3        EXT_STMT
         4        ADD                                              ~0      !0, !1
         5        ASSIGN                                                   !2, ~0
   5     6        EXT_STMT
         7      > RETURN                                                   !2
   6     8*       EXT_STMT
         9*     > RETURN                                                   null
我们首先要注意这个表格上面的一行文字：

compiled vars:  !0 = $a, !1 = $b, !2 = $c
前面已经介绍过compiled vars指的是在PHP程序中定义的变量。test2.php中的test函数有两个参数$a和$b，函数体中有一个变量$c，所以$a、$b和$c都是这个函数中定义的变量，它们会被编译到这个函数对应的OPArray中，并且会作为OPCode执行时的操作数，在OPCode中分别使用!0、!1和!2表示它们，这跟它们出现的顺序有个，实际上这些编译变量会被保存在一个数组中，所以你可以把这里的数字当成对应的变量在数组中的索引，而将"!"作为一个表示是编译变量的前缀说明。另外我们把compiled vars叫做编译变量，有时候也直接用它的缩写CV代指。

此时你心里一定有一个巨大的问号：~0这个又是个什么东西呢？这个问题问得很好，我们下面来慢慢道来。

OPCode的结构和操作数类型
要回答这个问题，我们就必须得先介绍下OPCode的结构：

1. OPArray
PHP编译器会把PHP程序编译为OPArray，它只所以叫做OPArray，就是因为它是OPCode的数组。实际上OPArray是一个C结构体，OPCode的数组是这个结构体中的一个指针字段，这个在那篇关于Zend执行引擎的文章中已经说的很清楚了。编译生成OPArray是有一定规则的，不是把所有代码都编译到一个OPArray中，而是每个函数（方法）都会被编译成一个OPArray，整个PHP脚本也会被编译为一个OPArray，还有就是传给eval的表示PHP代码的字符串也会被单独编译为一个OPArray，其实你会发现编译OPArray的规则是跟作用域有关系的，如果你了解C或者汇编程序的运行方式，你肯定也了解栈帧（stack frame）这个词，它就是函数调用的基本单位，OPArray跟栈帧有些类似。

每个OPArray不仅仅只有OPCode，它还包含编译变量、临时变量、文档注释、try-catch-finally信息、break-continue信息等，还有一个用于缓存临时信息的缓存槽，再提示一次，如果有兴趣进一步了解这些东西的话，请参考Zend执行引擎那篇文章。

2. OPCode结构
Zend的执行引擎一般情况下是从OPArray取出当前要执行的OPCode，执行完后再取下一个OPCode，这样子不断执行下去直到碰到RETURN这个OPCode才会返回退出，在PHP内部OPCode是用一个结构体表示的

struct _zend_op {
    opcode_handler_t handler;   /* opcode执行时会调用的处理函数，一个C函数 */
    znode_op op1; /* 操作数1 */
    znode_op op2; /* 操作数2 */
    znode_op result; /* 结果 */
    ulong extended_value; /* 额外信息 */
    uint lineno;
    zend_uchar opcode; /* opcode代号 */
    zend_uchar op1_type; /* 操作数1的类型 */
    zend_uchar op2_type; /* 操作数2的类型 */
    zend_uchar result_type; /* 结果类型 */
};
从这段代码可以看出，每个OPCode中都会这么几个东西（重要的）：

handler：实际完成OPCode工作的函数
op1：OPCode的操作数1，会传递给handler函数
op2：OPCode的操作数2，会传递给handler函数
result：OPCode执行后的结果
在这篇文章中我不对这几个东西做更详细的说明，在这里只强调一点，每个OPCode都会有一个handler，一个op1，一个op2，以及result，但是很显然op1、op2和result肯定都不是总会用到，而且它们肯定也不会都是编译变量（CV），所以我现在需要进一步了解操作数和结果的类型。

3.操作数类型
PHP的Zend引擎一共支持5种操作数类型（注意它也是result的类型），它们是：

IS_CV ：编译变量（Compiled Variable）：这个操作数类型表示一个PHP变量：以$something形式在PHP脚本中出现的变量，vld输出中以!0、!1形式出现
IS_VAR ： 供Zend引擎内部使用的变量，它可以被其他的OPCode重用，跟$php_variable很像，只是只能供Zend引擎内部使用，vld输出中以$0、$1、$2形式出现
IS_TMP_VAR ： Zend引擎内部使用的变量，但是不能被其他的OPCode重用，vld输出中以~0、~1、~2形式出现
IS_CONST ： 表示一个常量，它们都是只读的，值不可改变，vld中直接以常量值的形式出现
IS_UNUSED ：这个表示操作数没有被使用，我们可以从OPCode的结构体中看到它总是包含op1、op2、result，还有表示它们类型的字段，但是并非每个OPCode都需要用到它们，例如echo $a，显然它只有一个操作数$a，它是一个IS_CV的类型，而且不会返回结果，所以这个OPCode的op2的类型是IS_UNUSED，result的类型也是IS_UNUSED。
vld也可以把op1、op2、result的类型都输出出来，不管它们是有没有被用到，我们只需要在命令行中加上-dvld.verbosity=3，我们看看添加这个参数后test2.php中test函数的生成的OPCode的样子（其他部分不显示）：

line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   EXT_NOP                                           RES[  IS_UNUSED  ]         OP1[  IS_UNUSED  ] OP2[  IS_UNUSED  ]
         1        RECV                                              RES[  IS_CV !0 ]       OP1[  IS_UNUSED  ]
         2        RECV                                              RES[  IS_CV !1 ]       OP1[  IS_UNUSED  ]
   4     3        EXT_STMT                                          RES[  IS_UNUSED  ]         OP1[  IS_UNUSED  ] OP2[  IS_UNUSED  ]
         4        ADD                                               RES[  IS_TMP_VAR ~0 ]       OP1[  IS_CV !0 ] OP2[ ,  IS_CV !1 ]
         5        ASSIGN                                            RES[  ]         OP1[  IS_CV !2 ] OP2[ ,  IS_TMP_VAR ~0 ]
   5     6        EXT_STMT                                          RES[  IS_UNUSED  ]         OP1[  IS_UNUSED  ] OP2[  IS_UNUSED  ]
         7      > RETURN                                                    OP1[  IS_CV !2 ]
   6     8*       EXT_STMT                                          RES[  IS_UNUSED  ]         OP1[  IS_UNUSED  ] OP2[  IS_UNUSED  ]
         9*     > RETURN                                                    OP1[  IS_CONST (6471468) null ]
你可以看到每个OPCode的所有操作数和执行后的result的类型都被输出出来了，我们现在可以轻易的回答上面的那个问题——“~0”是干嘛的？它是一个临时变量，用于保存!0($a)和!1($b)相加后的结果，之后会把这个结果赋值给!2($c)，现在你也许会问为什么执行OPCode时不直接把!0和!1的结果赋值给!2，这是个很2的问题，也是个好问题。

说这个问题很2，是因为我们看过OPCode的结构，它只有op1、op2和result，result只是对op1和op2进行计算后保存临时值的地方，简单点说就是在执行ADD这个OPCode时，我们是不知道ADD后的结果会被赋值给$c，再确切点说是$c = $a + $b被编译成了两个OPCode：ADD和ASSIGN。那么为什么又说这是个好问题呢？因为很显然这个~0只会用于赋值给!2，所以把$a+$b的结果先保存到临时变量中显然有些浪费：时间和内存，所以为什么不直接存放到!2中呢？这个跟编译器优化方面的技术有些关系，我们现在不管它。

RETURN从何而来
在讨论第一个echo "Hello world"的示例时，我们看到这个程序只有一行代码，但是vld输出的OPCode表格中却显示了两行代码，另外一行代码是被编译为一个RETURN的OPCode，如下所示：

line    #* E I O op                            fetch          ext  return  operands
-------------------------------------------------------------------------------------
  4     2      > RETURN                                                    1
另外编译test2.php中的test函数生成的OPCode中的最后两行OPCode为：

line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   6     8*       EXT_STMT
         9*     > RETURN                                                   null
它对应的是第6行的代码，而实际上第6行没有代码（”}”不是代码），而且似乎这行返回的是null。ok，我先不解释这个是怎么来的，我们来看个非常简单的示例，我们写一个空的PHP程序，它只包含PHP开始标志，没有任何其他代码：

<?php

把它保存到文件中，然后使用vld查看下会输出：

line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E > > RETURN                                                   1
看到没，没有任何代码的PHP文件，它被编译后依然会有一个RETURN的OPCode，这是因为这个OPCode会告诉Zend引擎这个OPArray的执行工作可以正常结束了，任何一个OPArray中都必须得有一个RETURN的OPCode，要不然的话Zend引擎不能结束执行。对于PHP脚本，这个RETURN的操作数（op1）是1，IS_CONST类型，这个值会返回给Zend引擎，它表示正常结束。而在test()函数的OPCode中，RETURN的操作数（op1）的值则是null，它也是一个IS_CONST类型，这表示这个函数会返回null。这是很明显的，如果函数没有返回值（没有显式return语句），那么这个函数会返回null，这就是为什么没有return语句的函数会返回null的原因。

此时你可能又会有疑问了，因为test2.php中的test()函数是有return语句的，那么它为什么也会添加一个RETURN的OPCode呢，而且奇怪的是这个OPCode序号的前面会有一个'*'（星号）。好问题，我们下面就来解答这个问题。

#*中的*是什么意思
这个'*'是vld提供的一个牛叉的功能，它的意思是这段代码不会被执行到，或者专业点说叫做不可达的代码，永远都不会被执行。在test()函数中是很明显的，因为代码中有return $c; 这个语句，所以编译器添加的这个return null;肯定永远都不会被执行啊。

至于为什么已经有return语句了，编译器为何还会在最后添加一个return语句呢？这个你只有去问编译器了，不过有些OPCode的缓存工具，例如OPCache在缓存OPCode时会把这些不可达的代码删除，实际这些应该是编译器优化的工作。

分支 & 路径（branch & path）
现在我们已经搞清楚了vld输出的opcode表格中的大部分内容的意思：知道compiled vars是对应于PHP代码中以$variable形式定义的变量，知道可以设置vld.verbosity来输出每个opcode的op1、op2和result的类型信息，也知道OPArray、OPCode和OPCode的操作数类型等这些基础知识，到此本文基本上可以结束了。

如果就这么结束的话，显然是很不责任的。我们还没介绍opcode表格下面的一坨东西：branch（分支）和path（路径）。你可以通过命令行参数-dvld.dump_paths来控制是否输出它们，0表示不输出，1表示输出，默认为1。vld的作者Derick Rethans写过的一篇文章介绍过这两个东西（我能找到的少数几篇介绍它们的文章），下面我就以Derick的那篇文章中示例为例来介绍分支和路径，它的代码如下：

<?php
function test()
{
        for( $i = 0; $i < 10; $i++ )
        {
                if ( $i < 5 )
                {
                        echo "-";
                }
                else
                {
                        echo "+";
                }
        }
        echo "\n";
}
?>
假设这个文件被保存在test3.php中，使用下面的命令查看vld的输出：

php -dvld.active=1 -dvld.execute=0  -dvld.verbosity=0  test3.php
test()函数的opcode为（只显示这个函数的OPCode）：

Function test:
filename:       /Users/gongyong1/tmp/php/vld/test5.php
function name:  test
number of ops:  22
compiled vars:  !0 = $i
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   2     0  E >   EXT_NOP
   4     1        EXT_STMT
         2        ASSIGN                                                   !0, 0
         3    >   IS_SMALLER                                       ~1      !0, 10
         4        EXT_STMT
         5      > JMPZNZ                                        9          ~1, ->18
         6    >   POST_INC                                         ~2      !0
         7        FREE                                                     ~2
         8      > JMP                                                      ->3
   6     9    >   EXT_STMT
        10        IS_SMALLER                                       ~3      !0, 5
   7    11      > JMPZ                                                     ~3, ->15
   8    12    >   EXT_STMT
        13        ECHO                                                     '-'
   9    14      > JMP                                                      ->17
  12    15    >   EXT_STMT
        16        ECHO                                                     '%2B'
  14    17    > > JMP                                                      ->6
  15    18    >   EXT_STMT
        19        ECHO                                                     '%0A'
  16    20        EXT_STMT
        21      > RETURN                                                   null
branch: #  0; line:     2-    4; sop:     0; eop:     2; out1:   3
branch: #  3; line:     4-    4; sop:     3; eop:     5; out1:  18; out2:   9
branch: #  6; line:     4-    4; sop:     6; eop:     8; out1:   3
branch: #  9; line:     6-    7; sop:     9; eop:    11; out1:  12; out2:  15
branch: # 12; line:     8-    9; sop:    12; eop:    14; out1:  17
branch: # 15; line:    12-   14; sop:    15; eop:    16; out1:  17
branch: # 17; line:    14-   14; sop:    17; eop:    17; out1:   6
branch: # 18; line:    15-   16; sop:    18; eop:    21; out1:  -2
path #1: 0, 3, 18,
path #2: 0, 3, 9, 12, 17, 6, 3, 18,
path #3: 0, 3, 9, 15, 17, 6, 3, 18,
End of function test
vld还提供了一个生成可视化分支和路径的方法，你要先运行下面的命令把分支和路径信息保存到一个文件中，这个文件默认存放在/tmp/paths.dot。

php -dvld.dump_paths=1 -dvld.verbosity=0 -dvld.save_paths=1 -dvld.active=1 test2.php
上面的-dvld.save_paths=1这个参数就是控制是否保存分支和路径信息的，1则表示生成，0表示不生成，默认为0。你应该已经注意到了这个文件的扩展名有些奇怪：.dot，你可以使用GraphViz这个工具来生成一个png图片，使用这个工具的命令为（假设你已经安装了这个工具）：

dot -Tpng /tmp/paths.dot > /tmp/paths.png
需要提醒一下，上面给出的GraphViz的网站貌似打不开，而且我在mac上用homebrew也安装不了它（有这个包，就是装不成功，不过如果你使用linux的话可以尝试下安装这个包，windows怎么安装我就不知道了），所以这里我就直接把Derick文章中的图贴过来：



现在整个执行路径可以说是一目了然，现在我结合这个图和vld的输出来介绍branch和path到底是个什么东西，以及怎么阅读vld的输出信息：

1. 首先这个图中有两个框：__main和test，它们分别对应PHP脚本（类似C中的main函数，PHP编译说不定也把它们编译进一个main函数）和test()函数，我们不用管__main

2. test()中的每个框代表一个分支，所以这个函数有这些分支：0，3，6，9，12，15，17，18，数字是OPCode的序号（行号），这里的排列是按照OPCode序号由小到大排的。vld的输出的branch也是由小到大列出的，但从图中来看，这些分支的执行顺序显然不完全是按照OPCode序号从小到大的。

3. 图中每个方框里面还包括了对应的代码行的信息。例如#0这个方框中包含的代码行是2-4，这个意思是第一个分支的OPCode号是0，对应的代码是从第2行到第4行。对应的vld输出就是第一个branch：#0,  line:2-4（ok，这里输出的-和4间的空格有点多）；sop:0是指这个分支起始的OPCode序号，eop:2是分支结束的OPCode序号，sop和eop在图中没有展示。图中的箭头的方法表示分支的流向，我们看到第一个分支只会流向#3的OPCode，对应vld的输出就是out1:3。第二个分支对应的是#3的OPCode，代码位于第4行，sop为3，eop为5，所以这个分支的OPCode是从#3到#5，它可以流向两个分支：#18的OPCode和#9的OPCode

4. 从上面输出的OPCode表格中，你可以看到#5的JMPZNZ、#8的JMP 、#11的JMPZ、#14的JMP和#17的JMP，这些都是跳转指令。如果你熟悉汇编语言的话，这些指令很容易理解。JMP是无条件跳转，也就是执行这个指令后肯定会发生跳转，我们看到第8行OPCode（JMP）的操作数是->3，它表示跳转到第三行OPCode；JMPZNZ和JMPZ是条件跳转，只有在操作数op1满足某种条件才发生跳转，JMPZNZ是一种二元跳转指令（我自己起的名字），也就是满足条件会跳转到某个OPCode，不满足条件则会跳转到另外一个OPCode，我们把这行OPCode提取出来：

line     #* E I O op                           fetch          ext  return  operands
-----------------------------------------------------------------------------------
         3    >   IS_SMALLER                                       ~1      !0, 10
         4        EXT_STMT
         5      > JMPZNZ                                        9          ~1, ->18
首先第三行的IS_SMALLER这个OPCode是比较op1和op2的大小，或者说是判断op1是否小于op2，对应的程序就是i<10。如果i<10，将1保存到临时变量~1；如果i < 10不成立，则0会保存到~1。而JMPZNZ就是判断~1这个临时变量是否为0，如果为0则跳转到#18，反映在代码中就是：i >= 10，循环结束。而#18就保存于于JMPZNZ的操作数op2中的，如果~1不为0：i<10，循环继续，则跳转到#9，这个就是保存在OPCode的extended_value（还记得么）中的，vld输出的OPCode表格中ext列的值；剩下的JMPZ这个指令的含义，你自己去揣摩吧。

5. 相信你现在已经知道怎么阅读vld输出的branch的信息了吧，这里我就不再介绍为什么会有这么些分支了

6. 对于path来说，从图中可以直观地看出这些分支所组成的执行路径，实际上你可以把它看成是一个有向图的所有遍历路径

你现在已经知道了branch和path分别是什么意思了，那么你也可以分析这些branch对应的是什么代码，这样就可以真正的读懂表格中的OPCode了。

豁然开朗
还记得之前OPCode表格中的每一列的含义的介绍么？那个时候我确实还不清楚E/I/O，fetch和ext这几列什么意思，现在除了fetch外，其他几个我都搞明白了，似乎写文章的过程中也可以学到新的东西啊，哈哈。

ext实际上对应的就是OPCode结构体中的extended_value，我们已经在上面的示例中体会到了它的作用，至于还有其他什么情况下会有这个值，那只有碰到再说。

而对于E/I/O，它们是跟分支相关的，E表示这个OPArray的入口，也就是第一个（0号）OPCode，它是Entry Points的缩写；而I表示分支入口（标注'>'的），O自然就是会跳出分支的OPCode了（同样是标注'>'的）。

VLD的参数设置
我们已经介绍了大部分vld的参数了，它们是：vld.active、vld.execute、vld.verbosity、vld.dump_paths和vld.save_paths，你可以通过查看phpinfo()中输出的vld扩展部分的信息来查看vld的参数信息，在我的电脑上的情况是：



你也可以在命令行中使用php --re vld获取vld可设置参数的信息，从上图中你可以发现除了我们介绍的几个参数外，还有其他几个参数，它们分别是：vld.col_sep、vld.format、vld.save_dir、vld.skip_append和vld.skip_prepend，我们现在把所有这些参数的含义都介绍一下：

vld.active ：很显然这个参数用于控制是否激活vld，它是一个bool值，0表示不激活，1表示激活，默认为0，所以这也为何我们会在命令行中都带上这个参数，不过如果你闲麻烦的话，你也在ini中把它设置为总是激活，我测试了下，这么设置后访问web应用都会报504的错误，而使用php-cli则会输出当前执行脚本的OPCode，看来vld只能在命令行环境下使用
vld.execute ： 是否执行脚本，它也是一个bool值，0表示不执行，1表示执行，默认为1
vld.dump_paths：是否输出分支和路径信息，也是一个bool值，0表示不输出，1表示输出，默认为1
vld.save_paths：是否保存路径信息到文件中，注意这个参数含义不是保存路径，它也是一个bool，0表示不保存，1表示保存，默认为0
vld.save_dir：这个才是设置保存路径的参数，默认是/tmp，默认情况下路径信息保存在文件/tmp/paths.dot中，
vld.format：是否以自定义的格式显示，也是一个bool值，为1表示是，0则表示否，默认为0。vld的所谓自定义格式输出实际上就只是可以自定义输出间的间隔，间隔字符是以vld.col_sep指定的
vld.col_sep：指定间隔字符，默认是'\t'
vld.skip_append： 是否跳过php.ini配置文件中auto_append_file指定的文件，也是一个bool值，默认为0，即不跳过包含的文件，显示这些包含的文件中的代码所生成的中间代码。必须在vld.execute设置为0的前提下才能生效
vld.skip_prepend：是否跳过php.ini配置文件中auto_prepend_file指定的文件， 默认为0，即不跳过包含的文件，显示这些包含的文件中的代码所生成的中间代码。必须在vld.execute设置为0的前提下才能生效
vld.verbosity：是否显示更详细的信息，默认为1，其值可以为0,1,2,3 其实比0小的也可以，只是效果和0一样，比如0.1之类，但是负数除外，负数和效果和3的效果一样 比3大的值也是可以的，只是效果和3一样。我们看到在它的值为3的时候，它会输出op1、op2和result的类型信息，不管这个操作数是否被用到如否
总结

如果你用C写过底层系统，你肯定会在用gdb调试系统时查看C程序被编译成的汇编指令，你可以从中看到程序如何使用内存和寄存器，程序控制流的变化，在栈上和堆上存储的数据，以及随着函数调用而变化的栈等等，而且你还可以通过阅读这些汇编指令来分析程序的哪些地方可以进行优化以及如何优化。把C程序编译成汇编指令，就如同把PHP程序编译成OPCode，所以我们同样可以被编译出的OPCode中得到很多价值的信息，例如Julien Pauli的写的很多文章，都需要通过查看OPCode来分析PHP的特性。你也可以通过查看OPCode来判断不同代码性能的优劣。通过查看OPCode，你可以真正做到知其源，也知其所以源。

PHP是解释型语言，所以PHP程序的编译和执行是同时在Zend引擎上执行的，它不像C或者Java会被编译为一个中间文件，我们可以直接分析它们，所以我们必须借助于一些工具来查看编译PHP程序生成的OPCode，vld是其中的佼佼者。学会了怎么使用vld，你就可以随时通过查看OPCode来分析你的PHP代码，接下来我会写几篇这种文章。