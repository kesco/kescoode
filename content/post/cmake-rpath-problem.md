+++
date = "2016-01-05T18:02:06+08:00"
tags = ["C++", "CMake"]
title = "CMake解决动态链接库RPATH错误问题"

+++

最近在做C++开发，在MBP上用Cmake构建项目的时候，发现`make install`之后生成的执行文件会运行出错：

```sh
dyld: Library not loaded: @rpath/libadk.dylib
  Referenced from: /Users/kesco/Documents/workspaces/cpp/apue_practise/build/bin/./thread
  Reason: image not found
[1] 52362 trace trap  ./thread
```

提示我，动态链接库(`libadk.dylib`也是工程内的一个子项目)找不着。这就奇怪了，因为我`make install`安装的时候，是把动态链接库和执行文件都放在同一个目录下的。为啥会找不着呢，而之前在Windows下是没问题的，后来查了下Cmake的文档，原来是Cmake版本的问题=_+。

如果你的CMakeList.txt上写的`cmake_minium_required`为2.6的话，会报下面异常：

```sh
....
CMake Warning (dev):
  Policy CMP0042 is not set: MACOSX_RPATH is enabled by default.  Run "cmake
  --help-policy CMP0042" for policy details.  Use the cmake_policy command to
  set the policy and suppress this warning.
 
  MACOSX_RPATH is not specified for the following targets:
 
   cjson
   iniparser
   stemmer
   word2vec
 
This warning is for project developers.  Use -Wno-dev to suppress it.
 
-- Generating done
....
```

而在`CMP0042`更新，也就是Cmake 2.8.1.2之后，如果你声明的`cmake_minium_required`为2.8以上，`MACOSX_RPATH`会默认启动，这时候编译的执行文件在查找链接库的时候会往`@rpath`上搜索，所以就找不到要链接的库(`libadk.dylib`在同一个目录下)。我们可以用`otool -L <file>`命令来查看执行文件的链接库依赖。

```sh 
bin git:master ❯ otool -L thread
thread:
        @rpath/libadk.dylib (compatibility version 0.0.0, current version 0.0.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1226.10.1)
```

知道原因之后就好办了，我们只需在CMakeList.txt上把`MACOSX_RPATH`关掉就好了。

```cmake
set(CMAKE_MACOSX_RPATH 0)

add_subdirectory(adk)
add_subdirectory(thread)

....
```

重新用`cmake`生成MakeFile构建就不会报错了。不过，`RPATH`是啥呢？以前写C++的时候都是在Windows上开着VS来的，没遇到过这样的问题，现在遇到了，自然要查清楚。

## 什么是RPATH?

在Linux环境下，使用动态链接的程序在运行时会自动链接`ld.so`这个库(OS X上是`dyld`)，然后通过`ld.so`来查找链接其它的库。而`RPATH`就是编译的时候链接到执行文件的链接库路径。OS X在`RPATH`的设置上和Linux还是有点出入的，OS X的`RPATH`采用的是绝对路径。

`ld.so`搜索路径的优先级是这样的：  
1. `RPATH`，编译链接时加入`-rpath`参数指明所谓的`RUNPATH`，这样可执行文件（或者依赖其他动态链接库的动态链接库）就能告诉`ld.so`到哪里去搜索对应的动态链接库了。
2. `LD_LIBRARY_PATH`，对于没有设定`RPATH`的可执行文件或者动态链接库，我们可以用`LD_LIBRARY_PATH`这个环境变量通知`ld.so`往哪里查找链接库。
3. `/etc/ld.so.conf`，系统对`ld.so`的路径配置文件。
4. `/usr/lib`、`/lib`和`/usr/local/lib`，系统默认路径。

## Cmake和RPATH

在分发程序的时候，执行文件使用的链接库在系统内不一定会有，或者自带了的版本不对，一般都会在程序文件夹内都会附带相应的链接库，所以最好还是把`RPATH`加上。Cmake对RPATH提供了很多选项支持，我们一般只关注这几个变量就好了：`CMAKE_SKIP_BUILD_RPATH`、`CMAKE_BUILD_WITH_INSTALL_RPATH`、`CMAKE_INSTALL_RPATH`和`CMAKE_INSTALL_RPATH_USE_LINK_PATH`。

### 默认RPATH设置

```cmake
set(CMAKE_SKIP_BUILD_RPATH FALSE)                 # 编译时加上RPATH
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)         # 编译时RPATH不使用安装的RPATH
set(CMAKE_INSTALL_RPATH "")                       # 安装RPATH为空
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)      # 安装的执行文件不加上RPATH
```
Cmake在默认情况下，`make install`会把安装的执行文件的`RPATH`删掉的，所以就会出现上面我执行安装好的执行文件报错的问题。

### 加上完整的RPATH

Cmake的默认设置我们肯定是不能使用的，我们需要一个安装的时候也要带上`RPATH`的设置。

```cmake
set(INSTALL_LIB_DIR "${PROJECT_BINARY_DIR}/lib") # 假设安装目录在编译目录的lib子目录内

set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# 确保链接库不在系统默认安装的目录上时更改到项目lib上
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES ${CMAKE_INSTALL_RPATH} isSystemDir)
if("${isSystemDir}" STREQUAL "-1")
  set(CMAKE_INSTALL_RPATH "${INSTALL_LIB_DIR}")
endif("${isSystemDir}" STREQUAL "-1")
```

现在Cmake在C++项目上应用得越来越多了，但是Cmake的文档很分散，写Cmake构建脚本的时候会踩上很多坑，只能慢慢积累经验总结。