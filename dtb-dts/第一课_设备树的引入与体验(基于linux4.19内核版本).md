* 转载请注明文章地址 http://wiki.100ask.org/Linux_devicetree

本套视频面向如下三类学员：
1. 有Linux驱动开发基础的人, 可以挑感兴趣的章节观看；
2. 没有Linux驱动开发基础但是愿意学习的人,请按顺序全部观看,我会以比较简单的LED驱动为例讲解；
3. 完全没有Linux驱动知识，又不想深入学习的人, 比如应用开发人员，不得已要改改驱动, 等全部录完后，我会更新本文档，那时再列出您需要观看的章节。

# 第01节_字符设备的三种写法
##  怎么写驱动？

①看原理图：

　a.确定引脚；

　b.看芯片手册，确定如何操作引脚；

②写驱动程序；

　起封装作用；

③写测试程序；

如下原理图，VCC经过一个限流电阻到达LED的一端，再通向芯片的引脚上。

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_1_001.jpg)


当芯片引脚输出低电平时，电流从高电平流向低电平，LED灯点亮；
当芯片引脚输出高电平时，没有电势差，没有电流流过，LED灯不亮；
从原理图可以看出，控制了芯片引脚，就等于控制了灯。

在Linux里，操作硬件都是统一的接口，比如操作LED灯，需要先open，如果要读取LED状态就调用read，如果要操作LED就调用write函数，也可以通过ioctl去实现。
在驱动里，针对前面应用的每个调用函数，都写一个对应的函数，实现对硬件的操作。

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_1_002.jpg)


可以看出驱动程序起封装作用，它让应用程序访问硬件变得简单，屏蔽了硬件更加复杂的操作。


## 如何写驱动程序？

①分配一个file_operations结构体；
②设置：
　a. .open=led_open；把led引脚设置为输出引脚
　b. .read=led_read；根据APP传入的值设置引脚状态

③注册（告诉内核），register_chrdev(主设备号，file_operations，name)
④入口函数
⑤出口函数


## 在驱动中如何指定LED引脚？
有如下三种方法：
①传统方法：在代码led_drv.c中写死；
②总线设备驱动模型：
　a. 在led_drv.c里分配、注册、入口、出口等
　b. 在led_dev.c里指定引脚
③使用设备树指定引脚
　a. 在led_drv.c里分配、注册、入口、出口等
　b. 在jz2440.dts里指定引脚

可以看到，'''无论何种方法，驱动写法的核心不变，差别在于如何指定硬件资源'''。
对比下三种方法的优缺点。
假设这样一个情况，某公司用同一个芯片做了两款产品，其中一款是TV(电视盒子)，使用Pin1作为LED的指示灯控制引脚，其中一款是Cam(监控摄像头)，使用Pin2作为LED的指示灯控制引脚。

<table style="border-collapse:collapse;border-spacing:0" class="tg"><tr><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center;vertical-align:top"></th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center">TV设备</th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center">Cam设备</th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center">优缺点</th></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center;vertical-align:top">1.传统方法</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">led_drv.c<br>①分配一个file_operations结构体；<br>②设置：<br>　a .open=led_open；设置Pin1为输出引脚<br>　b .read=led_read；根据APP传入的值设置引脚状态<br>③注册（告诉内核）<br>④入口函数<br>⑤出口函数</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">led_drv.c<br><br>①分配一个file_operations结构体；<br>②设置：<br>　a. .open=led_open；设置Pin2为输出引脚<br>　b. .read=led_read；根据APP传入的值设置引脚状态<br>③注册（告诉内核）<br>④入口函数<br>⑤出口函数<br></td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">优点：简单<br><br>缺点：不易扩展，需要重新编译</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center;vertical-align:top">2.总线设备驱动模型</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">led_drv.c<br>①分配/设置/注册 platform_driver；<br>② .probe：<br>　a 分配一个file_operations结构体；<br>    b .open=led_open；设置平台设备总指定的引脚为输出引脚<br>　    .read=led_read；根据APP传入的值设置引脚状态<br>    c注册<br>③ .driver{ .name }<br><br>led_dev.c<br>①分配/设置/注册 platform_device；<br>② .resource：指定引脚；,name为Pin1<br><br></td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">led_dev.c<br><br>①分配/设置/注册 platform_driver；<br>② .resource：指定引脚；,name为Pin2</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left">优点：易扩展<br><br>缺点：稍复杂，冗余代码太多，需要重新编译</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:center;vertical-align:top">3.设备树</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left;vertical-align:top">led_drv.c<br>①分配/设置/注册 platform_driver；<br>② .probe：<br>　a 分配一个file_operations结构体；<br>    b .open=led_open；设置平台设备总指定的引脚为输出引脚<br>　    .read=led_read；根据APP传入的值设置引脚状态<br>    c注册<br>③ .driver{ .name }<br><br>.dts指定资源<br>内核根据dts生成的dtb文件分配/设置/注册platform_device</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left;vertical-align:top">.dts指定资源<br><br>内核根据dts生成的dtb文件分配/设置/注册platform_device</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:inherit;text-align:left;vertical-align:top">优点：易扩展<br><br>缺点：稍复杂，冗余代码太多，需要重新编译</td></tr></table>

# 第02节_字符设备驱动的传统写法

在上一节视频里我们介绍了三种编写驱动的方法，也对比了它们的优缺点，后面我们将使用比较快速的方法写出驱动程序，因为写驱动程序不是我们这套视频的重点，所以尽快的把驱动程序写出来，给大家展示一下。

这节视频我们使用传统的方法编写字符驱动程序，以最简单的点灯驱动程序为示例。
先回顾下写字符设备驱动的五个步骤：
1.2.3.分配/设置/注册file_operations 
4.入口
5.出口
所谓分配file_operations，我们可以定义一个file_operations结构体，就不需要分配了。
```c
static struct file_operations myled_oprs = {
	.owner = THIS_MODULE, //表示这个模块本身
	.open  = led_open,
	.write = led_write,
	.release = led_release,
};
```

定义好了file_operations结构体，再去入口函数注册结构体。
```c
static int myled_init(void)
{
	major = register_chrdev(0, "myled", &myled_oprs);

	return 0;
}
```

第一个参数：主设备号写0，让系统为我们分配；
第二个参数：设置名字，没有特殊要求；
第三个参数：file_operations结构体；
对应的出口操作进行相反向操作:
```c
static void myled_exit(void)
{
	unregister_chrdev(major, "myled");
}
```

然后用宏module_init对入口、出口函数进行修饰，表示它们和普通函数不一样：
```c
module_init(myled_init);
module_exit(myled_exit);
```

<code>module_init(myled_init)</code>实际就是<code>int init_module(void) __attribute__((alias("myled_init")))</code>，表示<code>myled_init</code>的别名是<code>init_module</code>，以后就可以使用<code>init_module</code>来引用<code>myled_init</code>。


此外，还要加上GPL协议：
```c
MODULE_LICENSE("GPL");
```


写到这里，驱动程序的框架已经搭建起来了，接下来实现具体的硬件操作函数：led_open()和led_write()。
在led_open()里把对应的引脚配置为输出引脚，在led_write()根据应用程序传入的数据点灯，让其输出高电平或低电平。
为了让程序更具有扩展性，把GPIO的寄存器放在一个数组里：
```c
static unsigned int gpio_base[] = {
	0x56000000, /* GPACON */
	0x56000010, /* GPBCON */
	0x56000020, /* GPCCON */
	0x56000030, /* GPDCON */
	0x56000040, /* GPECON */
	0x56000050, /* GPFCON */
	0x56000060, /* GPGCON */
	0x56000070, /* GPHCON */
	0,          /* GPICON */
	0x560000D0, /* GPJCON */
};
```
定义好了引脚的组，还得确定使用该组的哪个引脚，使用宏来确定哪个引脚：
```c
#define S3C2440_GPA(n)  (0<<16 | n)
#define S3C2440_GPB(n)  (1<<16 | n)
#define S3C2440_GPC(n)  (2<<16 | n)
#define S3C2440_GPD(n)  (3<<16 | n)
#define S3C2440_GPE(n)  (4<<16 | n)
#define S3C2440_GPF(n)  (5<<16 | n)
#define S3C2440_GPG(n)  (6<<16 | n)
#define S3C2440_GPH(n)  (7<<16 | n)
#define S3C2440_GPI(n)  (8<<16 | n)
#define S3C2440_GPJ(n)  (9<<16 | n)
```
后面就可以向对应宏传入对应位，得到对应组的对应引脚。
查看原理图，知道我们要使用的引脚是GPF5，因此定义 led_pin = s3c2440_GPF(5)。

```c
static int led_open (struct inode *node, struct file *filp)
{
	/* 把LED引脚配置为输出引脚 */
	/* GPF5 - 0x56000050 */
	int bank = led_pin >> 16;
	int base = gpio_base[bank];

	int pin = led_pin & 0xffff;
	gpio_con = ioremap(base, 8);  
	if (gpio_con) {
		printk("ioremap(0x%x) = 0x%x\n", base, gpio_con);
	}
	else {
		return -EINVAL;
	}
	
	gpio_dat = gpio_con + 1;

	*gpio_con &= ~(3<<(pin * 2));
	*gpio_con |= (1<<(pin * 2));  

	return 0;
}
```

在Linux中，不能直接操作基地址，需要使用ioremap()映射。

对于基地址，定义全局指针来表示，gpio_con表示控制寄存器，gpio_dat表示数据寄存器。

这里将GPF5的第二个引脚先清空，再设置为1，表示输出引脚。

接下来是写函数：
```c
static ssize_t led_write (struct file *filp, const char __user *buf, size_t size, loff_t *off)
{
	/* 根据APP传入的值来设置LED引脚 */
	unsigned char val;
	int pin = led_pin & 0xffff;
	
	copy_from_user(&val, buf, 1);

	if (val)
	{
		/* 点灯 */
		*gpio_dat &= ~(1<<pin);
	}
	else
	{
		/* 灭灯 */
		*gpio_dat |= (1<<pin);
	}

	return 1; /* 已写入1个数据 */
}
```
注意这里的__user宏起强调作用，告诉你buf来自应用空间，在内核里不能直接使用。
使用copy_from_user()将用户空间的数据拷贝到内核空间。
再根据传入的值，设置gpio_dat的值，来点亮或者熄灭pin所对应的灯。
至此，这个驱动程序已经具备操作硬件的功能，但我们还要增加一些内容，比如我们先注册驱动后，自动创建节点信息。
在入口函数里，使用class_create()创建class，并且使用device_create()创建设备。
```c
static int myled_init(void)
{
	major = register_chrdev(0, "myled", &myled_oprs);

	led_class = class_create(THIS_MODULE, "myled");
	device_create(led_class, NULL, MKDEV(major, 0), NULL, "led"); /* /dev/led */

	return 0;
}
```

出口函数需要进行相反操作：
```c
static void myled_exit(void)
{
	unregister_chrdev(major, "myled");
	device_destroy(led_class,  MKDEV(major, 0));
	class_destroy(led_class);
}
```

还有在release函数里，释放前面的iormap()的资源
```c
static int led_release (struct inode *node, struct file *filp)
{
	printk("iounmap(0x%x)\n", gpio_con);
	iounmap(gpio_con);
	return 0;
}
```

最后把以前的测试程序拷贝过来，简单修改一下，见网盘<code>led_driver/001_led_drv_traditional/ledtest.c</code>。
可以看出，这种传统写驱动程序的方法把硬件资源写在了代码里，换个LED，换个引脚，就得去修改<code> led_pin = s3c2440_GPF(5)</code>，然后重新编译，加载。

# 第03节_字符设备驱动的编译测试
这节课来讲解一下测试和编译的过程。
**驱动程序的编译依赖于内核，在驱动程序里的一堆头文件，是来自于内核的，因此我们需要先编译内核**。
接下来我们要编译驱动程序，编译测试程序，并在单板上测试一样。


首先从网盘下载：

<code>doc_and_sources_for_device_tree/source_and_images/source_and_images</code>下的内核源码和补丁；
<code>doc_and_sources_for_device_tree/source_and_images/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabi.tar.xz</code>编译内核和驱动的交叉编译工具链；
<code>doc_and_sources_for_device_tree/source_and_images/arm-linux-gcc-4.3.2.tar.bz2</code>编译测试程序的交叉编译工具链；
<code>doc_and_sources_for_device_tree/source_and_images/readme.txt</code>介绍了一些编译器、工具的使用、uboot等笔记，需要时可以看一看；

**1.编译内核**
将内核源码、补丁、编译内核的交叉工具链上传到Ubuntu，然后解压、打补丁。
再解压工具链，设置工具链环境，最后编译。
编译中遇到错误提示，尝试百度搜索，一般都能找到解决方法。

**2.编译驱动**
待内核编译完后，修改Makefile，编译驱动。
**3.编译应用程序**
解压编译应用程序的交叉编译工具链，修改环境变量，编译应用程序。
**4.加载驱动和运行测试程序**
使用nfs挂载该目录，加载驱动，运行测试程序。

# 第04节_总线设备驱动模型
## 总线驱动模型是为了解决什么问题呢？
* 使用之前的驱动模型，编写一个led驱动程序，如果需要修改gpio引脚，则需要修改驱动源码，重新编译驱动文件，假如驱动放在内核中，则需要重新编译内核
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_4_001.png)
bus总线是虚拟的概念，并非硬件，dev注册设置某个结构体，这个设备也就是平台设备
```c
    struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u32		num_resources;
	/*resource 里面确定使用那些资源*/
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
	char *driver_override; /* Driver name to force a match */

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
	};
```

drv那面定义platform_driver 去注册
```c
    struct platform_driver {
    	int (*probe)(struct platform_device *);
    	int (*remove)(struct platform_device *);
    	void (*shutdown)(struct platform_device *);
    	int (*suspend)(struct platform_device *, pm_message_t state);
    	int (*resume)(struct platform_device *);
    	struct device_driver driver;
    	const struct platform_device_id *id_table;
    	bool prevent_deferred_probe;
    };

```
## 设备和驱动如何进行通信呢
*通过bus进行匹配 platform_match函数确定(dev,drv)若匹配则调用drv中的probe函数
```c
    struct bus_type platform_bus_type = {
    	.name		= "platform",
    	.dev_groups	= platform_dev_groups,
    	.match		= platform_match,
    	.uevent		= platform_uevent,
    	.pm		= &platform_dev_pm_ops,
    };
```

这种模型只是一种编程技巧一种机制
并不是驱动程序的核心

## platform_match是如何判断dev drv是匹配的？
判断方法是比较dev 和drv 各自的name来进行匹配
* 平台设备platform_device这面有name
* platform_driver这面有 driver （里面含有name） 还有id_table（包含 name  driver_data）
* id_table里面的内容表示所支持一个或多个的设备名
```c
static int platform_match(struct device *dev, struct device_driver *drv)
{		
	/*省略部分无用代码*/
	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```
也就是优先比较 id_table中名字，如果没有则对比driver中名字
* 根据二期视频led代码进行修改
```c
/* 分配/设置/注册一个platform_device */
/*设置资源*/
static struct resource led_resource[] = {
    [0] = {
		/*指明了使用那个引脚*/
        .start = S3C2440_GPF(5),
		/*end并不重要，可以随意指定*/
        .end   = S3C2440_GPF(5),
        .flags = IORESOURCE_MEM,
    },
};

static void led_release(struct device * dev)
{
}


static struct platform_device led_dev = {
    .name         = "myled",
    .id       = -1,
    .num_resources    = ARRAY_SIZE(led_resource),
    .resource     = led_resource,
    .dev = { 
    	.release = led_release, 
	},
};

/*入口函数去注册平台设备*/
static int led_dev_init(void)
{
	platform_device_register(&led_dev);
	return 0;
}
/*出口函数去释放这个平台设备*/
static void led_dev_exit(void)
{
	platform_device_unregister(&led_dev);
}

module_init(led_dev_init);
module_exit(led_dev_exit);
```

* led_drv驱动文件
```c
static int led_probe(struct platform_device *pdev)
{
	struct resource		*res;

	/* 根据platform_device的资源进行ioremap */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	led_pin = res->start;

	major = register_chrdev(0, "myled", &myled_oprs);

	led_class = class_create(THIS_MODULE, "myled");
	device_create(led_class, NULL, MKDEV(major, 0), NULL, "led"); /* /dev/led */
	
	return 0;
}


struct platform_driver led_drv = {
	.probe		= led_probe,
	.remove		= led_remove,
	.driver		= {
		.name	= "myled",
	}
};


static int myled_init(void)
{
	platform_driver_register(&led_drv);
	return 0;
}

static void myled_exit(void)
{
	platform_driver_unregister(&led_drv);
}
```

Makefile文件
```c
KERN_DIR = /work/system/linux-4.19-rc3

all:
	make -C $(KERN_DIR) M=`pwd` modules 

clean:
	make -C $(KERN_DIR) M=`pwd` modules clean
	rm -rf modules.order

obj-m	+= led_drv.o
obj-m	+= led_dev.o
```
执行测试程序

如果我需要更换一个led
则只需要修改 led_dev led_resource结构体中的引脚即可
```c
static struct resource led_resource[] = {
    [0] = {
        .start = S3C2440_GPF(6),
        .end   = S3C2440_GPF(6),
        .flags = IORESOURCE_MEM,
    },
};
```

设备和驱动的匹配是如何完成的？
* dev这面有设备链表
* drv这面也有驱动的结构体链表
* 通过match函数进行对比，如果相同，则调用drv中的probe函数

# 第05节_使用设备树时对应的驱动编程
*  本节介绍怎么使用设备树怎么编写对应的驱动程序
*  只是平台设备的构建区别，以前构造平台设备是在.c文件中，使用设备树构造设备节点原本不存在，需要在dts文件中构造节点，节点中含有资源
*  dts被编译成dtb文件传给内核，内核会处理解析dtb文件得到device_node结构体，之后变成platform_device结构体，里面含有资源（资源来自dts文件）

* 我们定义的led设备节点
```c
    	led {
    		compatible = "jz2440_led";
    		reg = <S3C2410_GPF(5) 1>;
    	};
```

* 以后就使用compatible找到内核支持这个设备节点的平台driver <code>reg = <S3C2410_GPF(5) 1>; </code>就是寄存器地址的映射<br>

修改好后编译 设备树文件  make dtb

拷贝到tftp文件夹，开发板启动
*  进入<code> /sys/devices/platform </code>目录查看是否有5005.led平台设备文件夹
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_5_001.png)

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_5_002.png)

*  查看 reg 的地址，这里面是以大字节须来描述这些值的
![](http://photos.100ask.net/ldd/ldd_devicetree_chapter1_5_003.png)

这个属性有8个字节，对应两个数值
*   第一个值S3C2410_GPF(5)是我们的起始地址，对应 #define S3C2410_GPF(_nr)	((5<<16) + (_nr))
*  第二个值1 本意是指寄存器的大小

如何去写平台驱动？
通过bus总线去匹配设备驱动
在  platform_match函数中，通过
```c
	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;
```

进入 of_device.h中
```c
/**
 * of_driver_match_device - Tell if a driver's of_match_table matches a device.
 * @drv: the device_driver structure to test
 * @dev: the device structure to match against
 */
static inline int of_driver_match_device(struct device *dev,
					 const struct device_driver *drv)
{
	return of_match_device(drv->of_match_table, dev) != NULL;
}
```

of_match_table结构体
 include\linux\mod_devicetable.h
```c
/*
 * Struct used for matching a device
 */
struct of_device_id {
	char	name[32];
	char	type[32];
	char	compatible[128];
	const void *data;
};
```

* compatible  也就是从dts得到的platform_device里有compatible 属性，两者进行对比，一样就表示匹配
* 写led驱动，修改led_drv.c
* 添加
```c
static const struct of_device_id of_match_leds[] = {
	{ .compatible = "jz2440_led", .data = NULL },
	{ /* sentinel */ }
};
```

*修改
```c
struct platform_driver led_drv = {
	.probe		= led_probe,
	.remove		= led_remove,
	.driver		= {
		.name	= "myled",
		.of_match_table = of_match_leds, /* 能支持哪些来自于dts的platform_device */
	}
};
```

*修改Makefile并编译

* 如果修改灯怎么办？
* * 直接修改设备树中的led设备节点
```c
    	led {
    		compatible = "jz2440_led";
    		reg = <S3C2410_GPF(6) 1>;
    	};
```
上传编译，直接使用新的dtb文件
我们使用另外一种方法指定引脚
```c
	led {
		compatible = "jz2440_led";
		pin = <S3C2410_GPF(5)>;
	};
```
修改led_drv中的probe函数

在of.h中找到获取of属性的函数 of_property_read_s32
```c
static int led_probe(struct platform_device *pdev)
{
	struct resource		*res;

	/* 根据platform_device的资源进行ioremap */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (res) {
		led_pin = res->start;
	}
	else {
		/* 获得pin属性 */
		of_property_read_s32(pdev->dev.of_node, "pin", &led_pin);
	}

	if (!led_pin) 
	{
		printk("can not get pin for led\n");
		return -EINVAL;
	}
		

	major = register_chrdev(0, "myled", &myled_oprs);

	led_class = class_create(THIS_MODULE, "myled");
	device_create(led_class, NULL, MKDEV(major, 0), NULL, "led"); /* /dev/led */
	
	return 0;
}
```
* 从新编译设备树 和led驱动文件
在platform_device结构体中的struct device dev;中对于dts生成的platform_device这里含有of_node

of_node中含有属性，这取决于设备树，比如compatible属性
让后注册/配置/file_operation

# 第06节_只想使用设备树不想深入研究怎么办

寄希望于写驱动程序的人，提供了文档/示例/程序写得好适配性强<br>
根据之前写的设备树
    	led {
    		compatible = "jz2440_led";
    		reg = <S3C2410_GPF(6) 1>;
    	};
	led {
		compatible = "jz2440_led";
		pin = <S3C2410_GPF(5)>;
	};

可以通过reg指定引脚也可以通过pin指定引脚，我们在设备树中如何指定引脚完全取决于驱动程序
既可以获取pin属性值也可以获取reg属性值
```c
	/* 根据platform_device的资源进行ioremap */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (res) {
		led_pin = res->start;
	}
	else {
		/* 获得pin属性 */
		of_property_read_s32(pdev->dev.of_node, "pin", &led_pin);
	}

	if (!led_pin) 
	{
		printk("can not get pin for led\n");
		return -EINVAL;
	}
```
我们通过驱动程序再次验证了设备树的属性完全却决于写驱动程序的人<br>
commpatible属性必须是 jz2440_led 才可以和驱动匹配成功

我们写驱动的人应该写一个文档，告诉写应用程序的人设备树的节点应该怎么编写

对于内核自带的驱动文件，对应的设备树的文档一般放在<code>Documentation\devicetree\bindings</code>目录中,里面有各种架构的说明文档以及各种协议的说明文档，
这些驱动都能在 drivers 目录下找到对应的驱动程序.
比如查看<code>Documentation\devicetree\bindings\arm\samsung\exynos-chipid.txt</code>里面的内容如下
```c
 SAMSUNG Exynos SoCs Chipid driver.
 Required properties:（必须填写的内容）
 - compatible : Should at least contain "samsung,exynos4210-chipid".
 - reg: offset and length of the register set
 Example:
	chipid@10000000 {
		compatible = "samsung,exynos4210-chipid";
		reg = <0x10000000 0x100>;
	};
```
我们自己写的驱动说明文档自然没有适配到内核中去，所以只能期盼商家给你提供相应的说明文档

参考同类型单板的设备树文件
进入 <code>arch\arm\boot\dts </code>目录下里面是各种单板的设备树文件
比如
 am335x-boneblack.dts和am335x-boneblack-wireless.dts
发现多了wifi的信息,通过对比设备树文件，我们可以看出怎么写wifi设备节点,就知道如何添加设备节点.

网上搜索 
实在不行就研究驱动源码
一个好的驱动程序，它会尽量确定所用资源，只把不能确定的资源留给设备树，让设备树来指定。

