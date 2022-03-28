# x64的一些特性

### 寄存器

首先是寄存器数值的宽度，除去标志寄存器和段寄存器(段寄存器长度均为96位，其中有16位的可见部分和80位的不可见部分)外，均为8个字节64位宽度，在32位平台的基础上，增加了R8~R15共8个寄存器  

[<img src="/images/assets/post2/page1.PNG" width="100%"/>](/images/assets/post2/page1.PNG)  

* 易变寄存器:RAX,RCX,RDX,R8-R11
* 非易变寄存器：RBP,RBX,RSI,RDI,R12-R15

### FS与GS

32位的操作系统中，FS寄存器在3环执向TEB结构体，0环指向KPCR结构体，在X64中，则是使用GS寄存器  

[<img src="/images/assets/post2/page2.PNG" width="100%"/>](/images/assets/post2/page2.PNG)  

微软提供了IA32_FS_BASE(0xC0000100)和IA32_GS_BASE MSR(0xC0000101)寄存器用于获取Base,因为如果用段描述的方式去获取Base的话，在64位的段描述符中，是没法再放入64位的地址  

[<img src="/images/assets/post2/page3.PNG" width="100%"/>](/images/assets/post2/page3.PNG)  

FS则继续给Wow64提供服务，也就是32位的应用程序

### 内存布局与内存模型

x64下内存分配，理论可以使用2^64次大小的内存空间，但是实际操作系统设计时只用了其中的48位，x64采用了9-9-9-9-12分页，也就是PML4T分页，按照32位操作系统非PAE分页的套路，也即10-10-12分页，PDE-PTE-offset，总的内存1024x1024x4k=4Gb，其中用户和内核各使用了2GB,在x64下理论上支持，512x512x512x512x4k=256Tb，实际上windows支持16TB左右的空间
其中:
* 用户空间:0x00000000~00000000 --- 0x000007fff~ffffffff
* 内核空间:0xfffff800~00000000 --- 0xffffffff~ffffffff

IA-32架构下主要内存模型为:Basic Flat Module(基本的平坦模型)，Protect Flat Module(受保护的平坦模式)，Multi-Segment Module(分段内存模型)

* Basic Flat Module(基本的平坦模型)  
  
  最为简单的一种模型，操作系统和应用程序可以访问不受限制的内存空间，没有任何的分段机制，根据inter开发手册所说，一个最简单的平坦模型，至少建立两个段描述符，一个数据，一个代码，分别指向了数据段，代码段，但是都被映射到基址为0，限长为4gb的内存空间，即便访问的地址超过了实际物理内存的最大地址，也不会报错  

 [<img src="/images/assets/post2/page4.PNG" width="100%"/>](/images/assets/post2/page4.PNG)  

* Protect Flat Module(受保护的平坦模式)
  
  主要是在基本的平坦模型中，提供了内存访问保护，超出界限后会出发保护异常，同样有了特权级机制的段权限分配，也即用户模式和内核模式，这些段的基址都是从0开始，也就是说这些段在内存空间上来说都是重叠的，逻辑上划分为不同的用途  

[<img src="/images/assets/post2/page5.PNG" width="100%"/>](/images/assets/post2/page5.PNG)  

* Multi-Segment Module(分段内存模型)
  
  相较于前两种，稍稍复杂一点，这种对于不同用途的段在内存划分上也分离开来，实现真正的内存分段，对不同段，如代码段，数据段，内存访问采用seg base+offset的形式，采用了强制的硬件级的保护机制，每个应用程序可以有自己的私有段，也可以和其他程序共享，同样的，段权限检查和访问控制机制，都可以阻止如内存越界等非法访问操作  

[<img src="/images/assets/post2/page6.PNG" width="100%"/>](/images/assets/post2/page6.PNG)  

在x64中，根据inter开发手册:  

` In 64-bit mode, segmentation is generally (but not completely) disabled, creating a flat 64-bit linear-address  space. The processor treats the segment base of CS, DS, ES, SS as zero, creating a linear address that is equal to  the effective address. The FS and GS segments are exceptions. These segment registers (which hold the segment  base) can be used as additional base registers in linear address calculations. They facilitate addressing local data  and certain operating system data structures.`

CS, DS, ES, SS的段基址全部为0,base全部为0，但是FS和GS除外，它们的base可以通过前面说过的MSR寄存器获得
x64中，中断描述符和TSS段描述符均变为了128位，代码段和数据段描述符依然是64位
### 调用约定

X64的调用约定使用4寄存器的fast-call约定，整数参数在寄存器RCX,RDX,R8,R9,尽管使用了寄存器传参，但是系统还是会分配堆栈空间，而且堆栈平衡是由调用者完成
```c++
VOID x64_go(int a, int b, int c, int d) 
{
	printf("a:%d\n", a);
	printf("b:%d\n", b);
	printf("c:%d\n", c);
	printf("d:%d\n", d);

}


VOID main() 
{
	
	int a = 1;
	int b = 2;
	int c = 3;
	int d = 4;
	x64_go(a, b, c, d);
	printf("over\n");
	system("pause");
  
}

```
这段测试代码中，调用函数时所对应的汇编是这样的
```asm
mov         dword ptr [a],1  

mov         dword ptr [b],2  

mov         dword ptr [c],3  

mov         dword ptr [d],4  

mov         r9d,dword ptr [d]  
mov         r8d,dword ptr [c]  
mov         edx,dword ptr [b]  
mov         ecx,dword ptr [a]  
call        x64_go

```
可以看到当我们的参数长度不超过4个字节的时候，分别使用32位的寄存器进行传参，这个是编译器做的处理，长度足够存储数据的情况下，优先使用32位寄存器，因为操作64位寄存器(如MOV指令)的指令占4个字节，多一个48前缀指令，说明操作数是64位的，而32位寄存器只需要3个字节的指令长度

然后是堆栈分配的问题，调用者在调用子函数之前，根据函数参数的总长度，分配堆栈空间，这个堆栈空间至少要能容纳Regs*8，也就是所需保存的寄存器的总长度，如果有局部变量，则需要额外分配，并且要确保，在进入函数的时候，RSP要0x10对齐，这个堆栈空间用来进入子函数后，保存这些寄存器，以免后续可能会用到这些寄存器，从而改变导致后续出现异常

如果我想调用一个4个参数的函数，大概应该就是这样
```c++
  VOID x64_go2(int a, int b, int c, int d)
  {
    printf("a:%d\n", a);
    printf("b:%d\n", b);
    printf("c:%d\n", c);
    printf("d:%d\n", d);


  }

  test1 proc

      mov rcx,1
      mov rdx,2
      mov r8,3
      mov r9,4
      sub rsp,30h
      call x64_go2
      add rsp,30h
      ret

  test1 endp
```

#### 栈帧
x86函数下，一般以标志性的 push ebp,mov ebp,esp,sub esp,xxxx 三连起始,其中通过ebp被叫为帧指针，esp为栈指针，前者对应栈底，后者对应栈顶，而一般进行如局部变量访问，参数访问等，都是通过ebp来找的，例如一般情况下ebp+4是返回地址，ebp+8是第一个参数，函数堆栈图大概如下这样，从上到下地址逐渐减小  

[<img src="/images/assets/post2/page7.PNG" width="60%"/>](/images/assets/post2/page7.PNG)  

x64下使用RSP即为栈指针也为帧指针，所有的操作以RSP为基准进行，类似pop和push等可以更改RSP的操作，一般都在函数开始和尾部进行，RSP在同一个函数体内，保持不变

```c++
  VOID x64_go(int a, int b, int c)
  {
    printf("a:%d\n", a);
    printf("b:%d\n", b);
    printf("c:%d\n", c);
  }
  VOID main()
  {

    int a = 1;
    int b = 2;
    int c = 3;
    x64_go(a, b, c);
  }
```
上面的代码，对应如下的一个过程:  

[<img src="/images/assets/post2/page8.PNG" width="70%"/>](/images/assets/post2/page8.PNG)  

至于堆栈布局，其实和x86的是类似的
```c++
	VOID x64_go(ULONG64 a, ULONG64 b, ULONG64 c)
	{
		int x = 111;
		printf("a:%llx\n", a);
		printf("b:%llx\n", b);
		printf("c:%llx\n", c);



	}
	VOID main()
	{
		ULONG64 a = 1;
		ULONG64 b = 2;
		ULONG64 c = 3;		
		ULONG64 d = 4;
		x64_go(a, b, c);
		system("pause");

	}

```
把核心的汇编代码抠出来，用来做堆栈结构图
```asm
  sub     rsp, 48h
  mov     dword ptr [rsp+30h], 1
  mov     dword ptr [rsp+28h], 2
  mov     dword ptr [rsp+20h], 3
  mov     dword ptr [rsp+38h], 4
  mov     r8d, [rsp+20h]
  mov     rdx, [rsp+28h]
  mov     rcx, [rsp+30h]
  call    x64_go
  lea     rcx, Command    ; "pause"
  call    cs:system
  xor     eax, eax
  add     rsp, 48h
  retn

x64_go:

  mov     [rsp+18h], r8
  mov     [rsp+10h], rdx
  mov     [rsp+8h], rcx
  sub     rsp, 38h
  mov     dword ptr [rsp+20h], 6Fh ; 'o'
  mov     rdx, [rsp+40h]
  lea     rcx, aAD        ; "a:%d\n"
  call    sub_1400010B0
  mov     rdx, [rsp+48h]
  lea     rcx, aBD        ; "b:%d\n"
  call    sub_1400010B0
  mov     rdx, [rsp+50h]
  lea     rcx, aCD        ; "c:%d\n"
  call    sub_1400010B0
  add     rsp, 38h
  retn
```
堆栈结构如下:  

[<img src="/images/assets/post2/page9.PNG" width="60%"/>](/images/assets/post2/page9.PNG)  

### X64分页机制

x64分页和32位系统下的PAE分页分页是相似的，在原有的3级页映射的基础上，多了PML4T(4级页映射表)，因为x64使用了48位的虚拟地址，根据地址转换过程中虚拟地址的对应关系，也叫做9-9-9-9-12分页  

[<img src="/images/assets/post2/page10.PNG" width="100%"/>](/images/assets/post2/page10.PNG)  

以虚拟地址0x13fb7ff98为例，此处存了一个自定义字符串,因为实际使用了48位虚拟地址，所以最终拆分的地址为0x00013fb7ff98  

[<img src="/images/assets/post2/page11.PNG" width="100%"/>](/images/assets/post2/page11.PNG)  

拆分后得到9-9-9-9-12结构为

```h
0000 0000 0   0
0000 0010 0   0x4
1111 1110 1   0x1fd 
1011 1111 1   0x17f
1111 1001 1000  0xf98
```
在手动查找物理地址时，要注意高于48位以及低12位清0，这些是属性和保留位，只有中间的部分12-48位为PFN页帧，除了最后的12位页面偏移，其他的都要乘以8，因为每个PDE，PTE等页目录项都是8个字节为单位
```h
0: kd> !dq 130fd1000
#130fd1000 03000001`26616867 00000000`00000000
#130fd1010 00000000`00000000 00000000`00000000
#130fd1020 00000000`00000000 00000000`00000000
#130fd1030 00000000`00000000 00000000`00000000
#130fd1040 00000000`00000000 00000000`00000000
#130fd1050 00000000`00000000 00000000`00000000
#130fd1060 00000000`00000000 00000000`00000000
#130fd1070 00000000`00000000 00800001`2f312867
0: kd> !dq 00000001`26616000+0x4*8
#126616020 03100001`24a17867 00000000`00000000
#126616030 00000000`00000000 00000000`00000000
#126616040 00000000`00000000 00000000`00000000
#126616050 00000000`00000000 00000000`00000000
#126616060 00000000`00000000 00000000`00000000
#126616070 00000000`00000000 00000000`00000000
#126616080 00000000`00000000 00000000`00000000
#126616090 00000000`00000000 00000000`00000000
0: kd> !dq 00000001`24a17000+0x1fd*8
#124a17fe8 05200001`24818867 00000000`00000000
#124a17ff8 00000000`00000000 006f004c`00740076
#124a18008 00650052`002e0067 00720075`006f0073
#124a18018 00220073`00650063 00720065`00760020
#124a18028 006e006f`00690073 002e0036`0022003d
#124a18038 00360037`002e0031 0032002e`00310030
#124a18048 00320037`00300033 00720070`00200022
#124a18058 00730065`0063006f 00410072`006f0073
0: kd> !dq 00000001`24818000+0x17f*8
#124818bf8 9e100000`bdfba005 b2300000`71cf0005
#124818c08 b2400000`73af1005 95600001`2e45d005
#124818c18 a0000001`308d6005 b2500000`71df2005
#124818c28 b2600001`064f3005 b2700000`734f4005
#124818c38 b2800001`2f2f5005 b2900000`70bf6005
#124818c48 b2a00000`b8ff7005 b2b00000`b9ff8005
#124818c58 b2c00001`227f9005 b2d00001`2defa005
#124818c68 b2e00001`2cefb005 b2f00000`ba5fc005
0: kd> !dq 00000000`bdfba000+0xf98
#bdfbaf98 21747365`74343678 00000000`00002121
#bdfbafa8 786c6c25`000a7325 73756170`0000000a
#bdfbafb8 00000000`00000065 00000001`3fb800a0
#bdfbafc8 00000001`3fb801b0 00000001`3fb80308
#bdfbafd8 00000001`3fb80330 00000001`3fb80370
#bdfbafe8 00000001`3fb803a8 00000000`00000001
#bdfbaff8 00000001`00000001 00000000`00000000
#bdfbb008 00000000`00000000 00000000`00000000
0: kd> !db 00000000`bdfba000+0xf98
#bdfbaf98 78 36 34 74 65 73 74 21-21 21 00 00 00 00 00 00 x64test!!!......
#bdfbafa8 25 73 0a 00 25 6c 6c 78-0a 00 00 00 70 61 75 73 %s..%llx....paus
#bdfbafb8 65 00 00 00 00 00 00 00-a0 00 b8 3f 01 00 00 00 e..........?....
#bdfbafc8 b0 01 b8 3f 01 00 00 00-08 03 b8 3f 01 00 00 00 ...?.......?....
#bdfbafd8 30 03 b8 3f 01 00 00 00-70 03 b8 3f 01 00 00 00 0..?....p..?....
#bdfbafe8 a8 03 b8 3f 01 00 00 00-01 00 00 00 00 00 00 00 ...?............
#bdfbaff8 01 00 00 00 01 00 00 00-00 00 00 00 00 00 00 00 ................
#bdfbb008 00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00 ................
0: kd> !vtop 130fd1000 0x13fb7ff98
Amd64VtoP: Virt 000000013fb7ff98, pagedir 0000000130fd1000
Amd64VtoP: PML4E 0000000130fd1000
Amd64VtoP: PDPE 0000000126616020
Amd64VtoP: PDE 0000000124a17fe8
Amd64VtoP: PTE 0000000124818bf8
Amd64VtoP: Mapped phys 00000000bdfbaf98
Virtual address 13fb7ff98 translates to physical address bdfbaf98.
```

Enjoy, Good Luck
