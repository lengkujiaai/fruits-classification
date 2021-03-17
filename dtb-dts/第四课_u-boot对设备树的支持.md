* 转载请注明文章地址 http://wiki.100ask.org/Linux_devicetree

# 第01节_传递dtb给内核
先把设备树文件读到内存，在启动内核时把设备树的地址写到r2寄存器中
a. u-boot中内核启动命令:

>   bootm <uImage_addr>                            // 无设备树,bootm 0x30007FC0
   bootm <uImage_addr> <initrd_addr> <dtb_addr>   // 有设备树   

比如 :
>   nand read.jffs2 0x30007FC0 kernel;     // 读内核uImage到内存0x30007FC0
   nand read.jffs2 32000000 device_tree;  // 读dtb到内存32000000
   bootm 0x30007FC0 - 0x32000000          // 启动, 没有initrd时对应参数写为"-"

b. bootm命令怎么把dtb_addr写入r2寄存器传给内核?
在百度搜索<code>ARM程序调用规则(ATPCS)</code>
写一个c函数
 c_function(p0, p1, p2) // p0 => r0, p1 => r1, p2 => r2（3个参数分别保存到相应的寄存器）    
定义函数指针 the_kernel, 指向内核的启动地址,然后执行: the_kernel(0, machine_id, 0x32000000);

armlinux.c中
```c 
     /* 100ask for device tree, no initrd image used */
	if (argc == 4) {
		//第三个参数0x32000000就是设备树地址
		of_flat_tree = (char *) simple_strtoul(argv[3], NULL, 16);

		if  (be32_to_cpu(*(ulong *)of_flat_tree) == OF_DT_HEADER) {
			printf ("\nStarting kernel with device tree at 0x%x...\n\n", of_flat_tree);

			cleanup_before_linux ();
			//把dtb的地址传到r2寄存器里			
			theKernel (0, bd->bi_arch_number, of_flat_tree);
					
		} else {
			printf("Bad magic of device tree at 0x%x!\n\n", of_flat_tree);
		}
		
	}
```
c. dtb_addr 可以随便选吗?
&nbsp;&nbsp;&nbsp;&nbsp;c.1 不要破坏u-boot本身
&nbsp;&nbsp;&nbsp;&nbsp;c.2 不要挡内核的路: 内核本身的空间不能占用, 内核要用到的内存区域也不能占用
内核启动时一般会在它所处位置的下边放置页表, 这块空间(一般是0x4000即16K字节)不能被占用  
JZ2440内存使用情况:
```
                     ------------------------------
  0x33f80000       ->|    u-boot                  | 分析lds链接文件
                     ------------------------------
                     |    u-boot所使用的内存(栈等)|
                     ------------------------------
                     |                            |
                     |                            |
                     |        空闲区域            |
                     |                            |
                     |                            |
                     |                            |
                     |                            |
                     ------------------------------
  0x30008000       ->|      zImage                |
                     ------------------------------  uImage = 64字节的头部+zImage
  0x30007FC0       ->|      uImage头部            |
                     ------------------------------
  0x30004000       ->|      内核创建的页表        |  head.S
                     ------------------------------
                     |                            |
                     |                            |
              -----> ------------------------------
              |
              |
              --- (内存基址 0x30000000)
```

我如何知道内核放在 0x30008000
在内核目录下执行<code> mkimage -l arch/arm/boot/uImage</code>
里面显示内核的load address = 0x30008000 最终运行也在0x30008000位置


命令示例:
a. 可以启动:
> nand read.jffs2 30000000 device_tree
 nand read.jffs2 0x30007FC0 kernel
 bootm 0x30007FC0 - 30000000

b. 不可以启动: 内核启动时会使用0x30004000的内存来存放页表，dtb会被破坏
> nand read.jffs2 30004000 device_tree
 nand read.jffs2 0x30007FC0 kernel
 bootm 0x30007FC0 - 30004000
              
# 第02节_dtb的修改原理

如果修改设备树中的led设备引脚，有两种办法
* 修改dts文件，重新编译得到dtb并上传烧写
* 使用uboot提供的一些命令来修改dtb文件，修改后再把它保存到板子上，以后就使用这个修改后的dtb文件
移动值，也就是通过memmove处理
> memmove(dst,src,len)

拷贝值
> memcpy(dst,src,len)

## 例子1. 修改属性的值,
假设 老值: len
   新值: newlen (假设newlen > len)

* a. 把原属性val所占空间从len字节扩展为newlen字节:
   把老值之后的所有内容向后移动(newlen - len)字节
* b. 把新值写入val所占的newlen字节空间
* c. 修改dtb头部信息中structure block的长度: size_dt_struct
* d. 修改dtb头部信息中string block的偏移值: off_dt_strings
* e. 修改dtb头部信息中的总长度: totalsize

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter4_2_001.png)

扩充 string block
   并且修改dtb头部信息中string block的长度: size_dt_strings
   修改dtb头部信息中的总长度: totalsize

## 例子2. 添加一个全新的属性
a. 如果在string block中没有这个属性的名字,
   就在string block尾部添加一个新字符串: 属性的名
   并且修改dtb头部信息中string block的长度: size_dt_strings
   修改dtb头部信息中的总长度: totalsize

b. 找到属性所在节点, 在节点尾部扩展一块空间, 内容及长度为: 

	   TAG      // 4字节, 对应0x00000003
	   len      // 4字节, 表示属性的val的长度
	   nameoff  // 4字节, 表示属性名的offset 
	   val      // len字节, 用来存放val

c. 修改dtb头部信息中structure block的长度: size_dt_struct
d. 修改dtb头部信息中string block的偏移值: off_dt_strings
e. 修改dtb头部信息中的总长度: totalsize

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter4_2_002.png)

我们需要在 0000,0001与  0000,0002之间加入属性，我需要把0002后面这段空间移动若干字节
加入属性需要移动若干字节，

可以从u-boot官网源码下载一个比较新的u-boot, ftp://ftp.denx.de/pub/u-boot/ 查看它的cmd/fdt.c，里面构造了fdt的命令

fdt命令调用过程:
 fdt set    <path> <prop> [<val>] 
* 根据path找到节点
* 根据val确定新值长度newlen, 并把val转换为字节流
* fdt_setprop 
```c
		3.1 fdt_setprop_placeholder       // 为新值在DTB中腾出位置
		         fdt_get_property_w  // 得到老值的长度 oldlen
				 fdt_splice_struct_  // 腾空间
						fdt_splice_  // 使用memmove移动DTB数据, 移动(newlen-oldlen)
						fdt_set_size_dt_struct  // 修改DTB头部, size_dt_struct 
						fdt_set_off_dt_strings  // 修改DTB头部, off_dt_strings
						
		3.2 memcpy(prop_data, val, len);  // 在DTB中存入新值
```


# 第03节_dtb的修改命令fdt移植
我们仍然使用u-boot 1.1.6, 因为在这个版本上我们实现了很多功能: usb下载,菜单操作,网卡永远使能等, 不忍丢弃。
现在比较新的uboot，已经自带fdc命令，我们使用老版本需要在里面添加fdc命令, 这个命令可以用来查看、修改dtb。

从u-boot官网下载最新的源码, 把里面的 cmd/fdt.c移植过来.
u-boot官网源码:
ftp://ftp.denx.de/pub/u-boot/

如果不想看本节的移植过程，可以直接使用补丁文件打补丁，得到移植后的uboot。
最终的补丁存放在如下目录: <code>doc_and_sources_for_device_tree\source_and_images\u-boot\u-boot-1.1.6_device_tree_for_jz2440_add_fdt_20181022.patch</code>

## 补丁使用方法
1.设置交叉编译工具链
```
  export  PATH=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/work/system/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabi/bin
```

2.解压1.1.6版本的uboot
```
  tar xjf u-boot-1.1.6.tar.bz2  // 解压
```

3.进入解压的uboot
```
cd u-boot-1.1.6       
```

4.打补丁
```
patch -p1 < ../u-boot-1.1.6_device_tree_for_jz2440_add_fdt_20181022.patch   // 打补丁
```

5.重新配置，编译uboot
```
  make 100ask24x0_config     // 配置
  make                       // 编译, 可以得到u-boot.bin
```


## 移植fdt
**a.1 先把代码移过去, 修改Makefile来编译**
<code>u-boot-2018.11-rc2\lib\libfdt</code>主要用这个目录，它里面的大部分文件是直接包含<code>scripts\dtc\libfdt</code>中的同名文件,只有2个文件是自己的版本，即<code>fdt_region.c</code>和<code>fdt_ro.c</code>。
把新u-boot中<code>cmd/fdt.c</code>重命名为<code>cmd_fdt.c</code> , 和 <code>lib/libfdt/*</code>一起复制到老u-boot的<code>common/fdt</code>目录；
修改老u-boot中<code>u-boot/Makefile</code>，添加一行:<code>LIBS += common/fdt/libfdt.a</code>；
修改老u-boot中<code>u-boot/common/fdt/Makefile</code>， 仿照<code>drivers/nand/Makefile</code>来修改；

**a.2 根据编译的错误信息修改源码**

移植时常见问题:
i. **No such file or directory:**
```c 
   #include "xxx.h"  // 是在当前目录下查找xxx.h
   #include <xxx.h>  // 是在指定目录下查找xxx.h
```
这里的指定目录，在编译文件时可以用"-I"选项指定头文件目录，比如: <code>arm-linux-gcc -I <dir> -c -o ....</code>，对于u-boot来说, 一般就是源码的<code>include</code>目录。
   

* 解决方法:
确定头文件在哪, 把它移到include目录或是源码的当前目录。

**ii. xxx undeclared :**
宏, 变量, 函数未声明/未定义

* 解决方法:
对于宏, 去定义它;
对于变量, 去定义它或是声明为外部变量;
对于函数, 去实现它或是声明为外部函数;

**iii. 上述2个错误是编译时出现***

当一切都没问题时, 最后就是链接程序, 这时常出现: **undefined reference to `xxx'**
这表示代码里用到了xxx函数, 但是这个函数没有实现
   

* 解决方法:
去实现它, 或是找到它所在文件, 把这文件加入工程

## fdt命令使用示例
```shell
nand read.jffs2 32000000 device_tree  // 从flash读出dtb文件到内存(0x32000000)
fdt addr 32000000                     // 告诉fdt, dtb文件在哪
fdt print /led pin                    // 打印/led节点的pin属性
fdt get value XXX /led pin            // 读取/led节点的pin属性, 并且赋给环境变量XXX
print XXX                             // 打印环境变量XXX的值
fdt set /led pin <0x00050005>         // 设置/led节点的pin属性
fdt print /led pin                    // 打印/led节点的pin属性
nand erase device_tree                // 擦除flash分区
nand write.jffs2 32000000 device_tree // 把修改后的dtb文件写入flash分区
```

