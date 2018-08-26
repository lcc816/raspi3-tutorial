教程 0A - PC 屏幕字体
====================

绘制像素图很有意思, 但是肯定还需要显示字符. 基本地, 字体只不过是每个字符的位图. 在本教程中, 我选择了 PC Screen 字体格式, Linux Console 也使用这一格式.

Lfb.h, lfb.c
------------

`lfb_init()` 设置分辨率, 深度和颜色通道顺序, 并且查询帧缓冲区地址.

`lfb_print(x, y, s)` 在屏幕上显示一个字符串.

Font.psf
--------

字体文件. 可使用 /usr/share/kbd/consolefonts 中的任何文件. 不支持 Unicode 表, 使 Unicode 表把字符转换为字形索引 (而不是一对一关系) 作为你的一项作业. 此字体由原始 IBM PC VGA 8x16 Font ROM 生成，包含 127 个字形.

Makefile
--------

我已经添加了一个新对象文件, 生成自该 psf 文件. 这是一个如何在 C 中包含和引用二进制文件的好例子. 我使用了以下命令找出标签:

```sh
$ aarch64-elf-readelf -s font.o
        ... output removed for clearity ...
     2: 0000000000000820     0 NOTYPE  GLOBAL DEFAULT    1 _binary_font_psf_end
     3: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT    1 _binary_font_psf_start
     4: 0000000000000820     0 NOTYPE  GLOBAL DEFAULT  ABS _binary_font_psf_size
```

Main
----

很简单, 此文件中我们设置分辨率并显示字符串.
