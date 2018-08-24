树莓派 3 裸机编程
=================

这篇教程是为那些有意在树莓派上编译他们的裸机程序的开发者写作的.

教程的目标受众是那些使用这款新硬件的操作系统开发爱好者. 我将为你提供一些例子, 包括写串行控制台, 从串行控制台读取键盘输入, 设置屏幕分辨率和绘制到线性帧缓冲区. 我还会向你展示如何获取该硬件的序列(一串硬件支持的随机号码), 以及如何读取 boot 分区的文件.

这 *不是* 一篇讲述如何编写一个操作系统的教程, 其不涵盖内存管理, 虚拟文件系统和多任务实现等相关主题. 如果你计划自己为树莓派编写操作系统, 建议你在开始前进行一些搜索. 这篇教程和硬件接口紧密相关, 而非操作系统理论. 如果对操作系统理论感兴趣, 推荐访问 [raspberry-pi-os](https://github.com/s-matyukevich/raspberry-pi-os).

现在假设你具有良好的 GUN/Linux 知识, 知道如何编, 会创建硬盘和文件系统映像. 我将不会再详细讲述这些知识, 尽管我会给你一些关于如何设置此架构的交叉编译平台的提示.

为什么选择树莓派 3
-------------------

选择这款板子基于以下几点原因:
首先, 这款板子比较便宜且易获得.
第二, 它是 64 位机器. 我很久以前就摒弃了 32 位系统的编程. 64 位机器则有趣得多, 因为它的地址空间非常大, 大过了存储空间, 这使得我们可以使用一些有趣的新解决方案.
第三, 仅使用MMIO，使得它很容易编程.

关于 32 位的教程, 我推荐:

[Cambridge tutorials](http://www.cl.cam.ac.uk/projects/raspberrypi/tutorials/os/) (只使用 ASM, 32 位),

[David Welch's tutorials](https://github.com/dwelch67/raspberrypi) (大多数使用 C, 包含部分 64 位的例子),

[Peter Lemon's tutorials](https://github.com/PeterLemon/RaspberryPi) (只使用 ASM, 同时适用于 64 位)  和

[Leon de Boer's tutorials](https://github.com/LdB-ECM/Raspberry-Pi) (使用 C 和 ASM, 也适用于 64 位, 有更复杂的例子, 如 USB 和 OpenGL).

前提准备
--------

在开始之前, 你需要一个交叉编译器 (详见 00_crosscompiler 目录) 和一张包含了[固件文件](https://github.com/raspberrypi/firmware/tree/master/boot)在文件系统中的 Micro SD 卡. 感谢[@laroche](https://github.com/laroche)还使用 Clang 测试了本教程.

建议你找一个 [Micro SD 卡的 USB 适配器](http://media.kingston.com/images/products/prodReader-FCR-MRG2-img.jpg) (许多制造商都提供带有这种适配器的SD卡), 这使你可以将 SD 卡像 U 盘一样连接到电脑, 而不需要特殊的读卡器接口 (尽管现在很多笔记本电脑带有这样的接口).

你可以在SD卡上使用LBA FAT32 (键入 0x0C) 分区创建MBR分区方案，格式化该分区并将bootcode.bin, start.elf 和 fixup.dat 复制到其中. 或者你可以下载 rsapbian 镜像, 将其 `dd` 到 SD 卡, 挂载 SD 卡, 然后删除其中不需要的 .img 文件. 不论采用哪种方法, 重要的是, 你将必须创建一个用于此教程的 `kernel8.img` 文件并将其复制到 SD 卡的根目录中, 其他 `.img` 文件则不能存在.

另外建议找一根 [USB 串行调试电缆](https://www.adafruit.com/product/954). 将电缆连接到 GPIO 14/15 引脚, 在电脑上运行 minicom, 命令如下:

```sh
minicom -b 115200 -D /dev/ttyUSB0
```

仿真
----

很不幸, 官方的 qemu 二进制还不支持树莓派 3, 但好消息是, 我已经实现了它, 所以它即将被发布 (UPDATE: [qemu 2.12](https://wiki.qemu.org/ChangeLog/2.12#ARM) 已经可以获得). 在此之前，您必须从最新的源代码编译 qemu. 编译好后, 你可以这样使用:

```sh
qemu-system-aarch64 -M raspi3 -kernel kernel8.img -serial stdio
```

或者 (带文件系统)

```sh
qemu-system-aarch64 -M raspi3 -kernel kernel8.img -drive file=$(yourimagefile),if=sd,format=raw -serial stdio
```
**-M raspi3**
第一条参数告诉 qemu 模拟 Raspberry Pi 3 硬件.

**-kernel kernel8.img**
第二条参数指出要使用的内核文件.

**-drive file=$(yourimagefile),if=sd,format=raw**
第二种情况下, 这条参数也可以指出要使用的 SD 卡镜像文件, 它也可以是标准的 raspbian 镜像.

**-serial stdio**

**-serial null -serial stdio**
最后一个参数将模拟的 UART0 重定向到运行 qemu 的终端的标准 I/O, 以便所有发送到串行线路的内容都会被显示, 并且终端中的每次键入都将被 vm 接收.  仅适用于教程 05 及以上版本，因为默认情况下不会重定向 UART1. 为此，你必须添加类似 `-chardev socket,host=localhost,port=1111,id=aux -serial chardev:aux` 的命令 (感谢 [@godmar](https://github.com/godmar) 贡献此信息), 或者简单地使用两条 `-serial` 参数 (感谢  [@cirosantilli](https://github.com/cirosantilli)).

**!!!WARNING!!!** Qemu 只能进行基本的仿真, 它只能模拟那些最常见的外设! **!!!WARNING!!!**

关于硬件
--------

互联网上充斥着大量详细描述树莓派 3 的页面, 因此我仅简要介绍一些基础知识.

这块板子搭载一款 [BCM2837 SoC]() 芯片, 内含

 - VideoCore GPU,
 - ARM-Cortex-A53 CPU (ARMv8),
 - 一些 MMIO 映射外设.
 
 有趣的是, CPU 不是板上的主处理器, 当板子上电时, 首先运行的是 GPU. GPU 通过执行 bootcode.bin 中的代码完成初始化后, 再执行 start.elf 可执行文件. 该文件不是一个 ARM 可执行文件, 而是为 GPU 编译的. 我们感兴趣的是 start.elf 将查找不同的 ARM 可执行文件, 所有这些文件都以 `kernel` 开头, 以 `.img` 结尾. 由于我们要进行 AArch64 模式的 CPU 编程, 我们只需要 `kernel8.img`, 这是最后要查找的. 一旦完成加载, GPU 就会触发 ARM 处理器上的复位线, ARM 处理器然后开始执行地址 0x80000 处的代码 (或者更确切地说是 0 处的代码, 但 GPU 首先在该处放置了 ARM 跳转代码). 
 
 RAM (树莓派 3 具有 1G) 由 GPU 和 CPU 共享, 这意味着它们可以相互读取对方写入内存中的内容. 为避免冲突, 建立了一个定义良好的 [mailbox 接口](https://github.com/raspberrypi/firmware/wiki/Mailboxes). CPU 将消息写入邮箱, 并通知 GPU 读取它. GPU (知道消息完全在内存中) 解释它，并在同一地址放置响应消息. CPU 必须轮询内存以了解 GPU 何时完成, 然后它才能读取响应. 类似地, 所有外围设备都在内存中与 CPU 通信. 从 0x3F000000 开始, 每个外设占有其中一段专用的内存地址, 但这些地址并不真的存在于 RAM 中 (这称为 *内存映射 IO*). 外设没有 mailbox 接口, 但它们有自己的协议. 这些设备的共同之处在于它们的内存必须以 4 字节对齐的地址, 以 32 位为单位读写 (所谓的 *字*), 并且每个设备都有控制/状态数据字. 不幸的是, Broadcom 公司 (这款 SoC 的制造商) 提供产品文档方面表现不佳. 我们能获得的最好的文档是 BCM2835 文档, 已经足够接近我们的硬件. (UPDATE: 树莓派提供了更新版本 
[BCM2837 documentation](https://github.com/raspberrypi/documentation/files/1888662/)).

CPU 中还有存储管理单元, 允许创建虚拟地址空间. 此功能可以通过特定的 CPU 寄存器编程, 并且在你将这些 MMIO 地址映射到虚拟地址空间的过程中必须十分小心.

我们比较感兴趣的 MMIO 地址有:
```
0x3F003000 - System Timer
0x3F00B000 - Interrupt controller
0x3F00B880 - VideoCore mailbox
0x3F100000 - Power management
0x3F104000 - Random Number Generator
0x3F200000 - General Purpose IO controller
0x3F201000 - UART0 (serial port, PL011)
0x3F215000 - UART1 (serial port, AUX mini UART)
0x3F300000 - External Mass Media Controller (SD card reader)
0x3F980000 - Universal Serial Bus controller
```

有关更多信息请访问树莓派的 wiki 和 github 上的文档.

https://github.com/raspberrypi

祝你好运, 并享受 hack 树莓派的过程! :-)
