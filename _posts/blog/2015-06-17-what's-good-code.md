---
layout:     post
title: 什么样的代码才是好的代码
category: blog
description: xx
---

## 什么样的代码才是好的代码

*文章来源*：http://engineering.intenthq.com/2015/03/what-is-good-code-a-scientific-definition/

文章的统计结果来自于面试过程中对65个开发者的调查，这些开发者大多数使用java或者scala语言，同时多数具有5年以上的开发经验。

*  TOP 5
  + **Readable**（可读性）
  + **Tested/Testable**（可测试性）
  + **Simple**（简单）
  + **Works**（功能良好）
  + **Maintainable**（可维护性）

 > “Good code is written so that is <font size="5" color="red">readable</font>, <font size="5" color="green">understandable</font>, covered by <font size="5" color="blue">automated tests</font>, <font size="5" color="orange">not over complicated</font> and <font size="5" color="purple">does well</font> what is intended to do.”
    
	**Readable**应该体现在包，类，方法以及变量的命名上，同时方法调用逻辑，从上到下应该是可读的。写代码就像写故事一样，随着调用链的进行，应该是故事一步一步向前推进的过程。

* 其它
避免过早优化，[Knuth, Donald](http://en.wikipedia.org/wiki/Donald_Knuth)在40年前就这样说到： 
	> “We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil”

然而提到要**高效代码**的比提到**避免过早优化**的人要多出一倍。

 同时文中还提到了，self-document和comment，一部分人认为应该代码应该具有自文档性，不应该添加注释（**事实上，没有及时更新或者错误的注释比没有注释还让人烦恼**）; 而另一部分人觉得应该有添加comment，需要让别人知道你在干什么，就是java doc API一样。

 代码的自文档性能够提高代码的可读性，同时一定的注释也是必要的。对于那些有特殊默认值的方法，是需要使用注释来解释的，以避免使用者误用。并非所有的功能实现都能简单化，复杂的功能实现应该要有注释说明，甚至表明所使用到的相关理论以及文档，以供后来人参考。
  
*  统计数据
  下面是相应的统计信息：
  ![好代码的特征](https://github.com/hongbing/hongbing.github.io/tree/master/_site/images/goodcode_statistics.png)

* 测试标准
代码好坏的测试标准：
```
WTFs / minute
```  

![代码好坏的测试标准](https://github.com/hongbing/hongbing.github.io/tree/master/_site/images/standofgoodcode.png)

* 方法
 
如何写出好代码的方法：

![GoodCode](https://github.com/hongbing/hongbing.github.io/tree/master/_site/images/good_code.png)
