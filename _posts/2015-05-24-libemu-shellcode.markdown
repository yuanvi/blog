---
published: true
title: 基于libemu的shellcode检测(2)
layout: post
tags: [shellcode, 源码分析, 汇编, 虚拟化]
categories: [C/C++, 安全]
---

### EMU_MEMORY实现

接上篇，记录虚拟内存的实现。libemu中用了一个三维数组来模拟了分页内存中的目录->页表->内存区域的结构，默认设置中，每页内存的大小为2 ^ 12 = 4K，页表的大小为2 ^ 10 = 1024，那么页表数就是2 ^ (32 - 12 - 10) = 1024个，总共可以使用2 ^ 32 = 4G的内存。初始化时，使用emu_memory结构体中的pagetable字段来指向页表首地址，类似于CPU中的CR3寄存器。

```c++
struct emu_memory
{
    struct emu *emu;
    void ***pagetable;  // CR3
    
    uint32_t segment_offset;
    enum emu_segment segment_current;
    
    uint32_t segment_table[6];

    bool read_only_access;
    
    struct emu_breakpoint *breakpoint;
};
```

libemu并没有在初始化的时候申请4G的内存空间，仅是初始化了pagetable指向的1024个页表指针，并全部将它们置为0，在真正写数据的时候，才根据需要进行malloc，并且提供了三个宏来将一个写入地址转换为对应的页地址。

``` c++
#define PAGE_BITS 12 /* size of one page, 2^12 = 4096 */
#define PAGESET_BITS 10 /* number of pages in one pageset, 2^10 = 1024 */

#define PAGE_SIZE (1 << PAGE_BITS)
#define PAGESET_SIZE (1 << PAGESET_BITS)

// translate_addr macro
#define PAGESET(x) ((x) >> (PAGESET_BITS + PAGE_BITS))          /* remains highest 12 bits, means index of pagesets */
#define PAGE(x) (((x) >> PAGE_BITS) & ((1 << PAGESET_BITS) - 1))   /* remains 10 to 20 bits, means index of pages */
#define OFFSET(x) (((1 << PAGE_BITS) - 1) & (x))        /* lowest 10 bits, offset of page start address */
```

emu_memory结构体中大小为6的的segment_table字段，存储的是6个段选择器的值，在初始化的时候，FS值的被设置为0x7ffdf000，这个地址在较老的windows系统中直接存储了PEB结构，新的版本里这个结构地址做了随机化，但可以通过FS指向的TEB结构0x30处的值取到：``` mov eax, [FS:0x30]; mov PEB, eax```。

### shellcode检测

shellcode的具体信息不再详述，通过libemu来判定一段二进制数据是否是shellcode可以从三个方面来进行：

1.代码是否有自定位行为。通常来讲，一个实现得比较漂亮的，需要实现复杂功能的shellcode都是需要知道自己在内存中的位置，常用的有两种方式，call/pop和fstenv。检测可以通过判断调用的指令、监视栈顶或寄存器中的值是否落在shellcode的内存区间上等方式实现，这些对可以轻易获取cpu和内存状态的libemu来说都非常简单。

2.判断代码调用的API。libemu本身带了非常简单的hook机制，想监视某个API调用仅需要自己实现hook函数并修改定义的导出表结构。在某些情况下，如果eip指针指向了dll地址空间，就可以直接判定它是恶意的。比如说上篇文章中的简单shellcode：

```
STEP   0 : 0x00401000   8BEC                            mov ebp,esp
STEP   1 : 0x00401002   6865786520                      push dword 0x20657865
STEP   2 : 0x00401007   68636D642E                      push dword 0x2e646d63
STEP   3 : 0x0040100c   8D45F8                          lea eax,[ebp-0x8]
STEP   4 : 0x0040100f   50                              push eax 
STEP   5 : 0x00401010   B88D15867C                      mov eax,0x7c86158d
STEP   6 : 0x00401015   FFD0                            call eax 
STEP   7 : 0x7c86158d   0000                            add [eax],al
[debug] eip 7c86158d in kernel32
```

3.二进制是否可执行。某些情况下进行shellcode判定，正常的数据来源不应该出现可以被反汇编的数据，如果对某段数据模拟执行到了一定的步数，也可以认为它是恶意的。当然，这种情况需要做花指令剔除，防止误判。

附测试代码，供参考：

```c
int emu_test(char * buf, int len) {
    struct emu * e;
    struct emu_cpu * cpu;
    struct emu_env * env;

    e = emu_new();
    cpu = emu_cpu_get(e);

    // 初始化栈指针，向下增长，大小为1M
    tmp = (char *)malloc(0x10000);
    emu_memory_write_block(emu_memory_get(e), 0x100000, tmp, 0x10000);
    emu_cpu_reg32_set(cpu, esp, 0x100000);
    emu_cpu_reg32_set(cpu, ebp, 0x100000);
    free(tmp);

    // 在没开启ASLR的情况下，程序.text段首地址为0x401000，把shellcode填充在这个地方，并且设置eip
    emu_memory_write_block(emu_memory_get(e), 0x401000, buf, len);
    emu_cpu_eip_set(cpu, 0x401000);

    // 初始化环境，加载库文件到内存中
    env = emu_env_new(e);

    while (emu_cpu_parse(cpu) == 0) {

        // 做检测判断

        if (emu_cpu_step(cpu) != 0) {
            printf("[ERROR] emu_cpu_step, errno = %d, err desc : %s\n", emu_errno(e), emu_strerror(e));
            return -1;
        }
    }
    return 0;
}
```
