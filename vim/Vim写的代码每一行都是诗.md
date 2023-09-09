---

title: Vim写的代码,每一行都是诗
date: 2018-03-10
categories: 

- vim

---

> Vim写的代码，每一行都是诗 

![vim](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/vim.jpg)

# 缘起

从我开始接触编码的第一天起，就一直在寻找一个趁手的编辑器, 从最初的notepad++,ultraedit,sublime，以及后来的phpstrom等一系列的编辑工具，虽然功能丰富，五花八门，我也一度在繁杂的编辑器之中意乱情迷，左顾右盼。但是当我第一次见到vim那丑陋的终端界面，贫乏的UI设计以及陡峭的吓人的学习曲线之后，我心想，就是它了!

# Vi

![vi](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/vi.png)
<br>
**早期的vi界面**

## 作者

<br>
vim其实不是原创软件，而是从vi发展过来的，而vi的作者则另有其人:
> ![bil joy](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/Bill_joy.jpg)
<br>威廉·纳尔逊·乔伊（William Nelson Joy，1954年11月8日－），通称比尔·乔伊（Bill Joy），美国计算机科学家。与Vinod Khosla、Scott McNealy和Andy Bechtolsheim一起创立了太阳微系统公司，并作为首席科学家直到2003年。后来经营自己的风险投资公司HighBAR Ventures，也是知名投资公司Kleiner Perkins的合伙人。
主要成就：乔伊的童年是在密歇根州的乡村长大的，在密歇根大学获得电气工程学士学位之后，于1979年在加州大学伯克利分校获得电气工程与计算机科学硕士学位。学生期间，他开发了BSD操作系统。其他人以BSD为基础发展出了很多现代版本的BSD，最著名的有FreeBSD、OpenBSD和NetBSD，苹果电脑的Mac OS X操作系统也在很大程度上基于BSD。1986年，乔伊因他在BSD操作系统中所做的工作获得了Grace Murray Hopper奖。
除了BSD之外，他引人注目的贡献还包括TCP/IP、vi、NFS和C shell，如今这些软件都已经广泛的使用在Solaris、BSD、GNU/Linux等操作系统中，而且开放源代码给其他人无偿使用、改进，为自由软件的发展作出了极大的贡献。

<br>
### 了解更多
[Bill Joy In TED](https://www.ted.com/speakers/bill_joy)

## 一些你可能感兴趣的

### 1.为什么是hjkl

为什么是这个键位呢，为什么不是wasd这四个键位呢?
当 Bill Joy 创建 Vi 文本编辑器时，他使用的机器机器是 [ADM-3A](https://en.wikipedia.org/wiki/ADM-3A) 终端机，这机器就是把 HJKL 键作为方向键。自然而然，Bill Joy 也就用了相同的按键了。
![ADM-3A终端机上hjkl的键位](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/hjkl.jpg)

<br>而wasd这样的现在比较经典的键位，首先出现是在1996年的电子游戏雷神之锤,而vi诞生与1976年

![雷神之锤截图](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/leishen.jpg)
<br>
**雷神之锤截图**

### 2.为什么手指抽筋

为什么感觉Ctrl键，Esc键这么远，这么难用？
vi诞生与1976年，他的快捷键肯定不是为了现代电脑键盘布局设计的，所以使用起来肯定十分费劲，我们一起来看看ADM-3A的键盘布局

![ADM-3A全貌](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/adm3a.jpg)
<br>
**ADM-3A全貌**
<br>
![ADM-3A全貌](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/keyboard.jpg)
**ADM-3A终端机的键盘**
<br>
我们可以看到，esc在现在的tab键的位置，而ctrl键在caps lock键的位置上, 这也是unix键盘经典布局，ctrl在如此方便的位置上，这也就难怪vi会有很多的ctrl相关的快捷键了，更不用说另一个相同时期的emacs了.所以不要再抱怨vi每次ctrl组合键简直要把小指头按抽筋了，其实vi根本就不是为现代键位设计的

### 3.vi的衍生版本

由于bill joy是从 Ken Thompson 的[ed](https://en.m.wikipedia.org/wiki/Ed_(text_editor))编辑器基础上开发的,vi是衍生作品，所以除了拥有AT＆T源许可证的人之外，不能分发给其他人,所以直到1987年6月,才出现了实现部分vi功能的STEVIE，而我们现在的vim，则是从STEVIE发展来的

## vi的局限性

* vi此时还是没有buffer概念的，一次只能对一个文件进行编辑,并且功能单一
* 没有vim script的支持，无法语法高亮,自动缩进等
* 不支持插件,且不能跨平台

# vim

![vim页面](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/vimscreen.png)
<br>
**vim的启动页面**

## vim 作者

> ![vim作者](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/vimer.jpg)
> <br>
> Bram Moolenaar（生于1961年，Lisse）是荷兰的电脑程序员，也是开源软件社区的积极成员。他是Vim的作者，是一个在程序员和高级用户中非常流行的文本编辑器。Moolenaar是荷兰Unix用户组NLUUG的成员之一，NLUUG 在其25周年纪念期间向他颁发了一个奖项，因为他创建了Vim和他对开源软件的贡献。
> 截至2012年10月，Moolenaar在谷歌工作，自2006年7月起在苏黎世办事处工作。他还是Vim编辑的维护人员和发布经理。

## 一些你可能感兴趣的

### 1.vim为什么叫vim

vim是Vi IMproved的缩写，Bram的工作是从STEVIE开始的，并且决定当修改代码数量超过STEVIE的最初代码，就将其命名为vim

### 2.vim的发展历程

| 年代          | 版本   | 改动                                               |
|:----------- |:----:| ------------------------------------------------:|
| 1988年       | 1.0  | Bram Moolenaar 基于Stevie 创建了Amiga的Vi模拟，从未公开发布过    |
| 1991年11月2日  | 1.14 | 首次发布于 Fred Fish 适用于amiga电脑                       |
| 1992年       | 1.22 | 迁移到Unix。Vim此时开始与与vi竞争。                           |
| 1993年12月14日 | 2.0  | 这是使用名称Vi IMproved的第一个版本。                         |
| 1994年8月12日  | 3.0  | 支持多个buffer                                       |
| 1996年5月29日  | 4.0  | 图形用户界面                                           |
| 1998年2月19日  | 5.0  | 语法突出显示，基本脚本（用户定义的功能，命令等）                         |
| 1998年4月6日   | 5.1  | 错误修复，各种改进                                        |
| 1998年4月27日  | 5.2  | 长线支持，文件浏览器，对话框，弹出菜单，选择模式，会话文件，用户定义的功能和命令，Tcl接口等。 |
| 1998年8月31日  | 5.3  | 错误修复等                                            |
| 1999年7月25日  | 5.4  | 基本文件加密，各种改进                                      |
| 1999年9月19日  | 5.5  | 错误修复，各种改进                                        |
| 2000年1月16日  | 5.6  | 新的语法文件，错误修复等                                     |
| 2000年6月24日  | 5.7  | 新的语法文件，错误修复等                                     |
| 2001年5月31日  | 5.8  | 新的语法文件，错误修复等                                     |
| 2001年9月26日  | 6.0  | 折叠，插件，多语言等                                       |
| 2002年3月24日  | 6.1  | Bug修复                                            |
| 2003年6月1日   | 6.2  | GTK2，阿拉伯语言支持，尝试命令，次要功能，错误修复                      |
| 2004年6月7日   | 6.3  | 错误修复，翻译更新，标记改进                                   |
| 2005年10月15日 | 6.4  | 修复了错误，更新了Perl，Python和Ruby支持                      |
| 2006年5月7日   | 7.0  | 拼写检查，代码完成，标签页（多个视口/窗口布局），当前行和列高亮，撤消分支等           |
| 2007年5月12日  | 7.1  | 错误修复，新语法和运行时文件等                                  |
| 2008年8月9日   | 7.2  | 脚本中的浮点支持，重构的屏幕绘制代码，错误修复，新的语法文件等。                 |
| 2010年8月15日  | 7.3  | Lua支持，Python3支持，Blowfish加密，持久性撤销/重做              |
| 2013年8月10日  | 7.4  | 一个新的，更快的正则表达式引擎。                                 |
| 2016年9月12日  | 8.0  | 异步I / O支持，作业，lambdas等。                           |

### 3.为什么vim首页会有帮助乌干达儿童？

![乌干达可爱的小男孩](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/wuganda.jpg)
<br>
bram和其他一些热心的人去过基巴莱区的儿童救助中心（KCC），在那儿他们作为志愿者在KCC服务，在此过程中，他们了解了当地儿童的生存状态，知道他们需要帮助，于是回国后他们创立了ICCF Holland来帮助KCC，Bram Moolenaar自然就是创始人之一了。其中，基巴莱区是非洲中部国家乌干达的111个区之一，主要经济是农业，非常贫困。现在应该可以解释，为什么欢迎页面上是“帮助乌干达的可怜儿童”了。

# 资源

* [vimscript学习](http://learnvimscriptthehardway.onefloweroneworld.com/)
* [vim配色](http://vimcolors.com/)
* [vim早期源码](http://ftp.vim.org/ftp/pub/vim/old/)
* [vim无插件编程](https://nber1994.github.io/2017/12/13/%E6%97%A0%E6%8F%92%E4%BB%B6vim%E7%BC%96%E7%A8%8B%E6%8A%80%E5%B7%A7.html)

# 未完待续
