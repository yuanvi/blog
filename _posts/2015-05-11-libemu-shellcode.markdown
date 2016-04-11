---
published: true
title: 基于libemu的shellcode检测(1)
layout: post
tags: [shellcode, 源码分析, 汇编, 虚拟化]
categories: [C/C++, 安全]
---

### libemu介绍

libemu是一个32位的x86指令集模拟器，可以用来对一段二进制代码进行模拟执行，相当于简单的用软件模拟了32位机器的工作方式，在安全领域里通常用来做shellcode检测。这里主要分析一下它的基本原理和实现，顺便简单介绍一下如何做用它来做shellcode检测。

原本其git地址是```git://git.carnivore.it/libemu.git```，写这篇文章的时候已经无法clone了，需要的可以去github上搜别人克隆的镜像。clone完毕后，源码目录结构如下，简单标注一下各个文件内容：

```
.
├── emu_breakpoint.c
├── emu.c
├── emu_cpu.c               // emu_cpu的实现
├── emu_cpu_data.c
├── emu_getpc.c
├── emu_graph.c
├── emu_hashtable.c
├── emu_list.c
├── emu_log.c
├── emu_memory.c            // emu_memory实现
├── emu_queue.c
├── emu_shellcode.c         // shellcode检测逻辑
├── emu_source.c
├── emu_stack.c
├── emu_string.c
├── emu_track.c
├── environment              // 操作系统环境，运行时在虚拟内存中填充各库文件的文件头，
│   ├── emu_env.c            // 这里主要关心的是各个库中的导出函数，实现了一些重要函数的hook函数
│   ├── emu_profile.c        
│   ├── linux
│   │   ├── emu_env_linux.c
│   │   └── env_linux_syscall_hooks.c
│   └── win32
│       ├── dlls        // 硬编码了相应dll文件的PE头
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
├── functions       //指令函数的实现
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

从源码结构就可以看出基本实现思路了，核心的两个功能是对cpu和内存的模拟，emu_cpu主要分析二进制内容并调用对应的指令函数，emu_memory模拟内存供cpu读写，实现了分页机制并且写时malloc，避免了消耗大量内存。

### EMU_CPU实现

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

看这个结构体，抛开其他细节，看看它是怎么运行起来。其中eip寄存器和cpu_instr_info这个结构体比较关键，eip寄存器的值是一个指向要执行指令的指针，而cpu_instr_info结构体里记录了eip指向指令对应的处理函数。从源码中理一下它的运行流程:

```
emu_cpu_run: while(emu_cpu_parse(c) == 0) { ... emu_cpu_step(c) ...}
                       ↓                              ↓
     emu_cpu_parse: while(1) {...}      emu_cpu_step: c->cpu_instr_info->function(c, &c->instr.cpu);
                               ↓
            ret = emu_memory_read_byte(c->mem, c->eip++, &byte);
            c->cpu_instr_info = &ii_onebyte[byte];                          
            ...
```

在emu_cpu_run这个函数的循环中，每次调用emu_cpu_parse去从内存中读取一个字节，并且eip++，通过用获取到的字节做索引从ii_onebyte数组中去取到对应的emu_cpu_instruction_info结构。然后在emu_cpu_step中调用cpu_instr_info->function模拟函数来进行计算。实际上emu_cpu_parse是一个近400行的函数，并不像分析中这样简单，还处理了多字节指令，寻址模式，浮点数计算指令等。参考文档：[指令结构和寻址方式](http://tnovelli.net/ref/opx86.html), [浮点数指令](http://www.website.masmforum.com/tutorials/fptute/)。

用一个非常简单的shellcode来跟踪一下它的运行流程，这段shellcode的作用是在WinXP Sp2环境下执行cmd.exe程序，因为shellcode中call指令的WinExec地址是硬编码的，所以它只能在特定的环境下工作:

```asm
/*
\x8b\xec\x68\x65\x78\x65\x20\x68\x63\x6d\x64\x2e
\x8d\x45\xf8\x50\xb8\x8D\x15\x86\x7C\xff\xd0
*/
00402000  > 8BEC             MOV EBP,ESP
00402002  . 68 65786520      PUSH 20657865     // 20657865 -> "exe "
00402007  . 68 636D642E      PUSH 2E646D63     // 2E646D63 -> "cmd."
0040200C  . 8D45 F8          LEA EAX,DWORD PTR SS:[EBP-8]
0040200F  . 50               PUSH EAX
00402010  . B8 8D15867C      MOV EAX,kernel32.WinExec  //hardcode 8D15867C -> WinExc
00402015  . FFD0             CALL EAX
```

取```PUSH 20657865```这条指令跟踪，从运行流程中知道，0x68对应的emu_cpu_instruction_info结构在ii_onebyte[0x68]，从代码中找到这个结构和对应的函数实现，可以看到instr_push_68这个函数模拟了push指令在cpu中的工作方式，调用emu_memory_write_dword将立即数写入了模拟内存的栈顶，并且修改了影响的寄存器esp，其他指令的执行流程也都类似。

```c++
// file: include/emu/emu_cpu_itables.h
struct emu_cpu_instruction_info ii_onebyte[0x100] = { 
    ...
    /* 68 */ {instr_push_68, "push", {0, 0, 0, II_IMM, 0, 0, 0, 0}},
    ...
}

// file: src/function/push.c
int32_t instr_push_68(struct emu_cpu *c, struct emu_cpu_instruction *i) 
{
    if ( i->prefixes & PREFIX_OPSIZE )
    {   
        PUSH_WORD(c, *i->imm16)
    }   
    else
    {   
        PUSH_DWORD(c, i->imm)
    }
    return 0;
}


// file: include/emu/emu_cpu_stack.h
#define PUSH_DWORD(cpu, arg)                            \
{                                                       \
    uint32_t pushme;                                    \
    bcopy(&(arg),  &pushme, 4);                         \
    if (cpu->reg[esp] < 4)                              \
    {                                                   \
        emu_errno_set((cpu)->emu, ENOMEM);              \
        emu_strerror_set((cpu)->emu,                    \
        "ran out of stack space writing a dword\n");    \
        return -1;                                      \
    }                                                   \
    cpu->reg[esp]-=4;                                   \
    {                                                                           \
        int32_t memret = emu_memory_write_dword(cpu->mem, cpu->reg[esp], pushme);   \
        if (memret != 0)                                                        \
            return memret;                                                      \
    }                                                                           \
}
```

libemu是一个比较古老的项目了，对浮点数计算指令只是简单支持，也仅支持大部分的正常指令，64位的代码更是不在考虑范围内，这在shellcode检测中明显是不够用的，实际测试中很多代码会因为遇上不支持的指令而退出，但分析其代码，对理解操作系统和虚拟化原理还是有很大帮助。接下来准备记录一下模拟内存实现，一些小细节和shellcode检测原理。
