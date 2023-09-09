---
title: vim无插件编程技巧
date: 2017-12-13
categories: 
- vim
---

> 一般刚接触vim的时候，在熟悉了他的基本操作之后，一般都急于将自己的vim安装各种插件，已达到IDE的效果，个人认为，这些插件阻碍了vimer对vim的进一步学习，我也是这样，从熟悉了常用的vim操作之后，就迫不及待的将自己的vim堆砌插件改造成IDE，然后陷入无尽的关于vim与IDE那个好的无尽的争吵中,哈哈。但是过多的插件，反而会让你看不见vim真正的强大。突然有一天，我试着把vim的插件全部删除，探究如果没有插件，vim的使用是怎样的，后来发现，vim其实完全可以实现无插件编程，而且当你熟悉了这些操作之后，你会进一步领会vim的精妙。

# 浏览代码

```shell
:E

" =================================================================
" Netrw Directory Listing                                        (n
"   /Users/jingtianyou
"   Sorted by      name
"   Sort sequence: [\/]$,\<core\%(\.\d\+\)\=\>,\.h$,\.c$,\.cpp$,\~\
"   Quick Help: <F1>:help  -:go up dir  D:delete  R:rename  s:sort-
" =================================================================
../
./
.ShadowsocksX/
.Trash/
.bundle/
.cache/
.cheat/
.cisco/
.config/
.cups/
.gem/
.go/
.local/
.netease-musicbox/
.nvm/
.oh-my-zsh/
 [netrw] format: unix; [1,3]
 :E
```

> 不用使用类似与Nerdtree插件，vim原生即可实现改功能

这个界面中，你可以用 j, k 键上下移动，然后回车，进入一个目录，或是找开一个文件。你可以看到上面有一堆命令：

* 【–】 到上级目录
* 【D】删除文件（大写）
* 【R】改文件名（大写）
* 【s】对文件排序（小写）
* 【x】执行文件
  当然，打开的文件会把现有已打开的文件给冲掉——也就是说你只看到了一个文件。

## 补充命令

如果你要改变当前浏览的目录，或是查看当前浏览的目录，你可以使用和shell一样的命令：

```shell
:cd <dir> – 改变当前目录

:pwd  - 查看当前目录
```

# 缓冲区

其实，你用:E 浏览打开的文件都没有被关闭，这些文件都在缓冲区中。你可以用下面的命令来查看缓冲区：

```shell
:ls

 6 "   Sort sequence: [\/]$,\<core\%(\.\d\+\)\=\>,\.h$,\.c$,\.cpp$
  5 "   Quick Help: <F1>:help  -:go up dir  D:delete  R:rename  s:s
  4 " =============================================================
  3 ../
  2 ./
  1 applogs/
  0 htdocs/
  1 htdocs1/
  2 htdocs_dev/
  3 logs/
  4 molten/
  5 privdata/
  6 molten.log*
  7 molten.logtracing-20170912.log
  8 moltentracing-20170912.log
  9 test.php
 10 tracing-20170912.log
~
~
www [netrw] format: unix; [1,11]
:ls
  3      "contract-jingjirenpush/test.php" line 1
  5 %a   "contract-jingjirenpush/composer.json" line 1
  7      "contract-jingjirenpush/readme.txt" line 1
Press ENTER or type command to continue
```

你可以看到编号分别为3，5，7

## 打开某个文件

```shell
:buffer 5
//简单写法
:b5
```

## Tips:

* 你可以像在Shell中输入命令按Tab键补全一样补全Vim的命令
* 也可以用像gdb一样用最前面的几个字符，只要没有冲突。如：buff 
* 上图中，我们还可以看到5有一个%a，这表示当前文件，相关的标记如下：

```shell
- （非活动的缓冲区）
a （当前被激活缓冲区）
h （隐藏的缓冲区）
% （当前的缓冲区）
# （交换缓冲区）
= （只读缓冲区）
+ （已经更改的缓冲区）
```

## 补充命令

```shell
:bnext       缩写 :bn
:bprevious   缩写 :bp
:blast  缩写 :bl
:bfirst 缩写 :bf
```

# 窗口分屏浏览

其实，我更多的不是用拆分窗口的命令，而是用浏览文件的命令来分隔窗口。如：

```shell
:Ve
~/.o/l/completion.zsh  2 ~/                                                                                X
0 " ================================================|  7 " ================================================
1 " Netrw Directory Listing                         |  6 " Netrw Directory Listing
2 "   /home/jty                                     |  5 "   /home/jty
3 "   Sorted by      name                           |  4 "   Sorted by      name
4 "   Sort sequence: [\/]$,\<core\%(\.\d\+\)\=\>,\.h|  3 "   Sort sequence: [\/]$,\<core\%(\.\d\+\)\=\>,\.h
5 "   Quick Help: <F1>:help  -:go up dir  D:delete  |  2 "   Quick Help: <F1>:help  -:go up dir  D:delete
6 " ================================================|  1 " ================================================
7 ../                                               |  0 ../
8 ./                                                |  1 ./
9 .cache/                                           |  2 .cache/
10 .composer/                                       |  3 .composer/
11 .config/                                         |  4 .config/
12 .elinks/                                         |  5 .elinks/
13 .irssi/                                          |  6 .irssi/
14 .oh-my-zsh/                                      |  7 .oh-my-zsh/
15 .pki/                                            |  8 .pki/
16 .ssh/                                            |  9 .ssh/
17 .tmux/                                           | 10 .tmux/
18 .vim/                                            | 11 .vim/
19 .w3m/                                            | 12 .w3m/
20 .weechat/                                        | 13 .weechat/
21 .zsh/                                            | 14 .zsh/
22 LuaJIT-2.0.2/                                    | 15 LuaJIT-2.0.2/
23 Mail/                                            | 16 Mail/
24 Python-2.7.3/                                    | 17 Python-2.7.3/
25 Python-2.7.8/                                    | 18 Python-2.7.8/
[netrw] format: unix; [1,1]                            [netrw] format: unix; [1,8]

:Ve
```

## 相关命令

```shell
把当前窗口上下分屏，并在下面进行目录浏览：

:He   全称为 :Hexplore  （在下边分屏浏览目录）

如果你要在上面，你就在 :He后面加个 !，

:He!  （在上分屏浏览目录）

如果你要左右分屏的话，你可以这样：

:Ve 全称为 :Vexplore （在左边分屏间浏览目录，要在右边则是 :Ve!）

下图是分别用:He 和 :Ve搞出来的同时看三个文件：

~/.o/l/completion.zsh  3 ~/                                                                                X
5 "Bundle 'luochen1990/rainbow'                       |  8 Bundle 'vim-scripts/greenvision'
4 "let g:rainbow_active = 1                           |  7 Bundle 'morhetz/gruvbox'
3 Bundle 'bruth/vim-newsprint-theme'                  |  6 Bundle 'kudabux/vim-srcery-drk'
2                                                     |  5 "Bundle 'luochen1990/rainbow'
1 Bundle 'justinmk/vim-sneak'                         |  4 "let g:rainbow_active = 1
238 Bundle 'endel/vim-github-colorscheme'             |  3 Bundle 'bruth/vim-newsprint-theme'
1 Bundle 'nelstrom/vim-mac-classic-theme'             |  2
2 Bundle 'gkjgh/cobalt'                               |  1 Bundle 'justinmk/vim-sneak'
3 Bundle 'aunsira/macvim-light'                       |238 Bundle 'endel/vim-github-colorscheme'
4 Bundle 'vim-scripts/CSApprox'                       |  1 Bundle 'nelstrom/vim-mac-classic-theme'
5                                                     |  2 Bundle 'gkjgh/cobalt'
6                                                     |  3 Bundle 'aunsira/macvim-light'
vimrc [vim] format: unix; [1,238]                     |  4 Bundle 'vim-scripts/CSApprox'
12 " ================================================ |  5
11 ../                                                |  6
10 ./                                                 |  7
9 .cache/                                             |  8 Bundle 'jlanzarotta/bufexplorer'
8 .composer/                                          |  9 " 可以通过以下四种方式指定插件的来源--
7 .config/                                            | 10 " a)指定Github中vim-scripts仓库中的插件，直接指定>
6 .elinks/                                            |    插件名称即可，插件明中的空格使用“-”代替。--
5 .irssi/                                             | 11
4 .oh-my-zsh/                                         | 12 " b) 指定Github中其他用户仓库的插件，使用“用户名/>
3 .pki/                                               |    插件名称”的方式指定--
2 .ssh/                                               | 13
1 .tmux/                                              | 14 " c) 指定非Github的Git仓库的插件，需要使用git地址-
0 .vim/                                               |    -
                                               [netrw] format: unix; [1,19]                          vimrc [vim] format: unix; [1,238]
```

## 补充命令

```shell
 切换屏幕焦点
先按Ctrl + W，然后按方向键：h j k l

 改变屏幕位置
先按Ctrl + W，然后按方向键：H J K L(大写)

 调成各个屏幕大小
先按Ctrl + W，然后按: =
```

# 分屏同步移动

要让两个分屏中的文件同步移动，很简单，你需要到需要同步移动的两个屏中都输入如下命令（相当于使用“铁锁连环”）：

```shell
:set scb
```

如果你需要解开，那么就输入下面的命令：

```shell
:set scb!
```

> 注：set scb 是 set scrollbind 的简写。

# Tab页浏览目录

分屏可能会让你不爽，你可能更喜欢像Chrome这样的分页式的浏览，那么你可以用下面的命令：

```shell
:Te  全称是 :Texplorer
```

下图中，你可以看到我用Te命令打开了三页，就在顶端我们可以可以看到有三页，其中第一页Tab上的数字3表示那一页有3个文件。

```shell
|~/.o/l/completion.zsh |  ~/.o/l/clipboard.zsh                                                                X
1   # System clipboard integration
1 #
2 # This file has support for doing system clipboard copy and paste operations
3 # from the command line in a generic cross-platform fashion.
4 #
5 # On OS X and Windows, the main system clipboard or "pasteboard" is used. On other
6 # Unix-like OSes, this considers the X Windows CLIPBOARD selection to be the
7 # "system clipboard", and the X Windows `xclip` command must be installed.
8
9 # clipcopy - Copy data to clipboard
10 #
11 # Usage:
12 #
13 #  <command> | clipcopy    - copies stdin to clipboard
14 #
15 #  clipcopy <file>         - copies a file's contents to clipboard
16 #
17 function clipcopy() {
18   emulate -L zsh
19   local file=$1
20   if [[ $OSTYPE == darwin* ]]; then
21     if [[ -z $file ]]; then
22       pbcopy
23     else
24       cat $file | pbcopy
25     fi
```

## 切换tab

我们要在多个Tabe页中切换，在normal模式下，你可以使用下面三个按键（注意没有冒号）：

```shell
gt   – 到下一个页

gT  - 到前一个页

{i} gt   – i是数字，到指定页，比如：5 gt 就是到第5页

你可以以使用 【:tabm {n}】来切换Tab页。

gvim应该是：Ctrl+PgDn 和 Ctrl+PgUp 来在各个页中切换。

如果你想看看你现在打开的窗口和Tab的情况，你可以使用下面的命令：
```

## tabs列表

```shell
:tabs

于是你可以看到：
2 #
3 # On OS X and Windows, the main system clipboard or "pasteboard" is used. On other
4 # Unix-like OSes, this considers the X Windows CLIPBOARD selection to be the
5 # "system clipboard", and the X Windows `xclip` command must be installed.
6
7 # clipcopy - Copy data to clipboard
8 #
9 # Usage:
10 #
11 #  <command> | clipcopy    - copies stdin to clipboard
12 #
13 #  clipcopy <file>         - copies a file's contents to clipboard
14 #
15 function clipcopy() {
16   emulate -L zsh
17   local file=$1
18   if [[ $OSTYPE == darwin* ]]; then
19     if [[ -z $file ]]; then
20       pbcopy
21     else
22       cat $file | pbcopy
23     fi
clipboard.zsh [zsh] format: unix; [1,3]
:tabs
Tab page 1
~/.oh-my-zsh/lib/completion.zsh
Tab page 2
>   ~/.oh-my-zsh/lib/clipboard.zsh
Press ENTER or type command to continue
```

## 关闭tab

```shell
使用如下命令可以关闭tab：（当然，我更喜欢使用传统的:q, :wq来关闭）

:tabclose [i] – 如果后面指定了数字，那就关闭指定页，如果没有就关闭当前页
```

## Tips

```shell
最后提一下，如果你在Shell命令行下，你可以使用 vim 的 -p 参数来用Tab页的方式打开多个文件，比如：

vim -p cool.cpp shell.cpp haoel.cpp
vim -p *.cpp
```

# 保存会话

如果你用Tab或Window打开了好些文件的文件，还设置了各种滚屏同步，或是行号……，那么，你可以用下面的命令来保存会话：（你有兴趣你可以看看你的 mysession.vim文件内容，也就是一个批处理文件）

```shell
:mksession ~/.mysession.vim
```

如果文件重复，vim默认会报错，如果你想强行写入的话，你可以在mksession后加! ：

```shell
:mksession! ~/.mysession.vim
```

于是下次，你可以这样打开这个会话：

```shell
vim -S ~/.mysession.vim
```

保存完会话后，你也没有必要一个一个Tab/Windows的去Close。你可以简单地使用：

```shell
:qa   – 退出全部 

:wqa  -保存全部并退出全部
```

# 关键字补全

我们还是坚持不用任何插件。我们来看看是怎么个自动补全的。

在insert模式下，我们可以按如下快捷键：

```shell
【Ctrl +N】  - 当你按下这它时，你会发现Vim就开始搜索你这个目录下的代码，搜索完成了就会出现一个下拉列表

【Ctrl + P】 – 接下来你可以按这个键，于是回到原点，然后你可以按上下光标键来选择相应的Word。
```

## 补充的命令

与此类似的，还有更多的补齐，都在Ctrl +X下面：

```shell
Ctrl + X 和 Ctrl + D 宏定义补齐
Ctrl + X 和 Ctrl + ] 是Tag 补齐
Ctrl + X 和 Ctrl + F 是文件名 补齐
Ctrl + X 和 Ctrl + I 也是关键词补齐，但是关键后会有个文件名，告诉你这个关键词在哪个文件中
Ctrl + X 和 Ctrl +V 是表达式补齐
Ctrl + X 和 Ctrl +L 这可以对整个行补齐，变态吧。
```

# 其它技巧

## 字符相关

```shell
【guu 】 – 把一行的文字变成全小写。或是【Vu】

【gUU】 – 把一行的文件变成全大写。或是【VU】

按【v】键进入选择模式，然后移动光标选择你要的文本，按【u】转小写，按【U】转大写

【ga】 –  查看光标处字符的ascii码

【g8】 – 查看光标处字符的utf-8编码

【gf】  - 打开光标处所指的文件 （这个命令在打到#include头文件时挺好用的，当然，仅限于有路径的）

【*】或【#】在当前文件中搜索当前光标的单词
```

## 缩进相关

```shell
【>>】向右给它进当前行 【<<】向左缩进当前行

【=】  - 缩进当前行 （和上面不一样的是，它会对齐缩进）

【=%】 – 把光标位置移到语句块的括号上，然后按=%，缩进整个语句块（%是括号匹配）

【G=gg】 或是 【gg=G】  - 缩进整个文件（G是到文件结尾，gg是到文件开头）
```

## 复制粘贴相关

```shell
按【v】 键进入选择模式，然后按h,j,k,l移动光标，选择文本，然后按 【y】 进行复制，按 【p】 进行粘贴。

【dd】剪切一行（前面加个数字可以剪切n行），【p】粘贴

【yy】复制一行（前面加个数字可以复制n行），【p】粘贴
```

## 光标移动相关

```shell
【Ctrl + O】向后回退你的光标移动

【Ctrl + I 】向前追赶你的光标移动
```

这两个快捷键很有用，可以在Tab页和Windows中向前和向后trace你的光标键，这也方便你跳转光标。

读取Shell命令相关
【:r!date】 插入日期

上面这个命令，:r 是:read的缩写，!是表明要运行一个shell命令，意思是我要把shell命令的输出读到vim里来。

```

```
