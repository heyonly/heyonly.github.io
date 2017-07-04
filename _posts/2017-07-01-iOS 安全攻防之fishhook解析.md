---
layout: post
title: iOS 安全攻防之fishhook解析
categories: blog
description: fishhook
keywords: iOS安全攻防,fishhook
---


在iOS开发中，我们可以利用objective-c 的runtime 来实现函数的动态替换，这对修改系统函数行为来解决bug具有很重要的意义，特别是对于大型工程来说，修改bug更显得重要。然而在iOS开发中，我们还会用到大量的C  函数，在大型工程中，大量的C函数的使用，牵一发而动全身。因此我们也希望也能像OC 一样，能够动态修改C函数行为。此时，fishhook 就是Facebook 专门为此而开发的。fishhook 是安全的，没有用到私有API，在Yahoo的多个项目中都有用到。使用也非常简单。



通过\:[fishhook的官方文档](https://github.com/facebook/fishhook) 可以知道，fishhook很简单：






{%raw%}



```
#import <dlfcn.h>

#import <UIKit/UIKit.h>

#import "AppDelegate.h"
#import "fishhook.h"
 
static int (*orig_close)(int);
static int (*orig_open)(const char *, int, ...);
 
int my_close(int fd) {
  printf("Calling real close(%d)\n", fd);
  return orig_close(fd);
}
 
int my_open(const char *path, int oflag, ...) {
  va_list ap = {0};
  mode_t mode = 0;
 
  if ((oflag & O_CREAT) != 0) {
    // mode only applies to O_CREAT
    va_start(ap, oflag);
    mode = va_arg(ap, int);
    va_end(ap);
    printf("Calling real open('%s', %d, %d)\n", path, oflag, mode);
    return orig_open(path, oflag, mode);
  } else {
    printf("Calling real open('%s', %d)\n", path, oflag);
    return orig_open(path, oflag, mode);
  }
}
 
int main(int argc, char * argv[])
{
  @autoreleasepool {
    rebind_symbols((struct rebinding[2]){{"close", my_close, (void *)&orig_close}, {"open", my_open, (void *)&orig_open}}, 2);
 
    // Open our own binary and print out first 4 bytes (which is the same
    // for all Mach-O binaries on a given architecture)
    int fd = open(argv[0], O_RDONLY);
    uint32_t magic_number = 0;
    read(fd, &magic_number, 4);
    printf("Mach-O Magic Number: %x \n", magic_number);
    close(fd);
 
    return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
  }
}


```


{%endraw%}



