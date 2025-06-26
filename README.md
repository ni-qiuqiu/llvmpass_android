
[NDK r25加载 LLVM Pass 方案](https://xtuly.cn/article/ndk-load-llvm-pass-plugin)

[NDK r27 Clang 18 Hikari 混淆器 LLVM Pass 插件加载方案](https://eterocell.com/posts/7dbf3410)

[llvm学习（十一）：史上最优雅的NDK加载pass方案](https://leadroyal.cn/p/1008/)

### 首先请你阅读上述教程，如果按照上面的教程来，没什么问题，下面的就没什么必要看了

------

#### 个人环境
ubuntu 24.04 lts

#安装LLVM-19核心包和开发文件（这是我拿来跑x86的，也因此有了一个坑）

sudo apt install llvm-19 llvm-19-dev clang-19 libclang-19-dev

#ndk环境

我直接repo 的 llvm-toolchain（东西很全，但也很大，130G）

正常情况下，可以按教程中 获取llvm-project 以及prebuilt 的calng-r*  编译链，注意版本对应

#### 编译pass插件
这里没什么好说的，直接按照教程来
```
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.22)

set(ccc /home/qiu/llvm-toolchain/prebuilts/clang/host/linux-x86/clang-r522817)
set(CMAKE_C_COMPILER ${ccc}/bin/clang)
set(CMAKE_CXX_COMPILER ${ccc}/bin/clang++)
# 设置动态库输出位置
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)

project(Hikari-Obf-LLVM18-NDK27)
#先设置 LLVM_DIR 后续find_package 会从此目录下开始查
set(ENV{LLVM_HOME} ${ccc})

if (NOT DEFINED ENV{LLVM_HOME})
    message(FATAL_ERROR "$LLVM_HOME is not defined")
else ()
    set(ENV{LLVM_DIR} $ENV{LLVM_HOME}/lib/cmake/llvm)
endif ()

find_package(LLVM REQUIRED CONFIG)
add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})
link_directories(${LLVM_LIBRARY_DIRS})
set(CMAKE_CXX_STANDARD 17)


set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")

set(CMAKE_SKIP_RPATH ON)

add_subdirectory(Obfuscation)


```
仔细的人会发现有一处和教程中的不一样` -stdlib=libstdc++`，没错，这是一个小坑。正常是`-stdlib=libc++`

原因是在解决llvm符号问题时，我们需要重新编译llvm-project项目，替换掉ndk中的clang

问题在于我编译的时候用的是clang-19 `  -DCMAKE_C_COMPILER=clang-19 -DCMAKE_CXX_COMPILER=clang++-19 ` ，

然而这个链接的是libstdc++.so。正常情况下能够编译so成功，但加载就会`libc++.so.1 => not found`
```
~/llvm-toolchain/.repo/manifests$ ldd /usr/bin/clang-19
	libclang-cpp.so.19.1 => /lib/x86_64-linux-gnu/libclang-cpp.so.19.1 (0x00007a3121600000)
	libLLVM.so.19.1 => /lib/x86_64-linux-gnu/libLLVM.so.19.1 (0x00007a3119a00000)
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007a3119600000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007a31258b9000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007a3119200000)
	/lib64/ld-linux-x86-64.so.2 (0x00007a3125a0a000)
```
导致编译出来的clang也是链接`libstdc++`
```
~/llvm-toolchain/toolchain/llvm-project/build_r27_c18/bin$ ldd clang-18
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f3741200000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f3747348000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3740e00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f37473e0000)
```
正常情况，使用prebuilt的就`-stdlib=libc++`

#### 测试
经过测试发现r25c 可以编译个插件就行，没有llvm符号问题（完美符合预期，不污染ndk环境，只需一个pass so即可开启加密

r27 r28都有符号问题，需要重编llvm

此外，使用插件还可以加密lkm模块

![lkm_pass](https://github.com/user-attachments/assets/a522e212-42aa-4007-93cc-b329c0bd9f69)


#### 参考项目

[HikariObfuscator_Guide](https://github.com/OPSphystech420/HikariObfuscator_Guide/tree/build/android-ndk-llvm18)

[SsagePass](https://github.com/SsageParuders/SsagePass)
