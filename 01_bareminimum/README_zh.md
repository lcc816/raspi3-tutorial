教程 01 - 裸机最小系统
---------------------

好了, 现在我们不做其他的事, 只是单纯地测试我们的工具链. 最终 kernel8.img 应当被引导到树莓派上, 并且CPU 内核进入一个无限循环.

```sh
$ qemu-system-aarch64 -M raspi3 -kernel kernel8.img -d in_asm
        ... output removed for clearity, last line: ...
0x0000000000080004:  17ffffff      b #-0x4 (addr 0x80000)
```

Start
-----

当控制权转交到 kernel8.img 时, C 环境还没有准备好, 因此我们必须用汇编语言实现一小段前奏. 作为第一篇教程, 内容非常简单, 除了一段汇编程序外, 不存在 C 程序.

注意该 CPU 是 4 核, 现在所有这些内核都执行同样的死循环.

Makefile
--------

当前的 Makefile 文件非常简单. 我们只需要编译唯一的源码 start.S. 然后在链接阶段, 我们使用 linker.ld 脚本对其进行链接. 最后, 我们将得到的 elf 可执行文件转换为原始映像.

Linker 脚本
----------

不出预料, 此文件依然很简单. 我们只需要设置 kernel8.img 将被加载到的基地址, 并在此处放入唯一一段. 需要注意, 对于 AArch64, 加载地址是 **0x80000**, 而不是 AArch32 的 **0x8000**.
