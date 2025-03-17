# WebRTC-Compile
记录自己编译WebRTC静态库的过程，供大家进行参考


WebRTC源码在国外，而且更新频繁，又是Chromium中的一部分，导致国内编译较为麻烦。我参考了一些博客，也踩了一些坑，最后成功编译了WebRTC的静态库文件且成功运行了测试Demo，因此记录一下自己的编译测试过程。以下内容只能作为在我自己环境上编译WebRTC上的准确步骤，对于其他环境我不敢保证成功，而且我只对编译的静态库进行了简单的测试，其他功能尚不清楚，因此更多的则是提供一种参考价值。



参考文章：

1. [Windows下编译WebRTC在音视频领域中，WebRTC可以说是一个绕不开宝库，包括了音视频采集、编解码、传输、渲染 - 掘金](https://juejin.cn/post/7040726576572416036)
2. [webrtc编译 - never715 - 博客园](https://www.cnblogs.com/tangm421/p/15089911.html)
3. [【webRTC】一、windows编译webrtc_windows编译最新版webrtc-CSDN博客](https://blog.csdn.net/weixin_44353958/article/details/139003317)
4. [Windows平台WebRTC编译（持续更新） - 剑痴乎](https://blog.jianchihu.net/windows-webrtc-build.html)





---

# Windows 下编译 WebRTC 参考步骤

## 1. 准备环境

1. **安装 Visual Studio 2022**  

   - 必须勾选“使用C++的桌面开发”或类似的 MSVC 工具组件，确认 `vcvarsall.bat` 等文件存在。  
   - 如果缺少了组件去 `Visual Studio Installer` 补。我是在执行到后序的步骤中才发现自己缺少了Windows 11 SDK。（大致需要什么请参考第一篇博客）

2. **获取 depot_tools**  

   ```bash
   git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
   ```

   如果git拉取不下来也可以直接从 https://storage.googleapis.com/chrome-infra/depot_tools.zip 中下载解压

   - 例如将 depot_tools clone 到 `D:\Work_and_learn\Code\CPP\WebRTC\depot_tools`  
   - 确保把 `depot_tools` 路径加到环境变量 `PATH`，重启命令行生效。（博客中说需要将其置在环境变量的最顶部）

---

## 2. 拉取 WebRTC 源码

以下步骤在你准备好的空文件夹（例如 `D:\Work_and_learn\Code\CPP\WebRTC\webrtc`）下执行：

1. 在目标路径下的CMD中执行：

   ```bash
   set http_proxy=10.15.15.141:20172
   set https_proxy=10.15.15.141:20172
   ```

   这样 `fetch`、`gclient`、`git clone` 能走代理下载。如果你使用的是Clash，那么就是 `127.0.0.1:7890` 。这是Clash的默认端口号

1. **fetch webrtc**  

   ```bash
   fetch --nohooks webrtc
   ```

   这会创建一个 `.gclient` 并克隆 `webrtc.googlesource.com/src` 到 `webrtc/src`。

2. **初次同步**  

   ```bash
   gclient sync
   ```

   - 这一步可能会下载大量依赖（`src/third_party` 等），需要时间较长。  
   - 如果网络不稳，会出现重复重试或超时错误，尽量确保最后能 100% done。

在成功后，你的目录结构大致是：

```
D:\Work_and_learn\Code\CPP\WebRTC
 ├─ depot_tools (你之前下载的)
 ├─ webrtc
     ├─ .gclient
     ├─ src
         ├─ build
         ├─ third_party
         ├─ ...
```

---

## 3. 设置环境变量（使 GN/MSVC 正常工作）

可以在命令行里执行（或写到脚本里）：

```bash
set GYP_MSVS_VERSION=2022
set vs2022_install=D:\Software\Microsoft Visual Studio\2022\Commmon
set GYP_MSVS_OVERRIDE_PATH=D:\Software\Microsoft Visual Studio\2022\Common
set GYP_GENERATORS=msvs-ninja,ninja
set DEPOT_TOOLS_WIN_TOOLCHAIN=0
```

- 上面几行告诉 GN 用哪个版本的 VS 和 MSVC 工具链；`DEPOT_TOOLS_WIN_TOOLCHAIN=0` 禁用下载谷歌的预编译工具链。  
- 注意其中 `vcvarsall.bat` 必须能在 `...\Common\VC\Auxiliary\Build` 路径下找到，否则说明没安装 C++ 工具集或路径设置不对。帖子中说的是在`\Microsoft Visual Studio\2022\Community` ，但我的是在`Common` 中

---

## 4. 进入 `src` 并执行 gclient runhooks

如果前面 `gclient sync` 有提示某些 Hook 没跑完，可再次执行：

```bash
cd D:\Work_and_learn\Code\CPP\WebRTC\webrtc\src
gclient runhooks
```

- 如果报 `vpython3.bat ... not found` 之类错误，说明还需让 `depot_tools` 在你的 PATH 中，或者安装好 Python 并修复 vpython 问题。（其实我忽略这个错误了，直接往后走了）

---

## 5. 生成 GN 工程文件 (gn gen)

你多次使用了不同参数进行测试，这里举例两种常见配置。

### 5.1 用 MSVC (cl.exe) + 关闭组件化

示例：

```bash
cd D:\Work_and_learn\Code\CPP\WebRTC\webrtc\src


gn gen out\test --ide=vs2022 --args="is_debug=true use_custom_libcxx=false"
```

- 加 `use_custom_libcxx=false` 以免拉取 libc++，使用MSVC原生编译器/链接器，并且使用其自带的STL，因此通过参数显式关闭WebRTC的libc++；  
- 如果要 debug，可把 `is_debug=true`。

其实看网上好多博客这里都有许多其他参数，可以自行阅读其他博客进行了解。他们在Windows下编译都将 `is_clang=false` 置为false，表示使用MSVC。但我试了好几次生成的`all.sln` 都没能编译成功，或许是我其他地方可能没设置正确（[Windows平台WebRTC编译（持续更新） - 剑痴乎](https://blog.jianchihu.net/windows-webrtc-build.html)  这个帖子地下最新的评论说最新的webrtc如果设置 `is_clang=false` 就会报错，那现在只能使用了clang编译了？）

### 5.2 用 Clang (clang-cl) + 默认 libc++（GPT给的，仅供参考）

若想用官方推荐 clang-cl，则：

```bash
gn gen out\m1 --ide=vs2022 --args="
    is_clang=true
    is_debug=true
    target_cpu=\"x64\"
    rtc_build_examples=false
    rtc_include_tests=false
    ...
    (根据需要再加)
"
```

安装好 LLVM/Clang 后，这会生成 clang-cl + libc++ 风格的 .sln。

> 如果想裁剪功能，比如 `rtc_use_h264=true` / `rtc_libvpx_build_vp9=false` 等，都可以放进 args 里。

---

## 6. 编译

生成完后，你可以：

1. **在命令行用 ninja**  

   ```bash
   ninja -C out\test -d stats >> out\test\comp.log
   ```

   会把默认的全部 target（“all”）都编译出来，顺带生成编译log。这一步会比较慢，只需等到即可 。  

2. **查看编译产物**  

   - 大量 `.obj` 和 `.lib` 会放在 `.src\out\test\obj\...`  
   - 在这些lib中我们只需要 `webrtc.lib` 









# 测试Demo

## 测试代码

```C++
#include "rtc_base/ssl_adapter.h"  // For rtc::InitializeSSL, rtc::CleanupSSL
#include <iostream>

int main() {
    // 初始化 WebRTC 的 SSL 相关内容
    rtc::InitializeSSL();

    std::cout << "Hello, WebRTC static library test!" << std::endl;

    // 清理 WebRTC 的 SSL 相关内容
    rtc::CleanupSSL();
    return 0;
}
```



## 测试环境

- **IDE**：CLion 2024.2.2
- **编译器**：MSVC 2022





## CMakeLists

```cmake
cmake_minimum_required(VERSION 3.29)
project(WebRTCtest)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")  # /MTd

set(CMAKE_CXX_STANDARD 20)

add_definitions(
        -DWEBRTC_WIN
        -DWIN32_LEAN_AND_MEAN
        -DNOMINMAX
        -DWEBRTC_LITTLE_ENDIAN
        -DWEBRTC_ARCH_64_BITS   # 如果你是64位编译
        -D_WIN32_WINNT=0x0A00   # WebRTC/Chromium对 Windows 10 的要求
        # ... 还可以加其他 _CRT_SECURE_NO_WARNINGS 等
        -D_HAS_ITERATOR_DEBUGGING=0
        -D_ITERATOR_DEBUG_LEVEL=0
)

# 在 MSVC 下，将 Debug 模式改为 /MTd 而不是 /MDd
if(MSVC)
    # 这是 CMake 3.15 之前的常用写法
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
    set(CMAKE_C_FLAGS_DEBUG   "${CMAKE_C_FLAGS_DEBUG}   /MTd")

    # 如果是 CMake 3.15+，可以用更直观的：
    # set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")
endif()


# 指定 WebRTC 静态库所在目录 (示例)
set(WEBRTC_LIB_DIR "../webrtc/lib")

# 头文件
include_directories(
        "../webrtc/src"
        "../webrtc/src/third_party/abseil-cpp"
)

# 链接的库目录
link_directories(
        ${WEBRTC_LIB_DIR}
)

add_executable(WebRTCtest main.cpp)

# 链接 WebRTC 所需的静态库 + Windows 系统库
target_link_libraries(WebRTCtest
        # === 以下示例：你最少需要 rtc_base, system_wrappers, 以及一些其他库。===
        #     可根据实际 .lib 文件名进行增减。
        webrtc

        # === Windows系统库示例 ===
        ws2_32
        secur32
        winmm
        iphlpapi
        advapi32
        user32
        # 如果还有其他依赖，比如 dmoguids, msdmo 等，可继续添加
)
```



### CMakeLists解释

以下内容来自ChatGPT-o1模型

> 下面分 **两大部分** 来解释：第一部分是关于 **CMakeLists.txt** 中的编译器选项与宏定义，第二部分是为什么要使用 **WebRTC 源码真实目录** 来包含头文件，而不是去拷贝一份头文件。
>
> ---
>
> #### 1. CMakeLists.txt 中的编译器选项与宏定义
>
> 在这份 CMakeLists.txt 中，你能看到：
>
> ```cmake
> set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")  # /MTd
> 
> ...
> 
> add_definitions(
>         -DWEBRTC_WIN
>         -DWIN32_LEAN_AND_MEAN
>         -DNOMINMAX
>         -DWEBRTC_LITTLE_ENDIAN
>         -DWEBRTC_ARCH_64_BITS
>         -D_WIN32_WINNT=0x0A00
>         ...
>         -D_HAS_ITERATOR_DEBUGGING=0
>         -D_ITERATOR_DEBUG_LEVEL=0
> )
> 
> if(MSVC)
>     set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
>     set(CMAKE_C_FLAGS_DEBUG   "${CMAKE_C_FLAGS_DEBUG}   /MTd")
> 
>     # （注释掉的）set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")
> endif()
> ```
>
> ##### 1.1 `set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")  # /MTd`
>
> - **作用**：这是 CMake 3.15+ 提供的新选项，用于指定 MSVC 下使用哪种运行时库。  
>   - `"MultiThreadedDebug"` 对应 `/MTd` (静态链接调试 CRT)。  
>   - 也可以用 `"MultiThreaded"`（/MT）、`"MultiThreadedDLL"`（/MD）、`"MultiThreadedDebugDLL"`（/MDd）等。  
> - **为什么要这样**：因为你当前的 WebRTC 静态库是以 **/MTd**（静态调试 CRT）编译的。如果你的应用程序里用的却是 **/MDd**（动态调试 CRT），就会导致链接冲突（如 “RuntimeLibrary mismatch: MTd vs MDd”）。  
> - 通过这行设置，你的工程也会用 `/MTd`，与 WebRTC 保持一致，避免运行时库冲突。
>
> ##### 1.2 `add_definitions(...)` 中的一系列宏
>
> - **`-DWEBRTC_WIN`**、`-DWIN32_LEAN_AND_MEAN`、`-DNOMINMAX`、`-DWEBRTC_LITTLE_ENDIAN`、`-DWEBRTC_ARCH_64_BITS`、`-D_WIN32_WINNT=0x0A00` 等：  
>   - **作用**：显式告诉编译器当前目标平台是 Windows、使用 64 位架构、支持 Win10 API 等等。  
>   - **为什么要这样**：因为 WebRTC 在许多地方会根据 `WEBRTC_WIN`、`WEBRTC_ARCH_64_BITS` 来启用/禁用对应的代码路径。如果不定义，可能会报 “missing byte order functions” 或不识别 Windows 平台。  
>   - `WIN32_LEAN_AND_MEAN`、`NOMINMAX` 则是常见的 Windows 宏，用于减少 Windows 头文件包含的内容，以及避免 `min()`/`max()` 宏冲突。
> - **`-D_HAS_ITERATOR_DEBUGGING=0`、`-D_ITERATOR_DEBUG_LEVEL=0`**：  
>   - **作用**：这两个是 MSVC 特定的宏，用来控制 **“安全迭代器”**（iterator debugging）的等级。  
>   - 在 Debug 模式下，MSVC 默认会设置 `_ITERATOR_DEBUG_LEVEL=2`（启用安全迭代器，能检查越界等）。  
>   - 但 **Chromium/WebRTC** 通常在他们的 “Debug” 里禁用安全迭代器（=0）以减少性能损失，并保持二进制兼容性。  
>   - **为什么要这样**：如果 WebRTC 的库是用 `_ITERATOR_DEBUG_LEVEL=0` 编译，你的工程却是 `_ITERATOR_DEBUG_LEVEL=2`，就会出现 `LNK2038: mismatch detected` 之类的错误。通过显式设为 0，可以匹配 WebRTC 的设置。
> - 另外像 `_CRT_SECURE_NO_WARNINGS` 是 MSVC 在使用一些 “不安全函数”（`sprintf`、`strcpy`等）时去掉安全警告的常用宏。
>
> ##### 1.3 `if(MSVC) ... set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd") ...`
>
> - **作用**：这段是为了 **CMake 3.15 之前** 的传统写法。因为旧版本 CMake 不支持 `CMAKE_MSVC_RUNTIME_LIBRARY`，就只能手动往编译器标志里追加 `/MTd`。  
> - 同时你也看到注释中说，如果是 CMake 3.15+，可以用 `set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug")` 更直观地设置。
> - **为什么要这样**：同样是为了确保调试模式使用 /MTd，而不是 /MDd 或别的运行时。
>
> ---
>
> #### 2. 为什么使用 WebRTC 源码真实目录来包含头文件？
>
> 在这份 CMakeLists 中，你用了：
>
> ```cmake
> include_directories(
>         "../webrtc/src"
>         "../webrtc/src/third_party/abseil-cpp"
> )
> ```
>
> 而**不是**用之前提到的拷贝命令（BAT 文件），类似：
>
> ```bat
> :: xcopy src/*.h include/ /s /e /y
> ```
>
> 然后再 `include_directories("D:/Work_and_learn/Code/CPP/WebRTC/webrtc/include")`。
>
> ##### 2.1 原因：WebRTC 头文件不止是 “.h”，还有 `.inc`、子目录层级
>
> - WebRTC/Abseil/Chromium 中有很多 `.inc` 文件、或多层目录下的头文件。  
> - 如果你只拷贝了 “public” 的 .h 到一个 “include/” 目录，**可能缺失** `.inc` 或其他私有头文件（比如 `absl/numeric/int128_no_intrinsic.inc`）。编译时就会出现 `#include` 找不到文件的错误。  
> - 也有些头文件可能是 GN “flatten” 时复制到 `webrtc/include/`，但并不一定完整。  
>
> ##### 2.2 原因：源码中 “third_party/...” 依赖的头文件不在 `webrtc/include/` 里
>
> - 例如 Abseil 代码在 `webrtc/src/third_party/abseil-cpp/absl/...` 下，某些 `.inc` 文件只存在原始目录。  
> - 如果只复制 `.h` 到 `webrtc/include/third_party/abseil-cpp/`, 未必把 `.inc` 同步过来。  
> - 结果就会编译失败：`No such file or directory: "absl/numeric/int128_no_intrinsic.inc"` 之类。
>
> ##### 2.3 结论
>
> - **直接 include “D:/Work_and_learn/Code/CPP/WebRTC/webrtc/src”** 能让编译器找到 **所有** 原始位置的头文件/`.inc`/子目录，而不必人工拷贝。  
> - 如果以后 WebRTC 升级，头文件变动，你也不用重复维护拷贝脚本。  
> - 这在多模块、大型工程（Chromium/WebRTC）中很常见：源码自带许多 Third-Party 目录，必须直接包含它们才比较稳妥。
>
> ---
>
> #### 总结
>
> 1. **CMakeLists.txt 中的编译器选项/宏**：  
>    - `CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreadedDebug"`：强制使用 `/MTd`；  
>    - `add_definitions(-D_HAS_ITERATOR_DEBUGGING=0 -D_ITERATOR_DEBUG_LEVEL=0)`：与 WebRTC 对齐，避免迭代器调试级别冲突；  
>    - 其余宏如 `WEBRTC_WIN`, `NOMINMAX` 等是给 WebRTC 代码识别平台、避免 Windows 宏冲突等。
>
> 2. **为什么要直接包含 `webrtc/src` 而不从 “include/” 文件夹复制头文件**：  
>    - WebRTC 依赖大量子目录、`.inc` 文件，如果只复制 `.h`，容易丢失或不完整；  
>    - 源码路径里有所有需要的头文件，可一次性包含，避免文件缺失；  
>    - 不用维护多余的拷贝脚本，减少出错概率。





## 测试结果

```bash
D:\Work_and_learn\Code\CPP\CPPWork\WebRTCtest\cmake-build-debug\WebRTCtest.exe
Hello, WebRTC static library test!

进程已结束，退出代码为 0
```

