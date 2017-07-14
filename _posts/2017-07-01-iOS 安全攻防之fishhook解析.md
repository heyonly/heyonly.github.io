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



使用很简单，我们将会来看看是如何实现动态替换的。



我们看fishhook 的入口



```
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
  //调用prepend_rebindings 的函数，将整个rebindings数组添加到这个私有链表的头部
  int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
  if (retval < 0) {
    return retval;
  }
  // If this was the first call, register callback for image additions (which is also invoked for
  // existing images, otherwise, just run on existing images
  if (!_rebindings_head->next) {//判断是不是第一次调用
    _dyld_register_func_for_add_image(_rebind_symbols_for_image);
  } else {
    uint32_t c = _dyld_image_count();
    for (uint32_t i = 0; i < c; i++) {
      _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
    }
  }
  return retval;
}
```



对于prepend_rebindings函数，就是一个链表的头插法，就不解释了。但我们注意到函数_dyld_register_func_for_add_image，有函数名我们能猜个大概，当添加image（镜像）时，注册一个回调函数。(⊙v⊙)嗯，大概是这样的。可我们写程序不能靠猜。我们知道dyld 时开源的，我们不妨看看源码：



```
/*
 * _dyld_register_func_for_add_image registers the specified function to be
 * called when a new image is added (a bundle or a dynamic shared library) to
 * the program.  When this function is first registered it is called for once
 * for each image that is currently part of the program.
 */
void
_dyld_register_func_for_add_image(
void (*func)(const struct mach_header *mh, intptr_t vmaddr_slide))
{
	DYLD_LOCK_THIS_BLOCK;
	typedef void (*callback_t)(const struct mach_header *mh, intptr_t vmaddr_slide);
    static void (*p)(callback_t func) = NULL;

	if(p == NULL)
	    _dyld_func_lookup("__dyld_register_func_for_add_image", (void**)&p);
	p(func);
}
```


这个注释写得够详细了(英文不太好)，就是当一个添加一个image 时，注册一个回调函数，当这个函数注册后，每个(image)镜像都会调用这个函数一次。也就是说，我们只要注册了这个回调函数，程序中的image 都会调用这个回调函数。这也是fishhook 能够替换函数的前提。



接下来就是如何找到函数并替换了；


```
static void rebind_symbols_for_image(struct rebindings_entry *rebindings,
                                     const struct mach_header *header,
                                     intptr_t slide) {
  Dl_info info;
  if (dladdr(header, &info) == 0) {
    return;
  }

  segment_command_t *cur_seg_cmd;
  segment_command_t *linkedit_segment = NULL;
  struct symtab_command* symtab_cmd = NULL;
  struct dysymtab_command* dysymtab_cmd = NULL;

  uintptr_t cur = (uintptr_t)header + sizeof(mach_header_t);
  for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
    cur_seg_cmd = (segment_command_t *)cur;
    if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
      if (strcmp(cur_seg_cmd->segname, SEG_LINKEDIT) == 0) {
        linkedit_segment = cur_seg_cmd;
      }
    } else if (cur_seg_cmd->cmd == LC_SYMTAB) {
      symtab_cmd = (struct symtab_command*)cur_seg_cmd;
    } else if (cur_seg_cmd->cmd == LC_DYSYMTAB) {
      dysymtab_cmd = (struct dysymtab_command*)cur_seg_cmd;
    }
  }

  if (!symtab_cmd || !dysymtab_cmd || !linkedit_segment ||
      !dysymtab_cmd->nindirectsyms) {
    return;
  }

  // Find base symbol/string table addresses
  uintptr_t linkedit_base = (uintptr_t)slide + linkedit_segment->vmaddr - linkedit_segment->fileoff;
  nlist_t *symtab = (nlist_t *)(linkedit_base + symtab_cmd->symoff);
  char *strtab = (char *)(linkedit_base + symtab_cmd->stroff);

  // Get indirect symbol table (array of uint32_t indices into symbol table)
  uint32_t *indirect_symtab = (uint32_t *)(linkedit_base + dysymtab_cmd->indirectsymoff);

  cur = (uintptr_t)header + sizeof(mach_header_t);
  for (uint i = 0; i < header->ncmds; i++, cur += cur_seg_cmd->cmdsize) {
    cur_seg_cmd = (segment_command_t *)cur;
    if (cur_seg_cmd->cmd == LC_SEGMENT_ARCH_DEPENDENT) {
      if (strcmp(cur_seg_cmd->segname, SEG_DATA) != 0 &&
          strcmp(cur_seg_cmd->segname, SEG_DATA_CONST) != 0) {
        continue;
      }
      for (uint j = 0; j < cur_seg_cmd->nsects; j++) {
        section_t *sect =
          (section_t *)(cur + sizeof(segment_command_t)) + j;
        if ((sect->flags & SECTION_TYPE) == S_LAZY_SYMBOL_POINTERS) {
          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
        }
        if ((sect->flags & SECTION_TYPE) == S_NON_LAZY_SYMBOL_POINTERS) {
          perform_rebinding_with_section(rebindings, sect, slide, symtab, strtab, indirect_symtab);
        }
      }
    }
  }
}

```

对于这部分代码，我们得先看看其他的，再来解释。



<h6> Dl_info </h6>
```
/*
* Structure filled in by dladdr().
*/
typedef struct dl_info {
        const char      *dli_fname;    /* Pathname of shared object */
        void            *dli_fbase;    /* Base address of shared object */
        const char      *dli_sname;    /* Name of nearest symbol */
        void            *dli_saddr;    /* Address of nearest symbol */
} Dl_info;
```

通过dladdr 获取头部信息，判断是否是正确的可执行文件。



*  fname: 共享对象的路径，即framework 的加载路径。如：
```
7D69BB8F-8AB9-3AB1-ADD6-BACB312CE32D 0x0000000103b25000 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk//System/Library/Frameworks/Foundation.framework/Foundation
```


*  dli_fbase: 共享对象的起始地址，即framework的加载地址。如：0x0000000103b25000。
* dli_saddr: 符号的地址。
* dli_sname:符号的名字。


<h5>符号表</h5>
符号表是内存地址与函数名、文件名、行号的映射表。符号表元素如下所示：



<起始地址> <结束地址> <函数> [<文件名：行号>]



<h5>Mach-O可执行文件</h5>


mach-o 格式是OS X系统上的可执行文件格式，类似于Windows的PE与Linux的ELF，每个Mach-o文件都包含一个mach-o 头，然后是载入命令（Load Commands），最后是数据块（data）。



Mach-O 文件的格式如下图所示：



![](/images/blog/im1.jpg)







