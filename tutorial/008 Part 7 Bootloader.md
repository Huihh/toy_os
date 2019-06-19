# 第七章 引导程序



引导程序加载操作系统或者直接与硬件交互的应用。为了跑起来一个操作系统，我们要做的第一件事情是写一个引导程序（bootloader）。在这一章中，我们将要写一个初级 bootloader，因为我们主要的目标是去写一个操作系统，而不是 bootloader，更有趣的是，这一章将要展示一些既能应用到写 bootloader 也能应用到写操作系统的工具与技术。



## 7.1 x86 Boot 进程

当 POST 过程完成后，CPU 的程序计数器设置为 FFFF:0000h 以执行BIOS 代码。BIOS - *B*asic *I*nput / *O*utput *S*ystem 是一个执行硬件初始化并提供一组通用子程序来控制输入/输出设备的固件。



> 注：什么是 POST 过程——这篇文章中有详细的[解答](https://pc.net/helpcenter/answers/post_power_on_self_test)，大致是：POST 代表“开机自检（Power On Self Test）”。它是计算机硬件中内置的诊断程序，可在计算机启动之前测试不同的硬件组件。





BIOS 检查所有可用的存储设备（软盘和硬盘），如果有设备是可引导的，则检查第一个扇区的最后两个字节是否具有引导记录签名0x55，0xAA。如果是这样，BIOS 将第一个扇区加载到地址7C00h，将程序计数器设置为该地址并让 CPU 从那里执行代码。





第一个扇区称为 *M*aster *B*oot *R*ecord，或称为 MBR。在第一个扇区的程序叫做 MBR Bootloader。







## 使用 BIOS 服务

BIOS 提供了许多用于在引导阶段控制硬件的基本服务。这些基本服务是一组控制特定硬件设备或返回当前系统信息的例程。**每一个服务都有一个中断号**。





为了调用 BIOS，必须使用 int x 指令（x 是中断号）。每一个 BIOS 服务定义了属于他自己的所有例程的中断号，为了调用一个例程，必须向寄存器中写入一个特定的号码。所有 BIOS 中断列表都可以在Ralf Brown 的中断列表中找到：http://www.cs.cmu.edu/~ralf/files.html





![](https://upload-images.jianshu.io/upload_images/15548795-e95d24b1700d7e51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





![](https://upload-images.jianshu.io/upload_images/15548795-31b7e353f8ac4cb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





Example：中断 call 13h（软盘服务）需要读取扇区数，磁道号，扇区号，磁头号和从存储设备读取的驱动器号。



扇区的内容存储在由寄存器（ES:BX）定义的存储器地址中。参数存储在这样的寄存器中：





```assembly
; Store sector content in the buffer 10FF:00002
mov dx, 10FFh
mov es,dx
xor bx,bx
mov al, 2; read 2 sector
mov ch, 0; read track 0
mov cl, 2; 2nd sector is read
mov dh, 0; head number
mov dl, 0; drive number. Drive 0 is floppy drive.
mov ah, 0x02; read floppy sector function
int 0x13; call BIOS - Read the sector
```



BIOS 仅在实模式下可用。但是，当切换保护模式时，BIOS 将不再可用，并且操作系统代码负责控制硬件设备。



## 7.3 Boot process



来看看 Boot process 是怎么启动以及发挥作用的



+ BIOS 通过跳转到 0000:7c00h 将控制转移到 MBR 引导加载程序，其中假定引导加载程序已经存在

+ 通过正确初始化段寄存器以启用线性内存模型来设置引导机器环境

+ 加载内核
  1. 从磁盘中读内核
  2. 保存在主存中的某处
  3. 跳到内核开始代码处执行
+ 如果发生错误，打印消息以通知用户出错并停止





