# Intel VT-x mini VT 框架的实现

&emsp;&emsp;Intel VT-x是Intel芯片用于支持硬件虚拟化的一种技术，个人理解是在传统的操作系统之上，创建专用于虚拟化开发者的特权级环境，这种权限甚至高于操作系统的环境，也就是常说的Hypervisor层，同样的，新的环境也会引入一些新的特权指令等新的功能。在设计架构上和SVM类似，分为VMM和VMGUEST两种角色，分别处于VMX-ROOT和VMX-NO ROOT模式下  

[<img src="/images/assets/post1/page1.PNG" width="100%"/>](/images/assets/post1/page1.PNG)  

对于虚拟机来说，Hypervisor是完全透明的，当Guest客户机发生vmexit事件后会陷入到Hypervisor，此时控制权完全交与Hypervisor，而Hypervisor同样可以通过vmlaunch运行虚拟机，或者在处理完vmexit事件后，通过vmresume指令，继续运行虚拟机

### VT框架实现流程

本代码用于学习研究，运行于32位操作系统之上，在代码实现VT的时候，主要是需要遵循Intel开发手册的标准，我这里实现的时候，大概是以下的流程:  
* 判断当前CPU是否支持VT技术，通过执行cpuid指令，获取返回信息并判断对应的标志位，这是硬件层面的要求
* 读取MSR_IA32_FEATURE_CONTROL MSR寄存器对应标志位，用于判断在BIOS配置中是否开启了VT     
* 检查并设置CR4 VMXE标志位
* 分配4K对齐的内存，用于vmx操作，这块区域按照白皮书的定义一般叫做VMXON region
* 初始化VMXON region区域，主要是写入一个VMCS版本号，这个版本号通过读取 IA32_MSR_VMX_BASIC MSR寄存器获得
* 执行VMXON,并检查EFLAGE.CF位是否置0，判断是否执行成功
* 分配4K对齐的内存，用于逻辑处理器也即虚拟机的操作区域，叫做VMCS region
* 调用vmclear指令，将VMCS区域设置为清除状态，也即区域初始化
* 调用vmptrld指令，激活VMCS
* 为host以及guest分配内存空间
* 填充VMCS区域，这是最重要也是最复杂的一步
* 通过vmlaunch运行虚拟机

### 代码部分
这个代码写的也是一波三折，蓝屏N次，也是各种坑，虽然可以有参考代码，但实际上有很多因素，遇到比如像保存现场代码的优化，多核处理等等诸多问题

首先是开启VT的前期，检测阶段，首先就是通过CPUID指令，这个指令执行前，需要将EAX寄存器置为1，返回值会存在ECX寄存器中，这里CPUID置1是Intel的要求，如果EAX中是其他值，返回的信息也是不同的，只是在这里用作CPU VTZ支持检测，EAX需要置1  

[<img src="/images/assets/post1/page2.PNG" width="100%"/>](/images/assets/post1/page2.PNG)

```asm
get_cpuid_info proc,_para:dword
       
        pushad
        mov eax,1H
        cpuid
        mov esi,_para
        mov [esi],ecx
        popad
        ret

get_cpuid_info endp

```
然后是读取IA32_FEATURE_CONTROL MSR寄存器,如果在bios启用VT则bit1，bit2或者两者都必须为1，bit1和bit2主要用于设置是否处于SMX Mode（安全扩展模式）下，这里一般检测bit2就可以了
```C++
    ULONG64 msr =  __readmsr(MSR_IA32_FEATURE_CONTROL);
    if (!(msr & 4))                                              //4.VT 指令是否被锁定
    {
        DbgPrint("MSR_IA32_FEATURE_CONTROL VMXON Locked \n");
        return STATUS_UNSUCCESSFUL;
    }
```
检测并设置CR4.VMXE也就是bit13位，在VMXON和VMXOFF之间不许再修改此位
```C++
    set_cr4(X86_CR4_VMXE);                                 //3.设置CR4.VMXE
    cr4 = get_cr4(); 
    if ((cr4 & X86_CR4_VMXE) != X86_CR4_VMXE)
    {
        DbgPrint("CR4_VMXE set Error \n");
        return STATUS_UNSUCCESSFUL;
    }
```
分配VMXON Region区域，读取MSR寄存器，获取VMCS标识并用其初始化VMXON Region，白皮书中说设置前31位，这里需要注意分配非分页内存，然后就可以通过VMXON进入虚拟机模式了，这里首先要获取VMXON Region区域的物理地址，作为参数传递给VMXON，执行完VMXON后需要验证Eflags寄存器的CF位和ZF是否为0，0表示开启成功(参考 Intel-64-ia-32--system-programming-manual-vol-3 Chapter 24.1)  

[<img src="/images/assets/post1/page3.PNG" width="100%"/>](/images/assets/post1/page3.PNG)

```asm
vmx_on proc,LowPart:dword,HighPart:dword

        push HighPart
        push LowPart
        vmxon qword ptr [esp]
        add esp,8
        ret

vmx_on endp
```
```C++
    vmx_basic_msr = __readmsr(MSR_IA32_VMX_BASIC);                 //获取vmcs identifier
    VMX_Region = ExAllocatePoolWithTag(NonPagedPool, 4096, 'vmx');            
    RtlZeroMemory(VMX_Region, 4096);
    *(ULONG*)VMX_Region = (vmx_basic_msr & 0x7ffffff);
    VMXONRegion_PA = MmGetPhysicalAddress(VMX_Region);
    vmx_on(VMXONRegion_PA.LowPart,VMXONRegion_PA.HighPart);

```
前期检测的过程基本完了，接下来就是分配内存了，首先是分配VMCS Region,这块区域是最核心的，可以看为是虚拟化逻辑处理器做各种操作所需要的内存空间，VMCS Region中存在各种域，是需要手动指定填充的，同样需要为Host分配一块内存，用于guest发生vmexit回到host所需要的内存空间，同样分配内存4k对齐以及非分页，在驱动卸载时要释放这几块内存区域，在VMCS Region分配完后，需要调用vmclear以及vmptrld指令，用于初始化和激活VMCS Region

```C++
    VMXCS_Region = ExAllocatePoolWithTag(NonPagedPool, 4096,'vmcs');            
    DbgPrint("VMXCS_Region: %x\n", VMXCS_Region);
    RtlZeroMemory(VMXCS_Region, 4096);

    vmx_basic_msr = __readmsr(MSR_IA32_VMX_BASIC);
    *(ULONG*)VMXCS_Region = (vmx_basic_msr & 0x7ffffff);
    VMCSRegion_PA = MmGetPhysicalAddress(VMXCS_Region);

    vmxcs_clear(VMCSRegion_PA.LowPart, VMCSRegion_PA.HighPart）;    

    eflags = __readeflags();
    if ((eflags & 0x41) != 0)
    {
        DbgPrint("vmxcs_clear ERROR\n");
        ExFreePoolWithTag(VMX_Region, 'vmx');
        return STATUS_UNSUCCESSFUL;
    }

    vmx_vmptrld(VMCSRegion_PA.LowPart, VMCSRegion_PA.HighPart);  

    vmhost_Stack = ExAllocatePoolWithTag(NonPagedPool, 4096, 'hesp');    
    RtlZeroMemory(vmhost_Stack, 4096);
    DbgPrint("vmhost_Stack %x\n", vmhost_Stack);

    vmguest_Stack = ExAllocatePoolWithTag(NonPagedPool, 4096, 'gesp');
    RtlZeroMemory(vmguest_Stack, 4096);
    DbgPrint("vmguest_Stack %x\n", vmguest_Stack);

```
接下来时最关键是最关键的VMCS Region的填充，Intel规定了这块内存需要填充以下域 (参考 Intel-64-ia-32--system-programming-manual-vol-3 Chapter 24.3)：
* Guest-state area（虚拟机状态域）:进入CPU进入VM时需要从这里读取信息，VMExit时将当时的VM CPU状态信息保存在这，用以下次进入，保存状态例如各种CR寄存器，通用寄存器，GDTR等等
* Host-state  area（宿主机状态域）:VM发生EXIT进入Hypervisor时，恢复VMM的执行，保存HOST的CPU状态信息，通用寄存器，堆栈，控制寄存器等等
* VM-Execution Control Fields（VM执行控制域）: 用以控制VM在什么条件下产出VMEXIT事件，如读写CR3等等
* VMEntry Control Fields（VMEntry 控制域）:控制进入VM时的一些行为，如是否Load DR寄存器
* VM-Exit Control Fields （VMExit 控制域）:控制VM发生EXIT事件时的行为，
* VM-Exit Information Fields（VMExit 信息域）:表示了VM发生EXIT事件的原因，可以和VM-Execution Control Fields做配合调试       
  
这里学到一个小技巧，可以根据vmlaunch指令的运行，配合vmread VM_INSTRUCTION_ERROR，来判断这些域基本填写是否正确，错误码对应的原因可以查Chapter30.4  

[<img src="/images/assets/post1/page4.PNG" width="100%"/>](/images/assets/post4/page1.PNG)  

首先是填写Host Area,使用如下类似的填写方式，这里要注意的是必须使用vmwrite的方式，给对应的字段填写，不能自己通过指针啥的方式
```C++
    vmx_vmwrite(VMCS_HOSTAREA_CR0, __readcr0());
    vmx_vmwrite(VMCS_HOSTAREA_CR3, __readcr3());
    vmx_vmwrite(VMCS_HOSTAREA_CR4, __readcr4());
    vmx_vmwrite(VMCS_HOSTAREA_CS, readcs() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_DS, readds() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_ES, reades() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_SS, readss() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_FS, readfs() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_GS, readgs() & 0xfff8);
    vmx_vmwrite(VMCS_HOSTAREA_TR, readtr() & 0xfff8);

```
指定vmexit发生时，跳转到VMM的代码地址
```C++
vmx_vmwrite(VMCS_HOSTAREA_RIP, (ULONG)vmx_vmmhostentry);
```
Chapter24.5可以看到，Intel规定必须要填写的字段，像读取段寄存器，控制寄存器这种网上都是构建汇编如mov eax,es 这种也可以，不过比较推荐使用微软提供的库:#include <intrin.h>,微软出品，必属精品

Guest Area填写的东西就比较多了，比如像填写段寄存器，需要填写选择子，base,limit以及属性，这里参考了newbluepill的方式，动态的去GDT表里面解析段描述符，获取各种字段所需的值  

[<img src="/images/assets/post1/page5.PNG" width="100%"/>](/images/assets/post1/page5.PNG)  

```C++
	SegDesc = (PSEGMENT_DESCRIPTOR2)((PUCHAR)GdtBase + (Selector & ~0x7));
	// 段选择子
	SegmentSelector->sel = Selector;
	// 段基址15-39位 55-63位
	SegmentSelector->base = SegDesc->base0 | SegDesc->base1 << 16 | SegDesc->base2 << 24;
	// 段限长0-15位  47-51位
	SegmentSelector->limit = SegDesc->limit0 | (SegDesc->limit1attr1 & 0xf) << 16;
	// 段属性39-47 51-55 
	SegmentSelector->attributes.UCHARs = SegDesc->attr0 | (SegDesc->limit1attr1 & 0xf0) << 4;
```
这里注意，HOST和GUSET中都填写了类似如下字段：
```C++
    vmx_vmwrite(VMCS_GUSTAREA_SYSENTER_CS, __readmsr(MSR_IA32_SYSENTER_CS) & 0xFFFFFFFF);
    vmx_vmwrite(VMCS_GUSTAREA_SYSENTER_EIP, __readmsr(MSR_IA32_SYSENTER_EIP) & 0xFFFFFFFF);
    vmx_vmwrite(VMCS_GUSTAREA_SYSENTER_ESP, __readmsr(MSR_IA32_SYSENTER_ESP) & 0xFFFFFFFF);
```
这个就像正常的CPU模式下，3环进0环一样，需要切换环境，需要新的堆栈，以及入口EIP等，可以rdmsr MSR_IA32_SYSENTER_EIP(0x176)看看，机器如果支持快速调用，这里都是KifastCallEntry

在填写各种段寄存器的时候，host和guest中都有&0xFFF8的操作，这个是为了将最后的RPL清0，这个和段权限检查相关，访问某个段，RPL和CPL要在数值上小于等于DPL

```C++
vmx_vmwrite(VMCS_GUSTAREA_ES + Segreg * 2, Selector & 0xFFF8);
```
临时堆栈的设置，这里+0x1000,就相当于ESP+0x1000,vmguest_Stack我们之前分配了4k,栈是逆向增长的，这个操作就相当于ESP指向了分配的栈的栈底
```C++
 vmx_vmwrite(VMCS_GUSTAREA_RSP,vmguest_Stack + 0x1000);
```
虚拟机运行控制域，参考Chapter 24.6,这里填写的字段主要就是确定发生vmexit事件的原因，，这里的填写是有要求的，除了自定义填写之外，有些bit位是需要设置1，有些bit位是需要设置为0，这个根据对应的MSR寄存器来确定，具体AREA中填写的字段可以参考白皮书，这里VMCS_PROCESSOR_BASE_CONTTOL为例：
```C++
 vmx_vmwrite(VMCS_PROCESSOR_BASE_CONTTOL, AdjustControlBit(0, MSR_IA32_VMX_PROCBASED_CTLS));
```
我们以最小的发生vmexit事件的拦截原则，其中必须置位的根据IA32_VMX_PROCBASED_CTLS寄存器确定
```
0: kd> rdmsr 482
msr[482] = fff9fffe`0401e172
0: kd> .formats 0401e172
Evaluate expression:
  Hex:     0401e172
  Decimal: 67232114
  Octal:   00400360562
  Binary:  00000100 00000001 11100001 01110010
  Chars:   ...r
  Time:    Fri Feb 18 11:35:14 1972
  Float:   low 1.52674e-036 high 0
  Double:  3.32171e-316

```
我们可以看到对应msr寄存器的内容，第四个字节中的1代表该位必须置1，高四个字节中的0代表该位必须置0，在这里，15，16为1，也就对应一下情况发生时必须产生vmexit事件，然后VMM要对其进行拦截处理  

[<img src="/images/assets/post1/page6.PNG" width="100%"/>](/images/assets/post1/page6.PNG)  

其余的字段填写是类似的，根据需要对应的bit位置位就行

基本的area都按要求填写完毕，现在就可以vmlaunch启动你的虚拟机了，但是这个时候我们还没有对虚拟机的vmexit事件做处理，一个基本的情况就是，一运行之后，vmexit事件产出，代码就会跳转到先前指定的hostarea_rip的位置，说明你需要对这个vmexit世界进行处理，然后再vmresume，让vm继续运行

而vmexit产生的原因，可以通过vmread VM_EXIT_REASON得到，参考Intel-64-ia-32--system-programming-manual-vol-3 附录3 VMX Basic Exit Reasons 表格，一般基础的代码写完之后，遇到最多的一个vmexit原因就是CR3的读取和写入，需要注意的，在这个地方，接管vm过来的代码，处理vmexit时，要记得保存现场，就是一些寄存器要保存，在处理完事件之后，恢复即可
```asm
vmx_vmmhostentry proc
        
        pushad
        push esp
        call VmhostEntrydbg
        popad
        vmresume
        
vmx_vmmhostentry endp

```
这个地方保存的时候，保存之前Guest域中没有填写的就可以了，其他的会自己保存在那个域里面，这里我只保存了下通用寄存器，VmhostEntrydbg是定义的一个vmexit处理函数

以CR3的读写为例，部分代码：
```C++
    CrQulification = vmx_vmread(VMCS_EXIT_QUALIFICTION);
    CRNumber = (CrQulification & 0x0000000f);
    Accesstype = ((CrQulification & 0x00000030) >> 4);
    operandtype = ((CrQulification & 0x00000040) >> 6);
    MovrCRPurposeReg = ((CrQulification & 0x00000F00) >> 8);
    if (Accesstype == 0)
    {
        
        switch (MovrCRPurposeReg)
        {
        case 0:
        {
            vmx_vmwrite(VMCS_GUSTAREA_CR3, g_GUEST_REGS->eax);
            break;
        }
```
这里读取了一个QUALIFICTION这样的一个信息，这个信息可以理解为在发生对CR寄存器的访问和写入时的具体信息，如是从那个寄存器写入的，是读取还是写入，参考chapter 27.2  

[<img src="/images/assets/post1/page7.PNG" width="100%"/>](/images/assets/post1/page7.PNG)  

其中的bit位就对应了相关信息，只需要做一下解析就行了，例如通过bit postions 3:0 你可以获取到和那个CR寄存器发生了交互，在32位机器上bit3总为0，bit postions 5:4代表读取还是写入，0代表CR是目的操作数，此时写入，1代表源操作数，此时读取，这里的11：8要注意这个寄存器的顺序，因为我们要拦截对CR寄存器的操作，意味着我们要帮助vm完成这个过程，通过bit postions 11：8得到和CR寄存器交互的寄存器后一般如果构造这样一个结构体保存了这些通用寄存器的话，你需要注意你的寄存器的排列顺序

处理完vmexit事件后，就要返回VM了，这里的小细节：
```C++
    ULONG instructionlen = vmx_vmread(VMCS_INSTRUCTION_LENGH);
    ULONG geip = vmx_vmread(GUEST_RIP);
    ULONG guestnexteip = geip + instructionlen;
    vmx_vmwrite(VMCS_GUSTAREA_RIP, guestnexteip);
```
可以看到最后的返回eip加了一个值，这个VMCS_INSTRUCTION_LENGH，就是当时发生VMEXIT时，CPU执行的那条指令的长度,比如MOV eax,cr3 被我们拦截并处理完了，那么EIP也应当在原来的基础上加上这条指令的长度，接着执行下一条指令，最终调用vmresume继续执行VM即可

一个简单的VT框架就是这些吧，可能还有其他的点，后续遇到再深入学习吧！！！

### 扩展

1.VMEXIT导致GDT Limit发生改变，使用ARK工具查看GDT相关信息时出发BSoD

这个是在看zhouhe的VT教程看到的，至于原因，白皮书说,VMEXIT时会加载一些host的CR寄存器,msr寄存器,加载GDTR和IDTR时，limit会从3ff变为ffff:  

[<img src="/images/assets/post1/page8.PNG" width="100%"/>](/images/assets/post1/page8.PNG)  

在驱动加载,开启VT前可以保存一份GDTR，驱动卸载，关闭VT后恢复即可

2.VM第一次初始化运行，驱动继续执行的问题,就是说我们运行vmlaunch之后，进入虚拟机里，后续的驱动代码时不会执行的，卡死的状态，所以我们要构造一个上下文环境，在vmlaunch之后，我们的驱动代码可以正常执行完毕：
```C++
    VTenable_before();
    StartVMXCS();                                                          
    VTenable_after();
```
在VMCS初始化填写前后，构造两个函数，我这里汇编实现：
```asm
VTenable_before proc
        cli
        mov DriverEAX,eax
        mov DriverECX,ecx
        mov DriverEDX,edx
        mov DriverEBX,ebx
        mov DriverESP,esp
        mov DriverEBP,ebp
        mov DriverESI,esi
        mov DriverEDI,edi
        pushfd
        pop DriverEFL
        sti
        ret

VTenable_before endp
```
之前保存现场，之后恢复现场，第一次进入VM时，就会跳转到我们的vmx_GuestReturn代码执行，而这个vmx_GuestReturn功能就是调用VTenable_after()，确保驱动正常执行完毕
```C++
vmx_vmwrite(VMCS_GUSTAREA_RIP, (ULONG)vmx_GuestReturn);
```
3.多核问题
看雪小宝来了说过这个问题，就是说我们VT的开启，肯定是以CPU为基准的，单核那就在这个核上开启VT就行，但如果是多核的情况下，不可能仅在个别核上开启VT，那必然时蓝屏啊，所以需要扩展至多核兼容，比如分配各种内存时，需要每个核都分配，同样在执行VT代码的时候，每个核都要执行一边才行
```c++
    for (int i = 0; i < KeNumberProcessors; i++)
    {
        KeSetSystemAffinityThread((KAFFINITY)(1 << i));
        VT_Enable();
        KeRevertToUserAffinityThread();
        DbgPrint("stop vt on cpu [%d]...\n", i);
    }
```
具体API的使用以及用途,参考MSDN文档即可

Code: https://github.com/Jhinxs/x32_VT

Reference:

* https://github.com/zzhouhe/VT_Learn/tree/master/MinimalVT
* https://bbs.pediy.com/thread-211973.htm
* https://github.com/haidragon/newbluepill
* <<new blue pill:深入理解硬件虚拟机>>
* <<Intel 64-ia-32--system-programming-manual-vol-3>>

