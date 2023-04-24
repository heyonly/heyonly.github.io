---
layout: post   
title: 将pass加入到NewPassManager 
categories: LLVM
description: LLVM 将pass加入到NewPassManager
keywords: LLVM, LLVM Pass 
---





本文基于`llvm14.0.0`


1. 在 `llvm/lib/transforms/` 目录下创建 `Awesome` 修改`Transforms` 目录下的`CMakeLists.txt` ,添加一行：

```
.....
add_subdirectory(Awesome)
.....
```




2. 在 `Awesome` 目录下创建 `Awesome.cpp, CMakeLists.txt` 两个文件

`CMakeLists.txt` 内容：

```
# If we don't need RTTI or EH, there's no reason to export anything
# from the hello plugin.

if(WIN32 OR CYGWIN)
  set(LLVM_LINK_COMPONENTS Core Support)
endif()

add_llvm_pass_plugin( Awesome
  Awesome.cpp
  ADDITIONAL_HEADER_DIRS
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Transforms/Awesome

  DEPENDS
  intrinsics_gen
  BUILDTREE_ONLY
  )

```


`Awesome.cpp` 中内容：


```

#include "Awesome.h"
#include "llvm/InitializePasses.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"

#include <iostream>
#include <string>

using namespace llvm;
using namespace std;

static cl::opt<bool> awe("awe", cl::init(false),
                          cl::desc("wave good bye"));
namespace  {
    struct AwesomePass : PassInfoMixin<AwesomePass> {
        
        PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
            std::cout << "functionname: " << std::string(F.getName()) << std::endl;
            if (F.empty()) {
                return PreservedAnalyses::none();
            }
            
            return PreservedAnalyses::all();
        }
        
        static bool isRequired() { return true; }
        
    };
}


/* New PM Registration */
static bool callback(StringRef Name, llvm::FunctionPassManager &PM,
                     ArrayRef<llvm::PassBuilder::PipelineElement>) {
    if (Name == "awe") {
      PM.addPass(AwesomePass());
      return true;
    }
    return false;
}

static void callbackOpt(llvm::FunctionPassManager& PM, OptimizationLevel level) {
    PM.addPass(AwesomePass());
}

static void registerCallbacks(PassBuilder &PB) {
    PB.registerVectorizerStartEPCallback(callbackOpt);// clang 触发
    
    PB.registerPipelineParsingCallback(callback);// opt 触发
}



extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return {LLVM_PLUGIN_API_VERSION, "awe", LLVM_VERSION_STRING, registerCallbacks};
}

```


编译完后，检查：

```
$ nm -nm /Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib |grep llvmGetPassPluginInfo
0000000000002df0 (__TEXT,__text) weak external _llvmGetPassPluginInfo
```

注意：这里 `_llvmGetPassPluginInfo ` 必须为 `external ` 要不然会报这个错误：


```
$ ./opt -fpass-plugin=/Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib -passes awe main.bc 
WARNING: You're attempting to print out a bitcode file.
This is inadvisable as it may cause display problems. If
you REALLY want to taste LLVM bitcode first-hand, you
can force output with the `-f' option.

Failed to load passes from '/Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib'. Request ignored.
Expected<T> must be checked before access or destruction.
Unchecked Expected<T> contained error:
Plugin entry point not found in '/Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib'. Is this a legacy plugin?PLEASE submit a bug report to https://github.com/llvm/llvm-project/issues/ and include the crash backtrace.
Stack dump:
0.	Program arguments: ./opt -load-pass-plugin /Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib -passes awe main.bc
Stack dump without symbol names (ensure you have llvm-symbolizer in your PATH or set the environment var `LLVM_SYMBOLIZER_PATH` to point to it):
0  opt                      0x0000000111e1e9dd llvm::sys::PrintStackTrace(llvm::raw_ostream&, int) + 61
1  opt                      0x0000000111e1ef5b PrintStackTraceSignalHandler(void*) + 27
2  opt                      0x0000000111e1ce3a llvm::sys::RunSignalHandlers() + 138
3  opt                      0x0000000111e2091f SignalHandler(int) + 223
4  libsystem_platform.dylib 0x00007ff80ff9adfd _sigtramp + 29
5  libsystem_platform.dylib 0x0000000000000005 _sigtramp + 18446603370312913445
6  libsystem_c.dylib        0x00007ff80fed0d24 abort + 123
7  opt                      0x000000010ce149d2 llvm::Expected<llvm::PassPlugin>::fatalUncheckedExpected() const + 146
8  opt                      0x000000010ce1480d llvm::Expected<llvm::PassPlugin>::assertIsChecked() const + 45
9  opt                      0x000000010ce14799 llvm::Expected<llvm::PassPlugin>::~Expected() + 25
10 opt                      0x000000010cddd505 llvm::Expected<llvm::PassPlugin>::~Expected() + 21
11 opt                      0x000000010cddbb95 llvm::runPassPipeline(llvm::StringRef, llvm::Module&, llvm::TargetMachine*, llvm::TargetLibraryInfoImpl*, llvm::ToolOutputFile*, llvm::ToolOutputFile*, llvm::ToolOutputFile*, llvm::StringRef, llvm::ArrayRef<llvm::StringRef>, llvm::opt_tool::OutputKind, llvm::opt_tool::VerifierKind, bool, bool, bool, bool, bool) + 2645
12 opt                      0x000000010ce31115 main + 6661
13 dyld                     0x000000012e87e52e start + 462
Abort trap: 6
```


在 pass 中 要加入 下面这行代码


```
static bool isRequired() { return true; }
```

否则，就不会走 `run` 方法。或者使用 `opt -O3`, 不能使用 `-O0`

接下来，就可以使用opt 调试：

```
$ ./opt --load-pass-plugin=/Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Awesome.dylib --passes=awe /Users/heyonly/workspace/llvm/llvm-project/build/Debug/bin/main.ll -o /Users/heyonly/workspace/llvm/llvm-project/build/Debug/bin/after_main.ll


$ ./clang -O3  -fpass-plugin=/Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/Bye.dylib -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include -L/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/lib  /Users/heyonly/workspace/practise/llvmdemo2/llvmdemo2/main.c

```

至此，开发`pass` 流程 已走通，接下来就是研究利用`pass` 来定制需求了。

