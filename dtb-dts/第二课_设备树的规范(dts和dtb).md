* 转载请注明文章地址 http://wiki.100ask.org/Linux_devicetree
# 第01节_DTS格式
dts文件通过编译生成dtb格式文件<br>
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter2_1_001.jpg)
## 属性的定义
value取值类型
属性名=值只有三种取值
* 第一种  <1 0x3 0x123> (一个或多个32位数据)  arrays of cells
* 第二种 “字符串” (用双引号括起来的值)
* 第三种 [ 00 11 22] (byte string 是16进制表示的一个或者多个字节) 
*  一个 byte string必须用2位16进制数表示 byte之间的空格可以省略,可组合多种类型的值，之间用逗号分开

示例内容
示例: 
a. Arrays of cells : cell就是一个32位的数据<code>interrupts = <17 0xc>; </code><br>
b. 64bit数据使用2个cell来表示: <code>clock-frequency = <0x00000001 0x00000000>; </code><br>
c. A null-terminated string (有结束符的字符串): <code>compatible = "simple-bus"; </code><br>
d. A bytestring(字节序列) :<code>local-mac-address = [00 00 12 34 56 78]; </code> 每个byte使用2个16进制数来表示<br>
e. 可以是各种值的组合, 用逗号隔开:
> compatible = "ns16550", "ns8250";
 example = <0xf00f0000 19>, //"a strange property format";

##设备节点如何定义？
```c
[label:] node-name[@unit-address] {
	[properties definitions]
	[child nodes]
};
```
比如
```c
memory@30000000 {
		device_type = "memory";
		reg =  <0x30000000 0x4000000>;
};
```
其中memory@30000000就表示node-name[@unit-address]其中的unit-address是内存首地址用来区分其它同名的设备
可以把节点理解为目录，也就是同一目录下的子目录名称不能相同
## 有哪些需要注意的事项
比如2440设备树文件必须要包含的
	
	 model = "SMDK2440";
	 compatible = "samsung,smdk2440";
	 #address-cells = <1>;//表示子节点的地址宽度是32位
	 #size-cells = <1>;//表示子节点的位宽是32位

特殊的、默认的属性:
a.根节点:

	 #address-cells   // 在它的子节点的reg属性中, 使用多少个u32整数来描述地址(address)
	 #size-cells      // 在它的子节点的reg属性中, 使用多少个u32整数来描述大小(size)
	 compatible       // 定义一系列的字符串, 用来指定内核中哪个
例如<code> compatible = "samsung,smdk2440", "samsung,s3c24xx";<code> //它会优先去内核中寻找		    samsung,smdk2440，如果没有则寻找samsung,s3c24xx第二项，

	*machine_desc可以支持本设备
	 // 即这个板子兼容哪些平台	
	 // uImage : smdk2410 smdk2440 mini2440==> machine_desc		 			 
	 model  // 咱这个板子是什么
	 // 比如有2款板子配置基本一致, 它们的compatible是一样的
	 // 那么就通过model来分辨这2款板子

b. /memory
>device_type = "memory";
	 reg             // 用来指定内存的地址、大小

c. /chosen
> bootargs        // 内核command line参数, 跟u-boot中设置的bootargs作用一样

d. /cpus
> /cpus结点下有1个或多个cpu子结点, cpu子结点中用reg属性用来标明自己是哪一个cpu,
*所以 /cpus 中有以下2个属性:
 #address-cells   // 在它的子节点的reg属性中, 使用多少个u32整数来描述地址(address)
 #size-cells      // 在它的子节点的reg属性中, 使用多少个u32整数来描述大小(size)  必须设置为0

e. /cpus/cpu*

>device_type = "cpu";
reg             // 表明自己是哪一个cpu

引用其他节点:
a. phandle : // 节点中的phandle属性, 它的取值必须是唯一的(不要跟其他的phandle值一样)
```c
pic@10000000 {
	phandle = <1>;
	interrupt-controller;
};

another-device-node {
	interrupt-parent = <1>;   // 使用phandle值为1来引用上述节点
};
```
b. label:
```c
PIC: pic@10000000 {
	interrupt-controller;
};

another-device-node {
	interrupt-parent = <&PIC>;   // 使用label来引用上述节点, 
	                             // 使用lable时实际上也是使用phandle来引用, 
								 // 在编译dts文件为dtb文件时, 编译器dtc会在dtb中插入phandle属性
};
```

## 举例说明
如果我想在dts中包含dtsi文件

新建 jz2440.dtsi
拷贝jz2440.dts
dtsi文件时dts的父节点可以直接引用，语法格式相同，
在dts文件中引用dtsi,比如想修改某个引脚，但是又不想修改dtsi文件，则只需要在dts文件中覆盖掉原来的的配置即可
```c
#include "jz2440.dtsi"
/{
	led {
			ping = <S3C2410_GPF(6)>;
	}
	
}
```
上传文件，
设置环境变量，编译
如果我想反编译dtb文件怎么做？
当前目录下执行
>./scripts/dtc/dtc -I 输入文件dtb -O 输出文件dts -o tmp.dts（输出文件名） 指定dtb文件所在位置
./scripts/dtc/dtc -I dtb -O dts -o tmp.dts arch/arm/boot/dts/jz2440.dtb

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter2_1_002.png)

发现修改后寄存器值变了
再次修改
在dtsi中的led节点上添加lable
```c
LED:led {
	compatible = "jz2440_led";
	pin = <S3C2410_GPF(5)>;
};
```

在dts文件中覆盖
```c
&LED{
	pin = <S3C2410_GPF(7)>;
};
```


上传文件，
设置环境变量，编译,反编译dtb查看已经变化

官方文档:https://www.devicetree.org/specifications/
还可以查看内核目录\linux-4.19-rc3\Documentation\devicetree\usage-model.txt文件

* Linux uses DT data for three major purposes:
* * platform identification,
* * runtime configuration, and
* * device population.

比如你想保留某块内存,保留内存的起始地址以及大小

	/memreserve/ 0x33000000 0x10000
这些配置属于runtime configuration
比如led就属于device population.


# 第02节_DTB格式 

这节视频开始讲解设备树的DTB格式。

## DTS变成DTB
1. 在dtsi文件里，我们使用了各种C语言类似的宏，这些宏需要在被使用的地方展开；
2. dtsi和dts文件中，都是可读性非常强的代码，容易引入错误，需要检测这些错误；
3. 在dts文件里，可以包含一个或多个dtsi文件，这就意味着源文件有很多，需要将它们编译成一个唯一的文件；
4. dtsi和dts文件中，后面属性的值要覆盖前面同名的属性的值；

使用dtc工具将dtsi和dts变成dtb文件时，该工具就自动完成前面的四个操作。
本节视频的知识来源如下两个文档，可以阅读参考：

官方文档: https://www.devicetree.org/specifications/
内核文档: Documentation/devicetree/booting-without-of.txt

## DTB文件布局
DTB文件布局如下:
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter2_2_001.jpg)


可以看出整个DTB分为四个部分:<code>struct ftd_header</code>、<code>memory reservation block</code>、<code>structure block</code>、<code>strings block</code>；

* struct ftd_header:用来表明各个分部的偏移地址，整个文件的大小，版本号等；
* memory reservation block：在设备树中使用<code>/memreserve/ </code>定义的保留内存信息；
* structure block：保存节点的信息，节点的结构；
* strings block：保存属性的名字，单独作为字符串保存；

使用命令<code>make dts</code>编译JZ2440的设备树文件，生成DTB文件，再使用UltraEdit工具打开，方便查看16进制，进行分析dts和dtb的对应关系。
struct ftd_header结构体的定义如下：
```c
struct fdt_header {
uint32_t magic;
uint32_t totalsize;
uint32_t off_dt_struct;
uint32_t off_dt_strings;
uint32_t off_mem_rsvmap;
uint32_t version;
uint32_t last_comp_version;
uint32_t boot_cpuid_phys;
uint32_t size_dt_strings;
uint32_t size_dt_struct;
};
```

在DTB文件中，数据的存放格式是大端模式，即数值的高位存放在低地址。

* 补充知识:大端(big endian)小端(little endian)
对于一个值，比如0x12345678，存放方式如下：
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter2_2_002.jpg)
注意，大端模式和小端模式只针对数值，对于字符串<code> abc</code>，a在低地址，c在高地址。

## 分析DTB内容
下面开始分析DTB的内容：

1. 首先是ftd_header结构体中的magic，为0xd00dfeed；
2. 然后是totalsize，整个DTB文件的大小；
3. 再是off_dt_struct，即structure block的偏移地址；
4. 再是off_dt_strings，即strings block的偏移地址；
5. 再是off_mem_rsvmap，即memory reservation block的偏移地址；


因此，根据偏移，就能找到DTB每个部分的内容。
structure block保存节点的信息，节点的结构，和DTS中节点信息对应如下：
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter2_2_003.jpg)

其中节点信息结构体如下：
```c
struct {
	uint32_t len;
	uint32_t nameoff;
}
```

len表示val长度；
nameoff表示在string block的偏移；


## 总结 
最后总结一下：

1. DTB文件可以分为四个部分:<code>struct ftd_header</code>、<code>memory reservation block</code>、<code>structure block</code>、<code>strings block</code>；
2. 最开始的为struct ftd_header，包含其它三个部分的偏移地址；
3. memory reservation block记录保留内存信息；
4. structure block保存节点的信息，节点的结构；
5. strings block保存属性的名字，将属性名字单独作为字符串保存；


