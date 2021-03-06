内部架构原理
==============

TracedLayer的原理就是trace，相对简单，因此我们在这里不展开描述。本节将主要阐述ProgramTranslator基于源代码将动态图代码转化为静态图代码。


转化过程发生在用户开始调用被装饰的函数，转换过程在装饰器中实现。我们将内部涉及的过程分为以下几步：

函数与缓存
------------

动态图转静态图的主体是函数（Function）。对于函数内包含的PaddlePaddle接口，如果是仅计算相关算子代码语句，那么因为PaddlePaddle动态图和静态图接口一致，我们不需要额外转换这些代码为静态图代码。但是对于动态图，此类代码接口是直接运行计算和返回结果，而对于静态图此类代码接口其实是组网。那么如果被转化的函数被调用多次，动态图转静态图后会多次组网添加对应算子，这显然会导致问题。为了解决这个问题以及为了加速动转静转化过程，我们维护了被装饰器装饰的函数（Function）与其输入形状（shape），数据类型（dtype）映射到被转化后组网的Program的缓存（Cache）。当要被转化的函数命中缓存，我们直接用对应存储的Program运行静态图得到结果，否则我们才进行语句转化，并且转化成功后的Program存储进缓存。

动态图源码转AST（抽象语法树）
------------------------------

动态图转静态图的最核心部分类似一个编译器，解析动态图代码语句为AST，再对应AST进行改写，最后反转回成静态图代码。从函数转化为代码字符串可以使用Python的inspect.getsource。从字符串Python提供了自带的 `ast <https://docs.python.org/3/library/ast.html>`_ 库来解析字符串为AST，但是由于Python2，Python3的语法略有不同，为了避免我们需要额外处理这些Python2，Python3的不同情况，我们使用了统一Python2，Python3的开源AST处理 `gast库 <https://github.com/serge-sans-paille/gast>`_ 。这些接口使得函数转化为AST没有本质上的困难。

AST改写和静态图源码转换
-------------------------

这部分为动转静最核心的部分，我们对支持的各种语法进行ast转写。其中最重要的Python控制流，if-else，while，for循环被分别分析转化为PaddlePaddle静态图接口cond，while_loop等接口实现。我们对想转化的每一种主要语法创建一个Transformer（这里的Transformer是Python ast转写的概念，而不是自然语言处理NLP领域的Transformer），每个Transformer扫一遍AST并进行对应的改写。最后被转化完成的AST我们使用gast提供的接口转回成源码。

静态图源码作为动态图一部分运行的技术
--------------------------------------

为了动静转化更加易用和被转化的代码能在动态图中复用，我们在拥有源码后运行生成Program，并将这个Program作为一个大op，包装成动态图的一个op，这样既能把用户的代码转为静态图提速或者保存部署，另一方面如果用户想在Python层使用生成的静态图代码作为动态图的一部分继续训练或者别的动态图运算也是可以直接使用。

易用性与Debug功能在动转静过程的实现
-------------------------------------

正如AST转写类似编译器，而一般编译器都会提供debug断点，报错，输出一些中间代码等功能。我们在进行动转静时，万一用户的动态图代码出错，或者用户想断点调试，或者用户想看看被转化后的静态图代码是否符合其预期，我们也希望能够像编译器一样提供这些易用性功能，使得动转静兼顾性能和部署同时还具有易用性。我们这里将列出这些功能的实现方式

A. 报错对应到动态图代码行。由于被转化后的静态图代码和原动态图代码不同，Python运行出错时会报静态图的错误，因此我们在每一次AST转写时添加AST节点对应的原动态图代码行等信息，在Python报错栈中将静态图的报错转化成对应的动态图源码报错

B. 设置断点功能。我们保留了被转化后代码的中的pdb.set_trace(), 用户可以使用这种方式进行断点调试

C. 查看最后转化的静态图代码。我们输出为一个StaticLayer class，这个StaticLayer可以直接被调用，但是也存储转化后的代码，可以调用StaticLayer.code来获得转化后的代码。

D. 输出中间转化状态代码，甚至不同语法Transformer转化的代码，比如经过for循环转化后代码是什么样的。我们开放接口设定了log level来让用户可以打印中间状态转化的代码。


