---
layout: post
title:  "Windows下安装与使用GCC编译器"
date:   2015-04-22 10:21:32
categories: 其他
comments: true
---
### 什么是GCC?
我们在Windows系统下习惯使用诸如Windows Visual Stdio图形化IDE工具来编辑和编译代码，但在Unix/Linux系统下如何编译C++代码呢？答案是GCC(GUN Compiler Collection)。GCC源于一场自由软件计划，最初只能在不同操作系统上处理C语言，经过快速扩张后目前也支持C++/Objective-C/Java/Ada/Pascal/Fortran等多种语言的编译。

### 安装GCC
1. 下载安装MinGW
MinGW(Minimalist GNU on Windows)是Windows系统下GNU工具套装，使用MinGW来安装GCC。
<a href="http://sourceforge.net/projects/mingw/files/">点击下载</a>，并安装到C盘根目录。
2. 配置Windows环境变量
找到环境变量：控制面板->系统->高级系统设置->高级->环境变量；
在系统变量中选择<code>Path</code>，并在变量值中添加<code>C:\MinGW\bin</code>
在系统变量中新建<code>LIBRARY_PATH</code>，变量值是<code>C:\MinGW\lib</code>
在系统变量中新建<code>C_INCLUDE_PATH</code>，变量值是<code>C:\MinGW\include</code>
以上操作分别配置了标准库和头文件的存放路径。
3. 使用MinGW安装GCC
运行MinGW,在Basic Setup中选择mingw32-gcc-g++
选择Installation->Apply Changes
等待GCC相关环境安装完毕。
4. 以上步骤完成后，在cmd敲入命令
<code>gcc --version</code>
如果安装成功，会显示gcc版本号。

### 使用GCC
1. 创建GCC工作目录
<code>mkdir CPP</code>
2. 前往工作目录
<code>cd CPP</code>
3. 选择一种文本编辑器（比如Windows自带的notepad）
<code>notepad main.c</code>
4. 在自动弹出的文本编辑器中敲测试代码，保存

{% highlight ruby %}
include "stdlib.h";
int main() {
    printf("Hello World\n");
    return(0);
}
{% endhighlight %}

5. 使用GCC编译main.c，输出HelloWorld可执行文件
<code>gcc main.c -o HelloWorld</code>
6. 此时会报错“缺少libgcc_s_sjlj-1.dll文件”
<a href="http://www.dll-files.com/dllindex/dll-files.shtml?libgcc_s_sjlj-1">点击下载</a>32-bit版本，解压并放到C:\Windows\SysWOW64中
7. 重复步骤5，正常运行，此时可以看到工作目录下生成了HelloWorld.exe文件。最后在cmd中运行文件
<code>HelloWorld.exe</code>
终于看到第一个测试输出“HelloWorld”，大功告成！

[参考资料]
<ol>
	<li>http://www.wikihow.com/Compile-a-C-Program-Using-the-GNU-Compiler-(GCC)</li>
	<li>http://blog.csdn.net/firefoxbug/article/details/6724876</li>
</ol>
