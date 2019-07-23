---
layout: post   
title: Building LLVM IR   
categories: LLVM
description: Building LLVM IR  
keywords: LLVM, Building LLVM IR
---
高级编程语言便于人和机器交互。目前大多数的高级语言都包含 变量，循环，if-else 判断语句，块，函数这些基本元素。变量保存数据类型的值，一个基本块确定变量的作用域。一个if-else 语句决定代码执行路径。一个函数可以让代码块重用。高级语言可能在类型检查，类型转换，变量声明，复杂数据类型方面有所不同.但是，几乎所有其他语言都具有本节前面列出的基本构建块



一种语言可以有自己的解析器,该解析器解析token 并提取有意义的信息,如标识符及其数据类型，一个函数的声明，定义、调用，循环条件等等。这些信息可以存储在数据结构中，其中代码流可以轻松检索。抽象语法树（AST）经常作为代码呈现的一种树结构。抽象语法书将作为后面的转化和分析。



语言解析器可以通过各种工具(如 lex、yacc 等)以各种方式编写。编写一个高效的解析器是一门艺术。但是我们不会在本章介绍.我们希望重点关注 `LLVM IR` 以及如何使用 `LLVM` 将解析后的高级语言转换为 `LLVM IR`



本章将介绍如何构建LLVM 基本工作示例代码，包括以下内容：



* 创建一个LLVM模块

* 在一个模块中发射(emitting)一个函数

* 在函数找那个添加块

* 发射一个全局变量

* 发射一个返回语句

* 发射一个函数参数

* 在一个基本块中发射一个简单的算法语句

* 发射一个if-else 条件IR。

* 发射一个循环LLVM IR


<b>Creating an LLVM module</b>



在上一张，我们知道如何查看LLVMIR，在LLVM 中，一个模块表示要一起处理的单个代码单元。一个`LLVM moudle` 类十一所有`LLVM IR` 对象顶级容器。`LLVM module` 包含全局变量、函数、数据布局、三元组等等。让我们创建一个简单的`llvm module`



LLVM 提供`Module()` 构造函数来创建module，第一个参数是module 的名字，第二个参数是`LLVMContext` 如下：



```

#include "llvm/Bitcode/BitcodeWriter.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/DerivedTypes.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Instruction.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Type.h"
#include "llvm/Support/raw_ostream.h"
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/Verifier.h>
#include <iostream>
#include <vector>

using namespace llvm;

static LLVMContext Context;

```


```
int main(int argc, char *argv[]) {
  Module *M = new Module("test", Context);

    
  M->dump();
    
  return 0;
}
```

>我的是直接在xcode 中创建，可以直接利用xcode 已有的环境，编译直接使用xcode 编译。


命令行编译：

 


```
clang++ -O3 toy.cpp `llvm-config --ldflags --system-libs --libs core` -o toy
```

`./toy`


将会有如下输出：


```
; ModuleID = 'test'
source_filename = "test"
```


<b>Emitting a function in a module</b>

现在已经创建一个module了。下一步就是创建一个`function` LLVM  有一个`IRBuilder `类用于生成`LLVM IR `和打印函数的module 信息。代码如下：


```
Function *createFunc(IRBuilder<> &Builder, std::string name,Module *moduleObj){

    std::vector<Type *> Intergers(FunArgs.size(),Type::getInt32Ty(Context));
    FunctionType *functype = llvm::FunctionType::get(Builder.getInt32Ty(), Intergers,false);
    
    Function *fooFunc = llvm::Function::Create(functype, llvm::Function::ExternalLinkage, name,moduleObj);
    
    return fooFunc;
}
```


最终代码如下：


```

#include "llvm/Bitcode/BitcodeWriter.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/DerivedTypes.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/InstrTypes.h"
#include "llvm/IR/Instruction.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Type.h"
#include "llvm/Support/raw_ostream.h"
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/Verifier.h>
#include <iostream>
#include <vector>

using namespace llvm;

static LLVMContext Context;


Function *createFunc(IRBuilder<> &Builder, std::string name,Module *moduleObj){

    std::vector<Type *> Intergers(FunArgs.size(),Type::getInt32Ty(Context));
    FunctionType *functype = llvm::FunctionType::get(Builder.getInt32Ty(), Intergers,false);
    
    Function *fooFunc = llvm::Function::Create(functype, llvm::Function::ExternalLinkage, name,moduleObj);
    
    return fooFunc;
}


int main() {

    static IRBuilder<> Builder(Context);
    // Create the "module" or "program" or "translation unit" to hold the
    // function
    Module *M = new Module("test", Context);

    Function *foofunc = createFunc(Builder, "foo", M);

    
    bool isVerify = verifyFunction(*foofunc);
    
    M->dump();
    
  return 0;
}

```


输出如下：



```
; ModuleID = 'test'
source_filename = "test"

declare i32 @foo(i32, i32)
```



<b>Adding a block to a function</b>

一个函数包含了基本块，一个基本块有一个入口点。一个基本块包含了总舵的`IR` 指令，



在本章中,您学习了如何使用 LLVM 提供的丰富库创建简单的 LLVM IR,请记住 LLVM IR 是一个中间表示形式。使用自定义解析器将高级编程语言转换为 LLVM IR,该解析器将代码分解为原子部分,如变量、函数、函数返回类型、函数参数、if-else 条件、循环、指针、数组等。这些原子元素可以存储到自定义数据结构中,然后这些数据结构可用于发射LLVMIR,如本章所示。




• 在解析器阶段,可以进行句法分析,而词法分析和类型检查可以在解析后的中间阶段和发射 IR 之前进行

• 在实际使用中,很难发现 IR 以硬编码方式发射,如本章所示。相反,在抽象语法树中解析和表示语言。然后,树在 LLVM 库的帮助下用于发出 LLVM IR,如前面所示。LLVM 社区为编写解析器和发出 LLVM IR 提供了出色的教程。您可以访问http://llvm.org/docs/tutorial



在下一章中，我们将看到如何发出一些复杂的数据结构，如数组，指针。另外，我们将通过clang，C / C ++前端的一些例子，了解语义分析是如何完成的。
