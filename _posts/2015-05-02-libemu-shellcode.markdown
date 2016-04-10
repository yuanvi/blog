---
published: true
title: 基于libemu的shellcode检测原理(1)
layout: post
tags: [shellcode, 源码分析]
categories: [C/C++, 安全]
---

### libemu介绍

libemu是一个32位的x86指令集模拟器，可以用来对一段二进制代码进行模拟执行，相当于用纯代码的方式模拟了32位机器的工作方式。在安全领域里通常被用来做shellcode检测，这里主要分析一下它的基本原理和实现，顺便简单介绍如何做用它来做shellcode检测。

原本其git地址是```git://git.carnivore.it/libemu.git```，写这篇文章的时候已经无法clone了，需要的可以去github上搜别人克隆的镜像。clone完毕后，源码目录结构如下，简单标注一下各个文件内容：

```
.
├── emu_breakpoint.c
├── emu.c
├── emu_cpu.c
├── emu_cpu_data.c
├── emu_getpc.c
├── emu_graph.c
├── emu_hashtable.c
├── emu_list.c
├── emu_log.c
├── emu_memory.c
├── emu_queue.c
├── emu_shellcode.c
├── emu_source.c
├── emu_stack.c
├── emu_string.c
├── emu_track.c
├── environment
│   ├── emu_env.c
│   ├── emu_profile.c
│   ├── linux
│   │   ├── emu_env_linux.c
│   │   └── env_linux_syscall_hooks.c
│   └── win32
│       ├── dlls
│       │   ├── advapi32dll.c
│       │   ├── kernel32dll.c
│       │   ├── msvcrtdll.c
│       │   ├── ntdll.c
│       │   ├── shdocvwdll.c
│       │   ├── shell32dll.c
│       │   ├── shlwapidll.c
│       │   ├── urlmondll.c
│       │   ├── user32dll.c
│       │   ├── wininetdll.c
│       │   └── ws2_32dll.c
│       ├── emu_env_w32.c
│       ├── emu_env_w32_dll.c
│       ├── emu_env_w32_dll_export.c
│       ├── env_w32_dll_export_kernel32_hooks.c
│       ├── env_w32_dll_export_msvcrt_hooks.c
│       ├── env_w32_dll_export_shdocvw_hooks.c
│       ├── env_w32_dll_export_shell32_hooks.c
│       ├── env_w32_dll_export_urlmon_hooks.c
│       └── env_w32_dll_export_ws2_32_hooks.c
├── functions
│   ├── aaa.c
│   ├── adc.c
│   ...
│   ├── xchg.c
│   └── xor.c
├── libdasm.c
├── libdasm.h
├── Makefile.am
└── opcode_tables.h
```

从源码结构就可以看出基本实现思路了，核心的两个功能是对cpu和内存的模拟，emu_cpu主要分析二进制内容并调用对应的模拟函数，emu_memory模拟内存供cpu读写，模拟了分页机制并且写时malloc，避免了消耗大量内存。

模拟CPU的实现主要在emu_cpu.c中，先判断宿主机的endian，然后初始化emu_cpu结构体中的寄存器指针，在cpu中寄存器的关系是这样的：eax -> ax -> (ah, al)，eax寄存器是32位，ax是eax的低16位，ah和al又分别是ax的高8位和低8位，所以要将寄存器关系对应上，就需要知道宿主机数据存储方式是litte endian还是big endian。

判断endian：

```c++
int i = 1;
if (*(uint_t*)&i == 1) {...} else {...}
```

emu_cpu结构体:

```c++
struct emu_cpu
{
    struct emu *emu;
    struct emu_memory *mem;
    
    uint32_t debugflags;

    uint32_t eip;
    uint32_t eflags;
    uint32_t reg[8];      // eax, ebx, ecx...
    uint16_t *reg16[8];   // ax, bx, cx..., just a point to reg[8] low 16bit
    uint8_t *reg8[8];     // al, bl, cl..., just a point to reg[8] low 8bit

    struct emu_instruction instr;
    struct emu_cpu_instruction_info *cpu_instr_info;
    
    uint32_t last_fpu_instr[2];

    char *instr_string;
    bool repeat_current_instr;

    struct emu_track_and_source *tracking;
};
```

看上面emu_cpu结构体，先抛开其他细节，看看它是怎么运行起来。这个结构体中，eip寄存器和cpu_instr_info这个结构体比较关键，eip寄存器的值是一个指向要执行指令的指针，而cpu_instr_info这个结构体里就记录了eip指向指令对应的处理函数。分析一下它的运行过程:


```
emu_cpu_run: while(emu_cpu_parse(c) == 0) { ... emu_cpu_step(c) ...}
                       ↓                              ↓
     emu_cpu_parse: while(1) {...}      emu_cpu_step: c->cpu_instr_info->function(c, &c->instr.cpu);
                               ↓
            ret = emu_memory_read_byte(c->mem, c->eip++, &byte);
            c->cpu_instr_info = &ii_onebyte[byte];                          
            ...
```

待续。。。
