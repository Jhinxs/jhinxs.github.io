# Intel VT x64 EPT Hook框架的实现

主要记录下X64VT以及实现EPT Hook时的一些关键点
### 基本的VT开启

这部分和之前32位的没什么太大的区别，主要是在填写vmc字段时，需要额外填写如IA32_EFER，IA32E_MODE这种必须要填写的，还有像RDTSCP这种在新的win10上面系统上面做引发VMEXIT需要填写的，这种主要是为了兼容性考虑。
开启EPT的话需要在SECONDARY VM execution Control Field字段里面填写对应的enable ept字段，一般来讲不是太老的CPU都是支持的。  
而在X64上多核处理的话，主要是使用KeGenericCallDpc来调用我们的VT初始化或者你需要的其他需要进行多核同步的过程，这个API就是让多个CPU同时以特殊的DPC例程执行我们的代码。  
其他就是一些数据类型，寄存器的更改，换成64位的模式

### EPT内存管理  

EPT是intel为内存虚拟化而设计的一种硬件机制，主要是是为了实现Guest对于物理内存的访问控制  

[<img src="/images/assets/post3/ept.PNG" width="100%"/>](/images/assets/post3/ept.PNG)
  
因为EPT的存在，Guest对一个地址访问时，每当访问一个物理地址，都会走EPT物理地址转化的过程，这个过程和x64下的VA->PA的转换过程类似，例如访问Guest访问某个进程物理地址，首先找到CR3，此时访问CR3中的物理地址，就会触发GPA->HPA的EPT转换过程，然后访问PML4，PDE等物理地址时，都会触发一个EPT的转换，这也是VT开启EPT后性能损耗的原因之一，关于EPT各个表项的内容可以查看参考 Intel-64-ia-32--system-programming-manual-vol-3 Chapter 28.2

#### EPT的构建
EPT的构建，感觉就是实现一个具体的地址转换的过程，从EPTP的构建，以及其中PML4,PDPT,PDT其中各个表项的各个字段的填写
```C++
    pEptState->EptPageTable = PageTable;
    PageTable->PML4[0].all = 0;
    PageTable->PML4[0].Bits.read_access = 1;
    PageTable->PML4[0].Bits.write_access = 1;
    PageTable->PML4[0].Bits.exec_access_supervisor = 1;
    PageTable->PML4[0].Bits.PDPTPFN = MmGetPhysicalAddress(&PageTable->PML3[0]).QuadPart >> 12;

    for (int i = 0; i < VMM_EPT_PML3E_COUNT; i++)
    {
        PageTable->PML3[i].all = 0;
        PageTable->PML3[i].Bits.read_access = 1;
        PageTable->PML3[i].Bits.write_access = 1;
        PageTable->PML3[i].Bits.exec_access_supervisor = 1;
        PageTable->PML3[i].Bits.PDTPFN = MmGetPhysicalAddress(&PageTable->PML2[i][0]).QuadPart >> 12;

    }
```
上面的代码中可以看到对各个EPT表项的构建主要是对其中的对应字段进行构建填写，RWX权限位的设置，以及下一级EPT页表的地址的取值填写，例如PML4T中的每个PML4项的PFN对应一张PDPT表,所以就在PFN处填写PDPT物理地址去除属性位后的其余部分。  
构建PDE以及后面的页表时，最开始我是想以4kB为基准构建的，但是后来初始化EPT一直出错（其实是其他的代码有问题），我就采取了想hvpp，gbhv等项目的采用2MB大页的方式来构建
```C++
    for (int i = 0; i < VMM_EPT_PML3E_COUNT; i++)
    {
        for (int j = 0; j < VMM_EPT_PML2E_COUNT; j++)
        {
            PageTable->PML2[i][j].all = 0;
            PageTable->PML2[i][j].Bits.ExecuteAccess = 1;
            PageTable->PML2[i][j].Bits.ReadAccess = 1;
            PageTable->PML2[i][j].Bits.WriteAccess = 1;
            PageTable->PML2[i][j].Bits.LargePage = 1;
            PageTable->PML2[i][j].Bits.PhyPagePFN = (i * VMM_EPT_PML2E_COUNT) + j;
            SetMemMtrrInfo(PageTable->PML2[i][j], (i * VMM_EPT_PML2E_COUNT) + j);
        }

    }
```
也就是说到了构建PDT表的时候，其中每一项PDE的PFN不再是PTT的地址了（正常4k的构建下每个PDE指向一张PTT），而且直接指向一张物理页，采取大页的方式的话，构建整个EPT就效率有一定提高，而且后来搜了相关的东西，大页有助于减少缺页中断，TLB缓存未命中发生的次数。
#### MTRR
MTRR的全称是Memory Type Range Registers，意思就是说一组用来指定特定内存段的内存类型的特殊寄存器,这里所说的内存类型:  


[<img src="/images/assets/post3/mtrr.PNG" width="100%"/>](/images/assets/post3/mtrr.PNG)

同样MTRR是否支持也看CPU，一般新一点的CPU都是支持的，操作系统启动时BIOS会通过MTRR设置对应的内存访问属性，通过设置MTRR可以提高内存的访问效率，同时如果想在物理机上运行EPT这也是不可少的一个步骤,EPT在初始化时，应当先去获取MTRR的信息，然后并对对应的内存范围进行设置。

获取主机的MTRR信息
```C++
    IA32_MTRR_CAPABILITIES_REGISTER MTRRCap;
    IA32_MTRR_PHYSBASE_REGISTER CurrentPhysBase;
    IA32_MTRR_PHYSMASK_REGISTER CurrentPhysMask;
    PMTRR_RANGE_DESCRIPTOR Descriptor;
    ULONG NumberOfBitsInMask;
    MTRRCap.all = __readmsr(MSR_IA32_MTRR_CAPABILITIES);
    for (int i = 0; i < MTRRCap.Bits.VariableRangeCount; i++)
    {
        CurrentPhysBase.all = __readmsr(MSR_IA32_MTRR_PHYSBASE0 + (i * 2));
        CurrentPhysMask.all = __readmsr(MSR_IA32_MTRR_PHYSMASK0 + (i * 2));
        if (CurrentPhysMask.Bits.Valid)
        {
            Descriptor = &pEptState->MemoryRanges[pEptState->NumberOfEnabledMemoryRanges++];
            Descriptor->PhysBaseAddress = CurrentPhysBase.Bits.PhysBase * PAGE_SIZE;
            _BitScanForward64(&NumberOfBitsInMask, CurrentPhysMask.Bits.PhysMask * PAGE_SIZE);
            Descriptor->PhysEndAddress = Descriptor->PhysBaseAddress + ((1ULL << NumberOfBitsInMask) - 1ULL);
            Descriptor->MemoryType = (UCHAR)CurrentPhysBase.Bits.Type;
            Descriptor->enable = CurrentPhysMask.Bits.Valid;
            if (Descriptor->MemoryType == MEMORY_TYPE_WRITE_BACK)
            {
                pEptState->NumberOfEnabledMemoryRanges--;
            }
            DbgPrintLog("[+] MTRR Range: Base=0x%llx End=0x%llx Type=0x%x\n", Descriptor->PhysBaseAddress, Descriptor->PhysEndAddress, Descriptor->MemoryType);
        }
       

    }
    DbgPrintLog("[+] MTRR Build Over\n");
    DbgPrintLog("[+] MTRR Ranges: %d\n", pEptState->NumberOfEnabledMemoryRanges);
    return TRUE;
```
IA32_MTRR_PHYSBASE_REGISTER MSR寄存器可以获取对应的物理内存起始位置以及内存类型  
IA32_MTRR_PHYSMASK_REGISTER MSR寄存器可以获取对应物理内存的范围以及是否有效  
设置MTRR对应的内存范围的内存属性
```C++
    ULONG64 pageaddress = phyaddr * LargePage_Size;
    ULONG64 memorytype_mtrr = MEMORY_TYPE_WRITE_BACK;
    if (phyaddr == 0)
    {
        pde_2M.Bits.MemoryType = MEMORY_TYPE_UNCACHEABLE;
        return;
    }
    for (int i = 0; i < pEptState->NumberOfEnabledMemoryRanges; i++)
    {
        if (pEptState->MemoryRanges[i].enable)
        {

            if (pageaddress <= pEptState->MemoryRanges[i].PhysEndAddress)
            {
                if (pageaddress + LargePage_Size - 1 >= pEptState->MemoryRanges[i].PhysBaseAddress)
                {
                    memorytype_mtrr = pEptState->MemoryRanges[i].MemoryType;
                    if (memorytype_mtrr == MEMORY_TYPE_UNCACHEABLE)
                    {
                        break;
                    }
                }
            }
        }
    }
    pde_2M.Bits.MemoryType = memorytype_mtrr;
```
#### EPT Violation
这个是实现我们EPT Hook的关键，在通常的VA->PA的访问过程中,存在一个TLB的机制，就是说操作系统会缓存经常访问的虚拟地址与物理地址的映射关系，从而提高访问效率，而TLB又可以分为数据TLB和指令TLB，也是就分别对应于读写和执行，在此基础上，就出现了一种ShadowWalker的技术，具体可以自行GOOGLE，将读写和执行的行为分离开来,当触发读写异常时,分配一个真实的页面，当触发执行异常时，分配假的页面，这样看起来代码时正常的，但是实际执行的时候，执行的是另一份代码，[Github](https://github.com/Jhinxs/ProtectedModeCode/blob/main/shadowwalker.cpp)  这是我之前跟着zhouhe学着写的一份SW代码通过修改缺页中断，实现简易的效果 。  
在EPT中当产生EPT Violation时将会发生VMexit，此时需要在VM HOST中进行事件的接管并处理EPT Violation事件，那么这个时候就可以对RWX行为做不同的处理。
```C++
     PPTE hookpagepte = EptGetPTEENTRY(pEptState->EptPageTable, phyalign);
     PEPT_FAKE_PAGE fake_page = GetFakePage(phy);
     if (fake_page ==NULL)
     {
         DbgPrintLog("[!] Error: Try To Fake Page For Hookitem Failed\n");
         return FALSE;
     }

     if (!PQ->ExecuteAble && PQ->Execute)
     {
         fake_page->OriginalEntryAddress->all = fake_page->FakeEntryForX.all;
         InveptSingleContext(pEptState->EptPointer.all);
         DbgPrintLog("[+] Set hookpage PFN = %llx exec access to 1\n", hookpagepte->Bits.PhyPagePFN);
         return TRUE;
     }
     if (PQ->ExecuteAble && (PQ->Read|PQ->Write))
     {
         fake_page->OriginalEntryAddress->all = fake_page->FakeEntryForRW.all;
         InveptSingleContext(pEptState->EptPointer.all);
         DbgPrintLog("[+] Set hookpage PFN = %llx read access to 1,write access to 1\n", hookpagepte->Bits.PhyPagePFN);
         return TRUE;
     }
```
以上的代码首先获取了触发异常的真实页面的PTE，然后根据不同的行为，分配不同的页面，InveptSingleContext这个函数
```C++
VOID InveptSingleContext(ULONG64 EptPointer)
{
	INVEPT_DESCRIPTOR Descriptor = { EptPointer ,0 };
	return vmx_invept( VEPT_SINGLE_CONTEXT, &Descriptor);
}
```
主要执行invept指令，该执行可以对GPA->HPA的转换产生的Cache进行刷新，而invept支持两种方式进行刷新：  
SINGLE_CONTEXT：刷新描述符中给定的EPTP对应的所有缓存。
ALL_CONTEXT：忽略描述符，刷新所有EPTP对应的所有缓存。

### SSDT
这个也没啥，就是涉及到SSDT的查找以及一些内核函数地址的查找
```C++
ULONG64* GetSSDTBase()
{
	//先不考虑  KVAS
	ULONG64 lstar = __readmsr(0xC0000082);
	for (int i = 0; i < 1024; i++)
	{
		if (*(PUCHAR)(lstar + i) == 0x4c && *(PUCHAR)(lstar + i + 2) == 0x15)
		{
			if (*(PUCHAR)(lstar + i + 7) == 0x4c && *(PUCHAR)(lstar + i + 9) == 0x1d)
			{
				if (*(PUCHAR)(lstar + i + 14) == 0xf7 && *(PUCHAR)(lstar + i + 15) == 0x43)
				{
					ULONG64 KiSystemServiceRepeat = (ULONG64)(PUCHAR)lstar + i;
					ULONG offset = *(ULONG*)(KiSystemServiceRepeat + 3);
					ULONG64 SSDTBase = KiSystemServiceRepeat + offset + 7; //7= lea r10,[nt!KeServiceDescriptorTable]
					return SSDTBase;
				}

			}
		}
	}
	
	return NULL;

}
ULONG64 GetNTAPIAddress() 
{
	int SyscallNumber = 0x002c;     //NtTerminateProcess = 0x002c;
	ULONG64* ssdt = GetSSDTBase();
	ULONG64 address = ((*(ULONG*)(*ssdt + SyscallNumber * 4)) >> 4) + *ssdt;
	return address;
}
```
这里只是未开启内核页表隔离的情况，开启内核页表隔离lstar msr读取到的是KiSystemCall64Shadow地址，可能需要从KiSystemCall64Shadow开始往上搜了。  
关于API HOOK，测试时对NtTerminateProcess做了Inline Hook的测试  

Hook的步骤和通常的Hook没啥区别，在API开始的几条指令找合适的位置插入一条跳转指令，跳到我们的代码，执行完我们的代码后，跳转回当时插入的指令的下一条指令，也就是按着原路继续执行。  

这里利用ETP需要将首先将原本的函数对应的物理页拷贝一份过来，然后对拷贝的EPT的页面做HOOK通过ETP Violation的处理，就可以欺骗读写行为，绕过PG等扫描，直接修改原本的物理页是不仅需要关闭写保护而且是没有意义的。  

```C++
	/*
	    12 hook bytes:
	    mov rax,target64
		JMP rax                     
	*/
	TargetBuffer[0] = 0x48;
	TargetBuffer[1] = 0xb8;
	*(ULONG64*)&TargetBuffer[2] = TargetAddress;
	TargetBuffer[10] = 0xFF;
	TargetBuffer[11] = 0xE0;
```
这段代码就是构建简单的跳转指令，也可以使用PUSH，RET这种方式实现跳转  

```h
2: kd> u 0xffffdd8bfd4f4f40
ffffdd8b`fd4f4f40 48b820531b3900f8ffff mov rax,offset hvcv!NtTerminateProcessHook (fffff800`391b5320)
ffffdd8b`fd4f4f4a ffe0            jmp     rax
```
这里我们临时分配一个空间，将构造好的跳转指令以及原函数需要跳过的指令（跳过是为了不破坏原有的指令，这里使用了[LDE](https://github.com/BeaEngine/lde64),计算跳过的指令长度），首先将跳转的hook函数的指令拷贝到原本的跳转的位置
```h
2: kd> u 0xffffdd8bf5935e50
ffffdd8b`f5935e50 4c8bdc          mov     r11,rsp
ffffdd8b`f5935e53 49895b10        mov     qword ptr [r11+10h],rbx
ffffdd8b`f5935e57 49896b18        mov     qword ptr [r11+18h],rbp
ffffdd8b`f5935e5b 57              push    rdi
ffffdd8b`f5935e5c 48b84c3f6b3300f8ffff mov rax,offset nt!NtTerminateProcess+0xc (fffff800`336b3f4c)
ffffdd8b`f5935e66 ffe0            jmp     rax
```
然后将构造的返回正常函数流程的指令拷贝到实现定义好的函数指针处  
[<img src="/images/assets/post3/hook.PNG" width="100%"/>](/images/assets/post3/hook.PNG)  

大概的整个流程：
* 1.对目标函数对应的EPT页面RWX权限全部清0  
  
* 2.访问权限不足触发EPT Violation  
  
* 3.如果是执行则修改EPT页面挂上我们的HOOK，然后给执行的权限  
  
* 4.如果是读写则原封不动，然后给读写的权限  
  
* 5.HOOK执行完毕，走正常的函数流程  
  

整个VT框架具体的实现细节还是看代码比较好，不然字太多也打不过了。  

Code:[https://github.com/Jhinxs/hvcv](https://github.com/Jhinxs/hvcv)
