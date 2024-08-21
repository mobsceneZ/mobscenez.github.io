---
layout: post
title: CMake 基础入门
subtitle: 跟随CMake Tutorial学习基础的命令和设置
tags: [Build System, CMake]
---

虽然之前了解过跟`Makefile`相关的构建系统，但对于`CMake`一直是处于望而却步的状态，终于等到有足够的理由说服自己去看看`CMake`的基本使用方法，因此跟着官方的[CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)学习了一遍基础命令，在此做一记录。

### 第1步 从零构建基础项目

任何项目的顶层`CMakeLists.txt`都需要使用`cmake_minimum_required()`命令声明最低版本。

使用`project()`命令设置项目名称并通过`add_executable()`命令告知`CMake`利用声明的源代码文件构建可执行程序。

```cmake
# build a minimal project with only one source file.
cmake_minimum_required(VERSION 3.10)
project(Tutorial)
add_executable(Tutorial tutorial.cxx)
```

`CMake`中存在许多以`CMAKE_`开头、具有特殊含义的变量，例如`CMAKE_CXX_STANDARD`和`CMAKE_CXX_STANDARD_REQUIRED`，可以用`set()`命令设置它们。

```cmake
# enforce C++ 11 or higher version.
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

有时候将定义在`CMakelists.txt`中的变量导出到源文件中是很有用的，例如我们只需在`CMakeLists.txt`中维护版本号而避免在源文件的多处位置重复维护。

一种方法是使用配置头文件(**configured header file**)，将需要导入的变量以`@VAR@`的形式声明在配置头文件中并用`configure_file()`命令将对应变量值填充到头文件的相应位置。

```cmake
# configure project version and export it into source file.
set(Tutorial_VERSION_MAJOR 1)
set(Tutorial_VERSION_MINOR 0)
configure_file(TutorialConfig.h.in TutorialConfig.h)
target_include_directories(Tutorial PUBLIC ${PROJECT_BINARY_DIR})
```

### 第2步 添加库

使用`add_library()`命令向整个项目中添加库，若库文件组织为子目录的形式可用`add_subdirectory()`命令来即刻处理子目录中的`CMakeLists.txt`。

通过`target_link_libraries()`将库与目标源文件链接在一起，考虑到源文件要引用库的头文件，还需用`target_include_directories()`将库目录添加到目标的包含路径中。

```cmake
# MathFunctions/CMakeLists.txt
add_library(MathFunctions STATIC MathFunctions.cxx mysqrt.cxx)
# CMakeLists.txt
add_subdirectory(MathFunctions)
target_link_libraries(Tutorial MathFunctions)
target_include_directories(Tutorial PUBLIC "${PROJECT_SOURCE_DIR}/MathFunctions")
```

在构建大型项目时经常需要在不同实现之间做选择，`CMake`提供`option()`命令来设置一个用户可变的缓存变量(**cache variable**)。缓存变量会在首次`CMake`配置后保存到`CMakeCache.txt`中作为默认值，用户可通过传递命令行参数`-D...`更新缓存变量，注意`set()`命令在默认情况下不会覆写缓存变量，除非提供`FORCE`选项。

使用`target_compile_definitions()`命令向编译目标传递宏定义，以控制源文件中由宏定义管控的代码片段。`if()`命令根据条件判断的结果选择性执行若干命令。

```cmake
# MathFunctions/CMakeLists.txt
add_library(MathFunctions STATIC MathFunctions.cxx)
option(USE_MYMATH "Use customized implementation of sqrt() or not" ON)
if(USE_MYMATH)
	target_compile_definitions(MathFunctions PUBLIC USE_MYMATH)
	add_library(SqrtLibrary STATIC mysqrt.cxx)
	target_link_libraries(MathFunctions SqrtLibrary)
endif()
```

### 第3步 为库添加使用要求

在`CMake`中存在三类域关键字(**scope keywords**)，并且在不同的命令下有不同的含义：

1. 目标根据其自身的构建声明(**build specification**)以及从链接依赖传递而来的使用要求(**usage requirement**)来构建。构建声明形如`INCLUDE_DIRECTORIES`并只用于当前目标的构建，使用要求则是在构建声明前添加`INTERFACE_`前缀并允许通过依赖进一步传播，例如`INTERFACE_INCLUDE_DIRECTORIES`会被添加到依赖于当前目标的其他目标的`INCLUDE_DIRECTORIES`中。许多目标相关的指令会根据域关键字设置构建声明和使用要求：`PUBLIC`同时设置构建声明和使用要求；`PRIVATE`仅设置构建声明；`INTERFACE`仅设置使用要求；
2. `target_link_libraries()`命令中的域关键字含义稍有不同。假设现有库`B`依赖于库`C`，库`A`依赖于库`B`。如果库`A`不需要知道任何有关库`C`的知识，那么使用`PRIVATE`来将库`C`连接到库`B`因此库`B`不会将其作为链接接口(**link interface**)。如果库`B`不使用任何库`C`中的实现但是库`A`确实需要参考库`C`来完成所需功能，那么使用`INTERFACE`来将库`C`添加到库`B`的链接接口，从而后续依赖库`B`的目标会同时将库`C`链接进来。最后`PUBLIC`同时实现上述两个功能；

```cmake
# MathFunctions/CMakeLists.txt
target_include_directories(MathFunctions INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
```

`INTERFACE`库不会编译任何源文件也不会在硬盘上生成任何库文件，但其可被用来声明使用要求并被链接到其他目标中，从而使得这些属性被其他目标所强制执行。对于`target_compile_features()`这些命令来说，只允许使用`INTERFACE`关键字来设置`INTERFACE`库的属性。

```cmake
# CMakeLists.txt
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)
target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
# MathFunctions/CMakeLists.txt
target_link_libraries(SqrtLibrary PRIVATE tutorial_compiler_flags)
target_link_libraries(MathFunctions PRIVATE tutorial_compiler_flags)
```

### 第4步 添加生成器表达式

生成器表达式(**generator expressions**)会在构建系统生成的过程中被求值来生成特定信息，通用形式为`$<...>`。生成器表达式被允许用在目标属性的上下文中。

```cmake
# CMakeLists.txt
set(gcc_like_cxx $<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>)
set(msvc_cxx $<COMPILE_LANG_AND_ID:CXX,MSVC>)
target_compile_options(tutorial_compiler_flags INTERFACE
	$<${gcc_like_cxx}:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused> $<${msvc_cxx}:-W3>)
target_compile_options(tutorial_compiler_flags INTERFACE
	"$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
	"$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>")
```

### 第5步 安装与测试

`CMake`提供`install()`命令来方便我们将构建好的程序安装到指定的目录下，其存在多种形式，例如`install(TARGETS...)`负责安装目标输出文件(**target output artifacts**)，此外该命令还提供文件权限、安装目标路径等多种可配置选项。

```cmake
# MathFunctions/CMakeLists.txt
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
	list(APPEND installable_libs SqrtLibrary)
endif()
install(TARGETS ${installable_libs} DESTINATION lib)
install(FILES MathFunctions.h DESTINATION include)
# CMakeLists.txt
install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h" DESTINATION include)
```

`CTest`组件能够帮助管理项目中的测试。我们可通过`add_test()`命令添加测试，若测试命令返回状态码`0`那么认为是测试通过，此外还可以通过其他测试特性(**test property**)修改对测试成功与否的判断标准，例如`PASS_REGULAR_EXPRESSION`会将测试输出与正则表达式进行匹配并忽略先前提到的程序退出码。

```cmake
# CMakeLists.txt
enable_testing()
add_test(NAME Runs COMMAND Tutorial 25)
add_test(NAME Usage COMMAND Tutorial)
set_property(TEST Usage PROPERTY PASS_REGULAR_EXPRESSION "Usage.*number")
```

### 第6步 添加系统自检

在实际情形中或许项目所依赖的特性并非所有平台都支持。在`CMake`中可以导入模块来使用模块额外实现的功能，如`CheckCXXSourceCompiles`模块提供`check_cxx_source_compiles()`函数来检查给定代码是否能成功以`C++`源代码的形式被编译和链接。

```cmake
# MathFunctions/CMakeLists.txt
include(CheckCXXSourceCompiles)
check_cxx_source_compiles("
	#include <cmath>
	int main() {
	  std::log(1.0);
	  return 0;
	}" HAVE_LOG)
if(HAVE_LOG)
	target_compile_definitions(SqrtLibrary PRIVATE HAVE_LOG)
endif()
```

### 第7步 添加定制化命令和生成文件

`add_custom_command()`命令向生成的构建系统添加定制化的构建规则。该命令的一种用途是去生成指定的输出文件(**generate files**)，在同一目录下创建的目标如果将定制化命令的任何输出文件作为源文件，那么该目标在构建阶段会被给予规则去利用命令生成对应文件。

```cmake
add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
	COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
	DEPENDS MakeTable)
```

### 第8步 添加导出配置

若是希望其他`CMake`项目能使用我们的项目，我们可以向项目中添加导出信息。`install(TARGETS)`命令提供`export`选项将安装的目标文件与指定的导出名绑定在一起。`install(EXPORT)`生成并安装`CMake`文件，该文件包含将目标从安装树导入到其他项目的代码。

```cmake
# MathFunctions/CMakeLists.txt
install(TARGETS ${installable_libs}
	EXPORT MathFunctionsTargets
	DESTINATION lib)
# CMakeLists.txt
install(EXPORT MathFunctionsTargets
	FILE MathFunctionsTargets.cmake
	DESTINATION lib/cmake/MathFunctions)
```

然而，单纯这么做会导致`CMake`报错，这是由于在导出库时也会导出诸如`INTERFACE_INCLUDE_DIRECTORIES`等属性(这是符合情理的，试想需要使用`MathFunctions`库的项目不仅需要库文件本身，也需要知道头文件在何处)。但我们在为`MathFunctions`设置包含目录时使用的是当前机器上的绝对路径，这在其他机器上并不适用。因此在`target_include_directories`时需要区分究竟是构建还是安装。生成器表达式`$<BUILD_INTERFACE:...>`和`$<INSTALL_INTERFACE:...>`可用于区分上述两种情况。(后续包配置的步骤省略)

```cmake
target_include_directories(MathFunctions
	INTERFACE
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
	$<INSTALL_INTERFACE:include>)
```
