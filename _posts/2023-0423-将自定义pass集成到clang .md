---
layout: post   
title: 将自定义pass集成到clang 
categories: LLVM
description: 将自定义pass集成到clang 
keywords: LLVM, LLVM Pass 
---


<h4>一、构建 llvm 工程</h4>

1. 构建llvm 工程：
```
cmake -G Xcode -DLLVM_ENABLE_ASSERTIONS=On -DCMAKE_BUILD_TYPE=Debug -DLLVM_ENABLE_PROJECTS="clang;llvm;lld" ../llvm
```


2. 打开xcode工程：

```
$ open LLVM.xcodeproj/
```

可以自动创建`scheme`或者手动创建`scheme`,为了节省时间，我们选择手动创建

```clang, opt, llvmhello```


编译完后，就会在 `build/debug/bin` 目录下看到 `clang, opt` 工具，在 `build/debug/lib` 目录下看到`LLVMHello.dylib`



3.  使用opt 加载pass


```
$ ./opt -load /Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/NewPass.dylib --help |grep newpass
      --newpass 
```

```
$ ./opt -load /Users/heyonly/workspace/llvm/llvm-project/build/Debug/lib/NewPass.dylib -newpass -enable-new-pm=0 < main.bc 
WARNING: You're attempting to print out a bitcode file.
This is inadvisable as it may cause display problems. If
you REALLY want to taste LLVM bitcode first-hand, you
can force output with the `-f' option.

newpass: _ly_fun_b
newpass: _ly_fun_e
newpass: main

```

一定要记得添加 `-enable-new-pm=0` 这个选项


<h4>二、自定义pass</h4>

以创建 `BogusControlFlow` 为例，使用的是 `legacy-pass-manager`, 新版本的passmanager 正在研究中。

1. 在 `llvm/lib/Transforms/` 目录下创建子目录： `Obfuscation`
2. 在 `Obfuscation`  目录下创建文件： `BogusControlFlow.cpp, CMakeLists.txt, Obfuscation.exports`

3.  `CMakeLists.txt` 文件内容：

```
add_llvm_component_library(LLVMObfuscation
  BogusControlFlow.cpp
  
  ADDITIONAL_HEADER_DIRS
  ${LLVM_MAIN_INCLUDE_DIR}/llvm/Transforms/Obfuscation
  
  DEPENDS
  intrinsics_gen
  )

```


4. 在 `llvm/lib/Transforms/` 目录下的 `CMakeLists.txt` 添加一行：

```
add_subdirectory(Obfuscation)
```

5. 重新构建 xcode 工程

```
cmake -G Xcode -DLLVM_ENABLE_ASSERTIONS=On -DCMAKE_BUILD_TYPE=Debug -DLLVM_ENABLE_PROJECTS="clang;llvm;lld" ../llvm
```


6. 编辑 `BogusControlFlow.cpp` 文件内容如下：

```
#include "llvm/Transforms/Obfuscation/BogusControlFlow.h"
#include "llvm/InitializePasses.h"
#include "llvm/Pass.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Function.h"

namespace {
  struct BogusControlFlow : public FunctionPass {
    static char ID; // Pass identification
    bool flag;
    BogusControlFlow() : FunctionPass(ID) {}
    BogusControlFlow(bool flag) : FunctionPass(ID) {this->flag = flag; BogusControlFlow();}

    /* runOnFunction
     *
     * Overwrite FunctionPass method to apply the transformation
     * to the function. See header for more details.
     */
    virtual bool runOnFunction(Function &F){
      // Check if the percentage is correct
        errs()<<"BogusControlFlow:" << F.getName() << "\n";
      if(toObfuscate(flag,&F,"bcf")) {
          return true;
        }
      }

      return false;
    } // end of runOnFunction()

   }; // end of struct BogusControlFlow : public FunctionPass
}

char BogusControlFlow::ID = 0;
//static RegisterPass<BogusControlFlow> X("boguscf", "inserting bogus control flow");

INITIALIZE_PASS(BogusControlFlow, "boguscf", "Call graph flattening", false, false)

Pass *llvm::createBogusPass() {
  return new BogusControlFlow(true);
}

Pass *llvm::createBogusPass(bool flag) {
  return new BogusControlFlow(flag);
}

```

`llvm/Transforms/Obfuscation/BogusControlFlow.h` 文件：

```
// Namespace
namespace llvm {
    Pass *createBogusPass ();
    Pass *createBogusPass (bool flag);
}
```


接下来需要有几个步骤：


a) 在 `llvm/InitializePasses.h` 文件中添加如下代码：

```
void initializeBogusControlFlowPass(PassRegistry&);
```

b) 在 `llvm/lib/CodeGen/CodeGen.cpp` 文件中 添加如下代码：

```
initializeBogusControlFlowPass(Registry);
```


c) 在 `llvm/lib/IPO/CmakeLists.txt` 中加入以下代码：

```
LINK_COMPONENTS
  ......
  Obfuscation
```

d) 在 `PassManagerBuilder.cpp` 中添加代码：

```
#include "llvm/Transforms/Obfuscation/BogusControlFlow.h"


static cl::opt<bool> BogusControlFlow("bcf", cl::init(false),
                                      cl::desc("Enable bogus control flow"));
                                      


void PassManagerBuilder::addFunctionSimplificationPasses(
    legacy::PassManagerBase &MPM) {
    MPM.add(createBogusPass(BogusControlFlow));
}
                                      
```


e）编译 `LLVMObfuscation ` 

f) 设置`clang `参数：

```
-O3 -flegacy-pass-manager -mllvm -bcf -I/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include -c main.c -o main.o

```

注意添加 `-O3` 参数。


接下来运行 `clang`, 就可以打断点调试 自己的 `pass` 了。这样就可以直接使用`clang` 添加参数即可部署在生产环境中。


也可以使用 `opt` 调试更简单。