---
layout: post   
title: Swift 指针UnsafePointer  
categories: swift
description: UnsafePointer
keywords:  UnsafePointer
---

Swift 指针UnsafePointer



UnsafePointer




访问特定类型数据的指针



Declaration


`struct UnsafePointer<Pointee>`


OVerView


您可以使用`UnsafePointer` 来访问内存中的特定数据类型，这个数据类型是一个指针。`UnsafePointer` 并不提供自动内存管理或对齐方式。您需要自己完全掌握指针的生命周期以避免内存泄漏或者未定义的指针。   手动内存管理可以取消或者绑定数据到特定类型，使用`UnsafePointer` 类型来访问和管理特定的类型。





Understanding a Pointer's Memory State


`UnsafePointer` 可以是几种状态中的一种。一些指针操作必须在特定的状态中进行---您必须时刻了解指针的内存状态，以便对其有不通的操作。内存可以是`untyped` 、`uninitialized`、`bound to a type` 、`uninitialized` 、`bound to a type`、`initialized`这几种状态，最后，内存在`deallocated`之前必须先`allocated`，在销毁指针前必须先`unallocated memory`




Uninitilized Memory



内存通过`allocated` 或者在`deinitialized` 之后将处于`uninitialized` 状态，在`Uninitialized`的内存必须`initialized` 之后才可以被访问。




Initialized Memory



初始化后的内存可以通过指针的`pointee` 属性或者通过下表来读取，如下面的例子：




```
var number = 5
let numberPointer = UnsafePointer<Int>(&number)

print(numberPointer.pointee)
print(numberPointer[0])

log:
5
5

```



以不同类型访问指针内存



当访问`UnsafePointer` 实例指向的内存时，这个指针的`Pointee` 的类型必须和绑定的内存的类型相同。如果需要以其他方式访问这块内存，swift 的指针类型提供了`type-safe` 的方式暂时的或者永久的改变这个类型的类型，或者以`row memory` 的方式直接加载这块内存，如下：




当需要临时的使用不同的类型 访问 这块内存时，可以使用`withMemoryRebound(to:capacity:)`这个方法




```
let uint8Pointer = UnsafeMutablePointer<UInt8>.allocate(capacity: 1)
uint8Pointer.advanced(by: 0).pointee = CUnsignedChar(ascii: "a")
let length = uint8Pointer.withMemoryRebound(to: Int8.self, capacity: 8) { (len) in
      return strlen(len)
}
print(length)

var fullInteger = uint64Pointer.pointee
var firstByte = uint8Pointer.pointee
        
 print("\(fullInteger)  \(firstByte)")
        
uint8Pointer.deinitialize(count: 1)
uint8Pointer.deallocate()

//  length = 1
  
```



另外，也可以通过一种不确定的类型访问同一块内存，只要绑定类型和目标类型是普通类型，可以通过`UnsafeRawPointer` 实例和 `load(fromByteOffset:as:)`就可以读取值。如：




```
fullInteger = rawPointer.load(as: UInt64.self)   // OK
firstByte = rawPointer.load(fromByteOffset: 0, as: UInt8.self)      // OK

```


 
 
 指针计算
 
 
 
 
 
 指针计算是根据步长来进行计算的，当加上或减去一个指针，结果是一个新的额相同类型的指针。如：
 
 
 
 
 ```
 // 'intPointer' points to memory initialized with [10, 20, 30, 40]
let intPointer: UnsafePointer<Int> = ...

// Load the first value in memory
let x = intPointer.pointee
// x == 10

// Load the third value in memory
let offsetPointer = intPointer + 2
let y = offsetPointer.pointee
// y == 30


let numbers = [10, 20, 30, 40, 50, 60, 70]
print(numbers[..<3])
        
let intPointer = UnsafeMutablePointer<Int>.allocate(capacity: 4)
for i in 0..<4 {
     (intPointer + i).initialize(to: numbers[i])
 }
 for i in 0..<4 {
    print((intPointer + i).pointee)
  }
  
  
  intPointer.deinitialize(count: 4)
// When you allocate memory, always remember to deallocate once you're finished.
  intPointer.deallocate()
        
 ```
 
 
 
 
 
 也可以通过下标来访问：
 
 
 
 ```
 let z = intPointer[2]
// z == 30
 ```
 
 
 
隐式转换与桥接


使用 UnsafePointer 作为 函 数或方法的参数时,可以传递该特定指针类型的实例、传递兼容指针类型的实例或使用 Swift 的隐式桥接传递兼容指针。

```
func printInt(atAddress p: UnsafePointer<Int>) {
    print(p.pointee)
}

printInt(atAddress: intPointer)
// Prints "42"

```



因为在调用函数时，可变指针类型被隐式转换成不可变的指针类型`UnsafePointer`，可以如下：




```
let mutableIntPointer = UnsafeMutablePointer(mutating: intPointer)
printInt(atAddress: mutableIntPointer)
// Prints "42"
```



另外，也可以使用swift 隐式转换功能，传递一个指针或者数组




```
var value: Int = 23
printInt(atAddress: &value)
// Prints "23"
```



或者



```
let numbers = [5, 10, 15, 20]
printInt(atAddress: numbers)
// Prints "5"
```


也可以使用下面这种方式




```
var mutableNumbers = numbers
printInt(atAddress: &mutableNumbers)
```



以上是苹果文档提供的一些例子，下面这篇文章相当不错，而且这个网站博客和课程质量非常高，以后就不用到处找资料了。



[iOS&Swift Tutorials](https://www.raywenderlich.com/780-unsafe-swift-using-pointers-and-interacting-with-c)








[1]\:  [iOS&Swift Tutorials](https://www.raywenderlich.com/780-unsafe-swift-using-pointers-and-interacting-with-c)


[2]\:  [https://developer.apple.com/documentation/swift/unsafepointer](https://developer.apple.com/documentation/swift/unsafepointer)

