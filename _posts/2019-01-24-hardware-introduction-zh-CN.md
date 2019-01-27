# 简介

本文简要讲述嵌入式开发的一般性流程。

# SoC架构

一款SoC其内部除了包含CPU的部分，其中还有其他一些外设控制器、GPU或基带等电路集
成在SoC中。而这些外部电路都由CPU控制，以实现一些复杂控制逻辑。一般情况下，为了
降低CPU设计的复杂度，CPU被设计成只能访问内存，而控制这些外部电路就变成了一个棘
手的问题了。

## CPU内存映射

对于32位的CPU，其内存可寻址范围为0-4GB，但是对于嵌入式设备而言，4G内存又过于庞
大，浪费了大量内存寻址范围。如果将外部电路的控制逻辑映射为CPU读写内存，即解决
了外部电路控制的难题，又复用了过大的内存寻址范围。有了这种映射关系，控制某外部
电路只需要向其映射的内存某些地址中写入数据即可完成对该设备的控制。

由于不同的芯片厂对外设控制映射到内存地址不一，假设Qualcomm Snapdragon 845的USB
控制器控制逻辑被线性映射到地址0xFFFF0000，而MediaTek Helio P90的USB控制器控制
逻辑被线性映射到地址0xFFFFFF00，而且两个USB控制器的控制逻辑不一定是一致的。例
如对于Qualcomm平台，将USB速率提升到480Mbps是向控制器的第8个字节写入值
0x00000001；而对于MediaTek平台，将USB速率提升到480Mbps是向控制器的第16个字节写
入值0x00000002。很明显对于两个平台的USB控制器代码必须分开编译，或者使用条件编
译区分不同平台。实际上，对于相同平台，不同SoC型号，其外设控制器与内存地址的映
射关系都有所区分，这种平台或型号地址映射差异给代码可移植性带来了不小挑战。

不同平台USB控制器映射的地址不一样（示例代码）：
```
#if defined(CONFIG_VENDOR_QUALCOMM)
void *usb_controller_address = 0xffff0000;
#elif defined(CONFIG_VENDOR_MEDIATEK)
void *usb_controller_address = 0xffffff00;
#else
#error No vender specific!
#endif
```

不同平台USB控制器控制逻辑不一样（示例代码）：
```
void usb_set_speed_480mbps(void)
{
#if defined(CONFIG_VENDOR_QUALCOMM)
	*(int*)(usb_controller_address + 8) = 0x00000001;
#elif defined(CONFIG_VENDOR_MEDIATEK)
	*(int*)(usb_controller_address + 16) = 0x00000002;
#else
#error No vender specific!
#endif
}
```

# 选型

在一款电子产品开发验证阶段，用于程序员开发测试的板卡，叫做开发板。不同产品需求
所选择的开发板不尽相同，在产品定位阶段需要根据产品需求选择开发板或板级芯片，而
这种选择的过程一般称为“芯片选型”或“选型”。

假设某公司现在有一款家用级NAS产品及企业级NAS产品的需求，根据产品市场定位及其他
相关因素，在芯片选型阶段，家用级NAS产品选择了**[Marvell Armada 3720]**芯片（以
下简称3720），企业级NAS产品选择了**[Marvell Armada A388]**（以下简称A388）芯
片。

一旦芯片确认好以后，就需要利用搭载该芯片的开发板进行前期开发验证。基于以上芯
片，假设该公司选择了搭载[3720][Marvell Armada 3720]芯片的
**[GlobalScale ESPRESSObin]**开发板（以下简称ESPRESSObin）以及搭载
[A388][Marvell Armada A388]芯片的**[SolidRun ClearFog Pro]**开发板
（以下简称ClearFog）进行开发验证。

# 启动

先不考虑启动的事情，正常CPU在运行的时候都是根据编译好的二进制文件一条指令一条
顺序执行的（不考虑乱序执行、分支预测及超线程等因素）。CPU执行一条指令的流程简
而言可理解为分为三步（实际上不止这么简单）：取指（Fetch）-->执行（Excute）-->
回写（Write）。取指阶段CPU将从内存中读取出需要执行的指令并进行解析，最终将执行
流程告知执行器（ALU）；执行器执行完后，将结果写入内存中，完成一条执行的执行过
程。

而在指令执行之前，需要对指令进行定位：当前需要执行的指令位于内存的什么位置
（地址）。而完成这一项工作的通常是称为PC（Point Counter）或者
IP（Instructino Pointer）的寄存器。

## BOOT ROM

CPU在上电时，PC/IP寄存器一般会有一个默认值，通常是0x00000000。CPU上电执行的第
一条指令就会从这个默认值所指向的内存地址取出执行。根据前面章节的描述，CPU只能
访问到内存，板卡在上电时，内存中的数据都是随机数，甚至CPU的内存控制器都没有完
成内存初始化工作，CPU根本无法访问内存，更别说启动操作系统了。

此时，需要将存储在非易失性存储器中存储好的程序复制至内存中，然后将PC/IP寄存器
指向该内存区域的第一条指令，跳转至该程序中执行。

而完成这种复制数据至内存并跳转（修改PC/IP寄存器的值）至程序第一条指令所在内存
地址的工作由BOOT ROM完成。由于BOOT ROM完成的工作与硬件相关性比较大，SoC厂商一
般会内置好BOOT ROM程序并撰写好BOOT ROM文档供开发人员使用。即使SoC厂商未提供相
关BOOT ROM文档，板卡厂商也会有相应的文档对板卡BOOT ROM行为进行描述。普通开发人
员只需要寻找相应的文档，根据文档操作即可。

## Bootloader

一般情况下，BOOT ROM是写入SoC中的，考虑到SoC的生产成本，通常存储BOOT ROM的存储
容量不是很大，因此BOOT ROM的功能就相对比较有限，仅能完成对存储在非易失性存储器
中的二进制程序复制进内存的操作，而没有充足的空间去做其他外部电路的初始化工作。
因此，我们一般会将BOOT ROM处理不了的初始化工作整合起来，形成一段特殊的程序，专
门负责芯片及外部电路的初始化，以及内核或系统引导工作，这种特殊的程序称为
Bootloader。常见的开源Bootloader有：U-boot，Barebox等。

典型的芯片启动流程大致如下：

BOOT ROM ----> Bootloader ----> Kernel ----> Application

### ESPRESSObin BOOT ROM

一般情况下BOOT ROM是写入SoC中的，首先需要查阅SoC的文档，ESPRESSObin板载
[3720][Marvell Armada 3720]SoC，在Marvell官网上查找到的名为**《88F3710 and
88F3720 ARMADA® 3700 Family Single/Dual CPU System-on-Chip Hardware
Specifications》**的[芯片手册][Marvell Armada 3720 Manual]，在7.6节最后部分，
表37描述了3720的BOOT ROM行为描述，截取如下：

| GPIO1[7:5] |                      Boot Mode Description                    |
| :--------: | :------------------------------------------------------------ |
|    001b    | Serial NOR Flash Download Mode.                               |
|            |   Download boot loader or program code from Serial NOR flash  |
|            |   into CM3 or A53.                                            |
|    010b    | eMMC Download Mode.                                           |
|            |   Download boot loader or program code from eMMC flash into   |
|            |   CM3 or A53. Requires full initialization and command        |
|            |   sequence.                                                   |
|    011b    | eMMC Alternate Download Mode.                                 |
|            |   Download boot loader or program code from eMMC flash into   |
|            |   CM3 or A53. For cases when the eMMC Device is               |
|            |   pre-programmed to support eMMC Alternate Boot Mode.         |
|    100b    | SATA Download Mode.                                           |
|            |   Download boot loader or program code from AHCI interface    |
|            |   into CM3 or A53.                                            |
|    101b    | Serial NAND Flash Download Mode.                              |
|            |   Download boot loader or program code from SPI flash into    |
|            |   CM3 or A53.                                                 |
|    110b    | UART Mode.                                                    |
|            |   • Download boot loader or program code from UART interface  |
|            |     (WTPTP) into CM3 or A53.                                  |
|            |   • Simple UART monitor code is executed within the Boot ROM, |
|            |     allowing internal registers for read/write through        |
|            |     external UART terminal.                                   |
|    111b    | Reserved                                                      |

由上述表格可知，3720中的BOOT ROM是通过检测GPIO1控制器上，5、6、7引脚状态来加载
需要复制进内存的二进制程序。而三个GPIO引脚引出到ESPRESSObin开发板上什么位置，
就需要查阅板级的PCB设计图纸，确定上述三个GPIO具体引出到什么位置。但是对于这种
比较常用的BOOT ROM，一般情况下会写入ESPRESSObin开发板的板级开发文档中。

开发板比较少有规范化的开发手册，而是将需要注意的事项公布在其网站上供开发者查
阅，通过访问ESPRESSObin官方网站，有一篇名为“Ports and Interfaces”的
[页面][GlobalScale ESPRESSObin Manual]，页面上虽然没有描述板级各个引脚接驳于
CPU的哪个引脚，但是将SoC级别的BOOT ROM行为“抽象”为下表（仅适用用于ESPRESSObin
Revision 5 and earlier）：

| ESPRESSObin boot mode                  |    J10    |    J3     |    J11    |
| :------------------------------------- | :-------: | :-------: | :-------: |
| Serial NOR Flash Download Mode         |    1-2    |    2-3    |    2-3    |
| eMMC Download Mode                     |    2-3    |    1-2    |    2-3    |
| eMMC Alternate Download Mode           |    1-2    |    1-2    |    2-3    |
| SATA Download Mode                     |    2-3    |    2-3    |    1-2    |
| Serial NAND Flash Download Mode        |    1-2    |    2-3    |    1-2    |
| UART Mode                              |    2-3    |    1-2    |    1-2    |

阅读ESPRESSObin文档，其中板载非易失性存储器的接口有：

1. SPI NOR Flash（贴片式，自带4MB）
2. eMMC（贴片式，空贴）
3. SD（插入式）
4. SATA接口

### ClearFog BOOT ROM

ClearFog板载的A388在Marvell官网上查询到一份名为**《88F6810, 88F6820, and
88F6828 ARMADA® 380, 385, and 388 High-Performance Single/Dual CPU System on
Chip Hardware Specifications – Unrestricted》**的
[芯片手册][Marvell Armada A388 Manual]，在7.5节描述了A388芯片的BOOT ROM的启动
行为。由于该节描述过于复杂，在这里也不一一列举出来了。

对于这种过于复杂的BOOT ROM行为，板级文档一般会隐藏BOOT ROM复杂特性，转化为在开
发板上引脚插跳线帽或拨动开关来达到调整BOOT ROM复制二进制程序到内存的行为。

通过查阅ClearFog的[用户手册][SolidRun ClearFog Pro Manual]，在5.3节描述了板级
BOOT ROM行为，截取如下：

|                     | Switch 1 | Switch 2 | Switch 3 | Switch 4 | Switch 5 |
| :-----------------: | :------: | :------: | :------: | :------: | :------: |
|         SPI         |    Off   |    Off   |    Off   |    On    |    Off   |
|       SD/eMMC       |    Off   |    Off   |    On    |    On    |    On    |
|       M.2 SSD       |    On    |    On    |    On    |    Off   |    Off   |
|         UART        |    On    |    On    |    On    |    On    |    Off   |

由上表可知，ClearFog的BOOT ROM行为由开发板上5个独立的开关控制，通过调整这5个开
关的闭合，可以改变BOOR ROM的行为。

## 非易失性存储器

BOOT ROM的工作主要是将非易失性存储器中的二进制程序内容（通常是Bootloader）复制
进内存，并修改PC/IP的内容指向该段程序的第一条指令，完成对外部程序的引导功能。
本节简单描述一下常用于存储程序的一些特性及工作原理。

由前面章节描述，本节选取ESPRESSObin和ClearFog都支持的非易失性存储器进行讲解，
它们是：SPI，eMMC，UART。

### SPI

SPI全称Serial Peripheral Interface，是CPU跟外部设备连接时的一种通信协议。SPI作
为一种硬件传输协议，与CPU连接的设备不一定是存储设备，也可以是GPS定位装置、磁场
感应器等。

考虑到ESPRESSObin及ClearFog板载的均为SPI NOR Flash，本节就SPI NOR Flash相关特
性进行简单描述，对于SPI NAND Flash，不在本节讨论范围内。

对于通过SPI接口连接到CPU上的NOR Flash，其访问方法及通信协议已经被JEDEC组织标准
化了。直至目前位置，最新的SPI NOR Flash的标准文档为JESD216D；在Linux内核MTD子
系统下有控制SPI NOR Flash的源代码。本文不再详细描述SPI NOR Flash的工作标准及原
理，直接给出结论：**通过JESD216D标准，访问SPI NOR Flash可以像访问内存一样进行
寻址操作，所有数据存储都是线性存储在Flash中的。**简单而言就是，代码实现
JESD216D标准，将存储在SPI NOR Flash中的数据读取出来，并写入内存指定的区域。

ESPRESSObin及ClearFog的BOOT ROM行为，有从SPI启动的描述，若将部分跳线帽或拨动开
关设置成从SPI启动，在上电开机后，BOOT ROM将首先初始化内存（如果必要），随后初
始化SoC内部的SPI控制器，根据JESD216D的标准，将存储在SPI NOR Flash内的数据**全
部**复制到内存预先指定的位置（通常是0x00000000），随后将PC/IP的值修改为该预定
内存地址，完成引导流程。

伪代码如下：
```
#define PRE_DEFINED_ADDRESS 0x00000000
void boot_from_spi_nor_flash(void)
{
	void *address = PRE_DEFINED_ADDRESS;

	memory_controller_init();
	spi_controller_init();

	copy_all_data_from_spi_nor_flash_to(address);

	/* Jump to pre-defined address */
	(void(*)(void))(address);
}
```

### eMMC

eMMC为SD标准的一个分支，随着移动互联网的发展，各种手机、平板电脑内部基本上都使
用eMMC作为其存储设备。关于eMMC细节，可访问[这里][eMMC Documentation]查阅，本文
不再详细描述，直接给出结论：**eMMC设备内部有分区的概念，一般都包括2个boot分区
（出厂自带，boot1及boot2，大小通常在4MB左右），0-1个RPMB分区（出厂没有，用户新
建定义，大小可配置，通常不超过32MB），0-4个通用分区（通常是0个，也就是没有），
用户分区（主要存储区域）。**

|                 Description                  |         eMMC Partition      |
| :------------------------------------------: | :-------------------------: |
|              Boot Area Partition             |     Boot Area Partition 1   |
|                                              |     Boot Area Partition 2   |
|                 RPMB Partition               |         RPMB Partition      |
|            General Purpose Partition         | General Purpose Partition 1 |
|                                              | General Purpose Partition 2 |
|                                              | General Purpose Partition 3 |
|                                              | General Purpose Partition 4 |
|                User Data Area                |         User Data Area      |

开发板如果将BOOT ROM的引导行为配置为从eMMC启动，那么BOOT ROM首先初始化内存及内
存控制器（如果必要），随后初始化SoC内部的eMMC控制器，将存储在eMMC BOOT1分区或
者eMMC BOOT2分区内的数据**全部**复制到内存预先指定的位置（通常是0x00000000），
随后将PC/IP的值修改为该预定内存地址，完成引导流程。

伪代码如下：
```
#define PRE_DEFINED_ADDRESS 0x00000000
void boot_from_emmc(void)
{
	int partition = -1;
	void *address = PRE_DEFINED_ADDRESS;

	memory_controller_init();
	emmc_controller_init();

	/* Check which partition will be used as boot partition */
	partition = check_emmc_boot_partition_status();

	if ( partition < 0 )
		error("Can not check the boot partition of eMMC!");

	copy_all_data_from_emmc_boot_to(partition, address);

	/* Jump to pre-defined address */
	(void(*)(void))(address);
}
```

### UART

对于没有任何二进制程序存储在SPI NOR Flash或eMMC等非易失性存储器的开发板，称作
裸板。对于裸板而言，只能从开发板的外部将二进制程序（通常是Bootloader）导入到开
发板的内存中去，由BOOT ROM接收并处理接收到的二进制程序，并完成引导工作。

UART全称Universal Asynchronous Receiver/Transmitter，是一种CPU与外部电路传输数
据的硬件传输协议。这种传输协议通常被开发人员用来收发字符数据：将开发板通过UART
接口连接至电脑端，并建立到开发板的UART连接；将开发板一个UART控制器配置为输入输
出终端，所有标准输入输出即被重定向到该UART控制器，最终由电脑上的UART连接客户端
进行交互。

虽然UART不属于非易失性存储设备的范畴，但为了描述BOOT ROM的行为，在此将UART归类
为外部存储设备。UART常常用来收发字符数据，但并不是只能收发字符数据，也可以收发
二进制数据。

如果将开发板配置成从UART启动，在电脑上通过UART向开发板发送特定的前导数据，可以
使正在监听UART数据的BOOT ROM将后面所有的数据当成二进制程序，并写入内存中，当完
成传输后，经过BOOT ROM校验，引导从UART传输的二进制程序。

不同的SoC，其从UART启动所要求的传输方式都不尽相同，可能需要先传输一个字符告知
BOOT ROM进入程序下载阶段，随后再传输镜像大小，最后再传输最终的二进制程序文件。

伪代码如下：
```
#define PRE_CHAR 129
#define PRE_DEFINED_ADDRESS 0x00000000
void boot_from_uart(void)
{
	uint32_t image_size = 0;
	char pre_char = 0;

	void *address = PRE_DEFINED_ADDRESS;

	memory_controller_init();
	uart_controller_init();

	/**
	 * Wait for some symbols which indicate we need going to download
	 * image from UART
	 */
	while ( uart_read_char(&pre_char) ) {
		if ( PRE_CHAR == pre_char ) {
			/**
			 * Followed by a 32bit image size
			 */
			uart_read_uint32(&image_size);
			goto boot;
		} else
			continue;
	}

boot:
	/* Read bootloader image from uart */
	uart_read(address, image_size);

	/* Boot if we are good */
	if ( image_check_ok() ) {
		/* Jump to pre-defined address */
		(void(*)(void))(address);
	}

	reboot();
}
```

由于通过UART下载二进制程序到内存，不同开发板操作方法都不一样，具体需要查阅开发
板的文档。上述两款开发板的UART引导方法如下：

ESPRESSObin的UART引导方法请访问[这里][GlobalScale ESPRESSObin UART Boot]，
ClearFog的UART引导访问请访问[这里][SolidRun ClearFog Pro UART Boot]。

通过UART下载的程序是存在于内存中的，一旦断电将会丢失全部数据，但至少有程序在内
存，需要利用这一段小程序对开发板上的非易失性存储设备写入程序，以方便后续开发，
这种在板级内部对板级存储器写入数据也跟平台相关性比较强，具体也是需要查阅开发板
文档去操作。不过对于Marvell平台的U-boot，在编译时可选择bubt命令，通过UART下载
的U-boot程序可以使用bubt命令进行板级存储器烧录U-boot镜像。这一块不在本章讨论范
围内，就不详细描述了。

# 小结



******

Copyright (C) 2019 Ding Tao All Right Reserved.

[Marvell Armada 3720]: https://www.marvell.com/embedded-processors/armada/armada-3700/
[Marvell Armada 3720 Manual]: https://www.marvell.com/documents/qc8hltbjybmpjhx36ckw/ "88F3710 and 88F3720 ARMADA® 3700 Family Single/Dual CPU System-on-Chip Hardware Specifications"

[Marvell Armada A388]: https://www.marvell.com/embedded-processors/armada/armada-38x/
[Marvell Armada A388 Manual]: https://www.marvell.com/docs/embedded-processors/assets/marvell-embedded-processors-armada-38x-hardware-specifications-2017-03.pdf "88F6810, 88F6820, and 88F6828 ARMADA® 380, 385, and 388 High-Performance Single/Dual CPU System on Chip Hardware Specifications – Unrestricted"

[GlobalScale ESPRESSObin]: http://espressobin.net/
[GlobalScale ESPRESSObin Manual]: http://wiki.espressobin.net/tiki-index.php?page=Ports+and+Interfaces "Boot selection"
[GlobalScale ESPRESSObin UART Boot]: http://wiki.espressobin.net/tiki-index.php?page=Bootloader+recovery+via+UART "ESPRESSObin Boot from UART"

[SolidRun ClearFog Pro]: https://www.solid-run.com/marvell-armada-family/clearfog/
[SolidRun ClearFog Pro Manual]: https://developer.solid-run.com/download/clearfog-pro-user-manual/ "Boot selection"
[SolidRun ClearFog Pro UART Boot]: https://developer.solid-run.com/knowledge-base/a388-u-boot/#booting-from-uart "ClearFog Boot from UART"

[eMMC Documentation]: https://linux.codingbelief.com/zh/storage/flash_memory/emmc/ "eMMC Documentation"

