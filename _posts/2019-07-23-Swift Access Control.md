---
layout: post   
title: Access Control   
categories: swift
description: Access Control 
keywords:  Access Control 
---

访问控制



访问控制是 你的代码访问其他源文件和module 的约束。这个特性可以让你隐藏的代码的具体实现，和通过特定的接口就可以访问和使用你的代码。





你可以给你的单独类型（classes，structures and enumerations）指定特定的访问级别。如properties，methods、initializers和其他类型。协议可以限定特定上下文，如全局常量，变量，和函数。

swift 中由低至高提供了private、fileprivate、internal、public、open五中访问控制的权限。默认的internal 在绝大部分时候是使用的。另外由于它是swift中默认的控制级别，因此也是最为方便的。



除了提供不同级别的访问控制外,Swift 还减少了为典型方案提供默认访问级别来指定显式访问控制级别的需求。实际上,如果您正在编写单目标应用,则可能根本不需要指定显式访问控制级别。




模块和源文件（Modules and Files）

Swift 的访问控制模型基于模块和源文件的概念。



模块是代码分发的单个单元,它是一个框架或应用程序,它作为单个单元构建和交付,并且可以通过swift  的关键字 import keyword 来导入。




在swift 中每一个target（如APP bundle 或者framework）都是单独的模块来对待。如果你小组引用你的代码，或许是为了跨应用重用改代码，或在另一个框架中使用时。那么在导入和使用该框架时,在该框架中定义的所有内容都将是单独模块的一部分。




访问级别



Swift 为代码中的实体提供了五种不同的访问级别。这些访问级别与定义实体的源文件相关,也相对于源文件所属的模块。



* Open 和public 修饰的实体可以被任何源文件访问，也包括从其他模块中引入的模块，他们的区别下面会描述。

* internal 修饰的只能在模块内被访问，模块外的无法访问。当只允许在APP 内部或者framework内部访问时，可以使用internal来修饰。


* fileprivate 修饰的实体，只允许在他们自己定义的源文件中访问。使用fileprivate 修饰符，可以隐藏函数具体实现。


* private 访问权限修饰的实体，只允许闭包申明中和<b>同一个文件</b>的Extension 中访问。当这些详细信息仅在单个声明中使用时,使用私有访问来隐藏特定功能段的实现详细信息。



Open 是最高级别的访问权限（最少限制）,private  是最低级别访问权限（限制最多）



Open 仅仅用于类和类成员，和public  的区别如下：



* public 修饰的类，比open 有更多的限制，只能被模块内部继承（也就是说，open修饰的可以被模块外的继承）。


* public 修饰的类成员，只能被模块内部重写（也就是说open 修饰的可以被模块外部的重写）


* open 修饰类可以被模块外的继承

* open 修饰类成员可以被模块外的子类重写


使用open 显示标记的类表明，您必须考虑作为一个超类对其他模块的影响，从而更加严谨的设计您的代码。




访问级别的规则



Swift 中的访问级别遵循总体指导原则:任何实体都无法根据具有较低(限制性)访问级别的另一个实体进行定义。

for example:


* 一个public 类型的变量不能被定义成internal、fileprivate、或者private类型，因为这个变量有可能在其他地方已经用到。


* 一个函数访问级别不能高于他的参数类型和返回值类型。因为函数有可能被用于参数类型或者返回值类型不能使用的情况。


默认访问级别


所有的实体（个别除外，在后面的章节中会讲到）在不显示指定的情况下，都有一个默认的internal 访问级别。在很多情况下，不需要显示指定。




单个target APP 的访问级别



当您的APP 只有一个target 的时候，不需要提供给外界使用的时候，默认的internal 访问级别就已经满足要求。另外，可以使用private 或者fileprivate 来隐藏模块 内部具体实现。



framework 的访问级别



当开发一个framework 的时候，可以使用open 或者public 来标记公开接口，以便让其他模块访问，如APP 引入framework 。这些公开接口就是可编程接口。


> 在framework内部仍然可以使用默认界别的internal级别。或者使用private 、fileprivate 来对framework 其他部分隐藏具体实现。


单元测试的访问级别



使用单元测试目标编写应用时,需要将应用中的代码提供给该模块才能进行测试。默认情况下,只有标记为打开或公共的实体才能访问其他模块。但是,如果使用 @testable 属性标记产品模块的导入声明,并在启用测试的情况下编译该产品模块,则单元测试目标可以访问任何内部实体。




访问级别语法：



```
public class SomePublicClass {}

internal class SomeInternalClass {}

fileprivate class SomeFilePrivateClass {}

private class SomePrivateClass {}

public var somePublicVariable = 0

internal let someInternalConstant = 0

fileprivate func someFilePrivateFunction() {}

private func somePrivateFunction() {}
```



除非特别指定，否则都是默认级别internal。这就意味着SomeInternalClass 和someInternalConstant 可以不需要显示申明，可以写成下面这样：


```
class SomeInternalClass {}              // implicitly internal
let someInternalConstant = 0            // implicitly internal
```





自定义类型


当想为一个自定义类型显示声明一个访问级别。例如，您想定义一个file-private  的类，作为一个属性，或者函数参数或者函数返回值类型。



在这个自定义类型中对于他的成员（如 properties、methods、initializers和subscripts）仍然有一个默认的访问控制权限。如果您定义 的类型是private 或者fileprivate ，那她成员的默认权限将是private 或者fileprivate。如果这个自定义类型是internal 或者public，那么他的成员的默认权限将是internal。


> 一个public 类型，他的成员的默认权限是internal，不是public，如果想要他 的成员是public，您必须显示声明。此要求可确保类型面向公众的 API 是您选择发布的,并避免错误地将某种类型的内部工作形式呈现为公共 API。



```
public class SomePublicClass {                  // explicitly public class
    public var somePublicProperty = 0            // explicitly public class member
    var someInternalProperty = 0                 // implicitly internal class member
    fileprivate func someFilePrivateMethod() {}  // explicitly file-private class member
    private func somePrivateMethod() {}          // explicitly private class member
}

class SomeInternalClass {                       // implicitly internal class
    var someInternalProperty = 0                 // implicitly internal class member
    fileprivate func someFilePrivateMethod() {}  // explicitly file-private class member
    private func somePrivateMethod() {}          // explicitly private class member
}

fileprivate class SomeFilePrivateClass {        // explicitly file-private class
    func someFilePrivateMethod() {}              // implicitly file-private class member
    private func somePrivateMethod() {}          // explicitly private class member
}

private class SomePrivateClass {                // explicitly private class
    func somePrivateMethod() {}                  // implicitly private class member
}
```





元组类型



访问级别对于元组类型限制是最多的。例如，如果您用两个不同类型来组成一个元组，一个是internal，一个是private 没那么他组合后的访问级别是private。


> 元组并没有像类、结构体、枚举、函数那样有明确的定义。元组的访问级别是自动递减的，并且不能显示指定。




函数类型




函数类型的访问界别是根据函数的参数及返回值类型的访问级别计算的。如果函数计算出来的访问级别与实际的不匹配，则必须显示的指定函数的每一个部分的访问级别。







对于一个严格的项目来说，精确的最小化访问控制界别对于代码的维护来说还是相当重要的。我们想让同一module (或者说target) 中的其他代码访问的话，保持默认的internal 就可以了，如果我们再为其他开发者开发库的时候，可能希望用一些public 甚至open，因为在target 外只能调用到public 和open 的代码。public 和open 的区别在于，只有被open 编辑的内容才能在别的框架中被集成或者重写（如UIView）。因此，如果你只希望框架的用户使用某个类型和方法，而不希望被他们集成或者重写的话，应该将其限定为public 而非open。


1、private 和fileprivate 修饰属性




```
//ViewController.swift
class Person: NSObject {

    fileprivate var name:String = "man"
    
    private var age:Int = 1
}

class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
    
        let p = Person()
        print(p.name)
        
        //编译器不会提示age属性 如果强行写p.age
        //会报错'age' is inaccessible due to 'private' protection level
        //print(p.age)
    }
}
log:  man
```

> 在swift4.0中 fileprivate修饰的属性，能在当前文件内访问到，不管是否在本类的作用域;
private 只能在本类的作用域且在当前文件内能访问





