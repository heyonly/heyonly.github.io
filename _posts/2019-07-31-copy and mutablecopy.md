---
layout: post
title: copy and mutbalecopy
categories: Objective-C
description: copy&mutablecopy
keywords: copy mutablecopy,copy,mutablecopy
---



最近在整理知识的过程中，发现自己还是不能好好的理解copy 和mutablecopy ，所以抽时间整理一下。



copy 和mutablecopy的概念


`copy` 浅拷贝，不拷贝对象本身，仅仅是拷贝指向对象的指针


1、copy 一个不可变对象


```
NSString *str1 = @"asdf";
NSString *str2 = [str1 copy];


NSLog(@"str1 = %@ str1p = %p,str2 = %@ str2p = %p ",str1,str1,str2,str2);


log:

2019-07-30 18:20:27.440684+0800 Tutorial09[25941:5081781] str1 = asdf str1p = 0x104b980a8,str2 = asdf str2p = 0x104b980a8 
```

![](/images/blog/copy&mutablecopy/2701344-e4357fa0b6c8b5d1-2.png)



这样，可以清晰的看到，不可变对象使用`copy` 后，两个指针指向同一个内存块，那为什么要用`copy`呢。


如果这么写呢



```
NSString *str1 = @"asdf";
NSString *str2 = [str1 copy];
    
NSLog(@"str1 = %@ str1p = %p,str2 = %@ str2p = %p ",str1,str1,str2,str2);
//log: str1 = str1 str1p = 0x1028600a8,str2 = str1 str2p = 0x1028600a8

str1 = @"abcd";
NSLog(@"str1 = %@ str1p = %p,str2 = %@ str2p = %p ",str1,str1,str2,str2);
//log: str1 = abcd str1p = 0x1028600e8,str2 = str1 str2p = 0x1028600a8
```


这样写，很明显，只不过是将str1 的这个指针指向改变了，常量`asdf` 这块内存并没有改变。这时候，`str1 = @"abcd"` 重新开辟了一块内存，并且将`str1`指向这块内存。


可以使用lldb 命令查看这块内存：



```
(lldb) po 0x1028600a8
asdf
```


2、copy 一个可变对象：



```
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"1",@"2", nil];
NSMutableArray *array1 = [array copy];
NSLog(@"%p -- %p",array,array1);
NSLog(@"%@ %@",[array class],[array1 class]);

log: 
[28427:5249475] 0x28086cb10 -- 0x28063b0e0
[28427:5249475] __NSArrayM __NSArrayI
```

从结果可以看出，内存地址不一样，而且array1 也变成了不可变。这个为什么会指向不同的地址呢；


首先，array1 是通过array `copy`来的，所以array1 是不可变的。


另外，又有array 指向的内存空间是可变的，如果对array 进行修改，那么统一内存空间的内容就会发生变化。那么如果此时array1 还指向array的地址空间，显然达不到copy 的目的，因此，只能另外开辟空间。



3、copy 一个copy 修饰的属性


```
@property (nonatomic, copy) NSMutableArray *mArr;



NSMutableArray *array = [NSMutableArray arrayWithObjects:@"1",@"2", nil];
self.mArr = [array copy];
NSLog(@"array: %p ,self.mArr:%p",array,self.mArr);


log:
[28666:5253568] array: 0x280952a60 ,self.mArr:0x280701120
[28739:5254522] array: __NSArrayM ,self.mArr:__NSArrayI
```


可以看到，内存地址不一样，而且，_mArr 变成了不可变的数组。




因为，_mArr 声明的时候是用copy 修饰的，那么就限制了他为不可变数组，由于array 是可变的，因此，只能给_mArr 重新分配内存。





`mutablecopy` 深拷贝，是直接拷贝整个对象内存到另一块内存中。



![](/images/blog/copy&mutablecopy/2701344-91dab5d63efd2bac.png)


1、`mutablecopy`一个不可变对象

```
NSString *str5 = @"abcd";
NSString *str6 = [str5 mutableCopy];
NSLog(@"str5: %p, str6: %p",str5,str6);
NSLog(@"str5: %@, str6: %@",[str5 class],[str6 class]);

log:
[28827:5256797] str5: 0x104c4c0f0, str6: 0x28027f120
[28827:5256797] str5: __NSCFConstantString, str6: __NSCFString
```


可以看到，`mutablecopy` 拷贝一个对象，不管是地址还是class，都是变了，也就是说，mutablecopy 出来一个对象，是完完全全跟源对象无关的对象。



2、`mutablecopy` 一个可变对象


```
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"1",@"2", nil];


NSMutableArray *arr1 = [array mutableCopy];
NSLog(@"array: %p, arr1: %p",array,arr1);
NSLog(@"array: %@, arr1: %@",[array class],[arr1 class]);

log:
[28901:5258151] array: 0x2823d2760, arr1: 0x2823d56b0
[28901:5258151] array: __NSArrayM, arr1: __NSArrayM

```


可以看到跟上边完全一样的结果


`mutablecopy` 之后完全是一个新对象




3、mutablecopy 一个copy 修饰的属性


```
NSMutableArray *arr1 = [array mutableCopy];
NSLog(@"array: %p, arr1: %p",array,arr1);
NSLog(@"array: %@, arr1: %@",[array class],[arr1 class]);
    
self.mArr = [array mutableCopy];
NSLog(@"%p %@",self.mArr,[self.mArr class]);

log:

[28963:5258974] array: 0x28030d2f0, arr1: 0x28030a7c0
[28963:5258974] array: __NSArrayM, arr1: __NSArrayM
[28963:5258974] 0x280d4ecc0 __NSArrayI
```


结果还是一样。




其实，到现在看来，一开始说copy 浅拷贝是不准确的。

总结：



* 用`copy`修饰的属性或者变量，肯定是不可变的。
 
* 用`copy`赋值的，要看源对象是否可变，来决定之拷贝指针还是拷贝一个新对象到一块新的内存。

* 对象之间用`mutablecopy` 赋值的，肯定会拷贝整个对象到另一块内存。

* 对象之间赋值之后，再改变，遵循互不影响的原则。


