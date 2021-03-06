---
layout: post   
title: The Frontend 
categories: LLVM
description: The Frontend 
keywords: LLVM, LLVM 前端（Clang） 
---

以下来自  <b>Getting Started with LLVM Core Libraries</b> 翻译




在特定于目标的代码生成之前，编译器前端将源代码转换为编译器的IR。由于编程语言具有不同的语法和语义域，因此前端通常处理a
单一语言或一组类似的语言。例如，Clang处理C，C ++和Objective-C源代码输入。在本章中，我们将介绍以下主题：



* Clang 库如何链接程序以及如何使用`libclang`

* Clang 诊断以及Clang 前端阶段

* 使用libclang示例进行词法，句法和语义分析

* 如何编写使用C ++ Clang库的简化编译器驱动程序


Clang 介绍

Clang 是LLVM 的前端工程，您可以访问clang 官网`http://clang.llvm.org`,我么也已经在第一张介绍过如何设置、编译、和安装`llvm`



为了分析驱动和编译器是如何工作的，我们通过命令行命令来分析编译器驱动：



解析命令行参数后，Clang驱动程序通过使用-cc1选项生成其自身的另一个实例来调用内部编译器。通过在编译器驱动程序中使用-Xclang <option>，可以传递特定参数
对于这个工具，与驱动程序不同，它没有模仿GCC命令行界面的义务。例如，clang -cc1工具有一个特殊选项来打印Clang抽象语法树（AST）。要激活它，您可以使用以下命令结构：

`$ clang hello.c –o hello`

`$ clang -Xclang -ast-dump hello.c`

`$ clang -cc1 -ast-dump hello.c`


但是，请记住，编译器驱动程序的任务之一是使用所有必需参数初始化编译器的调用。使用 -  ###标志运行驱动程序，以查看它用于调用clang -cc1编译器的参数。例如，如果手动调用clang -cc1，则还需要通过-I标志自行提供所有系统头的位置。

前端功能




clang -cc1工具的一个重要方面（和混淆源）是它不仅实现了编译器前端，而且还通过LLVM库实例化了进行编译所需的所有其他LLVM组件。 LLVM可以。因此，它实现了几乎完整的编译器。通常，对于x86目标，clang -cc1在目标文件前沿停止，因为LLVM链接器仍处于实验状态且未集成。此时，它将控制权交还给驱动程序，驱动程序将调用外部工具来链接项目。 -  ###标志显示了Clang驱动程序调用的程序列表，并说明了这一点：



```
$ clang hello.c -o hello -###
Apple LLVM version 10.0.1 (clang-1001.0.46.4)
Target: x86_64-apple-darwin18.5.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
 "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang" "-cc1" "-triple" "x86_64-apple-macosx10.14.0" "-Wdeprecated-objc-isa-usage" "-Werror=deprecated-objc-isa-usage" "-emit-obj" "-mrelax-all" "-disable-free" "-disable-llvm-verifier" "-discard-value-names" "-main-file-name" "hello.c" "-mrelocation-model" "pic" "-pic-level" "2" "-mthread-model" "posix" "-mdisable-fp-elim" "-fno-strict-return" "-masm-verbose" "-munwind-tables" "-target-sdk-version=10.14" "-target-cpu" "penryn" "-dwarf-column-info" "-debugger-tuning=lldb" "-target-linker-version" "450.3" "-resource-dir" "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/10.0.1" "-isysroot" "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk" "-I/usr/local/include"
 .....
```




我们省略了驱动程序使用的完整参数列表。第一行显示clang -cc1进行从C源文件到目标代码的编译。然后，最后一行显示Clang仍然依赖于系统链接器来完成编译。



在内部，每次调用clang -cc1都由一个主要的前端动作控制。完整的动作集在源文件include / clang / Frontend / FrontendOptions.h中定义。下表包含一些示例，并描述了clang -cc1工具可以执行的不同任务：

Action            |   Description
:-:               |  	:-:
ASTView           |  Parse ASTs and view them in Graphviz 
EmitBC            |  Emit an LLVM bitcode .bc file
EmitObj           |  Emit a target-specific .o file
FixIt             |  Parse and apply any fixits to the source|
PluginAction      |  Run a plugin action
RunAnalysis       |  Run one or more source code analyses



`-cc1`选项触发`cc1_main`函数的执行（有关详细信息，请查看源代码文件`tools / driver / cc1_main.cpp`）。例如，当通过`clang hello.c -o hello`间接调用`-cc1`时，此函数初始化特定于目标的信息，设置诊断基础结构并执行`EmitObj`操作。此操作在`CodeGenAction`中实现，`CodeGenAction`是`FrontendAction`的子类。此代码将实例化所有`Clang`和`LLVM`组件并编排它们以构建目标文件。


不同前端操作的存在允许Clang为编译之外的目的运行编译管道，例如静态分析。仍然，取决于您通过`-target`命令行参数为clang指定的目标，
它将加载一个不同的`ToolChain`对象。这将通过执行不同的前端操作来更改-cc1应执行的任务，这些操作应由外部工具执行，以及使用哪些外部工具。例如，给定目标可以使用GNU汇编器和GNU链接器来完成编译，而另一个可以使用LLVM集成汇编器和GNU链接器。如果您对Clang用于目标的外部工具有疑问，可以使用 -  ###开关来打印驱动程序命令。我们将在第8章跨平台编译中讨论有关不同目标的更多信息。



库



从这一点开始，我们将把`Clang`作为一组实现编译器前端而不是驱动程序和编译器应用程序的库。从这个意义上说，Clang是模块化的，由几个库组成。 `libclang（http://clang.llvm.org/doxygen/group__CINDEX.html`）是外部`Clang`用户最重要的接口之一，并通过C API 提供广泛的前端功能。它包括几个Clang库，它们也可以单独使用并链接到您的项目中。本章最相关的库列表如下：



* <b>libclangLex :</b>      该库用于预处理和词法分析，处理宏，标记和 注解 结构

* <b>libclangAST :</b>       此库集成了构建，控制（manipulate）和 遍历抽象语法树的功能

* <b>libclangSema :</b>      此库用于语义分析，它为AST验证提供操作

* <b>libclangCodeGen :</b>   此库使用特定于目标的信息来 生成LLVM IR代码

* <b>libclangRewrite :</b>   该库允许支持代码重写并提供构建代码重构工具的基础结构（更多细节见第10章，使用LibTooling的Clang工具）

* <b>libclangBasic :</b>     该库提供了一组实用程序 - 内存分配抽象，源位置和诊断等。

    
    
 使用`libClang`
 
 
 
在本章中，我们将解释Clang前端的部分内容，并通过使用`libclang` C接口为您提供示例。即使它不是直接访问内部Clang类的C ++ API，使用`libclang`的一大优势来自于它的稳定性;由于许多客户都依赖它，因此Clang团队设计了它，考虑了与以前版本的向后兼容性。但是，您可以随时使用常规C ++ LLVM接口，就像在第3章工具和设计的示例中使用常规C ++ LLVM接口读取`bitcode`函数名时一样




在LLVM安装文件夹的include子文件夹中，检查子文件夹clang-c，即libclang C头所在的位置。要运行本章中的示例，您需要包含Index.h标头，它是Clang C接口的主要入口点。最初，开发人员创建此接口以帮助集成开发环境（如Xcode）导航C源文件并生成快速代码修复，代码完成和索引，这为主头文件指定了名称Index.h。我们还将说明如何将Clang与C ++接口一起使用，但我们将在本章末尾留下它。




与第3章工具和设计中的示例不同，我们使用`llvm-config`来帮助我们构建要链接的LLVM库列表，我们没有这样的工具对于Clang库。要链接`libclang`，您可以将`Makefile`从第3章工具和设计更改为以下列表。与上一章中的方法相同，请记住手动插入制表符以使`Makefile`正常工作。由于这是一个适用于所有示例的通用Makefile，请注意我们使用了`llvm-config --libs`标志而没有任何参数，这会返回完整的LLVM库列表。


```
$ build/Debug/bin/llvm-config --libs
-lLLVMLibDriver -lLLVMCoverage -lLLVMXRay -lLLVMLTO -lLLVMPasses 
...
```


理解Clang 诊断




诊断是编译器与其用户交互的重要部分。它们是编译器向用户发出错误，警告或建议信号的消息。 Clang具有非常好的编译诊断功能，具有漂亮的打印和C ++错误消息，并具有更高的可读性。在内部，Clang按类型划分诊断：每个不同的前端阶段都有不同的类型和自己的诊断集。例如，它定义了文件`include/clang/Basic/DiagnosticParseKinds.td`中解析阶段的诊断。 Clang还根据报告的问题的严重程度对诊断进行分类：注意，警告，扩展，`EXTWARN`和错误。它将这些严重性映射为`Diagnostic :: Level`枚举。




您可以通过在文件`include/clang/Basic/Diagnostic * Kinds.td`中添加新的`TableGen`定义并编写能够检查所需条件的代码并相应地发出诊断来引入新诊断。 LLVM源代码中的所有`.td`文件都是使用`TableGen`语言编写的。



`TableGen`是`LLVM`构建系统中使用的`LLVM`工具，用于为可以机械方式合成的编译器部分生成C ++代码。这个想法始于LLVM后端，它有大量代码可以根据目标机器的描述生成，现在也存在于整个LLVM项目中。 `TableGen`旨在以直接的方式表示信息：通过记录。例如，`DiagnosticParseKinds.td`包含表示诊断的记录的定义：


```
def err_invalid_token_after_toplevel_declarator : Error<
  "expected ';' after top level declarator">;
```




在此示例中，`def`是用于定义新记录的`TableGen`关键字。必须在这些记录中传达哪些字段完全取决于将使用哪个`TableGen`后端，并且每种类型的生成文件都有一个特定的后端。 `TableGen`的输出始终是包含在另一个`LLVM`源文件中的`.inc`文件。在这种情况下，`TableGen`需要使用宏定义生成DiagnosticsParseKinds.inc来解释每个诊断。



`err_invalid_token_after_toplevel_declarator`是记录标识符，而`Error`是`TableGen`类。请注意，语义与C ++略有不同，并不完全对应于C ++实体。每个`TableGen`类（与C ++不同）是一个记录模板，用于定义其他记录可以继承的信息字段。但是，与C ++一样，`TableGen`也允许类的层次结构。



类似模板的语法用于根据Error类指定定义的参数，`Error`类接收单个字符串作为参数。从该类派生的所有定义都将是`ERROR`类型的诊断，并且特定消息在类参数中编码，例如，`"expected ';' after top level declarator"`。虽然`TableGensyntax`非常简单，但由于数量很大，它很容易混淆读者TableGen条目中编码的信息。如有疑问 请参阅`http://llvm.org/docs/TableGen/LangRef`。



阅读诊断




我们现在提供一个C ++示例，它使用libclang C接口读取和转储Clang在读取给定源文件时生成的所有诊断。



```
extern "C" {
#include "clang-c/Index.h"
}
#include "llvm/Support/CommandLine.h"
#include <iostream>
using namespace llvm;
static cl::opt<std::string>
FileName(cl::Positional, cl::desc("Input file"), cl::Required);
int main(int argc, char** argv)
{
    cl::ParseCommandLineOptions(argc, argv, "Diagnostics Example");
    CXIndex index = clang_createIndex(0, 0);
    const char *args[] = {
        "-I/usr/include",
        "-I."
    };
    CXTranslationUnit translationUnit = clang_parseTranslationUnit(index, FileName.c_str(), args, 2, NULL, 0, CXTranslationUnit_None);
    unsigned diagnosticCount = clang_getNumDiagnostics(translationUnit);
    for (unsigned i = 0; i < diagnosticCount; ++i) {
        CXDiagnostic diagnostic = clang_getDiagnostic(translationUnit, i);
        CXString category = clang_getDiagnosticCategoryText(diagnostic);
        CXString message = clang_getDiagnosticSpelling(diagnostic);
        unsigned severity = clang_getDiagnosticSeverity(diagnostic);
        CXSourceLocation loc = clang_getDiagnosticLocation(diagnostic);
        CXString fName;
        unsigned line = 0, col = 0;
        clang_getPresumedLocation(loc, &fName, &line, &col);
        std::cout << "Severity: " << severity << " File: "
                << clang_getCString(fName) << " Line: "
                << line << " Col: " << col << " Category: \""
                << clang_getCString(category) << "\" Message: "
                << clang_getCString(message) << std::endl;
        
        clang_disposeString(fName);
        clang_disposeString(message);
        clang_disposeString(category);
        clang_disposeDiagnostic(diagnostic);
    }
    clang_disposeTranslationUnit(translationUnit);
    clang_disposeIndex(index);
    return 0;
}
```




在此C ++源代码中包含libclang C头文件之前，我们使用extern
“C”环境允许C ++编译器将此头编译为C代码。



我们重复使用前一章中的`cl namespace`来帮助我们解析程序的命令行参数。然后我们使用libclang接口中的几个函数（`http://clang.llvm.org/doxygen/group__CINDEX.html`）。首先，我们通过调用`clang_createIndex（）`函数创建一个索引，即libclang使用的顶级上下文结构。它接收两个整数编码的布尔值作为参数：如果我们想要从预编译头（PCH）中排除声明，则第一个为真，如果我们想要显示诊断，则第二个为真。我们设置两者都是假（零）因为我们想要自己显示诊断。



接下来，我们要求Clang通过`clang_parseTranslationUnit（）`解析翻译单元（参见`http://clang.llvm.org/doxygen/group__CINDEX__TRANSLATION__UNIT.html`）。它接收要解析的源文件的名称作为参数，我们从`FileName`全局检索它。此变量对应于用于启动我们的工具的字符串参数。我们还需要指定一组定义包含文件的位置的两个参数 - 您可以自由调整这些参数以适合您的系统。


>实现我们自己的Clang工具的难点在于缺乏驱动程序的参数预测能力，它为处理系统中的源文件提供了足够的参数。例如，如果您要创建Clang插件，则不必担心这一点。要解决此问题，可以使用第10章`use LibTooling`的Clang工具中讨论的编译命令数据库，它提供了用于处理要分析的每个输入源文件的确切参数集。在这种情况下，我们可以使用CMake生成数据库。但是，在我们的示例中，我们自己提供这些参数。



解析并将所有信息放入CXTranslationUnit后在C数据结构中，我们实现了一个循环，它遍历Clang生成的所有诊断并将它们转储到屏幕上。为此，我们首先使用`clang_ getNumDiagnostics（）`来检索解析此文件时生成的诊断数，并确定循环的边界（请参阅`http：//clang.llvm.org/doxygen/group__CINDEX__DIAG.html`）。其次，对于每个循环迭代，
我们使用`clang_getDiagnostic（）`来检索当前诊断，使`clang_getDiagnosticCategoryText（）`来检索描述此诊断类型的字符`clang_getDiagnosticSpelling（）`来检索要显示给用户的消息，并`clang_getDiagnosticLocation（）`来检索确切的代码位置发生了。我们`clang_getDiagnosticSeverity（）`来检索枚举成员，表示此诊断的严重性（`NOTE`，`WARNING`，`EXTENSION`，`EXTWARN`或`ERROR`），但我们将其转换为无符号值并打印它是一个简单的数字。



由于这是一个缺少C ++字符串类的C接口，因此在处理字符串时，函数通常会返回一个特殊的`CXString`对象，要求您调用`clang_getCString（）`来访问内部char指针以打印它，并且调用`clang_disposeString（）`以后再删除它。



请记住，您的输入源文件可能包含其他文件，要求诊断引擎还记录除行和列之外的文件名。文件，行和列的三重属性集允许您查找引用代码的哪个部分。一个特殊的对象`CXSourceLocation`代表了这个三元组
组。要将其转换为文件名，行和列号，必须将`clang_getPresumedLocation（）`函数与`CXString`和`int`一起用作相应填充的引用参数。



完成之后，我们通过调用`clang_disposeDiagnostic（）`，`clang_disposeTranslationUnit（）`和`clang_disposeIndex（）`删除我们的对象。

```
Let's test it with the file hello.c as follows:
int main() {
     printf("hello, world!\n")
     return 0;
}

```

这个C 文件，有两处错误: 缺失头文件和 缺失分号。来编译看看clang 会给我们报什么错误信息。


```
$ clang hello.c 
hello.c:2:6: warning: implicitly declaring library function 'printf' with type 'int (const char *, ...)' [-Wimplicit-function-declaration]
     printf("hello, world!\n")
     ^
hello.c:2:6: note: include the header <stdio.h> or explicitly provide a declaration for 'printf'
hello.c:2:31: error: expected ';' after expression
     printf("hello, world!\n")
                              ^
                              ;
```



学习前端阶段

要将源代码程序转换为LLVM IR bitcode，源代码必须通过一些中间步骤。下图说明了所有这些，它们是本节的主题：

![](/images/blog/LLVM/4/2019-07-18220636.png)



词法分析



第一个前端步骤通过将语言结构拆分为一组单词和标记来处理源代码的文本输入，删除诸如注释，空格和制表符之类的字符。每个单词或标记必须是语言子集的一部分，保留语言关键字将转换为内部编译器表示。保留字在`include/clang/Basic/TokenKinds.def`中定义。例如，请参阅`while`保留字的定义
和`<`符号，两个已知的C/C ++ Tokens，在`TokenKinds.def`摘录中突出显示：


```
KEYWORD(volatile                    , KEYALL)
KEYWORD(while                       , KEYALL)
KEYWORD(_Alignas                    , KEYALL)
```


```
PUNCTUATOR(percentequal,        "%=")
PUNCTUATOR(less,                "<")
PUNCTUATOR(lessless,            "<<")
```


此文件上的定义填充了tok名称空间。这样，只要编译器需要在词法处理之后检查是否存在保留字，就可以使用该命名空间来访问它们。例如，`{`，`<`，`goto`和`while`结构由枚举元素`tok :: l_brace`，`tok :: less`，`tok :: kw_ goto`和`tok :: kw_while`访问。



```
int min(int a, int b) {
    if (a < b)
      return a;
return b; }
```



每个`Token`都包含一个`SourceLocation`类的实例，用于保存
程序源代码中的位置。请记住，您使用的是C对应的`CXSourceLocation`，但两者都引用相同的数据。我们可以使用以下`clang -cc1`命令行从词法分析中转储`Token`及其`SourceLocation`结果：




```
$ clang -cc1 -dump-tokens main.c 
int 'int'	 [StartOfLine]	Loc=<main.c:1:1>
identifier 'min'	 [LeadingSpace]	Loc=<main.c:1:5>
l_paren '('		Loc=<main.c:1:8>
int 'int'		Loc=<main.c:1:9>
identifier 'a'	 [LeadingSpace]	Loc=<main.c:1:13>
comma ','		Loc=<main.c:1:14>
int 'int'	 [LeadingSpace]	Loc=<main.c:1:16>
identifier 'b'	 [LeadingSpace]	Loc=<main.c:1:20>
r_paren ')'		Loc=<main.c:1:21>
l_brace '{'	 [LeadingSpace]	Loc=<main.c:1:23>
if 'if'	 [StartOfLine] [LeadingSpace]	Loc=<main.c:2:6>
l_paren '('	 [LeadingSpace]	Loc=<main.c:2:9>
identifier 'a'		Loc=<main.c:2:10>
less '<'	 [LeadingSpace]	Loc=<main.c:2:12>
identifier 'b'	 [LeadingSpace]	Loc=<main.c:2:14>
r_paren ')'		Loc=<main.c:2:15>
return 'return'	 [StartOfLine] [LeadingSpace]	Loc=<main.c:3:8>
identifier 'a'	 [LeadingSpace]	Loc=<main.c:3:15>
semi ';'		Loc=<main.c:3:16>
return 'return'	 [StartOfLine]	Loc=<main.c:4:1>
identifier 'b'	 [LeadingSpace]	Loc=<main.c:4:8>
semi ';'		Loc=<main.c:4:9>
r_brace '}'	 [StartOfLine]	Loc=<main.c:5:1>
eof ''		Loc=<main.c:5:2>
```



分析词法错误
















文末，添加几个命令



`$ clang -emit-llvm –S -c main.c -o main.ll`

`$ clang -O0 -S -emit-llvm sample.c -o sample.ll`

`$ clang -emit-llvm -c main.c -o main.bc`

`$ clang -S -m32 test.c  -o test-32bit.S`

`$ clang -S  test.c  -o test-64bit.S`

`$ clang -emit-llvm -c test.c -o test.bc`

`$ build/Debug/bin/llc -stats test.bc -o test.S`

`$ build/Debug/bin/llc -filetype=obj test.bc -o test.o`

`$ build/Debug/bin/llvm-link simple.bc hello.bc -o test-hello.bc`

`$ /build/Debug/bin/clang -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/ -fobjc-arc -framework Foundation plugin.m -o plugin`


