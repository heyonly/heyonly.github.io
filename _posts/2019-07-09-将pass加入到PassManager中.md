---
layout: post   
title: LLVM 将pass加入到PassManager 
categories: LLVM
description: LLVM 将pass加入到PassManager
keywords: LLVM, LLVM Pass 
---



这篇本来是跟昨天那篇是一篇，但是由于昨天刚打完球回来太累了，没接上。干脆就重开一篇吧。


话不多说。

<h5>创建pass</h5>



首先创建一个pass。流程都烂熟于心了。




1、在lib/transforms 目录下创建一个目录`Obfuscation`,将hello 目录下的文件全部拷贝到这个目录，然后进行更改：



最终文件如下：


`lib/transform/CMakeLists.txt`



```
add_subdirectory(Utils)
add_subdirectory(Instrumentation)
add_subdirectory(AggressiveInstCombine)
add_subdirectory(InstCombine)
add_subdirectory(Scalar)
add_subdirectory(IPO)
add_subdirectory(Vectorize)
add_subdirectory(Hello)
add_subdirectory(ObjCARC)
add_subdirectory(Coroutines)
add_subdirectory(FirstPass)
add_subdirectory(FuncBlockCount)
add_subdirectory(FuncBlockCount)
add_subdirectory(Obfuscation)
```



`lib/Transforms/Obfuscation/CMakeLists.txt`




```
# If we don't need RTTI or EH, there's no reason to export anything
# from the hello plugin.
if( NOT LLVM_REQUIRES_RTTI )
  if( NOT LLVM_REQUIRES_EH )
    set(LLVM_EXPORTED_SYMBOL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/Obfuscation.exports)
  endif()
endif()

if(WIN32 OR CYGWIN)
  set(LLVM_LINK_COMPONENTS Core Support)
endif()

add_llvm_library( LLVMObfuscation MODULE BUILDTREE_ONLY
  SimplePass.cpp

  DEPENDS
  intrinsics_gen
  PLUGIN_TOOL
  opt
  )
```

`include/llvm/Transforms/Obfuscation/SimplePass.h` 文件如下，没有就创建.



```
#include "llvm/IR/Function.h"
#include "llvm/Pass.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Intrinsics.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
// Namespace
using namespace std;


namespace llvm {
    Pass *createSimplePass(bool flag);
}
```


`lib/Transforms/Obfscation/SimplePass.cpp`文件如下：



```
#include "llvm/Transforms/Obfuscation/SimplePass.h"
using namespace llvm;

namespace  {
    struct SimplePass : public FunctionPass {
        static char ID;
        bool flag;
        
        SimplePass() : FunctionPass(ID) {}
        
        SimplePass(bool flag) : FunctionPass(ID) {
            this->flag = flag;
        }
        
        bool runOnFunction(Function &F) override {
            Function *tmp = &F;
            errs() << F.getName() << "\n";
            for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++ bb) {
                for (BasicBlock::iterator inst = bb->begin(); inst != bb->end(); ++inst) {
                    if (inst->isBinaryOp()) {
                        if (inst->getOpcode() == Instruction::Add) {
                            ob_add(cast<BinaryOperator>(inst));
                        }
                    }
                }
            }
            return false;
        }
        
        void ob_add(BinaryOperator *bo) {
            BinaryOperator *op = NULL;
            if (bo->getOpcode() == Instruction::Add) {
                op = BinaryOperator::CreateNeg(bo->getOperand(1),"",bo);
                op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op,"",bo);
                
                op->setHasNoSignedWrap(bo->hasNoSignedWrap());
                op->setHasNoUnsignedWrap(bo->hasNoUnsignedWrap());
            }
            bo->replaceAllUsesWith(op);
        }
    };
}

char SimplePass::ID = 0;
static RegisterPass<SimplePass> X("simplepass","this is a simple pass");
Pass *llvm::createSimplePass(bool flag) {return new SimplePass(false);}

```


最后编译



写测试代码:


```
#include <stdio.h>
int main(int argc, char **argv) {
 int a = 5;
 int b = 4;
 return a + b;
}
```


编译:


生成中间代码



`clang -emit-llvm -c test.c -o test.bc`

使用opt 工具加载pass



`opt -load ../lib/LLVMSimplePass.dylib -simplepass < test.bc > after_test.bc`



反编译中间代码:


```
llvm-dis test.bc -o before_test.ll
llvm-dis after_test.bc -o after_test.ll

```
查看结果



```
befor_test.ll

define i32 @main(i32, i8**) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  %6 = alloca i32, align 4
  %7 = alloca i32, align 4
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  store i32 5, i32* %6, align 4
  store i32 4, i32* %7, align 4
  %8 = load i32, i32* %6, align 4
  %9 = load i32, i32* %7, align 4
  %10 = add nsw i32 %8, %9
  ret i32 %10
}
```

```
after_test.ll

define i32 @main(i32, i8**) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  %6 = alloca i32, align 4
  %7 = alloca i32, align 4
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  store i32 5, i32* %6, align 4
  store i32 4, i32* %7, align 4
  %8 = load i32, i32* %6, align 4
  %9 = load i32, i32* %7, align 4
  %10 = sub i32 0, %9
  %11 = sub nsw i32 %8, %10
  %12 = add nsw i32 %8, %9
  ret i32 %11
}
```




<h5>将pass 加入PassManager管理</h5>

上面是利用`opt` 工具去单独加载pass 动态库，这里我们将pass 加入PassManager，这样我们就不可以通过clang参数去加载我们的papss 了。


* 修改`llvm/lib/Transforms/IPO/PassManagerBuilder.cpp` 文件内容:



```
#include "llvm/Transforms/Obfuscation/SimplePass.h"
using namespace llvm;

static cl::opt<bool> SimplePass("simplepass",cl::init(false),cl::desc("Enable simple pass"));

static cl::opt<bool>
    RunPartialInlining("enable-partial-inlining", cl::init(false), cl::Hidden,
                       cl::ZeroOrMore, cl::desc("Run Partial inlinining pass"));
```



```
void PassManagerBuilder::populateModulePassManager(
    legacy::PassManagerBase &MPM) {
  // Whether this is a default or *LTO pre-link pipeline. The FullLTO post-link
if (PrepareForLTO) {
    MPM.add(createCanonicalizeAliasesPass());
    // Rename anon globals to be able to handle them in the summary
    MPM.add(createNameAnonGlobalPass());
  }
    MPM.add(createSimplePass(SimplePass));
 }
```



* 修改IPO 这个目录下的LLVMBuild.txt文件，添加库支持



```
;===- ./lib/Transforms/IPO/LLVMBuild.txt -----------------------*- Conf -*--===;
;
; Part of the LLVM Project, under the Apache License v2.0 with LLVM Exceptions.
; See https://llvm.org/LICENSE.txt for license information.
; SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
;
;===------------------------------------------------------------------------===;
;
; This is an LLVMBuild description file for the components in this subdirectory.
;
; For more information on the LLVMBuild system, please see:
;
;   http://llvm.org/docs/LLVMBuild.html
;
;===------------------------------------------------------------------------===;

[component_0]
type = Library
name = IPO
parent = Transforms
library_name = ipo
required_libraries = AggressiveInstCombine Analysis BitReader BitWriter Core InstCombine IRReader Linker Object ProfileData Scalar Support TransformUtils Vectorize Instrumentation Obfuscation
```



* 修改`Transforms`目录下的LLVMBuild.txt 文件



```
;===- ./lib/Transforms/LLVMBuild.txt ---------------------------*- Conf -*--===;
;
; Part of the LLVM Project, under the Apache License v2.0 with LLVM Exceptions.
; See https://llvm.org/LICENSE.txt for license information.
; SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
;
;===------------------------------------------------------------------------===;
;
; This is an LLVMBuild description file for the components in this subdirectory.
;
; For more information on the LLVMBuild system, please see:
;
;   http://llvm.org/docs/LLVMBuild.html
;
;===------------------------------------------------------------------------===;

[common]
subdirectories = AggressiveInstCombine Coroutines IPO InstCombine Instrumentation Scalar Utils Vectorize ObjCARC Obfuscation

[component_0]
type = Group
name = Transforms
parent = Libraries
```



最后修改`Obfuscation`目录下`CMakeLists.txt`



```
add_llvm_library( LLVMObfuscation
  SimplePass.cpp
  )
```

LLVMBuild.txt文件如下



```
[component_0]
type = Library
name = Obfuscation
parent = Transforms

```

最后，重新编译，不出意外，应该是成功的。
使用命令行编译：Xcode 直接编译报错
`cmake -G Xcode -DLLVM_ENABLE_ASSERTIONS=On -DCMAKE_BUILD_TYPE=Debug ../`

之后再用Xcode 后就好了。
最后测试：


```
clang -mllvm -simplepass test.c -o after_test
```


这样以后，就可以编写自己的pass 了。菜鸟正在学习中。。。。

参考：

[1]\:[Code_obfuscation](https://kunnan.github.io/2018/08/18/Code_obfuscation/)


[2]\:[LLVM CookBook](http://llvm.org/docs/WritingAnLLVMPass.html#setting-up-the-build-environment)




