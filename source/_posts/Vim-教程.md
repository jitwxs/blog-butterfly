---
title: Vim 教程
tags: Vim
categories: 开发工具
typora-root-url: ..
abbrlink: 8ab971cd
date: 2017-10-31 00:00:17
copyright_author: Jitwxs
---

## 一、Vim 简介

Vim是一个类似于Vi的著名的功能强大、高度可定制的文本编辑器，在Vi的基础上改进和增加了很多特性。代码补全、编译及错误跳转等方便编程的功能特别丰富。

它的最大特色是完全使用键盘命令进行编辑，脱离了鼠标操作虽然使得入门变得困难，但上手之后键盘流的各种巧妙组合操作却能带来极为大幅的效率提升。

## 二、Vim 多文本编辑

### 2.1 多个文件的打开与基本操作

命令：`vim fileName1 fileName2`

切换方法：

|命令|作用|
|-----|:---------|
|:n|切换到下一个文件|
|:N|切换到上一个文件|
|:n fileName2 |切换到文件fileName2|

除了使用上面的切换方法外，可以利用vim为每个文件提供的`buffer号`来进行切换

|命令|作用|
|-----|:---------|
|:ls    |查看所有打开的文件信息|
|:b2  |切换到buffer id为2的buffer|
|:bn   |切换到下一个buffer|
|:bp   |切换到前一个buffer|
|:bd |关闭当前buffer，并关闭对应文件|
|:bd2 |关闭buffer id为2的buffer，并关闭对应文件

### 2.2 窗口分割与基本操作

在打开窗口使就分割窗口：

|命令|作用|
|-----|:---------|
|vim  -o fileName1 fileName2 |水平分割窗口|
|vim  -O fileName1 fileName2 |垂直分割窗口|

在已经打开一个窗口的情况下，打开其他文件并分割窗口：

|命令|作用|
|-----|:---------|
|:e  fileName |不会分割窗口|
|:sp  fileName | 水平分割窗口|
|:vsp  fileName | 垂直分割窗口|

在多个窗口中使用命令 `Ctrl + W + W` 来切换窗口。

![多文件操作](/images/posts/20171030235938161.png)

### 2.3 vim和shell之间切换

**`:shell`**  ： 切换到shell，此时vim在后台运行，在shell中输入命令`exit`，切换回vim

## 三、Vim块操作

### 3.1 三种模式

块操作是利用了vim的可视模式，可视模式简单来说就是选中一块编辑区域，然后在上面执行一些操作，比如`删除`、`替换`、`改变大小写`等。

- 使用 `V` 进入基于字符的可视模式，模式类型为**字符文本**

- 使用 `Ctrl + V` 进入基于列的可视模式，模式类型为**块文本**

- 使用 `Shift + V` 进入基于行的可视模式，模式类型为**行文本**

|命令|说明|
|-----|:-------|
|v |基于字符的可视模式|
|V 或 shift + v|基于行的可视模式|
|ctrl + v|基于列（块）的可视模式|

### 3.2 基本操作

言归正传，首先我们根据需求选择了选区后，然后我们可以对其进行基本操作，基本操作主要有以下几种：

|命令|说明|
|-----|:-------|
|d	|删除选中文本|
|c	|修改选中文本|
|y	|复制选中文本|
|r	|替换选中文本|
|I	|在选中文本前插入|
|A	|在选中文本后插入|
|gu	|选中区域转为小写|
|gU	|选中区域转为大写|
|g~	|大小写互调|
|>	|向右缩进一个单位|
|<	|向左缩进一个单位|

值得注意的：**d只删除选中的字符，而D删除选中字符所在行的所有字符。C和c、y和Y同理**。

在使用可视模式的时候，一旦切换到可视模式以后，选中的区域是由两个端点来界定的（一个在左上角，一个在右下角），我们在默认情况下只可以控制右下角的端点，但是有些时候发现我们需要调整左上角的端点，这时我们可以使用`o`来在左上角和右下角之间进行切换。

### 3.3 实际演示

以上都是关于块操作的理论部分，下面让我们进入实战吧，下面演示几种情况。

（1）为以下代码每行行尾添加分号：
 
![](/images/posts/20170912164123918.png)

按键说明：

1. 使用基于列的块操作选中16 ~ 18行：`Ctrl +  V`

2. 使用 `$` 符号进入行末：`$`

3. 按A键在光标后面插入字符分号：`A + ;`

4. `ESC + ESC`

（2）注释以下代码：

![](/images/posts/20170912164137654.png)

按键说明：

1. 使用基于列的块操作选中11 ~ 19行：`Ctrl +  V`

2. 按I键在光标前插入注释符号：`I + //`

3. `ESC + ESC`

## 四、Vim 配置文件

### 4.1 功能说明

- INSERT模式下，`Ctrl +  N` 打开自动补全功能。

- 自动编译C/C++

- 括号匹配

- 自动注释

- 查看函数man定义（需修改配置文件中man.vim的路径）

### 4.2 编辑配置文件

- 如果仅对当前用户生效，编辑`~/.vimrc`文件即可

- 如果对所有用户生效，编辑`/etc/vimrc`文件即可

**配置内容：**

```bash
set backspace=indent,eol,start

syntax on " 自动语法高亮

set number " 显示行号

set tabstop=4 " 设定 tab 长度为 4

set cindent " 设定c语言自动缩进
set shiftwidth=4 "自动缩进空白符个数
set softtabstop=4 "tab键的一个
set autoindent " 设定自动缩进

" 设置man.vim插件(这里填写自己的man.vim路径)
" 程序中使用 'Man 函数名' 或 shift+k 
source /usr/share/vim/vim74/ftplugin/man.vim

" 括号匹配
inoremap { {<CR>}<ESC>i
"inoremap ( ()<Esc>i
"inoremap [ []<ESC>i

" F5编译和运行C程序，F6编译和运行C++程序
map <F5> :call CompileRunGcc()<CR>
func! CompileRunGcc()
exec "w"
exec "!gcc -Wall % -o %<"
exec "! ./%<"
endfunc

map <F6> :call CompileRunGpp()<CR>
func! CompileRunGpp()
exec "w"
exec "!g++ -Wall % -o %<"
exec "! ./%<"
endfunc

" Ctrl-t添加注释
map <C-t> :call Annotation()<CR>
func! Annotation()
exec "s#^#//#g"
endfunc

" Ctrl+c取消注释
map <C-c> :call CancelAnnotation()<CR>
func! CancelAnnotation()
exec "s#^//###g"
endfunc

"编码设置
set enc=utf-8
set fencs=utf-8,ucs-bom,shift-jis,gb18030,gbk,gb2312,cp936

"语言设置
set helplang=cn
set encoding=utf8 
set langmenu=zh_CN.UTF-8 
set imcmdline 
source $VIMRUNTIME/delmenu.vim 
source $VIMRUNTIME/menu.vim

"自动添加文件头
autocmd BufNewFile *.c,*.cpp,*.java exec ":call SetTitle()"

func SetTitle()  
     if &filetype == 'c'
        call setline(1, "#include <stdio.h>")  
        call append(line("."), "#include <stdlib.h>")  
        call append(line(".")+1, "")
        call append(line(".")+2, "int main(void) {")
        call append(line(".")+3, "     ")
        call append(line(".")+4, "    return 0;")
        call append(line(".")+5, "}")
    endif

    if &filetype == 'cpp'
        call setline(1, "#include <iostream>")  
        call append(line("."), "")
        call append(line(".")+1, "using namespace std;")  
        call append(line(".")+2, "")
        call append(line(".")+3, "int main(void) {")
        call append(line(".")+4, "     ")
        call append(line(".")+5, "    return 0;")
        call append(line(".")+6, "}")
    endif

    if &filetype == 'java'
        call setline(1, "class ".expand("%:r").expand("{"))
        call append(line("."), "    public static void main(String[] args) {")
        call append(line(".")+1, "         ")
        call append(line(".")+2, "    }")
        call append(line(".")+3, "}")
    endif
endfunc

"光标移动到指定位置
autocmd BufNewFile *.c normal 5G$
autocmd BufNewFile *.cpp normal 6G$
autocmd BufNewFile *.java normal 3G$
```

## 五、Vim 快捷键

### 5.1 移动光标

1、左移h、右移l、下移j、上移k
2、向下翻页ctrl + f，向上翻页ctrl + b
3、向下翻半页ctrl + d，向上翻半页ctrl + u
4、移动到行尾$，移动到行首0（数字），移动到行首第一个字符处^
5、移动光标到下一个句子 ），移动光标到上一个句子（
6、移动到段首{，移动到段尾}
7、移动到下一个词w，移动到上一个词b
8、移动到文档开始gg，移动到文档结束G
9、移动到匹配的{}.().[]处%
10、跳到第n行 ngg 或 nG 或 :n
11、移动光标到屏幕顶端H，移动到屏幕中间M，移动到底部L
12、读取当前字符，并移动到本屏幕内下一次出现的地方 *
13、读取当前字符，并移动到本屏幕内上一次出现的地方 #

### 5.2 查找替换

1、光标向后查找关键字 #或者g#
2、光标向前查找关键字 *或者g*
3、当前行查找字符 fx, Fx, tx, Tx
4、基本替换 :s/s1/s2 （将下一个s1替换为s2）
5、全部替换 :%s/s1/s2
6、只替换当前行 :s/s1/s2/g
7、替换某些行 :n1,n2 s/s1/s2/g
8、搜索模式为 /string，搜索下一处为n，搜索上一处为N
9、制定书签 mx, 但是看不到书签标记，而且只能用小写字母
10、移动到某标签处 x，1旁边的键
11、移动到上次编辑文件的位置 

### 5.3 编辑操作

1、光标后插入a, 行尾插入A
2、后插一行插入o，前插一行插入O
3、删除字符插入s， 删除正行插入S
4、光标前插入i，行首插入I
5、删除一行dd，删除后进入插入模式cc或者S
6、删除一个单词dw，删除一个单词进入插入模式cw
7、删除一个字符x或者dl，删除一个字符进入插入模式s或者cl
8、粘贴p，交换两个字符xp，交换两行ddp
9、复制y，复制一行yy
10、撤销u，重做ctrl + r，重复.
11、智能提示 ctrl + n 或者 ctrl + p
12、删除motion跨过的字符，删除并进入插入模式 c{motion}
13、删除到下一个字符跨过的字符，删除并进入插入模式，不包括x字符 ctx
14、删除当前字符到下一个字符处的所有字符，并进入插入模式，包括x字符，cfx
15、删除motion跨过的字符，删除但不进入插入模式 d{motion}
16、删除motion跨过的字符，删除但不进入插入模式，不包括x字符 dtx
17、删除当前字符到下一个字符处的所有字符，包括x字符 dfx
18、如果只是复制的情况时，将12-17条中的c或d改为y
19、删除到行尾可以使用D或C
20、拷贝当前行 yy或者Y
21、删除当前字符 x
22、粘贴 p
23、可以使用多重剪切板，查看状态使用:reg，使用剪切板使用”，例如复制到w寄存器，”wyy，或者使用可视模式v”wy
24、重复执行上一个作用使用.
25、使用数字可以跨过n个区域，如y3x，会拷贝光标到第三个x之间的区域，3j向下移动3行
26、在编写代码的时候可以使用]p粘贴，这样可以自动进行代码缩进
27、 >> 缩进所有选择的代码
28、 << 反缩进所有选择的代码
29、gd 移动到光标所处的函数或变量的定义处
30、K 在man里搜索光标所在的词
31、合并两行 J
32、若不想保存文件，而重新打开 :e!
33、若想打开新文件 :e filename，然后使用 `Ctrl +  ^` 进行文件切换
34、格式化代码 gg=G

### 5.4 窗口操作

1、分隔一个窗口:split或者:vsplit
2、创建一个窗口:new或者:vnew
3、在新窗口打开文件:sf {filename}
4、关闭当前窗口:close
5、仅保留当前窗口:only
6、到左边窗口 ctrl + w, h
7、到右边窗口 ctrl + w, l
8、到上边窗口 ctrl + w, k
9、到下边窗口 ctrl + w, j
10、到顶部窗口 ctrl + w, t
11、到底部窗口 ctrl + w, b

### 5.5 宏操作

1、开始记录宏操作q[a-z]，按q结束，保存操作到寄存器[a-z]中
2、@[a-z]执行寄存器[a-z]中的操作
3、@@执行最近一次记录的宏操作

### 5.6 可视操作

1、进入块可视模式 ctrl + v
2、进入字符可视模式 v
3、进入行可视模式 V
4、删除选定的块 d
5、删除选定的块然后进入插入模式 c
6、在选中的块同是插入相同的字符 I<String>ESC

### 5.7 跳到声明

1、[[ 向前跳到顶格第一个{  
2、[] 向前跳到顶格第一个}
3、]] 向后跳到顶格的第一个{
4、]] 向后跳到顶格的第一个}
5、[{ 跳到本代码块的开头
6、]} 跳到本代码块的结尾

### 5.8 挂起操作

1、挂起Vim ctrl + z 或者 :suspend
2、查看任务 在shell中输入 jobs
3、恢复任务 fg [job number]（将后台程序放到前台）或者 bg [job number]（将前台程序放到后台）
4、执行shell命令 :!command
5、开启shell命令 :shell，退出该shell exit
6、保存vim状态 :mksession name.vim
7、恢复vim状态 :source name.vim
8、启动vim时恢复状态 vim -S name.vim

![vi/vim键位图](/images/posts/20170606201929349.png)
