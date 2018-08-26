AArch64 交叉编译
===============

在开始这套教程之前, 你需要准备一些工具, 即编译器，用于编译AArch64体系结构及其配套实用程序.

**重要提示**: 这篇说明并不介绍如何编译一个通用的交叉编译器, 而只是介绍如何编译用于 *AArch64* 目标器件的交叉编译器. 如有疑问, 请谷歌搜索 "how to build a gcc cross-compiler" 或者访问你的操作系统相应的支持论坛. 我不能也不会对你的操作系统环境给出帮助, 你需要自己解决这些问题. 正如我在介绍中所说, 我假设你知道如何编译程序 (包括交叉编译器的编译).

**提示**: 如果不喜欢 gcc, 感谢 @laroche, 此教程也经过 Clang 测试.

编译系统
-------

为了组织编译, 我们将使用 GNU make. 由于只是在宿主机而非目标机上使用, 我们不必进行交叉编译. 在本教程中使用 GUN make 构建系统的原因是在编译编译器的时候也需要它.

下载和解压源码
-------------

首先, 下载 binutils 和 gcc 源码. 此例中, 我使用了文档编写时的最新版本. 建议你检查镜像, 并修改相应的文件名.

```sh
wget https://ftpmirror.gnu.org/binutils/binutils-2.30.tar.gz
wget https://ftpmirror.gnu.org/gcc/gcc-8.1.0/gcc-8.1.0.tar.gz
wget https://ftpmirror.gnu.org/mpfr/mpfr-4.0.1.tar.gz
wget https://ftpmirror.gnu.org/gmp/gmp-6.1.2.tar.bz2
wget https://ftpmirror.gnu.org/mpc/mpc-1.1.0.tar.gz
wget https://gcc.gnu.org/pub/gcc/infrastructure/isl-0.18.tar.bz2
wget https://gcc.gnu.org/pub/gcc/infrastructure/cloog-0.18.1.tar.gz
```

如果要校验下载, 可以下载相应的校验值和签名:

```sh
wget https://ftpmirror.gnu.org/binutils/binutils-2.30.tar.gz.sig
wget https://ftpmirror.gnu.org/gcc/gcc-8.1.0/gcc-8.1.0.tar.gz.sig
wget https://ftpmirror.gnu.org/mpfr/mpfr-4.0.1.tar.gz.sig
wget https://ftpmirror.gnu.org/gmp/gmp-6.1.2.tar.bz2.sig
wget https://ftpmirror.gnu.org/mpc/mpc-1.1.0.tar.gz.sig
wget https://gcc.gnu.org/pub/gcc/infrastructure/sha512.sum
```

下载完成后, 运行以下脚本校验下载:

```sh
sha512sum -c sha512.sum --ignore-missing
for i in *.sig; do gpg2 --auto-key-retrieve --verify-files "${i}"; done
```

对于 isl 和 cloog, 第一条命令应当输出 'OK'; 对于其他文件, 第二条命令应当输出 'Good signature'.

然后使用以下命令解压缩下载的 tar 包:

```sh
for i in *.tar.gz; do tar -xzf $i; done
for i in *.tar.bz2; do tar -xjf $i; done
```

删除那些我们不再需要的文件:

```sh
rm -f *.tar.* sha512.sum
```

在开始编译之前, 你需要创建一些符号链接:

```sh
cd binutils-*
ln -s ../isl-* isl
cd ..
cd gcc-*
ln -s ../isl-* isl
ln -s ../mpfr-* mpfr
ln -s ../gmp-* gmp
ln -s ../mpc-* mpc
ln -s ../cloog-* cloog
cd ..
```

编译源码
-------

现在我们需要编译两个程序包. 第一个称为 *binutils*, 其中包括连接器, 汇编器和其他有用的命令.

创建交叉编译工具的安装目录:
```sh
sudo mkdir /opt/cross-compiler/aarch64
sudo chown -R $USER /opt/cross-compiler
```

外部编译 *binutils* 源码:

```sh
mkdir build-binutils
cd build-binutils
../binutils-*/configure --prefix=/opt/cross-compiler/aarch64 --target=aarch64-elf \
--enable-shared --enable-threads=posix --enable-libmpx --with-system-zlib --with-isl --enable-__cxa_atexit \
--disable-libunwind-exceptions --enable-clocale=gnu --disable-libstdcxx-pch --disable-libssp --enable-plugin \
--disable-linker-build-id --enable-lto --enable-install-libiberty --with-linker-hash-style=gnu --with-gnu-ld\
--enable-gnu-indirect-function --disable-multilib --disable-werror --enable-checking=release --enable-default-pie \
--enable-default-ssp --enable-gnu-unique-object
make -j4
make install
```

其中, 第一个参数告诉 configure 脚本将构建结果安装到 `/usr/local/cross-complier`. 第二个参数指定我们要为其编译工具的目标架构. 其他参数打开或关闭特定的选项, 不必改动, 只要知道它们适合嵌入式系统的编译就行了.

第二个需要编译的程序包自然就是 *gcc complier* 本身了.

```sh
mkdir build-gcc
cd build-gcc
../gcc-*/configure --prefix=/opt/cross-compiler/aarch64 --target=aarch64-elf \ --enable-languages=c \
--enable-shared --enable-threads=posix --enable-libmpx --with-system-zlib --with-isl --enable-__cxa_atexit \
--disable-libunwind-exceptions --enable-clocale=gnu --disable-libstdcxx-pch --disable-libssp --enable-plugin \
--disable-linker-build-id --enable-lto --enable-install-libiberty --with-linker-hash-style=gnu --with-gnu-ld\
--enable-gnu-indirect-function --disable-multilib --disable-werror --enable-checking=release --enable-default-pie \
--enable-default-ssp --enable-gnu-unique-object
make -j4 all-gcc
make install-gcc
```

这里我们指定同先前一样的安装目录和架构. 我们还指定只编译 C 编译器, 以免 gcc 会支持大量我们不需要的语言, 这样可以极大地减少编译时间. 余下参数同编译 binutils 一样.

现在查看 `/usr/local/cross-complier` 目录下的 `bin` 文件夹, 应当存在大量可执行文件. 你或许希望将此目录添加到 PATH 环境变量.

```sh
ls /usr/local/cross-compiler/bin
aarch64-elf-addr2line  aarch64-elf-elfedit    aarch64-elf-gcc-ranlib  aarch64-elf-ld       aarch64-elf-ranlib
aarch64-elf-ar         aarch64-elf-gcc        aarch64-elf-gcov        aarch64-elf-ld.bfd   aarch64-elf-readelf
aarch64-elf-as         aarch64-elf-gcc-7.2.0  aarch64-elf-gcov-dump   aarch64-elf-nm       aarch64-elf-size
aarch64-elf-c++filt    aarch64-elf-gcc-ar     aarch64-elf-gcov-tool   aarch64-elf-objcopy  aarch64-elf-strings
aarch64-elf-cpp        aarch64-elf-gcc-nm     aarch64-elf-gprof       aarch64-elf-objdump  aarch64-elf-strip
```

我们感兴趣的可执行文件有:
- aarch64-elf-as - 汇编器
- aarch64-elf-gcc - 编译器
- aarch64-elf-ld - 链接器
- aarch64-elf-objcopy - 将 ELF 可执行文件转换为 IMG 格式
- aarch64-elf-objdump - 用于反汇编可执行文件的实用程序（用于调试）
- aarch64-elf-readelf - 转存可执行文件中的节和段的有用实用程序 (用于调试)

如果你拥有了以上 6 个可执行文件, 并且运行时没有错误, 恭喜! 至此你已经获得所有需要的工具, 可以开始使用我的教程.
