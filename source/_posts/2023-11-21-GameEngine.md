---
title: GameEngine
date: 2023-11-21 14:40:42
tags:
- GameEngine
---

# overview

- EntryPoint
- Appliacation layer
- Window layer
  - Input
  - Event
- Render
- Render API abstraction
- Debugging support
- Scripting language
- Memory Systems
- Entity Componment system
- Physics
- File I/O VFS
- Build systems

# Project setup

hazel引擎会编译成dll的文件，我们添加exe的Sandbox,依赖于hazel。我们还设置了取消32位和输出目录设置

> 为什么链接

```cpp Hazel/src/Test.h
#pragma once

namespace Hazel {
	_declspec(dllexport) void Print();
}
```

```cpp Hazel/src/Test.cpp
#include "Test.h"
#include <stdio.h>

namespace Hazel {
	void Print()
	{
		printf("Welcome to Hazel Engine!\n");
	}
}
```
```cpp Sandbox/src/Application.cpp
namespace Hazel {
	_declspec(dllimport) void Print();
}

void main()
{
	Hazel::Print();
}
```


# entry point

github的第一次提交可见代码。

# logging

把spdlog用给git以子模块的方式加到Hazel/vendor/spdlog。然后添加include path。再给Sandbox的exe添加include path。

接下来把spdlog封装成一个log类。

然后定义了宏使用

# premake

下载解压拷贝premake5.exe和license到Hazel/vendor/premake里。

编写脚本premake5.lua

编写脚本执行permake脚本。

# event sys

# precompile header

添加 hzpch.h 和 hzpch.cpp(windows必须)。

> hzpch.h必须放在最前面！

# windows class and GLFW

在fork的glfw中添加permake5.lua。git拉取成submodule。然后再修改premake5.lua中增加子module。

# window event

我们定义的一些Application的callbac函数，实现了windowswindow和application的互不知晓。低耦合。

# layer stack

layer stack是用户自定义的集合，可以响应event。

> overlayer 是什么？

# GLad

GLFW必须放在glad头文件后面。 如果不这样做的话需要添加预编译的宏GLFW_INCLUDE_NONE保证不会预加载gl函数。

# imgui

imgui很强大，我们可以把他看作是debug的工具！！！

# imgui event 

我们可以学习imgui的glfw的例子并偷一些代码给imgui event。这些event的代码其实可以自己写。

# imput polling

input的轮询

# key and mouse code

方案1：通过编译时的宏来控制code
方案2：runtime查询。这样可以序列化

不要自作聪明的过早优化！runtime不一定耗时。

# math

如果要想写好的math lib需要SIMD

我们选择GLM，这是一个header-only的库，不需要premake。我们把glm/glm下的header和hpp文件包含到hazel的project里！

# imgui docking and Viewports

我们在imgui的子模块确定switch到docking子分支（包含了premake文件）。然后再把在hazel中编译对应的例子。然后重写imguilayer。

> ingui是static的，在hazel这个dll中可以被很好的使用。但是在exe文件中会链接出错，这是为什么？我们的sandbox只链接hazel，而hazel是dll，他**只链接使用过**的静态库的符号，并且在`imgui.h`**修改**export符号的宏`IMGUI_API`编译才可以。
> 我们可以hack一下。在imgui预编译输入`IMGUI_API=_declspec(dllexport)`,然后再hazel定义`IMGUI_API=_declspec(dllexport)`然后再使用相关的符号来import符号到dll来
> 还可以定义hazel的linker的module definition file的`.def`文件来import所有的符号，
> 所以我们选择把hazel编译成静态库。因为我们不太需要动态库

# intrduction to rendering

opengl是最简单的。为了实现多api兼容。我们要设计一个抽象层render line。并且opengl没有多线程，我们要自己**实现**这个render thread！！


# render architecture

{% asset_img img01.png %}

我们还需要一个render command queue。vlkan有，而opengl没有。我们就要实现OpenGL的

{% asset_img img02.png %}

# rendering and maintenance

这里讨论了把hazel修改成静态库的原因，和上次链接出错的原因。

# static libraries and ZERO warnings

`permake5.lua`中的`staticruntime`动态库时候是`off`的，原因是dll有很多版本，比如debug的版本等，运行时需要链接不同的版本。而静态库就不同了。


我们修改了很多的lua文件来编译成静态库！并且解决了大部分的warning！amazing！

# render context

先看的`windowswindow.cpp`的`glfwMakeContextCurrent()`学习一下OpenGL的window的context class的用法

# first triangle

一个简单的三角形
