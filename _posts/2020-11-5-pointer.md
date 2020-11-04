本周进度：

1.汇编中指针及多级指针的实现

2.一些简单逆向题目的复现

```c
1、列出每一行的反汇编代码：			
			
char a = 10;			
short b = 20;			
int c = 30;			
			
char* pa = &a;			
short* pb = &b;			
int* pc = &c;			
			
char** ppa = &pa;			
short** ppb = &pb;			
int** ppc = &pc;
```

```assembly
;1

mov byte ptr [ebp-4], A
mov word ptr[ebp-8], 14
mov dword[ebp-c] 1E

lea eax, dword ptr [ebp-4]
mov dword ptr [ebp-10], eax
lea eax dword ptr [ebp-8]
mov dword ptr [ebp-14], eax
lea eax, dowrd ptr [ebp-c]
mov dword ptr [ebp-18] eax

lea eax, dword ptr [ebp-10]
mov dowrd ptr [ebp-1C], eax
lea eax, dword ptr [ebp-14]
mov dword ptr [ebp-20], eax
lea eax, dword ptr [ebp-18]
mov dword ptr [ebp-24], eax
```

```c
2、列出每一行的反汇编代码：			
			
int p = 10;			
			
int******* p7;			
int****** p6;			
int***** p5;			
int**** p4;			
int*** p3;			
int** p2;			
int* p1;			
			
p1 = &p;			
p2 = &p1;			
p3 = &p2;			
p4 = &p3;			
p5 = &p4;			
p6 = &p5;			
p7 = &p6;		
```

```assembly
;2
mov dword ptr [ebp-4], A	;int p = 10;
lea eax, dword ptr [ebp-4]	;&p
mov dword ptr [epb-8], eax	;p1 = &p
lea eax, dword ptr [ebp-8]	;&p1
mov dword ptr [ebp-C], eax	;p2 = &p1
lea eax, dword ptr [ebp-C]	;&p2
mov dword ptr [ebp-10], eax	;p3 = &p2
;...

```

攻防世界csaw2013reversing2的复现

昨天看了wp，今天独立复现一下

题目描述：听说运行就能拿到Flag，不过菜鸡运行的结果不知道为什么是乱码

运行程序：

![1](https://tva2.sinaimg.cn/large/007s2Gz9ly1ghoca8ihiaj30xh0hoaai.jpg)

发送到IDA，F5看一下伪代码

```c
int __cdecl __noreturn main(int argc, const char **argv, const char **envp)
{
  int v3; // ecx
  CHAR *lpMem; // [esp+8h] [ebp-Ch]
  HANDLE hHeap; // [esp+10h] [ebp-4h]

  hHeap = HeapCreate(0x40000u, 0, 0);
  lpMem = (CHAR *)HeapAlloc(hHeap, 8u, MaxCount + 1);
  memcpy_s(lpMem, MaxCount, &unk_409B10, MaxCount);
  if ( sub_40102A() || IsDebuggerPresent() )
  {
    __debugbreak();
    sub_401000(v3 + 4, lpMem);
    ExitProcess(0xFFFFFFFF);
  }
  MessageBoxA(0, lpMem + 1, "Flag", 2u);
  HeapFree(hHeap, 0, lpMem);
  HeapDestroy(hHeap);
  ExitProcess(0);
}
```

从if语句看，`if ( sub_40102A() || IsDebuggerPresent() )`，括号内有一个值为1，就执行if语句中的内容，而`sub_40102A()`函数的返回值恒为0，所以它对if语句是否执行不产生影响

**一些知识储备：**

1.`IsDebuggerPresent() `函数读取当前进程的PEB里`BeingDebugged`的值（值为1，正在被调试）用于判断程序是否处于调试状态

2.`__debugbreak()`：中断程序

```c
main() 
{
   __debugbreak();
}
```

is similar to:

```c
main() 
{
   __asm 
   {
      int 3
   }
}
```



**流程分析：**

我们可以看到，如果直接运行程序，即跳过if语句，MessageBox的flag是乱码，那么，想要得到flag，应该是要让if语句中的解密函数执行：

- 让if语句执行：修改`sub_40102A()`附近的反汇编，直接jmp；

- 将`__debugbreak`nop掉；

- 执行`sub_401000(v3 + 4, lpMem);`后，直接跳到`MessageBox`，打印解密后的flag。

  

具体操作就不放上来了，用IDA打补丁也比较方便。

**本题学到的一些知识：**

PEB结构

```c++
typedef struct _PEB 
{
  BYTE                          Reserved1[2];
  BYTE                          BeingDebugged;
  BYTE                          Reserved2[1];
  PVOID                         Reserved3[2];
  PPEB_LDR_DATA                 Ldr;
  PRTL_USER_PROCESS_PARAMETERS  ProcessParameters;
  PVOID                         Reserved4[3];
  PVOID                         AtlThunkSListPtr;
  PVOID                         Reserved5;
  ULONG                         Reserved6;
  PVOID                         Reserved7;
  ULONG                         Reserved8;
  ULONG                         AtlThunkSListPtr32;
  PVOID                         Reserved9[45];
  BYTE                          Reserved10[96];
  PPS_POST_PROCESS_INIT_ROUTINE PostProcessInitRoutine;
  BYTE                          Reserved11[128];
  PVOID                         Reserved12[1];
  ULONG                         SessionId;
} PEB, *PPEB;
```

我们暂时只需要重点关注第二个`BeingDebugged`

