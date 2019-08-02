---
layout: post   
title: AdVanced LLVM IR   
categories: LLVM
description: AdVanced LLVM IR  
keywords: LLVM, AdVanced LLVM IR
---


```
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include <vector>

using namespace llvm;

static LLVMContext Context;
static Module *ModuleObj;

static std::vector<std::string> FunArgs;


Function *createFunc(IRBuilder<> &Builder, std::string name) {
    Type *u32Ty = Type::getInt32Ty(Context);
    Type *vecTy = VectorType::get(u32Ty, 2);
    Type *ptrTy = vecTy->getPointerTo(0);
    
    FunctionType *funcType = FunctionType::get(Builder.getInt32Ty(), ptrTy, false);
    
    Function *fooFunc = Function::Create(funcType, Function::ExternalLinkage, name,ModuleObj);
    
    return fooFunc;
}

void setFuncArgs(Function *fooFunc,std::vector<std::string> FunArgs) {
    unsigned Idx = 0;
    Function::arg_iterator AI,AE;
    for (AI = fooFunc->arg_begin(),AE = fooFunc->arg_end(); AI != AE; ++AI,++Idx) {
        AI->setName(FunArgs[Idx]);
    }
}

BasicBlock *createBB(Function *fooFunc, std::string Name) {
    return BasicBlock::Create(Context,Name,fooFunc);
}

Value *getGEP(IRBuilder<> &Builder,Value *Base,Value *Offset) {
    return Builder.CreateGEP(NULL, Base,Offset,"a1");
}

int main() {
    ModuleObj = new Module("my compiler",Context);
    static IRBuilder<> Builder(Context);
    FunArgs.push_back("a");
    
    Function *fooFunc = createFunc(Builder, "foo");
    setFuncArgs(fooFunc, FunArgs);
    Value *Base = fooFunc->arg_begin();
    
    BasicBlock *entry = createBB(fooFunc, "entry");
    Builder.SetInsertPoint(entry);
    Value *gep = getGEP(Builder, Base, Builder.getInt32(1));
    
    verifyFunction(*fooFunc);
    ModuleObj->dump();
    
    return 0;
}
```



```
; ModuleID = 'my compiler'
source_filename = "my compiler"

define i32 @foo(<2 x i32>* %a) {
entry:
  %a1 = getelementptr <2 x i32>, <2 x i32>* %a, i32 1
}
```