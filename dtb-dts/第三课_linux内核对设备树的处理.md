* 转载请注明文章地址 http://wiki.100ask.org/Linux_devicetree

这一课是设备树中最重要的一课。
前面我们从内核文档了解到，对于设备树，它里面描述的信息可以分为这三部分：
Linux uses DT data for three major purposes:
1) platform identification,
2) runtime configuration, and
3) device population.
事实上，内核对设备树的处理，也会分为与其对应的三部分：
对于<code>platform identification</code>，将在<code>第02节_对设备树中平台信息的处理(选择machine_desc)</code>进行分析；
对于<code>runtime configuration</code>，将在<code>第03节_对设备树中运行时配置信息的处理</code>进行分析；
对于<code>device population</code>，将在<code>第04-06节</code>进行分析；

# 第01节_从源头分析_内核head.S对dtb的简单处理
现在我们开始第一节，我们要从源头分析，uboot将一些参数，设备树文件传给内核，那么内核如何处理这些设备树文件呢？
我们需要从内核的第一个执行文件<code>head.S</code>开始分析。

## r0,r1,r2三个寄存器的设置
bootloader启动内核时,会设置r0,r1,r2三个寄存器,

r0一般设置为0；
r1一般设置为machine id (在使用设备树时该参数没有被使用)；
r2一般设置ATAGS或DTB的开始地址；


这里的machine id，是让内核知道是哪个CPU，从而调用对应的初始化函数。
以前没有使用设备树时，需要bootloader传一个machine id给内核，现在使用设备树的话，这个参数就不需要设置了。
r2要么是以前的ATAGS开始地址，要么是现在使用设备树后的DTB文件开始地址。
对于ATAGS传参方法, 可以参考我们的"毕业班视频-自己写bootloader"
   从www.100ask.net下载页面打开百度网盘,
   打开如下目录:
        100ask分享的所有文件
            006_u-boot_内核_根文件系统(新1期_2期间的衔接)
                视频
                    第002课_从0写bootloader_更深刻理解bootloader

##  head.S的内容 
内核<code>head.S</code>所做工作如下：
a. __lookup_processor_type : 使用汇编指令读取CPU ID, 根据该ID找到对应的proc_info_list结构体(里面含有这类CPU的初始化函数、信息)
b. __vet_atags             : 判断是否存在可用的ATAGS或DTB
c. __create_page_tables    : 创建页表, 即创建虚拟地址和物理地址的映射关系
d. __enable_mmu            : 使能MMU, 以后就要使用虚拟地址了
e. __mmap_switched         : 上述函数里将会调用__mmap_switched
f. 把bootloader传入的r2参数, 保存到变量__atags_pointer中
g. 调用C函数start_kernel

##最终效果 
head.S和head-common.S最终效果：
把bootloader传来的r1值, 赋给了C变量: __machine_arch_type
把bootloader传来的r2值,

# 第02节_对设备树中平台信息的处理(选择machine_desc)
这节讲解内核对设备树中平台设备信息是如何处理的。
## 内核是如何选择对应的machine_desc? 
前面讲解到，一个编译成uImage的内核镜像文件，可以支持多个单板，这里假设支持smdk2410、smdk2440、jz2440(其中smdk2410、smdk2440是厂家的公板，国内的厂家参考公板设计出了自己的板子，比如jz2440)。

这些板子的配置稍有不同，需要做一些单独的初始化，在内核里面，对于这些单板，都构造了一个machine_desc结构体，里面有.init和.nr。
对于JZ2440，它源自smdk2440，内核没有它的单独文件，它使用smdk2440的相关文件，代码。
在上一节视频里面我们说过，以前uboot使用ATAGS给内核传参数时，它会传入一个机器ID，内核会使用这个机器ID找到最合适的machine_desc。即机器ID与machine_desc里面的.nr比较，相等就表示找到了对应的machine_desc。
当我们的uboot不使用ATAGS传参数，而使用DTB文件时，那么这时内核是如何选择对应的machine_desc呢？
在设备树文件的根节点里，有如下两行：
```c
	model = "SMDK24440";
	compatible = "samsung,smdk2440"，"samsung,smdk24140"，"samsung,smdk24xx";
```

这里的<code>compatible</code>属性声明想要什么<code>machine_desc</code>，属性值可以是一系列字符串，依次与<code>machine_desc</code>匹配。
内核最好支持<code>samsung,smdk2440</code>，如果不支持，再尝试是否支持<code>samsung,smdk24140</code>，再不支持，最后尝试<code>samsung,smdk24xx</code


* 总结如下：
a. 设备树根节点的compatible属性列出了一系列的字符串,
&nbsp;&nbsp;&nbsp;&nbsp;表示它兼容的单板名，从"最兼容"到次之；

b. 内核中有多个machine_desc,
&nbsp;&nbsp;&nbsp;&nbsp;其中有dt_compat成员, 它指向一个字符串数组, 里面表示该machine_desc支持哪些单板；

c. 使用compatile属性的值, 跟'''每一个machine_desc.dt_compat'''比较,
&nbsp;&nbsp;&nbsp;&nbsp;成绩为"吻合的compatile属性值的位置",
&nbsp;&nbsp;&nbsp;&nbsp;成绩越低越匹配, 对应的machine_desc即被选中

# start_kernel的调用过程
上节视频里，head.S会把DTB的位置保存在变量<code>__atags_pointer</code>里，最后调用<code>start_kernel</code>。
<code>start_kernel</code>的调用过程如下：

```c
start_kernel // init/main.c
    setup_arch(&command_line);  // arch/arm/kernel/setup.c
        mdesc = setup_machine_fdt(__atags_pointer);  // arch/arm/kernel/devtree.c
                    early_init_dt_verify(phys_to_virt(dt_phys)  // 判断是否有效的dtb, drivers/of/ftd.c
                                    initial_boot_params = params;
                    mdesc = of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);  // 找到最匹配的machine_desc, drivers/of/ftd.c
                                    while ((data = get_next_compat(&compat))) {
                                        score = of_flat_dt_match(dt_root, compat);
                                        if (score > 0 && score < best_score) {
                                            best_data = data;
                                            best_score = score;
                                        }
                                    }
                    
        machine_desc = mdesc;
```

# 第03节_对设备树中运行时配置信息的处理
**设备树只是起一个信息传递的作用，对这些信息配置的处理，也比较简单，即从设备树的DTB文件中，把这些设备信息提取出来赋给内核中的某个变量即可。**

函数调用过程如下:
```c
start_kernel // init/main.c
    setup_arch(&command_line);  // arch/arm/kernel/setup.c
        mdesc = setup_machine_fdt(__atags_pointer);  // arch/arm/kernel/devtree.c
                    early_init_dt_scan_nodes();      // drivers/of/ftd.c
                        /* Retrieve various information from the /chosen node */
                        of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);

                        /* Initialize {size,address}-cells info */
                        of_scan_flat_dt(early_init_dt_scan_root, NULL);

                        /* Setup memory, calling early_init_dt_add_memory_arch */
                        of_scan_flat_dt(early_init_dt_scan_memory, NULL);
```


里面主要对三种类型的信息进行处理，分别是：/chosen节点中<code> bootarg</code>s属性，根节点的<code> #address-cells</code> 和<code> #size-cells</code>属性，/memory中的<code> reg</code>属性。

1./chosen节点中bootargs属性就是内核启动的命令行参数，它里面可以指定根文件系统在哪里，第一个运行的应用程序是哪一个，指定内核的打印信息从哪个设备里打印出来。

2./memory中的reg属性指定了不同板子内存的大小和起始地址。

3.根节点的#address-cells和#size-cells属性指定属性参数的位数，比如指定前面memory中的reg属性的地址是32位还是64位，大小是用一个32位表示，还是两个32位表示。


* 总结：
a. /chosen节点中bootargs属性的值, 存入全局变量： boot_command_line
b. 确定根节点的这2个属性的值: #address-cells, #size-cells
&nbsp;&nbsp;&nbsp;&nbsp; 存入全局变量: dt_root_addr_cells, dt_root_size_cells
c. 解析/memory中的reg属性, 提取出"base, size", 最终调用memblock_add(base, size);

# 第04节_dtb转换为device_node(unflatten)

在讲解之前，我们先想一个问题，我们的uboot把设备树DTB文件随便放到内存的某一个地方就可以使用，为什么内核运行中，他不会去覆盖DTB所占用的那块内存呢？
在前面我们讲解设备树格式时，我们知道，在设备树文件中，可以使用<code>/memreserve/</code>指定一块内存，这块内存就是保留的内存，内核不会占用它。即使你没有指定这块内存，当我们内核启动时，他也会把设备树所占用的区域保留下来。
如下就是函数调用过程:

```c
start_kernel // init/main.c
    setup_arch(&command_line);  // arch/arm/kernel/setup.c
        arm_memblock_init(mdesc);   // arch/arm/kernel/setup.c
            early_init_fdt_reserve_self();
                    /* Reserve the dtb region */
                    // 把DTB所占区域保留下来, 即调用: memblock_reserve
                    early_init_dt_reserve_memory_arch(__pa(initial_boot_params),
                                      fdt_totalsize(initial_boot_params),
                                      0);           
            early_init_fdt_scan_reserved_mem();  // 根据dtb中的memreserve信息, 调用memblock_reserve
            
        unflatten_device_tree();    // arch/arm/kernel/setup.c
            __unflatten_device_tree(initial_boot_params, NULL, &of_root,
                        early_init_dt_alloc_memory_arch, false);            // drivers/of/fdt.c
                
                /* First pass, scan for size */
                size = unflatten_dt_nodes(blob, NULL, dad, NULL);
                
                /* Allocate memory for the expanded device tree */
                mem = dt_alloc(size + 4, __alignof__(struct device_node));
                
                /* Second pass, do actual unflattening */
                unflatten_dt_nodes(blob, mem, dad, mynodes);
                    populate_node
                        np = unflatten_dt_alloc(mem, sizeof(struct device_node) + allocl,
                                    __alignof__(struct device_node));
                        
                        np->full_name = fn = ((char *)np) + sizeof(*np);
                        
                        populate_properties
                                pp = unflatten_dt_alloc(mem, sizeof(struct property),
                                            __alignof__(struct property));
                            
                                pp->name   = (char *)pname;
                                pp->length = sz;
                                pp->value  = (__be32 *)val;
```

可以看到，先把dtb中的memreserve信息告诉内核，把这块内存区域保留下来，不占用它。

然后将扁平结构的设备树提取出来，构造成一个树，这里涉及两个结构体：<code>device_node</code>结构体和<code>property</code>结构体。弄清楚这两个结构体就大概明白这节视频的主要内容了。

在dts文件里，每个大括号<code>{ }</code>代表一个节点，比如根节点里有个大括号，对应一个device_node结构体；memory也有一个大括号，也对应一个device_node结构体。
节点里面有各种属性，也可能里面还有子节点，所以它们还有一些父子关系。
根节点下的memory、chosen、led等节点是并列关系，兄弟关系。
对于父子关系、兄弟关系，在device_node结构体里面肯定有成员来描述这些关系。

打开<code>include/linux/Of.h</code>可以看到device_node结构体的定义如下：
<syntaxhighlight lang="c" >
        struct device_node {
            const char *name;  // 来自节点中的name属性, 如果没有该属性, 则设为"NULL"
            const char *type;  // 来自节点中的device_type属性, 如果没有该属性, 则设为"NULL"
            phandle phandle;
            const char *full_name;  // 节点的名字, node-name[@unit-address]
            struct fwnode_handle fwnode;

            struct  property *properties;  // 节点的属性
            struct  property *deadprops;    /* removed properties */
            struct  device_node *parent;   // 节点的父亲
            struct  device_node *child;    // 节点的孩子(子节点)
            struct  device_node *sibling;  // 节点的兄弟(同级节点)
        #if defined(CONFIG_OF_KOBJ)
            struct  kobject kobj;
        #endif
            unsigned long _flags;
            void    *data;
        #if defined(CONFIG_SPARC)
            const char *path_component_name;
            unsigned int unique_id;
            struct of_irq_controller *irq_trans;
        #endif
        };
```
device_node结构体表示一个节点，property结构体表示节点的具体属性。


properties结构体的定义如下：
```c
        struct property {
            char    *name;    // 属性名字, 指向dtb文件中的字符串
            int length;       // 属性值的长度
            void    *value;   // 属性值, 指向dtb文件中value所在位置, 数据仍以big endian存储
            struct property *next;
        #if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
            unsigned long _flags;
        #endif
        #if defined(CONFIG_OF_PROMTREE)
            unsigned int unique_id;
        #endif
        #if defined(CONFIG_OF_KOBJ)
            struct bin_attribute attr;
        #endif
        };
```

两个结构体与dts内容的对于关系如下：

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter3_4_001.jpg)

具体的代码分析，参考视频内容。

# 第05节_device_node转换为platform_device
内核如何把device_node转换成platfrom_device 
## 两个问题
a.那些device_node可以转换为platform_device 
```c
/ {
	model = "SMDK24440";
	compatible = "samsung,smdk2440";

	#address-cells = <1>;
	#size-cells = <1>;
	//内存设备不会	
	memory@30000000 {
		device_type = "memory";
		reg =  <0x30000000 0x4000000>;
	};
/*
	cpus {
		cpu {
			compatible = "arm,arm926ej-s";
		};
	};
*/	//只是设置一些启动信息
	chosen {
		bootargs = "noinitrd root=/dev/mtdblock4 rw init=/linuxrc console=ttySAC0,115200";
	};

/*只有这个led设备才对转换成platfrom_device */	
	led {
		compatible = "jz2440_led";
		reg = <S3C2410_GPF(5) 1>;
	};
/************************************/
};
```


* a. 内核函数of_platform_default_populate_init, 遍历device_node树, 生成platform_device
* b. 并非所有的device_node都会转换为platform_device只有以下的device_node会转换:
* * b.1 该节点必须含有compatible属性
* * b.2 根节点的子节点(节点必须含有compatible属性)
* * b.3 含有特殊compatible属性的节点的子节点(子节点必须含有compatible属性):
    这些特殊的compatilbe属性为:<code> "simple-bus","simple-mfd","isa","arm,amba-bus "</code>

根节点是例外的，生成platfrom_device时，即使有compatible属性也不会处理

举例
cpu可以访问很多外设，spi控制器 I2c控制器，led

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter3_5_001.png)

如何在设备树中描述这些硬件？
b.4 示例: 比如以下的节点, 
/mytest会被转换为platform_device, 
因为它兼容"simple-bus", 它的子节点/mytest/mytest@0 也会被转换为platform_device

 /i2c节点一般表示i2c控制器, 它会被转换为platform_device, 在内核中有对应的platform_driver;
 
/i2c/at24c02节点不会被转换为platform_device, 它被如何处理完全由父节点的platform_driver决定, 一般是被创建为一个i2c_client。

类似的也有/spi节点, 它一般也是用来表示SPI控制器, 它会被转换为platform_device, 在内核中有对应的platform_driver;

/spi/flash@0节点不会被转换为platform_device, 它被如何处理完全由父节点的platform_driver决定, 一般是被创建为一个spi_device。
   
 ```c  
    / {
          mytest {
              compatile = "mytest", "simple-bus";
              mytest@0 {
                    compatile = "mytest_0";
              };
          };
          
          i2c {
              compatile = "samsung,i2c";
              at24c02 {
                    compatile = "at24c02";                      
              };
          };

          spi {
              compatile = "samsung,spi";              
              flash@0 {
                    compatible = "winbond,w25q32dw";
                    spi-max-frequency = <25000000>;
                    reg = <0>;
                  };
          };
      };
```
b.怎么转换
函数调用过程: 
a. 入口函数 of_platform_default_populate_init (drivers/of/platform.c) 被调用到过程:

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter3_5_003.png)

里面有段属性，编译内核段属性的变量会被集中放在一起
 vim arch/arm/kernel/vmlinux.lds
```c
start_kernel     // init/main.c
    rest_init();
        pid = kernel_thread(kernel_init, NULL, CLONE_FS);
                    kernel_init
                        kernel_init_freeable();
                            do_basic_setup();
                                do_initcalls();
                                    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
                                        do_initcall_level(level);  // 比如 do_initcall_level(3)
                                                                               for (fn = initcall_levels[3]; fn < initcall_levels[3+1]; fn++)
                                                                                    do_one_initcall(initcall_from_entry(fn));  // 就是调用"arch_initcall_sync(fn)"中定义的fn函数
```
b. of_platform_default_populate_init  (drivers/of/platform.c) 生成platform_device的过程:
遍历device树
图3
```c
of_platform_default_populate_init
    of_platform_default_populate(NULL, NULL, NULL);
        of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL)
            for_each_child_of_node(root, child) {
                rc = of_platform_bus_create(child, matches, lookup, parent, true);  // 调用过程看下面
                            dev = of_device_alloc(np, bus_id, parent);   // 根据device_node节点的属性设置platform_device的resource
                if (rc) {
                    of_node_put(child);
                    break;
                }
            }
 
```

c. of_platform_bus_create(bus, matches, ...)的调用过程(处理bus节点生成platform_devie, 并决定是否处理它的子节点):
```c
        dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);  // 生成bus节点的platform_device结构体
        if (!dev || !of_match_node(matches, bus))  // 如果bus节点的compatile属性不吻合matches成表, 就不处理它的子节点
            return 0;

        for_each_child_of_node(bus, child) {    // 取出每一个子节点
            pr_debug("   create child: %pOF\n", child);
            rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);   // 处理它的子节点, of_platform_bus_create是一个递归调用
            if (rc) {
                of_node_put(child);
                break;
            }
        }
```


        
d. I2C总线节点的处理过程:

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter3_5_004.gif)

   /i2c节点一般表示i2c控制器, 它会被转换为platform_device, 在内核中有对应的platform_driver;
   platform_driver的probe函数中会调用i2c_add_numbered_adapter:
 ```c 
   i2c_add_numbered_adapter   // drivers/i2c/i2c-core-base.c
        __i2c_add_numbered_adapter
            i2c_register_adapter
                of_i2c_register_devices(adap);   // drivers/i2c/i2c-core-of.c
                    for_each_available_child_of_node(bus, node) {
                        client = of_i2c_register_device(adap, node);
                                        client = i2c_new_device(adap, &info);   // 设备树中的i2c子节点被转换为i2c_clien
```

# 第06节_platform_device跟platform_driver的匹配

![](http://photos.100ask.net/ldd/ldd_devicetree_chapter3_6_001.png)

 drivers/base/platform.c
a. 注册 platform_driver 的过程:

```c 
platform_driver_register
    __platform_driver_register
        drv->driver.probe = platform_drv_probe;
        driver_register
            bus_add_driver
                klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);    // 把 platform_driver 放入 platform_bus_type 的driver链表中
                driver_attach
                    bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);  // 对于plarform_bus_type下的每一个设备, 调用__driver_attach
                        __driver_attach
                            ret = driver_match_device(drv, dev);  // 判断dev和drv是否匹配成功
                                        return drv->bus->match ? drv->bus->match(dev, drv) : 1;  // 调用 platform_bus_type.match
                            driver_probe_device(drv, dev);
                                        really_probe
                                            drv->probe  // platform_drv_probe
                                                platform_drv_probe
                                                    struct platform_driver *drv = to_platform_driver(_dev->driver);
                                                    drv->probe
```
b. 注册 platform_device 的过程:
```c 
platform_device_register
    platform_device_add
        device_add
            bus_add_device
                klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices); // 把 platform_device 放入 platform_bus_type的device链表中
            bus_probe_device(dev);
                device_initial_probe
                    __device_attach
                        ret = bus_for_each_drv(dev->bus, NULL, &data, __device_attach_driver); // // 对于plarform_bus_type下的每一个driver, 调用 __device_attach_driver
                                    __device_attach_driver
                                        ret = driver_match_device(drv, dev);
                                                    return drv->bus->match ? drv->bus->match(dev, drv) : 1;  // 调用platform_bus_type.match
                                        driver_probe_device
```
匹配函数是platform_bus_type.match, 即platform_match,
匹配过程按优先顺序罗列如下:
* 比较 platform_dev.driver_override 和 platform_driver.drv->name
* 比较 platform_dev.dev.of_node的compatible属性 和 platform_driver.drv->of_match_table
* 比较 platform_dev.name 和 platform_driver.id_table
* 比较 platform_dev.name 和 platform_driver.drv->name
有一个成功, 即匹配成功

# 第07节_内核中设备树的操作函数

include/linux/目录下有很多of开头的头文件:
<code>dtb -> device_node -> platform_device</code>
a. 处理DTB
 of_fdt.h           // dtb文件的相关操作函数, 我们一般用不到, 因为dtb文件在内核中已经被转换为device_node树(它更易于使用)
b. 处理device_node
	
	 of.h               // 提供设备树的一般处理函数, 比如 of_property_read_u32(读取某个属性的u32值), *of_get_child_count(获取某个device_node的子节点数)
	 of_address.h       // 地址相关的函数, 比如 of_get_address(获得reg属性中的addr, size值)
	 of_match_device(从matches数组中取出与当前设备最匹配的一项)
	 of_dma.h           // 设备树中DMA相关属性的函数
	 of_gpio.h          // GPIO相关的函数
	 of_graph.h         // GPU相关驱动中用到的函数, 从设备树中获得GPU信息
	 of_iommu.h         // 很少用到
	 of_irq.h           // 中断相关的函数
	 of_mdio.h          // MDIO (Ethernet PHY) API
	 of_net.h           // OF helpers for network devices. 
	 of_pci.h           // PCI相关函数
	 of_pdt.h           // 很少用到
	 of_reserved_mem.h  // reserved_mem的相关函数

以中断相关的作为例子
一个设备可以发出中断，必须包含中断号和中断触发方式

官方设备树规格书里面的设备示例
```c 
soc {
#address-cells = <1>;
#size-cells = <1>;
serial {
compatible = "ns16550";
reg = <0x4600 0x100>;
clock-frequency = <0>;
interrupts = <0xA 0x8>;
interrupt-parent = <&ipic>;
};
};
```
里面的属性里面有中断值

通过
```c 
int of_irq_parse_one(struct device_node *device, int index,
			  struct of_phandle_args *out_irq);
```
解析某一对值，或者我们可以解析原始数据
```c 
int of_irq_parse_raw(const __be32 *addr, struct of_phandle_args *out_irq);
```
addr就指向了某一对值，把里面的中断号中断触发方式解析出来，保存在of_phandle_args结构体中

c. 处理 platform_device
 of_platform.h      // 把device_node转换为platform_device时用到的函数, 
```c 
/* Platform drivers register/unregister */
extern struct platform_device *of_device_alloc(struct device_node *np,
					 const char *bus_id,
					 struct device *parent);
```
文件涉及的函数在 device_node -> platform_device 中大量使用

	 // 比如of_device_alloc(根据device_node分配设置platform_device), 
	 //     of_find_device_by_node (根据device_node查找到platform_device),
	 //     of_platform_bus_probe (处理device_node及它的子节点)
	 of_device.h        // 设备相关的函数, 比如 of_match_device
	可以通过of_match_device找出哪一项最匹配，

of文件分为三类
* 处理DTB
* 处理device_node
* 处理 platform_device 设备相关信息

# 第08节_在根文件系统中查看设备树(有助于调试)
a. <code>/sys/firmware/fdt</code>        // 查看原始dtb文件
 hexdump -C /sys/firmware/fdt
b.<code> /sys/firmware/devicetree</code> // 以目录结构程现的dtb文件, 根节点对应base目录, 每一个节点对应一个目录, 每一个属性对应一个文件
比如查看 #address-cells 的16进制
 hexdump -C "#address-cells"
查看compatible

	cat compatible
 
如果你在设备树设备节点中设置一个错误的中断属性，那么就导致led对应的平台设备节点没办法创建
c.<code> /sys/devices/platform</code>    // 系统中所有的platform_device, 有来自设备树的, 也有来有.c文件中注册的<br>
对于来自设备树的platform_device,   可以进入<code> /sys/devices/platform/<设备名>/of_node <code>查看它的设备树属性<br>
d.<code>  /proc/device-tree</code> 是链接文件, 指向<code> /sys/firmware/devicetree/base</code>
