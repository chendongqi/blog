---
layout:      post
title:      "系统升级系列六"
subtitle:   "Recovery之启动模式"
navcolor:   "invert"
date:       2018-12-26
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - Recovery
    - Boot Mode
    - 启动模式
---

### 1 前言

在系列五中讲到了请求升级，通过framework的RecoverySystem.bootCommand将升级请求写入到了/cache/recovery/command中格式为```--update_package=<filename>\n--locale=language```，而后面的升级任务就交给了recovery来真正执行了，那么这一篇的重点就是来介绍下，为什么写了这个command，重启之后就能决定系统进入到Recovery中并开始安装升级包的。  

### 2 uboot介绍

uboot是写在flash里的一段启动引导程序，设备上电后，会把uboot程序从flash拷到内存中先跑起来，这里会有启动模式的判断，来决定是进入到Android系统还是Recovery系统。我们来看下uboot启动的大致流程，主要关注和启动模式相关的部分。  

### 3 uboot代码获取

如果你在原厂或是品牌商/OEM/ODM等类型公司工作，拿到的源码里都是带uboot的，因为uboot是要结合硬件板子来做定制的，所以google的源码里是不包含这部分的。因为我这边用的是freescale的imx系列芯片，开发的源码中是带uboot代码的，但是因为一些原因，原生写/cache/recovery/command的方式被废弃了，而改用了另一种写到param分区里的方式来实现，虽然思路相同，但是这边我还是想用原生的方式来梳理这套流程。所以我去找了fsl原厂的uboot代码，获取的方式为  

```xml
git clone git://git.freescale.com/imx/uboot-imx.git
git checkout maddev-imx-android-r13.2// 切下分支
```

另外提一下MTK的uboot方案是定制的preloader+lk的方式，所以代码结构也不太一样，所以还是用fsl的来做分析基础吧。  

### 4 uboot启动

代码位置：bootable/bootloader/uboot-imx  

uboot启动入口是在start.S这个汇编文件中，此文件在目录下的cpu/目录下存在很多，不同的架构都有一份，我们挑一个cpu/arm_cortexa8/start.S来看  

```S
	...
	ldr	pc, _start_armboot	@ jump to C code
_start_armboot: .word start_armboot
```

前面的操作都忽略，走到这里，将_start_armboot指令移到pc寄存器中，而\_start_armboot指向到了C代码中，代码位于lib_arm/board.c。  

### 5 检查recovery模式

```c
void start_armboot (void)
{
    ...
    #ifdef CONFIG_ANDROID_RECOVERY
	check_recovery_mode();
	#endif
    ...
}
```

我们只来看如何决定进入recovery，所以来看下check_recovery_mode调用，这个函数的定义在board/freescale/common/recovery.c中  

```c
/* export to lib_arm/board.c */
void check_recovery_mode(void)
{
	if (check_key_pressing())// 检查按键
		setup_recovery_env();// 如果是组合按键方式则设置recovery的启动环境
	else if (check_recovery_cmd_file()) {// 如果按键不符合，则检查启动命令
		puts("Recovery command file founded!\n");
		setup_recovery_env();// 如果检查到有recovery命令，则设置recovery启动环境
	}
}
```

这里我们先来看下两种检查方式，检查按键和recovery command，先来看第一种  

```c
#ifdef CONFIG_MXC_KPD//按键驱动，相反的对应qwerty

#define PRESSED_HOME	0x01// home键值
#define PRESSED_POWER	0x02// power键值
#define RECOVERY_KEY_MASK (PRESSED_HOME | PRESSED_POWER)// recovery组合键的定义是power+home，这个是可以定制的

inline int test_key(int value, struct kpp_key_info *ki)
{
	return (ki->val == value) && (ki->evt == KDepress);
}

int check_key_pressing(void)
{
	struct kpp_key_info *key_info;
	int state = 0, keys, i;

	mxc_kpp_init();// mxc驱动初始化

	puts("Detecting HOME+POWER key for recovery ...\n");

	/* Check for home + power */
	keys = mxc_kpp_getc(&key_info);// 获取按键
	if (keys < 2)
		return 0;

	for (i = 0; i < keys; i++) {
		if (test_key(CONFIG_POWER_KEY, &key_info[i]))// 测试home键
			state |= PRESSED_HOME;
		else if (test_key(CONFIG_HOME_KEY, &key_info[i]))// 测试power键
			state |= PRESSED_POWER;
	}

	free(key_info);

	if ((state & RECOVERY_KEY_MASK) == RECOVERY_KEY_MASK)// 如果检测到home+power，则跟定义的recovery组合键符合
		return 1;

	return 0;
}
#else
/* If not using mxc keypad, currently we will detect power key on board */
int check_key_pressing(void)
{
	return 0;
}
#endif
```

如果检测组合键无果后还有个途径就是检测recovery/command，这也是恢复出厂会用的方式，因为这个跟不同的芯片设计有关，所以会有很多客制化，先找一个来看下board/freescale/mx53_evk/mx53_evk.c  

```c
int check_recovery_cmd_file(void)
{
	disk_partition_t info;
	ulong part_length;
	int filelen;
	char *env;

	/* For test only */
	/* When detecting android_recovery_switch,
	 * enter recovery mode directly */
	env = getenv("android_recovery_switch");
	if (!strcmp(env, "1")) {
		printf("Env recovery detected!\nEnter recovery mode!\n");
		return 1;
	}

	printf("Checking for recovery command file...\n");
	switch (get_boot_device()) {// 获取启动设备
	case MMC_BOOT:
	case SD_BOOT:
		{
			block_dev_desc_t *dev_desc = NULL;
            // 取到所有注册的mmc设备的设备号是0的
			struct mmc *mmc = find_mmc_device(0);

			dev_desc = get_dev("mmc", 0);// 取到块设备描述信息

			if (NULL == dev_desc) {
				puts("** Block device MMC 0 not supported\n");
				return 0;
			}

			mmc_init(mmc);
			// 从块设备描述信息了找到cache分区的信息
			if (get_partition_info(dev_desc,
					CONFIG_ANDROID_CACHE_PARTITION_MMC,
					&info)) {
				printf("** Bad partition %d **\n",
					CONFIG_ANDROID_CACHE_PARTITION_MMC);
				return 0;
			}

            // 计算cache分区大小
			part_length = ext2fs_set_blk_dev(dev_desc,
					CONFIG_ANDROID_CACHE_PARTITION_MMC);
			if (part_length == 0) {
				printf("** Bad partition - mmc 0:%d **\n",
					CONFIG_ANDROID_CACHE_PARTITION_MMC);
				ext2fs_close();
				return 0;
			}

            // 挂载cache分区
			if (!ext2fs_mount(part_length)) {
				printf("** Bad ext2 partition or "
					"disk - mmc 0:%d **\n",
					CONFIG_ANDROID_CACHE_PARTITION_MMC);
				ext2fs_close();
				return 0;
			}

            // #define CONFIG_ANDROID_RECOVERY_CMD_FILE "/recovery/command"
            // 打开cache/recovery/command文件
			filelen = ext2fs_open(CONFIG_ANDROID_RECOVERY_CMD_FILE);

			ext2fs_close();
		}
		break;
	case NAND_BOOT:
		return 0;
		break;
	case SPI_NOR_BOOT:
		return 0;
		break;
	case UNKNOWN_BOOT:
	default:
		return 0;
		break;
	}

    // 如果有command文件，则返回1，标示找到了recovery command文件，但是这里是不去判断command的内容的
	return (filelen > 0) ? 1 : 0;

}
```

### 6 启动recovery

在上面提供了两种检测方式，检查组合键和recovery command的方式。组合键我们比较容易理解，我们平时操作就是按组合键开机，然后就进入了recovery。如果让我们自己设计uboot的话我们可能也是这个逻辑，检查到组合键之后就去设置启动进入recovery（如何启动recovery我们在设计阶段可以先不考虑技术细节）。但是这里检查recovery command的方式感觉就没有结束，只是检查到了cache分区里有/recovery/command，而没有去检查其内容，那么有个疑问就是难道只有存在command就会启动recovery吗？那么command的内容是干什么用的？确实检查到组合键或者command存在都会启动recovery，而command的内容是在recovery模式下读取然后进行判断再来决定进入recovery之后做什么。这里就把两种检查机制返回true之后的“setup_recovery_env”来介绍下  

代码位于board/freescale/common/recovery.c  

```c
void setup_recovery_env(void)
{
	char *env, *boot_args, *boot_cmd;
	int bootdev = get_boot_device();// 取到启动设备，MMC、SDCARD、NAND、SPI等

    // 取到驱动设备对应的启动命令
    // boot_cmd 实际就是
	// CONFIG_ANDROID_RECOVERY_BOOTCMD_MMC="run bootargs_android_recovery;mmc read ${loadaddr} ${kernel_offset} ${kernel_size};mmc read ${rd_loadaddr} ${recovery_offset} ${recovery_size};bootm ${loadaddr} ${rd_loadaddr}"
	boot_cmd = supported_reco_envs[bootdev].cmd;
    // 启动参数这里没有用到，有时候也会用，如下
    // boot_args = supported_reco_envs[bootdev].args;

	if (boot_cmd == NULL) {
		printf("Unsupported bootup device for recovery\n");
		return;
	}

	printf("setup env for recovery..\n");

	env = getenv("bootcmd_android_recovery");
	if (!env)
        // 将启动命令设置到环境变量中
		setenv("bootcmd_android_recovery", boot_cmd);
    // 设置bootcmd，去运行启动recovery的命令
	setenv("bootcmd", "run bootcmd_android_recovery");
}
```

### uboot调试

#### 添加日志

添加日志使用c语言标准的printf  

#### 编译

删除out下的u-boot.bin，先source然后make bootloader  

#### 烧录

fastboot flash bootloader_nor u-boot.bin

#### 查看日志

需要连接串口，在串口能看到uboot启动时输出的日志  