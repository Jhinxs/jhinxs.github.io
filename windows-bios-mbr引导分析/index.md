# Windows BIOS MBR引导分析

### 流程概述
Windows BIOS开机引导流程概览：

* 1.开机主板加电，BIOS运行
* 2.运行硬盘开头的MBR代码
* 3.MBR代码引导运行Bootmgr（也有可能引导VBR，然后VBR引导Bootmgr）
* 4.Bootmgr根据BCD配置，自动运行或让用户选择不同的启动项，并加载运行Winload.exe
* 5.Winload.exe加载ntoskrnl.exe内核模块
* 6.ntoskrnl.exe启动操作系统

### MBR与VBR结构概览  
[<img src="/images/assets/post6/mbrvbr.png" width="100%"/>](/images/assets/post6/mbrvbr.png)

### MBR
MBR主引导记录，用于引导后续运行的代码，一般用作引导bootmgr等第二阶段的引导程序

MBR一般位于每块硬盘的最开头的512个字节，也就是一个扇区的大小，其中前446个字节为引导程序，然后是64个字节的分区表，其中每个分区项为16个字节，最后是2个字节的Magic,统一为0X55,0XAA,操作系统用此Magic确定和查找引导记录，如果没有，则无法进行引导。

其中446个字节的引导程序中：

* 前162个字节为MBR可执行代码

* 163-1B2这80个字节为错误信息字符串

* 1B3-1BD这11字节包含2个byte 00 padding,3个字节63 7B 9A分别代表了错误信息字符串的offset，然后是4个字节的磁盘签名，然后是2个字节的padding

* 磁盘签名可以通过注册表查看：reg query HKEY_LOCAL_MACHINE\SYSTEM\MountedDevices

#### MBR结构
```c++
//主引导记录
typedef struct  _MASTER_BOOT_RECORD
{
	CHAR bootcode[446];   //实际的引导代码
	MBR_PARTITION_TABLE partitiontable[4];   //分区表
	USHORT MBRMagic;      // 0x55AA 
}MASTER_BOOT_RECORD,*PMASTER_BOOT_RECORD;

//分区表项
typedef struct _MBR_PARTITION_TABLE 
{
	BYTE ActiveStatus;         //活动状态 0x80=active 0x00=noactive，表示当前分区是否为活动分区，只能有一个活动分区
	BYTE StartHead;
	WORD StartingSectCylinder; //起始扇区柱面
	BYTE PARTITION_SYSTEMID;   //标识ID，指示当前分区的分区类型，比如常见的有：NTFS,FAT32等等
	BYTE EndHead;
	WORD EndingSectCylinder;
	DWORD RelativeSector;
	DWORD TotalSectors;        //分区中的总扇区数
}MBR_PARTITION_TABLE,*PMBR_PARTITION_TABLE;

```
#### 分析MBR代码

MBR的字节码如下图，测试环境为WIN10,21H2  
[<img src="/images/assets/post6/mbrhex.png" width="100%"/>](/images/assets/post6/mbrhex.png)
IDA静态分析：
```asm
seg000:7C00                         seg000          segment byte public 'CODE' use16
seg000:7C00                                         assume cs:seg000
seg000:7C00                                         ;org 7C00h
seg000:7C00                                         assume es:nothing, ss:nothing, ds:nothing, fs:nothing, gs:nothing
seg000:7C00
seg000:7C00                         loc_7C00:                               ; CODE XREF: seg000:7D2A↓J
seg000:7C00 33 C0                                   xor     ax, ax
seg000:7C02 8E D0                                   mov     ss, ax          ; 段地址为0
seg000:7C04 BC 00 7C                                mov     sp, 7C00h
seg000:7C07 8E C0                                   mov     es, ax
seg000:7C09 8E D8                                   mov     ds, ax
seg000:7C0B BE 00 7C                                mov     si, 7C00h
seg000:7C0E BF 00 06                                mov     di, 600h
seg000:7C11 B9 00 02                                mov     cx, 200h
seg000:7C14 FC                                      cld
seg000:7C15 F3 A4                                   rep movsb               ; 从ds:si 7C00h拷贝0x200个字节到 es:di 600h处；
seg000:7C15                                                                 ; 7C00h地址刚好是我们的MBR代码起始地址，大小为0x200,
seg000:7C15                                                                 ; 也就是说这段代码用于备份我们的MBR代码到600h处
seg000:7C17 50                                      push    ax
seg000:7C18 68 1C 06                                push    61Ch            ; cs:ip = 00:61Ch;
seg000:7C18                                                                 ; retf 处理器先从栈中弹出一个字到IP，再弹出一个字到CS;
seg000:7C18                                                                 ; 跳转到61ch处执行；之前把MBR代码拷贝到了600h
seg000:7C1B CB                                      retf
seg000:7C1C                         ; ---------------------------------------------------------------------------
seg000:7C1C FB                                      sti                     ; 代码接着从这运行
seg000:7C1D B9 04 00                                mov     cx, 4           ; 分区表内一共四个分区项
seg000:7C20 BD BE 07                                mov     bp, 7BEh        ; 7BE = 600+1BE BP为第一个分区项
seg000:7C23
seg000:7C23                         loc_7C23:                               ; CODE XREF: seg000:7C30↓j
seg000:7C23 80 7E 00 00                             cmp     byte ptr [bp+0], 0 ; 比较分区表结构体中的活动标志位，用来查找活动分区
seg000:7C27 7C 0B                                   jl      short loc_7C34  ; 有符号数比较，小于0跳转
seg000:7C29 0F 85 0E 01                             jnz     loc_7D3B        ; 跳转到错误处理，活动标志要么0x80要么0x00出去别的，说明出错
seg000:7C2D 83 C5 10                                add     bp, 10h         ; bp+10h 到下一个分区表项进行查看是否为活动分区
seg000:7C30 E2 F1                                   loop    loc_7C23        ; 比较分区表结构体中的活动标志位，用来查找活动分区
seg000:7C32 CD 18                                   int     18h             ; TRANSFER TO ROM BASIC
seg000:7C32                                                                 ; causes transfer to ROM-based BASIC (IBM-PC)
seg000:7C32                                                                 ; often reboots a compatible; often has no effect at all
seg000:7C34
seg000:7C34                         loc_7C34:                               ; CODE XREF: seg000:7C27↑j
seg000:7C34                                                                 ; seg000:7CAE↓j
seg000:7C34 88 56 00                                mov     [bp+0], dl      ; 磁盘驱动器号，用于AX=41H 下的INT13,例如第一块硬盘则DL=80H,由BIOS主板设定完毕
seg000:7C37 55                                      push    bp
seg000:7C38 C6 46 11 05                             mov     byte ptr [bp+11h], 5
seg000:7C3C C6 46 10 00                             mov     byte ptr [bp+10h], 0
seg000:7C40 B4 41                                   mov     ah, 41h ; 'A'
seg000:7C42 BB AA 55                                mov     bx, 55AAh       ; 构造用来测试是否支持INT13扩展 所需要的参数，当AH=41的时候，
seg000:7C42                                                                 ; int13则用于检测是否支持int13扩展，
seg000:7C42                                                                 ; CF Set On Not Present, Clear If Present
seg000:7C42                                                                 ; https://en.wikipedia.org/wiki/INT_13H#INT_13h_AH=41h:_Check_Extensions_Present
seg000:7C45 CD 13                                   int     13h             ; DISK - Check for INT 13h Extensions
seg000:7C45                                                                 ; BX = 55AAh, DL = drive number
seg000:7C45                                                                 ; Return: CF set if not supported
seg000:7C45                                                                 ; AH = extensions version
seg000:7C45                                                                 ; BX = AA55h
seg000:7C45                                                                 ; CX = Interface support bit map
seg000:7C47 5D                                      pop     bp
seg000:7C48 72 0F                                   jb      short loc_7C59  ; CF=1 跳转，此时表示不支持INT13扩展
seg000:7C4A 81 FB 55 AA                             cmp     bx, 0AA55h
seg000:7C4E 75 09                                   jnz     short loc_7C59  ; bx是否为AA55h，不是则不支持
seg000:7C50 F7 C1 01 00                             test    cx, 1           ; cx=1 表示支持Device Access using the packet structure
seg000:7C54 74 03                                   jz      short loc_7C59
seg000:7C56 FE 46 10                                inc     byte ptr [bp+10h] ; 到这表示支持，标志位加1
seg000:7C59
seg000:7C59                         loc_7C59:                               ; CODE XREF: seg000:7C48↑j
seg000:7C59                                                                 ; seg000:7C4E↑j ...
seg000:7C59 66 60                                   pushad
seg000:7C5B 80 7E 10 00                             cmp     byte ptr [bp+10h], 0 ; 判断标志位，是否支持int13扩展
seg000:7C5F 74 26                                   jz      short loc_7C87  ; 不支持跳转到标准INT13处
seg000:7C61 66 68 00 00 00 00                       push    large 0         ; 构造Packet Structure
seg000:7C61                                                                 ; struct DiskPacket
seg000:7C61                                                                 ; {
seg000:7C61                                                                 ;     .packet_size     db 0x10 ; use_transfer_64 ? 10h : 18h
seg000:7C61                                                                 ;     .reserved        db 0x00 ; always zero
seg000:7C61                                                                 ;     .block_cout      dw 0x00 ; number of sectors to read
seg000:7C61                                                                 ;     .transfer_buffer dd 0x00 ; address to load in ram
seg000:7C61                                                                 ;     .lba_value       dq 0x00 ; LBA addres value
seg000:7C61                                                                 ; }
seg000:7C67 66 FF 76 08                             push    large dword ptr [bp+8] ; 读取的源地址，以扇区为单位，这里是0x800扇区，也就是0x100000字节处
seg000:7C6B 68 00 00                                push    0
seg000:7C6E 68 00 7C                                push    7C00h           ; 写入的目的地址，完成拷贝
seg000:7C71 68 01 00                                push    1               ; 读取一个扇区
seg000:7C74 68 10 00                                push    10h             ; packet 大小
seg000:7C77 B4 42                                   mov     ah, 42h ; 'B'   ; AH=42H Extended Read Sectors From Drive
seg000:7C79 8A 56 00                                mov     dl, [bp+0]
seg000:7C7C 8B F4                                   mov     si, sp          ; 传入packet的地址
seg000:7C7E CD 13                                   int     13h             ; DISK - IBM/MS Extension - EXTENDED READ (DL - drive, DS:SI - disk address packet)
seg000:7C80 9F                                      lahf
seg000:7C81 83 C4 10                                add     sp, 10h
seg000:7C84 9E                                      sahf
seg000:7C85 EB 14                                   jmp     short loc_7C9B  ; 恢复通用寄存器
seg000:7C87                         ; ---------------------------------------------------------------------------
seg000:7C87
seg000:7C87                         loc_7C87:                               ; CODE XREF: seg000:7C5F↑j
seg000:7C87 B8 01 02                                mov     ax, 201h        ; AH=02 功能号为02 Read Sectors From Drive
seg000:7C87                                                                 ; AL=01 读取一个扇区 Sectors To Read Count
seg000:7C8A BB 00 7C                                mov     bx, 7C00h
seg000:7C8D 8A 56 00                                mov     dl, [bp+0]      ; 磁盘号
seg000:7C90 8A 76 01                                mov     dh, [bp+1]      ; 磁头
seg000:7C93 8A 4E 02                                mov     cl, [bp+2]      ; 扇区
seg000:7C96 8A 6E 03                                mov     ch, [bp+3]      ; 柱面
seg000:7C99 CD 13                                   int     13h             ; DISK - READ SECTORS INTO MEMORY
seg000:7C99                                                                 ; AL = number of sectors to read, CH = track, CL = sector
seg000:7C99                                                                 ; DH = head, DL = drive, ES:BX -> buffer to fill
seg000:7C99                                                                 ; Return: CF set on error, AH = status, AL = number of sectors read
seg000:7C9B
seg000:7C9B                         loc_7C9B:                               ; CODE XREF: seg000:7C85↑j
seg000:7C9B 66 61                                   popad                   ; 恢复通用寄存器
seg000:7C9D 73 1C                                   jnb     short loc_7CBB  ; int13 读取成功 CF=0 JNB跳转
seg000:7C9F FE 4E 11                                dec     byte ptr [bp+11h] ; bp+11=5 一共会读取5次磁盘
seg000:7CA2 75 0C                                   jnz     short loc_7CB0
seg000:7CA4 80 7E 00 80                             cmp     byte ptr [bp+0], 80h ; 比较磁盘驱动器号是否为80h
seg000:7CA8 0F 84 8A 00                             jz      loc_7D36        ; 比较失败，进入打印错误的代码
seg000:7CAC B2 80                                   mov     dl, 80h
seg000:7CAE EB 84                                   jmp     short loc_7C34  ; 磁盘驱动器号，用于AX=41H 下的INT13,例如第一块硬盘则DL=80H,由BIOS主板设定完毕
seg000:7CB0                         ; ---------------------------------------------------------------------------
seg000:7CB0
seg000:7CB0                         loc_7CB0:                               ; CODE XREF: seg000:7CA2↑j
seg000:7CB0 55                                      push    bp
seg000:7CB1 32 E4                                   xor     ah, ah
seg000:7CB3 8A 56 00                                mov     dl, [bp+0]
seg000:7CB6 CD 13                                   int     13h             ; DISK - RESET DISK SYSTEM
seg000:7CB6                                                                 ; DL = drive (if bit 7 is set both hard disks and floppy disks reset)
seg000:7CB8 5D                                      pop     bp
seg000:7CB9 EB 9E                                   jmp     short loc_7C59
seg000:7CBB                         ; ---------------------------------------------------------------------------
seg000:7CBB
seg000:7CBB                         loc_7CBB:                               ; CODE XREF: seg000:7C9D↑j
seg000:7CBB 81 3E FE 7D 55 AA                       cmp     ds:word_7DFE, 0AA55h ; 再次检测是否包含0X55AA
seg000:7CC1 75 6E                                   jnz     short loc_7D31  ; 不包含则进入打印错误流程，CPU执行hlt指令并暂停运行
seg000:7CC3 FF 76 00                                push    word ptr [bp+0]
seg000:7CC6 E8 8D 00                                call    sub_7D56        ; 这个call里面的核心就是in al, 64h，从64H端口读1byte到al
seg000:7CC9 75 17                                   jnz     short loc_7CE2  ; AX=0BB00 INT 1A 时为， TCG_StatusCheck
seg000:7CCB FA                                      cli                     ; 关闭中断
seg000:7CCC B0 D1                                   mov     al, 0D1h
seg000:7CCE E6 64                                   out     64h, al         ; 8042 keyboard controller command register.
seg000:7CCE                                                                 ; Write output port (next byte to port 60h):
seg000:7CCE                                                                 ; 7:  1=keyboard data line pulled low (inhibited)
seg000:7CCE                                                                 ; 6:  1=keyboard clock line pulled low (inhibited)
seg000:7CCE                                                                 ; 5:  enables IRQ 12 interrupt on mouse IBF
seg000:7CCE                                                                 ; 4:  enables IRQ 1 interrupt on keyboard IBF
seg000:7CCE                                                                 ; 3:  1=mouse clock line pulled low (inhibited)
seg000:7CCE                                                                 ; 2:  1=mouse data line pulled low (inhibited)
seg000:7CCE                                                                 ; 1:  A20 gate on/off
seg000:7CCE                                                                 ; 0:  reset the PC (THIS BIT SHOULD ALWAYS BE SET TO 1)
seg000:7CD0 E8 83 00                                call    sub_7D56
seg000:7CD3 B0 DF                                   mov     al, 0DFh
seg000:7CD5 E6 60                                   out     60h, al         ; 8042 keyboard controller data register.
seg000:7CD7 E8 7C 00                                call    sub_7D56
seg000:7CDA B0 FF                                   mov     al, 0FFh
seg000:7CDC E6 64                                   out     64h, al         ; 8042 keyboard controller command register.
seg000:7CDC                                                                 ; Pulse output port.
seg000:7CDC                                                                 ; Bits 0-3 indicate ports to pulse.
seg000:7CDE E8 75 00                                call    sub_7D56
seg000:7CE1 FB                                      sti                     ; 开启中断
seg000:7CE2
seg000:7CE2                         loc_7CE2:                               ; CODE XREF: seg000:7CC9↑j
seg000:7CE2 B8 00 BB                                mov     ax, 0BB00h      ; AX=0BB00 INT 1A 时为， TCG_StatusCheck
seg000:7CE5 CD 1A                                   int     1Ah             ; Trusted Computing Group call - TCG_StatusCheck
seg000:7CE5                                                                 ; Return: EAX = 0 if supported
seg000:7CE5                                                                 ; EBX = 41504354h ('TCPA')
seg000:7CE5                                                                 ; CH:CL = TCG BIOS Version
seg000:7CE5                                                                 ; EDX = BIOS TCG Feature Flags
seg000:7CE5                                                                 ; ESI = Pointer to Event Log
seg000:7CE5                                                                 ;
seg000:7CE7 66 23 C0                                and     eax, eax        ; TCG 状态检测结果放在eax,eax!=0则说明不支持
seg000:7CEA 75 3B                                   jnz     short loc_7D27  ; dx = 0x80 驱动器号
seg000:7CEC 66 81 FB 54 43 50 41                    cmp     ebx, 41504354h  ; TCG检测通过会在EBX中放入"TCPA"的Hex字节码
seg000:7CF3 75 32                                   jnz     short loc_7D27  ; dx = 0x80 驱动器号
seg000:7CF5 81 F9 02 01                             cmp     cx, 102h        ; TCG检查返回后会在ECX中放入TCG BIOS Version
seg000:7CF9 72 2C                                   jb      short loc_7D27  ; dx = 0x80 驱动器号
seg000:7CFB 66 68 07 BB 00 00                       push    large 0BB07h    ; 又是跟TCG相关的一些东西
seg000:7D01 66 68 00 02 00 00                       push    large 200h
seg000:7D07 66 68 08 00 00 00                       push    large 8
seg000:7D0D 66 53                                   push    ebx
seg000:7D0F 66 53                                   push    ebx
seg000:7D11 66 55                                   push    ebp
seg000:7D13 66 68 00 00 00 00                       push    large 0
seg000:7D19 66 68 00 7C 00 00                       push    large 7C00h
seg000:7D1F 66 61                                   popad
seg000:7D21 68 00 00                                push    0
seg000:7D24 07                                      pop     es
seg000:7D25 CD 1A                                   int     1Ah
seg000:7D27
seg000:7D27                         loc_7D27:                               ; CODE XREF: seg000:7CEA↑j
seg000:7D27                                                                 ; seg000:7CF3↑j ...
seg000:7D27 5A                                      pop     dx              ; dx = 0x80 驱动器号
seg000:7D28 32 F6                                   xor     dh, dh
seg000:7D2A EA 00 7C 00 00                          jmp     far ptr loc_7C00 ; 7C00经过之前的拷贝已经变为了VBR的代码，现在开始转入执行VBR
seg000:7D2F                         ; ---------------------------------------------------------------------------
seg000:7D2F CD 18                                   int     18h             ; TRANSFER TO ROM BASIC
seg000:7D2F                                                                 ; causes transfer to ROM-based BASIC (IBM-PC)
seg000:7D2F                                                                 ; often reboots a compatible; often has no effect at all
seg000:7D31
seg000:7D31                         loc_7D31:                               ; CODE XREF: seg000:7CC1↑j
seg000:7D31 A0 B7 07                                mov     al, ds:7B7h     ;7B7内容为9A，也是就获取700+9A处的字符串
seg000:7D34 EB 08                                   jmp     short loc_7D3E  ; 这些都是打印错误信息
seg000:7D36                         ; ---------------------------------------------------------------------------
seg000:7D36
seg000:7D36                         loc_7D36:                               ; CODE XREF: seg000:7CA8↑j
seg000:7D36 A0 B6 07                                mov     al, ds:7B6h
seg000:7D39 EB 03                                   jmp     short loc_7D3E
seg000:7D3B                         ; ---------------------------------------------------------------------------
seg000:7D3B
seg000:7D3B                         loc_7D3B:                               ; CODE XREF: seg000:7C29↑j
seg000:7D3B A0 B5 07                                mov     al, ds:7B5h
seg000:7D3E
seg000:7D3E                         loc_7D3E:                               ; CODE XREF: seg000:7D34↑j
seg000:7D3E                                                                 ; seg000:7D39↑j
seg000:7D3E 32 E4                                   xor     ah, ah
seg000:7D40 05 00 07                                add     ax, 700h
seg000:7D43 8B F0                                   mov     si, ax
seg000:7D45
seg000:7D45                         loc_7D45:                               ; CODE XREF: seg000:7D51↓j
seg000:7D45 AC                                      lodsb
seg000:7D46 3C 00                                   cmp     al, 0
seg000:7D48 74 09                                   jz      short loc_7D53
seg000:7D4A BB 07 00                                mov     bx, 7
seg000:7D4D B4 0E                                   mov     ah, 0Eh
seg000:7D4F CD 10                                   int     10h             ; - VIDEO - WRITE CHARACTER AND ADVANCE CURSOR (TTY WRITE)
seg000:7D4F                                                                 ; AL = character, BH = display page (alpha modes)
seg000:7D4F                                                                 ; BL = foreground color (graphics modes)
seg000:7D51 EB F2                                   jmp     short loc_7D45
seg000:7D53                         ; ---------------------------------------------------------------------------
seg000:7D53
seg000:7D53                         loc_7D53:                               ; CODE XREF: seg000:7D48↑j
seg000:7D53                                                                 ; seg000:7D54↓j
seg000:7D53 F4                                      hlt
seg000:7D54                         ; ---------------------------------------------------------------------------
seg000:7D54 EB FD                                   jmp     short loc_7D53
seg000:7D56
seg000:7D56                         ; =============== S U B R O U T I N E =======================================
seg000:7D56
seg000:7D56
seg000:7D56                         sub_7D56        proc near               ; CODE XREF: seg000:7CC6↑p
seg000:7D56                                                                 ; seg000:7CD0↑p ...
seg000:7D56 2B C9                                   sub     cx, cx
seg000:7D58
seg000:7D58                         loc_7D58:                               ; CODE XREF: sub_7D56+8↓j
seg000:7D58 E4 64                                   in      al, 64h         ; 8042 keyboard controller status register
seg000:7D58                                                                 ; 7:  PERR    1=parity error in data received from keyboard
seg000:7D58                                                                 ;    +----------- AT Mode ----------+------------ PS/2 Mode ------------+
seg000:7D58                                                                 ; 6: |RxTO    receive (Rx) timeout  | TO      general timeout (Rx or Tx)|
seg000:7D58                                                                 ; 5: |TxTO    transmit (Tx) timeout | MOBF    mouse output buffer full  |
seg000:7D58                                                                 ;    +------------------------------+-----------------------------------+
seg000:7D58                                                                 ; 4:  INH     0=keyboard communications inhibited
seg000:7D58                                                                 ; 3:  A2      0=60h was the port last written to, 1=64h was last
seg000:7D58                                                                 ; 2:  SYS     distinguishes reset types: 0=cold reboot, 1=warm reboot
seg000:7D58                                                                 ; 1:  IBF     1=input buffer full (keyboard can't accept data)
seg000:7D58                                                                 ; 0:  OBF     1=output buffer full (data from keyboard is available)
seg000:7D5A EB 00                                   jmp     short $+2
seg000:7D5C                         ; ---------------------------------------------------------------------------
seg000:7D5C
seg000:7D5C                         loc_7D5C:                               ; CODE XREF: sub_7D56+4↑j
seg000:7D5C 24 02                                   and     al, 2
seg000:7D5E E0 F8                                   loopne  loc_7D58
seg000:7D60 24 02                                   and     al, 2
seg000:7D62 C3                                      retn
seg000:7D62                         sub_7D56        endp
seg000:7D62
seg000:7D62                         ; ---------------------------------------------------------------------------
seg000:7D63 49                                      db  49h ; I
seg000:7D64 6E 76 61 6C 69 64 20 70+aNvalidPartitio db 'nvalid partition table',0
seg000:7D7B 45                                      db  45h ; E
seg000:7D7C 72 72 6F 72 20 6C 6F 61+aRrorLoadingOpe db 'rror loading operating system',0
seg000:7D9A 4D                                      db  4Dh ; M
seg000:7D9B 69 73 73 69 6E 67 20 6F+aIssingOperatin db 'issing operating system',0
seg000:7DB3 00 00 63 7B 9A 73 0D 64+                db 2 dup(0), 63h, 7Bh, 9Ah, 73h, 0Dh, 64h, 36h, 2 dup(0)
seg000:7DB3 36 00 00 80 20 21 00 07+                db 80h, 20h, 21h, 0, 7, 7Fh, 39h, 6, 0, 8, 3 dup(0), 90h
seg000:7DB3 7F 39 06 00 08 00 00 00+                db 1, 2 dup(0), 7Fh, 3Ah, 6, 7, 0FEh, 2 dup(0FFh), 0, 98h
seg000:7DB3 90 01 00 00 7F 3A 06 07+                db 1, 0, 52h, 85h, 0ECh, 4, 0, 0FEh, 2 dup(0FFh), 27h
seg000:7DB3 FE FF FF 00 98 01 00 52+                db 0FEh, 2 dup(0FFh), 0, 20h, 0EEh, 4, 0, 0D0h, 11h, 11h dup(0)
seg000:7DFE 55 AA                   word_7DFE       dw 0AA55h               ; DATA XREF: seg000:loc_7CBB↑r
seg000:7DFE                         seg000          ends
seg000:7DFE
seg000:7DFE
seg000:7DFE                                         end
```
### VBR

VBR(Volume boot record)卷引导记录：

由于操作系统逐渐复杂，仅仅靠MBR是无法成功引导操作系统的运行，因此实际情况中，MBR则是引导活动分区的VBR,VBR读取并执行IPL(初始程序加载器)模块代码，IPL则负责解析文件系统并加载Bootmgr模块，引导操作系统初始化。

#### VBR结构
```c++
typedef struct _BOOTSTRAP_CODE
{

	BYTE Bootcode[426];   //引导扇区代码，JMP后会跳转到这
	WORD bootSignature;   //0x55AA

}BOOTSTRAP_CODE,*PBOOTSTRAP_CODE;


typedef struct _BIOS_PARAMETER_BLOCK
{
	WORD BytesPerSector;
	BYTE SectorsPerCluster;
	WORD ReservedSectors;
	CHAR Reserved1[5];
	BYTE MediaId;
	CHAR Reserved2[2];
	WORD SectorsPerTrack;
	WORD NumberOfHeader;
	DWORD HiddenSectors;            
	CHAR Reserved3[8];
	QWORD TotalSectorsNumber;
	QWORD LogicalClusterMFT;
	QWORD LogicalClusterMFTMirror;
	DWORD ClustersPerFileRecSegment;
	DWORD ClustersPerIndexBlock;
	QWORD SerialNumber;
	DWORD Checksum;

}BIOS_PARAMETER_BLOCK, * PBIOS_PARAMETER_BLOCK;

typedef struct _VOLUME_BOOT_RECORD 
{
	WORD JMP;
	BYTE NOP;
	CHAR OEM_NAME[8];                       // OEM NAME 
	BIOS_PARAMETER_BLOCK BPB;               //BIOS参数块
	BOOTSTRAP_CODE BootStrap;
}VOLUME_BOOT_RECORD,*PVOLUME_BOOT_RECORD;
```
#### 分析VBR代码

VBR代码一共占用一个扇区512个字节

* 前两个字节为一个JMP指令，用于跳转到VBR主引导代码段

* 第三个字节 90为一个NOP

* OEM_ID占用8个字节，这个跟磁盘格式有关NTFS基本都是4E 54 46 53 20 20 20 20，如果是FAT32等别的，值会不一样

* 然后是72个字节的BPB块

* 0x54h-0x189h 则为实际的VBR引导代码

* 然后是一些00字节padding

* 8A 01 A7 01 BF 01 这些分别是错误字符串的OFFSET，例如018a则是A disk read error.....

* 最后两个字节0x55AA
[<img src="/images/assets/post6/vbrhex.png" width="100%"/>](/images/assets/post6/vbrhex.png)

IDA静态分析：
```asm
seg000:0000                         ; Segment type: Pure code
seg000:0000                         seg000          segment byte public 'CODE' use16
seg000:0000                                         assume cs:seg000
seg000:0000                                         assume es:nothing, ss:nothing, ds:nothing, fs:nothing, gs:nothing
seg000:0000 EB 52                                   jmp     short loc_54    ; 调整ss段寄存器，关闭中断
seg000:0000                         ; ---------------------------------------------------------------------------
seg000:0002 90                                      db 90h
seg000:0003 4E                                      db  4Eh ; N
seg000:0004 54 46 53 20 20 20 20 00 aTfs            db 'TFS    ',0          ; DATA XREF: seg000:00F8↓o
seg000:0004                                                                 ; seg000:00A9↓r
seg000:000C 02                                      db    2
seg000:000D 08 00 00 00 00 00 00 00+byte_D          db 8, 7 dup(0), 0F8h, 2 dup(0), 3Fh, 0, 0FFh, 2 dup(0)
seg000:000D F8 00 00 3F 00 FF 00 00+                                        ; DATA XREF: sub_11D+20↓r
seg000:000D 08 00 00 00 00 00 00 80+                                        ; seg000:00AF↓w ...
seg000:000D 00 80 00 FF 8F 01 00 00+                db 8, 6 dup(0), 80h, 0, 80h, 0, 0FFh, 8Fh, 1, 5 dup(0)
seg000:000D 00 00 00 AA 10 00 00 00+                db 0AAh, 10h, 6 dup(0), 2, 7 dup(0), 0F6h, 3 dup(0), 1
seg000:000D 00 00 00 02 00 00 00 00+                db 3 dup(0), 95h, 0BCh, 0B5h, 0AAh, 0E0h, 0B5h, 0AAh, 16h
seg000:000D 00 00 00 F6 00 00 00 01+                db 4 dup(0)
seg000:0054                         ; ---------------------------------------------------------------------------
seg000:0054
seg000:0054                         loc_54:                                 ; CODE XREF: seg000:0000↑j
seg000:0054 FA                                      cli                     ; 调整ss段寄存器，关闭中断
seg000:0055 33 C0                                   xor     ax, ax
seg000:0057 8E D0                                   mov     ss, ax
seg000:0059 BC 00 7C                                mov     sp, 7C00h       ; 跳转SP到7C00，在mbr中把VBR拷贝到了这个位置
seg000:005C FB                                      sti
seg000:005D 68 C0 07                                push    7C0h
seg000:0060 1F                                      pop     ds              ; 调整DS段寄存器 DS=7C0H
seg000:0061                                         assume ds:nothing
seg000:0061 1E                                      push    ds
seg000:0062 68 66 00                                push    66h
seg000:0065 CB                                      retf                    ; IP=66H CS=7C0H
seg000:0066                         ; ---------------------------------------------------------------------------
seg000:0066 88 16 0E 00                             mov     ds:0Eh, dl      ; DATA XREF: seg000:00D9↓r
seg000:0066                                                                 ; seg000:010B↓r
seg000:0066                                                                 ; DL=0X80H 驱动器号
seg000:006A 66 81 3E 03 00 4E 54 46+                cmp     dword ptr ds:3, 5346544Eh ; 比较oemid是否是NTFS
seg000:0073 75 15                                   jnz     short loc_8A
seg000:0075 B4 41                                   mov     ah, 41h ; 'A'
seg000:0077 BB AA 55                                mov     bx, 55AAh       ; 继续测试是否支持INT13扩展
seg000:007A CD 13                                   int     13h             ; DISK - Check for INT 13h Extensions
seg000:007A                                                                 ; BX = 55AAh, DL = drive number
seg000:007A                                                                 ; Return: CF set if not supported
seg000:007A                                                                 ; AH = extensions version
seg000:007A                                                                 ; BX = AA55h
seg000:007A                                                                 ; CX = Interface support bit map
seg000:007C 72 0C                                   jb      short loc_8A    ; CF=1 不支持跳转打印错误信息
seg000:007E 81 FB 55 AA                             cmp     bx, 0AA55h
seg000:0082 75 06                                   jnz     short loc_8A    ; 校验BX，不通过跳转打印错误信息
seg000:0084 F7 C1 01 00                             test    cx, 1
seg000:0088 75 03                                   jnz     short loc_8D    ; 检验CX,判断是否支持 Device Access using the packet structure，不支持继续流程并打印错误信息，支持则跳转
seg000:008A
seg000:008A                         loc_8A:                                 ; CODE XREF: seg000:0073↑j
seg000:008A                                                                 ; seg000:007C↑j ...
seg000:008A E9 DD 00                                jmp     loc_16A
seg000:008D                         ; ---------------------------------------------------------------------------
seg000:008D
seg000:008D                         loc_8D:                                 ; CODE XREF: seg000:0088↑j
seg000:008D 1E                                      push    ds
seg000:008E 83 EC 18                                sub     sp, 18h         ; 提升堆栈
seg000:0091 68 1A 00                                push    1Ah
seg000:0094 B4 48                                   mov     ah, 48h ; 'H'   ; AH=48H INT13为 Extended Read Drive Parameters
seg000:0096 8A 16 0E 00                             mov     dl, ds:0Eh      ; DL为驱动器号0x80h
seg000:009A 8B F4                                   mov     si, sp
seg000:009C 16                                      push    ss
seg000:009D 1F                                      pop     ds              ; ss=ds=0
seg000:009E                                         assume ds:nothing
seg000:009E CD 13                                   int     13h             ; DISK - IBM/MS Extension - GET DRIVE PARAMETERS (DL - drive, DS:SI - buffer)
seg000:00A0 9F                                      lahf                    ; flags register -> AH
seg000:00A1 83 C4 18                                add     sp, 18h
seg000:00A4 9E                                      sahf                    ; AH -> flag register
seg000:00A5 58                                      pop     ax              ; INT13的结果放在DS:SI处，不同offset内容不同
seg000:00A5                                                                 ; 此时add sp,18h,说明去offset为18h的地方找数据，18h-19h两个字节存放着 磁盘一个扇区有多少数据，一般为512bytes
seg000:00A5                                                                 ; offset range    size    description
seg000:00A5                                                                 ; 00h..01h    2 bytes size of Result Buffer (set this to 1Eh)
seg000:00A5                                                                 ; 02h..03h    2 bytes information flags
seg000:00A5                                                                 ; 04h..07h    4 bytes physical number of cylinders = last index + 1
seg000:00A5                                                                 ; (because index starts with 0)
seg000:00A5                                                                 ; 08h..0Bh    4 bytes physical number of heads = last index + 1
seg000:00A5                                                                 ; (because index starts with 0)
seg000:00A5                                                                 ; 0Ch..0Fh    4 bytes physical number of sectors per track = last index
seg000:00A5                                                                 ; (because index starts with 1)
seg000:00A5                                                                 ; 10h..17h    8 bytes absolute number of sectors = last index + 1
seg000:00A5                                                                 ; (because index starts with 0)
seg000:00A5                                                                 ; 18h..19h    2 bytes bytes per sector
seg000:00A5                                                                 ; 1Ah..1Dh    4 bytes optional pointer to Enhanced Disk Drive (EDD) configuration parameters which may be used for subsequent interrupt 13h Extension calls (if supported)
seg000:00A6 1F                                      pop     ds              ; 恢复DS
seg000:00A7 72 E1                                   jb      short loc_8A    ; int13 返回CF=1 ERROR,跳转打印错误
seg000:00A9 3B 06 0B 00                             cmp     ax, word ptr ds:aTfs+7 ; cmp ax,word ptr [0xb],0xb的地方对应BPB结构体的BytesPerSector字段
seg000:00AD 75 DB                                   jnz     short loc_8A    ; 如果一个扇区不等于512bytes,跳转打印错误
seg000:00AF A3 0F 00                                mov     word ptr ds:byte_D+2, ax ; mov [0000fh],512
seg000:00B2 C1 2E 0F 00 04                          shr     word ptr ds:byte_D+2, 4 ; 0x200h 右移4位 0x20h
seg000:00B7 1E                                      push    ds
seg000:00B8 5A                                      pop     dx              ; dx=ds=7C0H
seg000:00B9 33 DB                                   xor     bx, bx
seg000:00BB B9 00 20                                mov     cx, 2000h
seg000:00BE 2B C8                                   sub     cx, ax          ; CX=1e00h
seg000:00C0 66 FF 06 11 00                          inc     dword ptr ds:byte_D+4 ; [0x11] =1
seg000:00C5
seg000:00C5                         loc_C5:                                 ; CODE XREF: seg000:00D4↓j
seg000:00C5 03 16 0F 00                             add     dx, word ptr ds:byte_D+2 ; dx=07c0h+20h=7e0h
seg000:00C5                                                                 ; 15th :dx = 9A0
seg000:00C9 8E C2                                   mov     es, dx
seg000:00CB FF 06 16 00                             inc     word ptr ds:byte_D+9 ; [0x16]+1
seg000:00CF E8 4B 00                                call    sub_11D
seg000:00D2 2B C8                                   sub     cx, ax          ; CX=1E00-200H=1C00
seg000:00D2                                                                 ; 这里CX递减15次跳出循环
seg000:00D4 77 EF                                   ja      short loc_C5    ; dx=07c0h+20h=7e0h
seg000:00D4                                                                 ; 15th :dx = 9A0
seg000:00D6 B8 00 BB                                mov     ax, 0BB00h      ; TCG Status Check
seg000:00D9 CD 1A                                   int     1Ah             ; Trusted Computing Group call - TCG_StatusCheck
seg000:00D9                                                                 ; Return: EAX = 0 if supported
seg000:00D9                                                                 ; EBX = 41504354h ('TCPA')
seg000:00D9                                                                 ; CH:CL = TCG BIOS Version
seg000:00D9                                                                 ; EDX = BIOS TCG Feature Flags
seg000:00D9                                                                 ; ESI = Pointer to Event Log
seg000:00D9                                                                 ;
seg000:00DB 66 23 C0                                and     eax, eax
seg000:00DE 75 2D                                   jnz     short loc_10D
seg000:00E0 66 81 FB 54 43 50 41                    cmp     ebx, 41504354h
seg000:00E7 75 24                                   jnz     short loc_10D
seg000:00E9 81 F9 02 01                             cmp     cx, 102h        ; 三次比较均为校验TCG的支持状态，不支持直接跳入下一流程
seg000:00ED 72 1E                                   jb      short loc_10D
seg000:00EF 16                                      push    ss
seg000:00F0 68 07 BB                                push    0BB07h
seg000:00F3 16                                      push    ss
seg000:00F4 68 52 11                                push    1152h
seg000:00F7 16                                      push    ss
seg000:00F8 68 09 00                                push    (offset aTfs+5) ; "  "
seg000:00FB 66 53                                   push    ebx
seg000:00FD 66 53                                   push    ebx
seg000:00FF 66 55                                   push    ebp
seg000:0101 16                                      push    ss
seg000:0102 16                                      push    ss
seg000:0103 16                                      push    ss
seg000:0104 68 B8 01                                push    (offset aOotmgrIsCompre+0Eh) ; "ressed"
seg000:0107 66 61                                   popad
seg000:0109 0E                                      push    cs
seg000:010A 07                                      pop     es
seg000:010B CD 1A                                   int     1Ah             ; DL=0X80H 驱动器号
seg000:010D
seg000:010D                         loc_10D:                                ; CODE XREF: seg000:00DE↑j
seg000:010D                                                                 ; seg000:00E7↑j ...
seg000:010D 33 C0                                   xor     ax, ax
seg000:010F BF 0A 13                                mov     di, 130Ah
seg000:0112 B9 F6 0C                                mov     cx, 0CF6h
seg000:0115 FC                                      cld
seg000:0116 F3 AA                                   rep stosb               ; 将EAX的值拷贝到ES:DI的位置
seg000:0116                                                                 ; AX=0 CX=0CF6
seg000:0116                                                                 ; 所以这是将9A00 + 130A = AD0A 清零
seg000:0118 E9 FE 01                                jmp     near ptr 319h   ; 跳转到BootStrap代码运行Bootmgr loader
seg000:011B                         ; ---------------------------------------------------------------------------
seg000:011B 90                                      nop
seg000:011C 90                                      nop
seg000:011D
seg000:011D                         ; =============== S U B R O U T I N E =======================================
seg000:011D
seg000:011D
seg000:011D                         sub_11D         proc near               ; CODE XREF: seg000:00CF↑p
seg000:011D 66 60                                   pushad
seg000:011F 1E                                      push    ds
seg000:0120 06                                      push    es
seg000:0121
seg000:0121                         loc_121:                                ; CODE XREF: sub_11D+46↓j
seg000:0121 66 A1 11 00                             mov     eax, dword ptr ds:byte_D+4 ; eax =1
seg000:0125 66 03 06 1C 00                          add     eax, dword ptr ds:byte_D+0Fh ; EAX=[7C11]
seg000:0125                                                                 ; [7C11]+[7C1C]
seg000:0125                                                                 ; [7C1C]处为BPB结构体中的HiddeSectors字段
seg000:0125                                                                 ; 该字段指示了硬盘开始到NTFS卷引导（也就是VBR）的扇区数量
seg000:0125                                                                 ; 此数量加1x512bytes，即为IPL代码位置
seg000:0125                                                                 ; 1h+0x800h
seg000:012A 1E                                      push    ds              ; ds=0
seg000:012B 66 68 00 00 00 00                       push    large 0
seg000:0131 66 50                                   push    eax
seg000:0133 06                                      push    es              ; es=7e0h
seg000:0134 53                                      push    bx              ; bx=offset
seg000:0135 68 01 00                                push    1               ; copy 1 sector
seg000:0138 68 10 00                                push    10h             ; Disk ACCESS Packet size SET TO 10H
seg000:013B B4 42                                   mov     ah, 42h ; 'B'
seg000:013D 8A 16 0E 00                             mov     dl, ds:byte_D+1
seg000:0141 16                                      push    ss
seg000:0142 1F                                      pop     ds              ; ds=0
seg000:0143 8B F4                                   mov     si, sp          ; 此时的栈结构:
seg000:0143                                                                 ; -----0x0 00 10H DAP SIZE 10 + Reserved 00
seg000:0143                                                                 ; -----0X4 01 00  number of sectors to be read
seg000:0143                                                                 ; -----0X8 00 00 E0 07 07E0:0000 transfer buffer segment:offset = 7e0h:0000h
seg000:0143                                                                 ; -----0xc 01 08 LBA地址=801h
seg000:0145 CD 13                                   int     13h             ; DISK - IBM/MS Extension - EXTENDED READ (DL - drive, DS:SI - disk address packet)
seg000:0147 66 59                                   pop     ecx
seg000:0149 5B                                      pop     bx
seg000:014A 5A                                      pop     dx
seg000:014B 66 59                                   pop     ecx
seg000:014D 66 59                                   pop     ecx
seg000:014F 1F                                      pop     ds
seg000:0150 0F 82 16 00                             jb      loc_16A         ; 读取失败，跳转打印错误信息
seg000:0154 66 FF 06 11 00                          inc     dword ptr ds:byte_D+4 ; 1->2
seg000:0154                                                                 ; 15th [0711]变成16
seg000:0159 03 16 0F 00                             add     dx, word ptr ds:byte_D+2 ; 7e0+20h
seg000:0159                                                                 ; 15th DX变成9C0h
seg000:015D 8E C2                                   mov     es, dx
seg000:015F FF 0E 16 00                             dec     word ptr ds:byte_D+9 ; [7c16] = 0
seg000:0163 75 BC                                   jnz     short loc_121   ; 这里判断了标志，跳转读取下一个扇区
seg000:0165 07                                      pop     es
seg000:0166 1F                                      pop     ds
seg000:0167 66 61                                   popad
seg000:0169 C3                                      retn
seg000:016A                         ; ---------------------------------------------------------------------------
seg000:016A
seg000:016A                         loc_16A:                                ; CODE XREF: seg000:loc_8A↑j
seg000:016A                                                                 ; sub_11D+33↑j
seg000:016A A1 F6 01                                mov     ax, ds:word_1F6
seg000:016D E8 09 00                                call    sub_179
seg000:0170 A1 FA 01                                mov     ax, ds:word_1FA
seg000:0173 E8 03 00                                call    sub_179
seg000:0176
seg000:0176                         loc_176:                                ; CODE XREF: seg000:0177↓j
seg000:0176 F4                                      hlt
seg000:0176                         sub_11D         endp
seg000:0176
seg000:0177                         ; ---------------------------------------------------------------------------
seg000:0177 EB FD                                   jmp     short loc_176
seg000:0179
seg000:0179                         ; =============== S U B R O U T I N E =======================================
seg000:0179
seg000:0179
seg000:0179                         sub_179         proc near               ; CODE XREF: sub_11D+50↑p
seg000:0179                                                                 ; sub_11D+56↑p
seg000:0179 8B F0                                   mov     si, ax
seg000:017B
seg000:017B                         loc_17B:                                ; CODE XREF: sub_179+E↓j
seg000:017B AC                                      lodsb
seg000:017C 3C 00                                   cmp     al, 0
seg000:017E 74 09                                   jz      short locret_189
seg000:0180 B4 0E                                   mov     ah, 0Eh
seg000:0182 BB 07 00                                mov     bx, 7
seg000:0185 CD 10                                   int     10h             ; - VIDEO - WRITE CHARACTER AND ADVANCE CURSOR (TTY WRITE)
seg000:0185                                                                 ; AL = character, BH = display page (alpha modes)
seg000:0185                                                                 ; BL = foreground color (graphics modes)
seg000:0187 EB F2                                   jmp     short loc_17B
seg000:0189                         ; ---------------------------------------------------------------------------
seg000:0189
seg000:0189                         locret_189:                             ; CODE XREF: sub_179+5↑j
seg000:0189 C3                                      retn
seg000:0189                         sub_179         endp
seg000:0189
seg000:0189                         ; ---------------------------------------------------------------------------
seg000:018A 0D                                      db  0Dh
seg000:018B 0A                                      db  0Ah
seg000:018C 41                                      db  41h ; A
seg000:018D 20 64 69 73 6B 20 72 65+aDiskReadErrorO db ' disk read error occurred',0
seg000:01A7 0D                                      db  0Dh
seg000:01A8 0A                                      db  0Ah
seg000:01A9 42                                      db  42h ; B
seg000:01AA 4F 4F 54 4D 47 52 20 69+aOotmgrIsCompre db 'OOTMGR is compressed',0
seg000:01AA 73 20 63 6F 6D 70 72 65+                                        ; DATA XREF: seg000:0104↑o
seg000:01BF 0D                                      db  0Dh
seg000:01C0 0A                                      db  0Ah
seg000:01C1 50                                      db  50h ; P
seg000:01C2 72 65 73 73 20 43 74 72+aRessCtrlAltDel db 'ress Ctrl+Alt+Del to restart',0Dh,0Ah,0
seg000:01E1 00 00 00 00 00 00 00 00+                db 15h dup(0)
seg000:01F6 8A 01                   word_1F6        dw 18Ah                 ; DATA XREF: sub_11D:loc_16A↑r
seg000:01F8 A7                                      db 0A7h
seg000:01F9 01                                      db 1
seg000:01FA BF 01                   word_1FA        dw 1BFh                 ; DATA XREF: sub_11D+53↑r
seg000:01FC 00 00 55 AA                             db 2 dup(0), 55h, 0AAh
seg000:01FC                         seg000          ends
seg000:01FC
seg000:01FC
seg000:01FC                                         end
```

### MBR/VBR Bochs 动态调试
我这里使用了Bochs+IDA的方式，安装Bochs Windows版，Bochsrc内容如下:
```
megs: 256
romimage:file="..\BIOS-bochs-latest"
vgaromimage:file="..\VGABIOS-lgpl-latest"
ata0-master: type=disk, path="..\c.img", mode=flat, cylinders=20, heads=16, spt=63
boot: disk
```
镜像制作过程，利用Bochs自带的bximage制作即可，选择1，然后一路回车即可。  
[<img src="/images/assets/post6/hximage.png" width="100%"/>](/images/assets/post6/hximage.png)  
之后用二进制编辑器如010editor，讲自己系统硬盘的MBR/VBR代码拷贝到制作好的img镜像即可，拷贝的地址偏移均不变。
IDA中在主菜单中debugger选项，选择本地bochs调试，选择上面新建好的Bochsrc文件即可。  
可能会遇到cfg文件报错，只需要在IDA文件夹下的cfg/dbg_bochs.cfg中编辑BOCHSDBG的路径即可。

### 总结
综上结果，可以简单总结一下windows引导过程：

* 加载MBR，MBR将自己备份到0x600H处，并更改IP为61CH开始执行代码

* 循环4次，在分区表中通过匹配活动分区标志，找到活动分区

* 通过int13中断，从0x800扇区处，读取一个扇区的代码大小，也就是读取512bytes的VBR，写入0x7C00H处

* 做一些端口读取和写入检测，然后代码进入VBR执行，MBR执行过程中，每次涉及到检测，只要错误，就会通过INT10打印错误信息，并停止运行代码

* VBR代码执行，JMP跳转，并关闭中断，调整段寄存器

* 验证OEMID是否为NTFS

* 通过读取BPB结构体中HiddeSectors字段，加1得到IPL代码所在的扇区数，并读取0x2000个字节的IPL代码

* 跳转到IPL代码处，控制权交给IPL，开始后续的加载Bootmgr的过程
