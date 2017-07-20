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



<h5> Dl_info </h5>


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



* fname: 共享对象的路径，即framework 的加载路径。如：


```
	7D69BB8F-8AB9-3AB1-ADD6-BACB312CE32D 0x0000000103b25000 /Applications/	Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/	SDKs/iPhoneSimulator.sdk//System/Library/Frameworks/Foundation.framework/	Foundation
```
	
	
* dli_fbase: 共享对象的起始地址，即framework的加载地址。如：0x0000000103b25000。
* dli_saddr: 符号的地址。
* dli_sname:符号的名字。



<h5>Mach-O可执行文件</h5>

mach-o 格式是OS X系统上的可执行文件格式，类似于Windows的PE与Linux的ELF，每个Mach-o文件都包含一个mach-o 头，然后是载入命令（Load Commands），最后是数据块（data）.

Mach-O 文件的格式如下图所示:


![](/images/blog/852671-9fde036a1ce9d902.jpg)



<h5>Header 的结构</h5>
通过Mach-O的头部，可以快速确认一些信息，比如当前文件用于32位还是64位。当前文件是fat文件 还是thin文件。下面是Mach-O头部的定义：



```
/*
 * The 32-bit mach header appears at the very beginning of the object file for
 * 32-bit architectures.
 */
struct mach_header {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
};

/* Constant for the magic field of the mach_header (32-bit architectures) */
#define	MH_MAGIC	0xfeedface	/* the mach magic number */
#define MH_CIGAM	0xcefaedfe	/* NXSwapInt(MH_MAGIC) */

/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */

```



注释很详细，也很容易看懂，只是有一个reserved 字段，是64位特有的保留字段。



如果还不明确，可以使用otool 或者MachOView 查看:




```
$ otool -h AlipayWallet
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedface      12          9  0x00           2    75       7580 0x00010085
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    75       8400 0x00210085
 
```

从以上结果可以知道，输出两个header ，表明这是一个fat 文件。在machine.h 文件中可以找到定义


* magic 值是0xfeedface 表示该二进制支持32位
* cputype 值12 表示arm 可以看看定义：



	```
	#define CPU_TYPE_ARM		((cpu_type_t) 12)
	```

* cpusubtype 值 9 表示 v7


	```
	#define CPU_SUBTYPE_ARM_V7		((cpu_subtype_t) 9)
	```
* magic 值0xfeedfacf 表示支持64位
* cputype 值 16777228 先看看定义
	
	
	```
	#define CPU_ARCH_ABI64	0x01000000	/* 64 bit ABI */
	#define CPU_TYPE_ARM		((cpu_type_t) 12)
	#define CPU_TYPE_ARM64          (CPU_TYPE_ARM | CPU_ARCH_ABI64)
	```
	(CPU_TYPE_ARM | CPU_ARCH_ABI64)  对应的十进制位 16777228
	
* cpusubtype 值 0


	```
	/*
	*  ARM64 subtypes
	*/
	#define CPU_SUBTYPE_ARM64_ALL           ((cpu_subtype_t) 0)
	#define CPU_SUBTYPE_ARM64_V8            ((cpu_subtype_t) 1)
	```
	
* filetype 值  2，表示MH_EXECUTE，代表可执行文件


	```
	#define	MH_OBJECT	0x1		/* relocatable object file */
	#define	MH_EXECUTE	0x2		/* demand paged executable file */
	#define	MH_FVMLIB	0x3		/* fixed VM shared library file */
	#define	MH_CORE		0x4		/* core file */
	#define	MH_PRELOAD	0x5		/* preloaded executable file */
	#define	MH_DYLIB	0x6		/* dynamically bound shared library */
	#define	MH_DYLINKER	0x7		/* dynamic link editor */
	#define	MH_BUNDLE	0x8		/* dynamically bound bundle file */
	#define	MH_DYLIB_STUB	0x9		/* shared library stub for static */
					/*  linking only, no section contents */
	#define	MH_DSYM		0xa		/* companion file with only debug */
					/*  sections */
	#define	MH_KEXT_BUNDLE	0xb		/* x86_64 kexts */
	```

* flags 定义太多了，就不贴代码了。就贴其中几个


	```
	#define MH_TWOLEVEL	0x80		/* the image is using two-level name
					   space bindings */
	#define	MH_PIE 0x200000			/* When this bit is set, the OS will
					   load the main executable at a
					   random address.  Only used in
					   MH_EXECUTE filetypes. */
	```
	
	
	
<h5>ASLR</h5>
ASLR（Address Space Layout Randomization）：地址空间布局随机化，镜像会在随机的地址上加载。这其实是一二十年前的旧技术了。

进程每一次启动，地址空间都会简单地随机化。

对于大多数应用程序来说，地址空间随机化是一个和他们完全不相关的实现细节，但是对于黑客来说，它具有重大的意义。

如果采用传统的方式，程序的每一次启动的虚拟内存镜像都是一致的，黑客很容易采取重写内存的方式来破解程序。采用ASLR可以有效的避免黑客攻击。

当然，你也可以将其去掉，在这篇文章中会教你怎么做.\:[http://codedigging.com/blog/2016-04-27-debugging-ios-binaries-with-lldb/](http://codedigging.com/blog/2016-04-27-debugging-ios-binaries-with-lldb/)


<h5>二级名称空间</h5>
The two-level namespace feature of OS X v10.1 and later adds the module name as part of the symbol name of the symbols defined within it. This approach ensures a module’s symbol names don’t conflict with the names used in other modules.

为了避免不同module 之间符号冲突而在OSX 10.1 以后引入的一项技术。与其对应的是平坦名称空间。





<h5>符号表</h5>
符号表是内存地址与函数名、文件名、行号的映射表。符号表元素如下所示：



&lt;起始地址&gt; &lt;结束地址&gt; &lt;函数&gt; [&lt;文件名：行号&gt;]



![](/images/blog/852671-9fde036a1ce9d902.jpg)



