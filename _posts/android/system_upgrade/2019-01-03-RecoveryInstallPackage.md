---
layout:      post
title:      "系统升级系列七"
subtitle:   "升级包安装流程"
navcolor:   "invert"
date:       2019-01-03
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - System Upgrade
    - OTA
    - Recovery
    - 安装升级包
---
### 1 Recovery启动

在系列六里介绍了uboot如何读取启动参数，然后决定进入主系统（Main System）还是Recovery系统，这里就从启动了Recovery开始。Recovery的代码位于bootable/recovery目录下，我们先来看recovery.cpp，这个是Recovery启动的入口。  

先不着急看代码流程，在recovery.cpp开头，有一段注释非常重要，介绍了recovery安装升级包的流程，写代码就要这样。  

```java
/*   
 * OTA INSTALL    
 * 1. main system downloads OTA package to /cache/some-filename.zip    
 * 2. main system writes "--update_package=/cache/some-filename.zip"  
 * 3. main system reboots into recovery    
 * 4. get_args() writes BCB with "boot-recovery" and "--update_package=..."      
 *    -- after this, rebooting will attempt to reinstall the update --      
 * 5. install_package() attempts to install the update      
 *    NOTE: the package install must itself be restartable from any point      
 * 6. finish_recovery() erases BCB       
 *    -- after this, rebooting will (try to) restart the main system --      
 * 7. ** if install failed **      
 *    7a. prompt_and_wait() shows an error icon and waits for the user      
 *    7b; the user reboots (pulling the battery, etc) into the main system      
 * 8. main() calls maybe_install_firmware_update()        
 *    ** if the update contained radio/hboot firmware **:            
 *    8a. m_i_f_u() writes BCB with "boot-recovery" and "--wipe_cache"       
 *        -- after this, rebooting will reformat cache & restart main system --    
 *    8b. m_i_f_u() writes firmware image into raw cache partition   
 *    8c. m_i_f_u() writes BCB with "update-radio/hboot" and "--wipe_cache"      
 *        -- after this, rebooting will attempt to reinstall firmware --     
 *    8d. bootloader tries to flash firmware       
 *    8e. bootloader writes BCB with "boot-recovery" (keeping "--wipe_cache")     
 *        -- after this, rebooting will reformat cache & restart main system --       
 *    8f. erase_volume() reformats /cache     
 *    8g. finish_recovery() erases BCB        
 *        -- after this, rebooting will (try to) restart the main system --       
 * 9. main() calls reboot() to boot main system       
 */        
```

前面还介绍了recovery和主系统的交互方式和恢复出厂设置的流程，这里扣了一段OTA升级的。  

1. 下载升级包到cache目录下，这个是可以定制的，只要使用的目录一直统一就可以了  
2. 主系统写入升级命令，这个在系列五中已经介绍过了  
3. 重启进入到recovery，这个具体的在系列六中已经介绍过了  
4. 获取启动参数和安装包目录  
5. 尝试安装升级包  
6. 结束安装，清除启动命令，重启进入到主系统  
7. 如果失败了，提示失败信息等待用户操作；用户重启进入到主系统  
8. 升级radio或者hboot，不常用，这里不做重点介绍
9. 升级完radio或者hboot后重启  

以上从4开始就是今天会介绍的内容，而主要内容在安装升级包。  

#### 1.1 recovery.cpp::main  

进入正题，recovery入口在recovery.cpp的main函数中  

```cpp
int
main(int argc, char **argv) {
    time_t start = time(NULL);

    // If these fail, there's not really anywhere to complain...  
    
    // static const char *TEMPORARY_LOG_FILE = "/tmp/recovery.log";  
    
    // 所以在串口调试时可以通过cat /tmp/recovery.log的方式来查看日志  
    
    freopen(TEMPORARY_LOG_FILE, "a", stdout); setbuf(stdout, NULL);
    freopen(TEMPORARY_LOG_FILE, "a", stderr); setbuf(stderr, NULL);

    // If this binary is started with the single argument "--adbd",
    
    // instead of being the normal recovery binary, it turns into kind
    
    // of a stripped-down version of adbd that only supports the
    
    // 'sideload' command.  Note this must be a real argument, not
    
    // anything in the command file or bootloader control block; the
    
    // only way recovery should be run with this argument is when it
    
    // starts a copy of itself from the apply_from_adb() function.
    
    // sideload升级的方式，这里先不管
    
    if (argc == 2 && strcmp(argv[1], "--adbd") == 0) {
        adb_main();
        return 0;
    }

    printf("Starting recovery on %s", ctime(&start));
	// 加载recovery分区表，详见2.1  
	
    load_volume_table();
    // #define LAST_LOG_FILE "/cache/recovery/last_log"
    
    // last_log目录挂载，ensure_path_mounted比较重要，用于目录的挂载，U盘升级的方式也是通过这个来挂载U盘，详见2.2  
    
    ensure_path_mounted(LAST_LOG_FILE);
    // 保存旧的日志，详见2.3
    
    // Rename last_log -> last_log.1 -> last_log.2 -> ... -> last_log.$max
    
    // 这里最多存在11个，last_log--last_log.10
    
    rotate_last_logs(10);
    // 取到启动参数，这个很关键，详见3.1
    
    get_args(&argc, &argv);

    int previous_runs = 0;
    const char *send_intent = NULL;
    const char *update_package = NULL;
    int wipe_data = 0, wipe_cache = 0, show_text = 0;
    bool just_exit = false;

	// 这一段是调用linux的getopt_long函数来解析参数，详细用法可以参考"man getopt_long"
	
	// OPTIONS是option结构体数组，其中一条例如下：
	
	// { "update_package", required_argument, NULL, 'u' }
	
	// 意思就是从argv中解析到update_package时，将附加的参数读到optarg中，arg赋值为'u'
	
    int arg;
    while ((arg = getopt_long(argc, argv, "", OPTIONS, NULL)) != -1) {
        switch (arg) {
        case 'p': previous_runs = atoi(optarg); break;
        case 's': send_intent = optarg; break;
        // 将optarg赋值给update_package，其实就是command里带的升级包路径
        
        case 'u': update_package = optarg; break;
        case 'w': wipe_data = wipe_cache = 1; break;
        case 'c': wipe_cache = 1; break;
        case 't': show_text = 1; break;
        case 'x': just_exit = true; break;
        case 'l': locale = optarg; break;
        case '?':
            LOGE("Invalid command argument\n");
            continue;
        }
    }

    if (locale == NULL) {
    	// 去读/cache/recovery/last_local
    	
        load_locale_from_cache();
    }
    printf("locale is [%s]\n", locale);

	// 实例化一个Device，用来做ui显示，要研究recovery的ui可以从这里入手
	
    Device* device = make_device();
    ui = device->GetUI();
    gCurrentUI = ui;
	
	// ui初始化部分，多语言界面的客制化可以从这里入手
	
    ui->Init();
    ui->SetLocale(locale);
    ui->SetBackground(RecoveryUI::NONE);
    if (show_text) ui->ShowText(true);

	// selinux权限
	
    struct selinux_opt seopts[] = {
      { SELABEL_OPT_PATH, "/file_contexts" }
    };

    sehandle = selabel_open(SELABEL_CTX_FILE, seopts, 1);

    if (!sehandle) {
        ui->Print("Warning: No file_contexts\n");
    }

    device->StartRecovery();

    printf("Command:");
    for (arg = 0; arg < argc; arg++) {
        printf(" \"%s\"", argv[arg]);
    }
    printf("\n");
	// 兼容老版本的cache路径
	
    if (update_package) {
        // For backwards compatibility on the cache partition only, if
        
        // we're given an old 'root' path "CACHE:foo", change it to
        
        // "/cache/foo".
        
        if (strncmp(update_package, "CACHE:", 6) == 0) {
            int len = strlen(update_package) + 10;
            char* modified_path = (char*)malloc(len);
            strlcpy(modified_path, "/cache/", len);
            strlcat(modified_path, update_package+6, len);
            printf("(replacing path \"%s\" with \"%s\")\n",
                   update_package, modified_path);
            update_package = modified_path;
        }
    }
    printf("\n");

    property_list(print_property, NULL);
    property_get("ro.build.display.id", recovery_version, "");
    printf("\n");

    int status = INSTALL_SUCCESS;

    if (update_package != NULL) {
    	// 这里是流程的重点，参考4.1，进入到安装升级包，
    	
    	// 三个参数
    	
    	// update_package--升级路径
    	
    	// wipe_cache--是否清cache
    	
    	// TEMPORARY_INSTALL_FILE="/tmp/last_install"--临时升级日志目录
    	
        status = install_package(update_package, &wipe_cache, TEMPORARY_INSTALL_FILE);
        if (status == INSTALL_SUCCESS && wipe_cache) {
        	// 如果升级成功再根据wipe_cache的值擦除cache分区，那么问题来了，升级成功后我们还能看到cache/recovery/last_log，这个是怎么来的？
        	
        	// 在erase_volume中，会先把last开头的文件写入到内存中，擦除完cache分区之后再写回来        	
            if (erase_volume("/cache")) {
                LOGE("Cache wipe (requested by package) failed.");
            }
        }
        if (status != INSTALL_SUCCESS) {
            ui->Print("Installation aborted.\n");

            // If this is an eng or userdebug build, then automatically
            
            // turn the text display on if the script fails so the error
            
            // message is visible.
            
            char buffer[PROPERTY_VALUE_MAX+1];
            property_get("ro.build.fingerprint", buffer, "");
            if (strstr(buffer, ":userdebug/") || strstr(buffer, ":eng/")) {
                ui->ShowText(true);
            }
        }
    } else if (wipe_data) {
        if (device->WipeData()) status = INSTALL_ERROR;
        if (erase_volume("/data")) status = INSTALL_ERROR;
        if (wipe_cache && erase_volume("/cache")) status = INSTALL_ERROR;
        if (status != INSTALL_SUCCESS) ui->Print("Data wipe failed.\n");
    } else if (wipe_cache) {
        if (wipe_cache && erase_volume("/cache")) status = INSTALL_ERROR;
        if (status != INSTALL_SUCCESS) ui->Print("Cache wipe failed.\n");
    } else if (!just_exit) {
        status = INSTALL_NONE;  // No command specified
        ui->SetBackground(RecoveryUI::NO_COMMAND);
    }

    if (status == INSTALL_ERROR || status == INSTALL_CORRUPT) {
        copy_logs();
        ui->SetBackground(RecoveryUI::ERROR);
    }
    if (status != INSTALL_SUCCESS || ui->IsTextVisible()) {
    	// 如果只是进入recovery，而没有升级命令，例如组合键的方式进入，等待的是用户操作，交给prompt_and_wait函数来等待用户操作，参考后面第6节手动模式
    	
        prompt_and_wait(device, status);
    }

    // Otherwise, get ready to boot the main system...
    
    // 这个也是安装流程的核心，用来做收尾工作，主要包括了拷贝日志，清除boot message，清除command，详见5.1
    
    finish_recovery(send_intent);
    ui->Print("Rebooting...\n");
    property_set(ANDROID_RB_PROPERTY, "reboot,");
    return EXIT_SUCCESS;
}
```



### 2 环境初始化

#### 2.1 roots.cpp::load_volume_table

```cpp
void load_volume_table()
{
    int i;
    int ret;
	// 加载分区表
    
    fstab = fs_mgr_read_fstab("/etc/recovery.fstab");
    if (!fstab) {
        LOGE("failed to read /etc/recovery.fstab\n");
        return;
    }
	// 额外添加/tmp目录用于生成临时日志之类的
    
    ret = fs_mgr_add_entry(fstab, "/tmp", "ramdisk", "ramdisk", 0);  
    if (ret < 0 ) {
        LOGE("failed to add /tmp entry to fstab\n");
        fs_mgr_free_fstab(fstab);
        fstab = NULL;
        return;
    }

    printf("recovery filesystem table\n");
    printf("=========================\n");
    for (i = 0; i < fstab->num_entries; ++i) {
        Volume* v = &fstab->recs[i];
        printf("  %d %s %s %s %lld\n", i, v->mount_point, v->fs_type,
               v->blk_device, v->length);
    }
    printf("\n");
}
```

recovery的分区表信息为recovery.fstab，在设备中位于/etc目录下，在编译时，生成在out/target/product/\<product_name\>/recovery/root/etc下，例如  

```xml
# Android fstab file.
#<src>                                                  <mnt_point>         <type>    <mnt_flags and options>                       <fs_mgr_flags>
# The filesystem that contains the filesystem checker binary (typically /system) cannot
# specify MF_CHECK, and must come before any filesystems that do specify MF_CHECK

#/dev/block/mmcblk0p1 /misc emmc defaults 0  
/dev/block/mmcblk0p6 /cache ext4 defaults 0  
/dev/block/mmcblk0p5 /data ext4 defaults 0  
/dev/block/mmcblk0p8 /private ext4 defaults 0  
```

#### 2.2 roots.cpp::ensure_path_mounted

```cpp
int ensure_path_mounted(const char* path) {
    Volume* v = volume_for_path(path);
    if (v == NULL) {
        LOGE("unknown volume for path [%s]\n", path);
        return -1;
    }
    if (strcmp(v->fs_type, "ramdisk") == 0) {
        // the ramdisk is always mounted.
        
        return 0;
    }

    int result;
    result = scan_mounted_volumes();
    if (result < 0) {
        LOGE("failed to scan mounted volumes\n");
        return -1;
    }

    const MountedVolume* mv =
        find_mounted_volume_by_mount_point(v->mount_point);
    if (mv) {
        // volume is already mounted
        
        return 0;
    }

    mkdir(v->mount_point, 0755);  // in case it doesn't already exist

    if (strcmp(v->fs_type, "yaffs2") == 0) {
        // mount an MTD partition as a YAFFS2 filesystem.
        
        mtd_scan_partitions();
        const MtdPartition* partition;
        partition = mtd_find_partition_by_name(v->blk_device);
        if (partition == NULL) {
            LOGE("failed to find \"%s\" partition to mount at \"%s\"\n",
                 v->blk_device, v->mount_point);
            return -1;
        }
        return mtd_mount_partition(partition, v->mount_point, v->fs_type, 0);
    } else if (strcmp(v->fs_type, "ext4") == 0 ||
               strcmp(v->fs_type, "vfat") == 0) {
        result = mount(v->blk_device, v->mount_point, v->fs_type,
                       MS_NOATIME | MS_NODEV | MS_NODIRATIME, "");
        if (result == 0) return 0;

        LOGE("failed to mount %s (%s)\n", v->mount_point, strerror(errno));
        return -1;
    }

    LOGE("unknown fs_type \"%s\" for %s\n", v->fs_type, v->mount_point);
    return -1;
}
```

#### 2.3 recovery.cpp::rotate_last_logs

```cpp
// Rename last_log -> last_log.1 -> last_log.2 -> ... -> last_log.$max

// Overwrites any existing last_log.$max.

static void
rotate_last_logs(int max) {
    char oldfn[256];
    char newfn[256];

    int i;
    // 把last_log.9写到last_log.10中，把8写到9中...把last_log写到last_log.1中
    
    for (i = max-1; i >= 0; --i) {
        snprintf(oldfn, sizeof(oldfn), (i==0) ? LAST_LOG_FILE : (LAST_LOG_FILE ".%d"), i);
        snprintf(newfn, sizeof(newfn), LAST_LOG_FILE ".%d", i+1);
        // ignore errors
        
        rename(oldfn, newfn);
    }
}
```

### 3 获取升级参数

#### 3.1 recovery.cpp::get_args

```cpp
// command line args come from, in decreasing precedence:

//   - the actual command line

//   - the bootloader control block (one per line, after "recovery")

//   - the contents of COMMAND_FILE (one per line)

static void
get_args(int *argc, char ***argv) {
    struct bootloader_message boot;
    memset(&boot, 0, sizeof(boot));
    get_bootloader_message(&boot);  // this may fail, leaving a zeroed structure

    if (boot.command[0] != 0 && boot.command[0] != 255) {
        LOGI("Boot command: %.*s\n", sizeof(boot.command), boot.command);
    }

    if (boot.status[0] != 0 && boot.status[0] != 255) {
        LOGI("Boot status: %.*s\n", sizeof(boot.status), boot.status);
    }

    // --- if arguments weren't supplied, look in the bootloader control block
    
    // 从bootloader的启动命令中找到recovery参数
    
    if (*argc <= 1) {
        boot.recovery[sizeof(boot.recovery) - 1] = '\0';  // Ensure termination
        
        const char *arg = strtok(boot.recovery, "\n");
        if (arg != NULL && !strcmp(arg, "recovery")) {
            *argv = (char **) malloc(sizeof(char *) * MAX_ARGS);
            (*argv)[0] = strdup(arg);
            for (*argc = 1; *argc < MAX_ARGS; ++*argc) {
                if ((arg = strtok(NULL, "\n")) == NULL) break;
                // 把命令放到argv中带回到main函数里
                
                (*argv)[*argc] = strdup(arg);
            }
            LOGI("Got arguments from boot message\n");
        } else if (boot.recovery[0] != 0 && boot.recovery[0] != 255) {
            LOGE("Bad boot message\n\"%.20s\"\n", boot.recovery);
        }
    }

    // --- if that doesn't work, try the command file
    
    // 从/cache/recovery/command里再找一次
    
    if (*argc <= 1) {
        FILE *fp = fopen_path(COMMAND_FILE, "r");
        if (fp != NULL) {
            char *token;
            char *argv0 = (*argv)[0];
            *argv = (char **) malloc(sizeof(char *) * MAX_ARGS);
            (*argv)[0] = argv0;  // use the same program name

            char buf[MAX_ARG_LENGTH];
            for (*argc = 1; *argc < MAX_ARGS; ++*argc) {
                if (!fgets(buf, sizeof(buf), fp)) break;
                token = strtok(buf, "\r\n");
                if (token != NULL) {
                    (*argv)[*argc] = strdup(token);  // Strip newline.
                    
                } else {
                    --*argc;
                }
            }

            check_and_fclose(fp, COMMAND_FILE);
            LOGI("Got arguments from %s\n", COMMAND_FILE);
        }
    }

    // --> write the arguments we have back into the bootloader control block
    
    // always boot into recovery after this (until finish_recovery() is called)
    
    // 回写到bootloader启动命令控制块中，后续每次重启都会进入到recovery，直到完成了finish_recovery
    
    strlcpy(boot.command, "boot-recovery", sizeof(boot.command));
    strlcpy(boot.recovery, "recovery\n", sizeof(boot.recovery));
    int i;
    for (i = 1; i < *argc; ++i) {
        strlcat(boot.recovery, (*argv)[i], sizeof(boot.recovery));
        strlcat(boot.recovery, "\n", sizeof(boot.recovery));
    }
    set_bootloader_message(&boot);
}
```

##### 3.1.1 bootloader_message

位于bootloader.h，存储bootloader启动recovery的参数  

```cpp
/* Bootloader Message

 *
 
 * This structure describes the content of a block in flash
 
 * that is used for recovery and the bootloader to talk to
 
 * each other.
 
 *
 
 * The command field is updated by linux when it wants to
 
 * reboot into recovery or to update radio or bootloader firmware.
 
 * It is also updated by the bootloader when firmware update
 
 * is complete (to boot into recovery for any final cleanup)
 
 *
 
 * The status field is written by the bootloader after the
 
 * completion of an "update-radio" or "update-hboot" command.
 
 *
 
 * The recovery field is only written by linux and used
 
 * for the system to send a message to recovery or the
 
 * other way around.
 
 */

struct bootloader_message {
    char command[32];
    char status[32];
    char recovery[1024];
};

```

##### 3.1.2 get_bootloader_message

位于bootloader.cpp中  

```cpp
int get_bootloader_message(struct bootloader_message *out) {
    Volume* v = volume_for_path("/misc");//获取misc分区
    
    if (v == NULL) {
      LOGE("Cannot load volume /misc!\n");
      return -1;
    }
    // 根据分区类型再来做分别调用，我们设备使用的是emmc，来看下get_bootloader_message_block
    
    if (strcmp(v->fs_type, "mtd") == 0) {
        return get_bootloader_message_mtd(out, v);
    } else if (strcmp(v->fs_type, "emmc") == 0) {
        return get_bootloader_message_block(out, v);
    }
    LOGE("unknown misc partition fs_type \"%s\"\n", v->fs_type);
    return -1;
}
```

##### 3.1.3 get_bootloader_message_block

位于bootloader.cpp中  

```cpp
static int get_bootloader_message_block(struct bootloader_message *out,
                                        const Volume* v) {
    wait_for_device(v->blk_device);
    // 打开misc块设备
    
    FILE* f = fopen(v->blk_device, "rb");
    if (f == NULL) {
        LOGE("Can't open %s\n(%s)\n", v->blk_device, strerror(errno));
        return -1;
    }
    // 将misc分区内容读到temp中
    
    struct bootloader_message temp;
    int count = fread(&temp, sizeof(temp), 1, f);
    if (count != 1) {
        LOGE("Failed reading %s\n(%s)\n", v->blk_device, strerror(errno));
        return -1;
    }
    if (fclose(f) != 0) {
        LOGE("Failed closing %s\n(%s)\n", v->blk_device, strerror(errno));
        return -1;
    }
    // 将temp写到out中，也就是把bootloader message的内容通过指针带回
    
    memcpy(out, &temp, sizeof(temp));
    return 0;
}
```

### 4 安装升级包

#### 4.1 install.cpp::install_package

```cpp
int
install_package(const char* path, int* wipe_cache, const char* install_file)
{
    // install_log记录了升级包路径
    
    FILE* install_log = fopen_path(install_file, "w");
    if (install_log) {
        fputs(path, install_log);
        fputc('\n', install_log);
    } else {
        LOGE("failed to open last_install: %s\n", strerror(errno));
    }
    int result;
    if (setup_install_mounts() != 0) {
        LOGE("failed to set up expected mounts for install; aborting\n");
        result = INSTALL_ERROR;
    } else {
        // 这里进入真正安装，见下一章
        
        result = really_install_package(path, wipe_cache);
    }
    // install_log记录了升级结果，1成功，0失败
    
    if (install_log) {
        fputc(result == INSTALL_SUCCESS ? '1' : '0', install_log);
        fputc('\n', install_log);
        fclose(install_log);
    }
    return result;
}
```

#### 4.2 setup_install_mounts

代码roots.cpp，作用是在安装之前初始化安装需要的分区挂载工作，这里原生代码中，如果从分区表读到了挂载点为/tmp或者/cache分区，则去尝试挂载，其他分区尝试卸载。如果我们做定制，例如添加分区，或者是不希望卸载某些分区的时候，可以修改这里的逻辑，例如只要是分区表中的分区，都挂载即可。  

```cpp
int setup_install_mounts() {
    if (fstab == NULL) {
        LOGE("can't set up install mounts: no fstab loaded\n");
        return -1;
    }
    for (int i = 0; i < fstab->num_entries; ++i) {
        Volume* v = fstab->recs + i;

        if (strcmp(v->mount_point, "/tmp") == 0 ||
            strcmp(v->mount_point, "/cache") == 0) {
            if (ensure_path_mounted(v->mount_point) != 0) return -1;

        } else {
            if (ensure_path_unmounted(v->mount_point) != 0) return -1;
        }
    }
    return 0;
}
```

#### 4.3 really_install_package

```cpp
static int
really_install_package(const char *path, int* wipe_cache)
{
    // ui设置部分，这里会涉及到ui定制，改背景图片之类
    
    ui->SetBackground(RecoveryUI::INSTALLING_UPDATE);
    ui->Print("Finding update package...\n");
    // Give verification half the progress bar...
    
    ui->SetProgressType(RecoveryUI::DETERMINATE);
    ui->ShowProgress(VERIFICATION_PROGRESS_FRACTION, VERIFICATION_PROGRESS_TIME);
    LOGI("Update location: %s\n", path);

    // 挂载安装文件所在，如果是U盘升级，这里是测试出错最多了，经常会因为个别U盘识别问题这里就报错
    
    if (ensure_path_mounted(path) != 0) {
        LOGE("Can't mount %s\n", path);
        return INSTALL_CORRUPT;
    }

    ui->Print("Opening update package...\n");

    int numKeys;
    // 从/res/key中加载公钥，见4.4
    
    Certificate* loadedKeys = load_keys(PUBLIC_KEYS_FILE, &numKeys);
    if (loadedKeys == NULL) {
        LOGE("Failed to load keys\n");
        return INSTALL_CORRUPT;
    }
    LOGI("%d key(s) loaded from %s\n", numKeys, PUBLIC_KEYS_FILE);

    ui->Print("Verifying update package...\n");

    int err;
    // 用公钥来校验升级包签名，详见4.5
    
    err = verify_file(path, loadedKeys, numKeys);
    free(loadedKeys);
    LOGI("verify_file returned %d\n", err);
    if (err != VERIFY_SUCCESS) {
        LOGE("signature verification failed\n");
        return INSTALL_CORRUPT;
    }

    /* Try to open the package.
     */
    
    ZipArchive zip;
    // 打开并扫描内容到zip中，详见4.6
    
    err = mzOpenZipArchive(path, &zip);
    if (err != 0) {
        LOGE("Can't open %s\n(%s)\n", path, err != -1 ? strerror(err) : "bad");
        return INSTALL_CORRUPT;
    }

    /* Verify and install the contents of the package.
     */
    
    ui->Print("Installing update...\n");
    // 准备用updater程序来做升级，详见4.7
    
    return try_update_binary(path, &zip, wipe_cache);
}
```

#### 4.4 load_keys

公钥的位置在recovery系统的/res/key下，/res下也包括了ui用到的图片和文字（png形式），key的内容如下例：  

```xml
v2 {64,0xac8263c1,{2623558591,4129352496,3619307294,1733525894,4184181140,2183609138,359385070,3499603564,1253115871,2020329408,1454169246,4060812049,4258124289,2274749027,1738697311,2044927431,3044780914,3802674911,2715853980,2244528683,865151561,2675609242,3775154336,1651550970,2867900749,867136141,2276139121,3305470757,85756060,3246695085,129965540,1001987497,2191172590,4027738750,2928439848,1770137053,2738354340,3762332136,956784980,3548168872,1433597255,1436685120,725702207,485702277,1471687611,695364983,1184367274,3962731269,3792044367,3320874508,400804384,3701678800,2498230266,3591842649,1834798600,4045751497,3729832285,2368730451,455921334,2217268269,1848486709,3684416332,516130221,3298408301},{3897649612,1697031415,2195092620,1647893902,2113204748,1977064309,4148631423,2176654807,1740691286,618480656,90450848,278990949,1882739812,3017823331,3560709577,2840070122,162099387,1119124189,1042078873,717416586,1043473661,4095431750,1895841744,621978925,3316697647,64768160,3710469994,3984239669,1184110615,3599177883,1760404633,431525418,3444881611,911243604,130584453,2944218772,2712559993,180982919,3457641495,3008060663,4288802231,3241373031,3612338378,2209930564,1214234090,3683533724,1638861259,4278020420,134732044,719677914,3295618569,3483583029,3689171431,2336665568,2978790550,1521057868,2361311657,622759589,4098544847,2788104477,954107361,3427306801,87936696,336451220}}
```

load_keys函数会根据key的类型，将它们读到内存中  

```cpp
// Reads a file containing one or more public keys as produced by  

// DumpPublicKey:  this is an RSAPublicKey struct as it would appear  

// as a C source literal, eg:  

//

//  "{64,0xc926ad21,{1795090719,...,-695002876},{-857949815,...,1175080310}}"  

//  

// For key versions newer than the original 2048-bit e=3 keys  

// supported by Android, the string is preceded by a version  

// identifier, eg:  

//  

//  "v2 {64,0xc926ad21,{1795090719,...,-695002876},{-857949815,...,1175080310}}"  

//  

// (Note that the braces and commas in this example are actual  

// characters the parser expects to find in the file; the ellipses  

// indicate more numbers omitted from this example.)  

//  

// The file may contain multiple keys in this format, separated by  

// commas.  The last key must not be followed by a comma.  

//  

// A Certificate is a pair of an RSAPublicKey and a particular hash  

// (we support SHA-1 and SHA-256; we store the hash length to signify  

// which is being used).  The hash used is implied by the version number.  

//  

//       1: 2048-bit RSA key with e=3 and SHA-1 hash  

//       2: 2048-bit RSA key with e=65537 and SHA-1 hash  

//       3: 2048-bit RSA key with e=3 and SHA-256 hash  

//       4: 2048-bit RSA key with e=65537 and SHA-256 hash  

//  

// Returns NULL if the file failed to parse, or if it contain zero keys.   

Certificate*
load_keys(const char* filename, int* numKeys) {
    Certificate* out = NULL;
    *numKeys = 0;

    FILE* f = fopen(filename, "r");
    if (f == NULL) {
        LOGE("opening %s: %s\n", filename, strerror(errno));
        goto exit;
    }

    {
        int i;
        bool done = false;
        while (!done) {
            ++*numKeys;
            out = (Certificate*)realloc(out, *numKeys * sizeof(Certificate));
            Certificate* cert = out + (*numKeys - 1);
            cert->public_key = (RSAPublicKey*)malloc(sizeof(RSAPublicKey));

            char start_char;
            if (fscanf(f, " %c", &start_char) != 1) goto exit;
            if (start_char == '{') {
                // a version 1 key has no version specifier.
                
                cert->public_key->exponent = 3;
                cert->hash_len = SHA_DIGEST_SIZE;
            } else if (start_char == 'v') {
                int version;
                if (fscanf(f, "%d {", &version) != 1) goto exit;
                //读取版本号 
                
                switch (version) {
                    case 2:
                        cert->public_key->exponent = 65537;
                        cert->hash_len = SHA_DIGEST_SIZE;
                        break;
                    case 3:
                        cert->public_key->exponent = 3;
                        cert->hash_len = SHA256_DIGEST_SIZE;
                        break;
                    case 4:
                        cert->public_key->exponent = 65537;
                        cert->hash_len = SHA256_DIGEST_SIZE;
                        break;
                    default:
                        goto exit;
                }
            }
			// 读取key的内容
			
            RSAPublicKey* key = cert->public_key;
            if (fscanf(f, " %i , 0x%x , { %u",
                       &(key->len), &(key->n0inv), &(key->n[0])) != 3) {
                goto exit;
            }
            if (key->len != RSANUMWORDS) {
                LOGE("key length (%d) does not match expected size\n", key->len);
                goto exit;
            }
            for (i = 1; i < key->len; ++i) {
                if (fscanf(f, " , %u", &(key->n[i])) != 1) goto exit;
            }
            if (fscanf(f, " } , { %u", &(key->rr[0])) != 1) goto exit;
            for (i = 1; i < key->len; ++i) {
                if (fscanf(f, " , %u", &(key->rr[i])) != 1) goto exit;
            }
            fscanf(f, " } } ");// 一个key以{{}}包含分割

            // if the line ends in a comma, this file has more keys.
            
            switch (fgetc(f)) {
            case ',':
                // more keys to come.
                
                break;

            case EOF:// 读到文件尾结束
            
                done = true;
                break;

            default:
                LOGE("unexpected character between keys\n");
                goto exit;
            }

            LOGI("read key e=%d hash=%d\n", key->exponent, cert->hash_len);
        }
    }

    fclose(f);
    return out;

exit:
    if (f) fclose(f);
    free(out);
    *numKeys = 0;
    return NULL;
}
```

#### 4.5 verify_file

关于签名校验，我在系列五中RecoverySystem校验包的时候已经完整介绍过了，跟这里是完全相同的，只是描述语言的区别，可以直接参考[升级包签名校验](http://chendongqi.me/2018/12/25/InstallRequest/#33-verifypackage)来阅读这个函数。  

```cpp
// Look for an RSA signature embedded in the .ZIP file comment given

// the path to the zip.  Verify it matches one of the given public

// keys.

//

// Return VERIFY_SUCCESS, VERIFY_FAILURE (if any error is encountered

// or no key matches the signature).


int verify_file(const char* path, const Certificate* pKeys, unsigned int numKeys,unsigned int offset)
{
    ui->SetProgress(0.0);

    FILE* f = fopen(path, "rb");
    if (f == NULL) {
        LOGE("failed to open %s (%s)\n", path, strerror(errno));
        return VERIFY_FAILURE;
    }

    // An archive with a whole-file signature will end in six bytes:
    
    //
    
    //   (2-byte signature start) $ff $ff (2-byte comment size)
    
    //
    
    // (As far as the ZIP format is concerned, these are part of the
    
    // archive comment.)  We start by reading this footer, this tells
    
    // us how far back from the end we have to start reading to find
    
    // the whole comment.
    
#define FOOTER_SIZE 6

    if (fseek(f, -1*(FOOTER_SIZE+offset), SEEK_END) != 0) {
        LOGE("failed to seek in %s (%s)\n", path, strerror(errno));
        fclose(f);
        return VERIFY_FAILURE;
    }

    unsigned char footer[FOOTER_SIZE];
    if (fread(footer, 1, FOOTER_SIZE, f) != FOOTER_SIZE) {
        LOGE("failed to read footer from %s (%s)\n", path, strerror(errno));
        fclose(f);
        return VERIFY_FAILURE;
    }

    if (footer[2] != 0xff || footer[3] != 0xff) {
        LOGE("footer is wrong\n");
        fclose(f);
        return VERIFY_FAILURE;
    }

    size_t comment_size = footer[4] + (footer[5] << 8);
    size_t signature_start = footer[0] + (footer[1] << 8);
    LOGI("comment is %d bytes; signature %d bytes from end\n",
         comment_size, signature_start);

    if (signature_start - FOOTER_SIZE < RSANUMBYTES) {
        // "signature" block isn't big enough to contain an RSA block.
        
        LOGE("signature is too short\n");
        fclose(f);
        return VERIFY_FAILURE;
    }

#define EOCD_HEADER_SIZE 22

    // The end-of-central-directory record is 22 bytes plus any
    
    // comment length.
    
    size_t eocd_size = comment_size + EOCD_HEADER_SIZE;

    if (fseek(f, -1*(eocd_size+offset), SEEK_END) != 0) {
        LOGE("failed to seek in %s (%s)\n", path, strerror(errno));
        fclose(f);
        return VERIFY_FAILURE;
    }

    // Determine how much of the file is covered by the signature.
    
    // This is everything except the signature data and length, which
    
    // includes all of the EOCD except for the comment length field (2
    
    // bytes) and the comment data.
    
    size_t signed_len = ftell(f) + EOCD_HEADER_SIZE - 2;

    unsigned char* eocd = (unsigned char*)malloc(eocd_size);
    if (eocd == NULL) {
        LOGE("malloc for EOCD record failed\n");
        fclose(f);
        return VERIFY_FAILURE;
    }
    if (fread(eocd, 1, eocd_size, f) != eocd_size) {
        LOGE("failed to read eocd from %s (%s)\n", path, strerror(errno));
        fclose(f);
        return VERIFY_FAILURE;
    }

    // If this is really is the EOCD record, it will begin with the
    
    // magic number $50 $4b $05 $06.
    
    if (eocd[0] != 0x50 || eocd[1] != 0x4b ||
        eocd[2] != 0x05 || eocd[3] != 0x06) {
        LOGE("signature length doesn't match EOCD marker\n");
        fclose(f);
        return VERIFY_FAILURE;
    }

    size_t i;
    for (i = 4; i < eocd_size-3; ++i) {
        if (eocd[i  ] == 0x50 && eocd[i+1] == 0x4b &&
            eocd[i+2] == 0x05 && eocd[i+3] == 0x06) {
            // if the sequence $50 $4b $05 $06 appears anywhere after
            
            // the real one, minzip will find the later (wrong) one,
            
            // which could be exploitable.  Fail verification if
            
            // this sequence occurs anywhere after the real one.
            
            LOGE("EOCD marker occurs after start of EOCD\n");
            fclose(f);
            return VERIFY_FAILURE;
        }
    }

#define BUFFER_SIZE 4096

    bool need_sha1 = false;
    bool need_sha256 = false;
    for (i = 0; i < numKeys; ++i) {
        switch (pKeys[i].hash_len) {
            case SHA_DIGEST_SIZE: need_sha1 = true; break;
            case SHA256_DIGEST_SIZE: need_sha256 = true; break;
        }
    }

    SHA_CTX sha1_ctx;
    SHA256_CTX sha256_ctx;
    SHA_init(&sha1_ctx);
    SHA256_init(&sha256_ctx);
    unsigned char* buffer = (unsigned char*)malloc(BUFFER_SIZE);
    if (buffer == NULL) {
        LOGE("failed to alloc memory for sha1 buffer\n");
        fclose(f);
        return VERIFY_FAILURE;
    }

    double frac = -1.0;
    size_t so_far = 0;
    fseek(f, 0, SEEK_SET);
    while (so_far < signed_len) {
        size_t size = BUFFER_SIZE;
        if (signed_len - so_far < size) size = signed_len - so_far;
        if (fread(buffer, 1, size, f) != size) {
            LOGE("failed to read data from %s (%s)\n", path, strerror(errno));
            fclose(f);
            return VERIFY_FAILURE;
        }
        if (need_sha1) SHA_update(&sha1_ctx, buffer, size);
        if (need_sha256) SHA256_update(&sha256_ctx, buffer, size);
        so_far += size;
        double f = so_far / (double)signed_len;
        if (f > frac + 0.02 || size == so_far) {
            ui->SetProgress(f);
            frac = f;
        }
    }
    fclose(f);
    free(buffer);

    const uint8_t* sha1 = SHA_final(&sha1_ctx);
    const uint8_t* sha256 = SHA256_final(&sha256_ctx);

    for (i = 0; i < numKeys; ++i) {
        const uint8_t* hash;
        switch (pKeys[i].hash_len) {
            case SHA_DIGEST_SIZE: hash = sha1; break;
            case SHA256_DIGEST_SIZE: hash = sha256; break;
            default: continue;
        }

        // The 6 bytes is the "(signature_start) $ff $ff (comment_size)" that
        
        // the signing tool appends after the signature itself.
        
        if (RSA_verify(pKeys[i].public_key, eocd + eocd_size - 6 - RSANUMBYTES,
                       RSANUMBYTES, hash, pKeys[i].hash_len)) {
            LOGI("whole-file signature verified against key %d\n", i);
            free(eocd);
            return VERIFY_SUCCESS;
        } else {
            LOGI("failed to verify against key %d\n", i);
        }
    }
    free(eocd);
    LOGE("failed to verify whole-file signature\n");
    return VERIFY_FAILURE;
}
```

#### 4.6 mzOpenZipArchive

这里是去解zip文件，用到了recovery/minzip模块来完成这部分功能

```cpp
int mzOpenZipArchive(const char* fileName, ZipArchive* pArchive)
{
    MemMapping map;// 内存映射
    
    int err;

    LOGV("Opening archive '%s' %p\n", fileName, pArchive);

    // 初始化
    
    map.addr = NULL;
    memset(pArchive, 0, sizeof(*pArchive));

    // 打开升级包文件拿到文件描述符
    
    pArchive->fd = open(fileName, O_RDONLY, 0);
    if (pArchive->fd < 0) {
        err = errno ? errno : -1;
        LOGV("Unable to open '%s': %s\n", fileName, strerror(err));
        goto bail;
    }

    // 通过mmap内存应声的方式将文件写入到map中
    
    if (sysMapFileInShmem(pArchive->fd, &map) != 0) {
        err = -1;
        LOGW("Map of '%s' failed\n", fileName);
        goto bail;
    }

    // ENDHDR=22，就是eocd的长度，如果小于这个值，从文件数据格式上就可以判断不是一个zip文件
    
    if (map.length < ENDHDR) {
        err = -1;
        LOGV("File '%s' too small to be zip (%zd)\n", fileName, map.length);
        goto bail;
    }

    // 解析文件内容
    
    if (!parseZipArchive(pArchive, &map)) {
        err = -1;
        LOGV("Parsing '%s' failed\n", fileName);
        goto bail;
    }

    err = 0;
    // 从内存中拷贝到数据变量中保存，带回
    
    sysCopyMap(&pArchive->map, &map);
    map.addr = NULL;

bail:
    if (err != 0)
        mzCloseZipArchive(pArchive);
    if (map.addr != NULL)
        sysReleaseShmem(&map);
    return err;
}
```

#### 4.7 try_update_binary

```cpp
// If the package contains an update binary, extract it and run it.  

static int
try_update_binary(const char *path, ZipArchive *zip, int* wipe_cache) {
	// 真正安装升级包是交给updater这个程序，它的代码在recovery/updater下，编译生成updater，会被打到升级包中META-INF/com/google/android/update-binary，system/bin下也会有一份  
	
	// 这里的ASSUMED_UPDATE_BINARY_NAME就是META-INF/com/google/android/update-binary  
	
    const ZipEntry* binary_entry =
            mzFindZipEntry(zip, ASSUMED_UPDATE_BINARY_NAME);
    // 如果没找到update-binary，直接就出错了  
    
    if (binary_entry == NULL) {
        mzCloseZipArchive(zip);
        return INSTALL_CORRUPT;
    }
	
	// 将升级包中的update_binary解压到/tmp/update_binary  
	
    const char* binary = "/tmp/update_binary";
    unlink(binary);
    int fd = creat(binary, 0755);
    if (fd < 0) {
        mzCloseZipArchive(zip);
        LOGE("Can't make %s\n", binary);
        return INSTALL_ERROR;
    }
    bool ok = mzExtractZipEntryToFile(zip, binary_entry, fd);
    close(fd);
    mzCloseZipArchive(zip);

    if (!ok) {
        LOGE("Can't copy %s\n", ASSUMED_UPDATE_BINARY_NAME);
        return INSTALL_ERROR;
    }

    int pipefd[2];
    pipe(pipefd);

    // When executing the update binary contained in the package, the  
    
    // arguments passed are:  
    
    //  
    
    //   - the version number for this interface  
    
    //  
    
    //   - an fd to which the program can write in order to update the  
    
    //     progress bar.  The program can write single-line commands:  
    
    //  
    
    //        progress <frac> <secs>  
    
    //            fill up the next <frac> part of of the progress bar  
    
    //            over <secs> seconds.  If <secs> is zero, use  
    
    //            set_progress commands to manually control the  
    
    //            progress of this segment of the bar  
    
    //  
    
    //        set_progress <frac>  
    
    //            <frac> should be between 0.0 and 1.0; sets the  
    
    //            progress bar within the segment defined by the most  
    
    //            recent progress command.  
    
    //  
    
    //        firmware <"hboot"|"radio"> <filename>  
    
    //            arrange to install the contents of <filename> in the  
    
    //            given partition on reboot.  
    
    //  
    //            (API v2: <filename> may start with "PACKAGE:" to  
    
    //            indicate taking a file from the OTA package.)  
    
    //  
    
    //            (API v3: this command no longer exists.)  
    
    //  
    
    //        ui_print <string>  
    
    //            display <string> on the screen.  
    
    //  
    
    //   - the name of the package zip file.  
    
    //  

	// 后续在专门介绍updater篇章中会看到main函数第一步就是判断参数，如果不是5个就报错
	
    const char** args = (const char**)malloc(sizeof(char*) * 6);
    args[0] = binary;// update-binary程序
    
    args[1] = EXPAND(RECOVERY_API_VERSION);   // defined in Android.mk
    
    char* temp = (char*)malloc(10);
    sprintf(temp, "%d", pipefd[1]);
    args[2] = temp;// 管道通信，升级进度，升级执行中需要打印的信息等
    
    args[3] = (char*)path;// 升级包路径  
    
    args[4] = (char *)&system_type;// 系统类型
    
    args[5] = NULL;

    pid_t pid = fork();// fork子进程来跑update-binary
    
    if (pid == 0) {
        close(pipefd[0]);
        execv(binary, (char* const*)args);
        fprintf(stdout, "E:Can't run %s (%s)\n", binary, strerror(errno));
        _exit(-1);
    }
    close(pipefd[1]);

    *wipe_cache = 0;

    char buffer[1024];
    FILE* from_child = fdopen(pipefd[0], "r");
    while (fgets(buffer, sizeof(buffer), from_child) != NULL) {// 获取子进程传来的信息
    
        char* command = strtok(buffer, " \n");
        if (command == NULL) {
            continue;
        } else if (strcmp(command, "progress") == 0) {
            char* fraction_s = strtok(NULL, " \n");
            char* seconds_s = strtok(NULL, " \n");

            float fraction = strtof(fraction_s, NULL);
            int seconds = strtol(seconds_s, NULL, 10);

            ui->ShowProgress(fraction * (1-VERIFICATION_PROGRESS_FRACTION), seconds);
        } else if (strcmp(command, "set_progress") == 0) {
            char* fraction_s = strtok(NULL, " \n");
            float fraction = strtof(fraction_s, NULL);
            ui->SetProgress(fraction);
        } else if (strcmp(command, "ui_print") == 0) {
            char* str = strtok(NULL, "\n");
            if (str) {
                ui->Print("%s\n", str);
            } else {
                ui->Print("\n");
            }
            fflush(stdout);
        } else if (strcmp(command, "wipe_cache") == 0) {
            char* param = strtok(NULL, " \n");
            *wipe_cache = strtoul(param,0,0);
			fprintf(stderr,"wipe_cache: %s\n",param);
        } else if (strcmp(command, "clear_display") == 0) {
            ui->SetBackground(RecoveryUI::NONE);
        } else {
            LOGE("unknown command [%s]\n", command);
        }
    }
    fclose(from_child);

    int status;
    waitpid(pid, &status, 0);
    if (!WIFEXITED(status) || WEXITSTATUS(status) != 0) {
        LOGE("Error in %s\n(Status %d)\n", path, WEXITSTATUS(status));
        return INSTALL_ERROR;
    }

    return INSTALL_SUCCESS;
}
```





### 5 结束安装

#### 5.1 finish_recovery

```cpp
// clear the recovery command and prepare to boot a (hopefully working) system,

// copy our log file to cache as well (for the system to read), and

// record any intent we were asked to communicate back to the system.

// this function is idempotent: call it as many times as you like.

static void
finish_recovery(const char *send_intent) {
    // By this point, we're ready to return to the main system...
    
    // send_intent会被写入到/cache/recovery/intent，用于跟主系统交互，后续看看是否能够多利用这个参数来做升级结果的交流
    
    // INTENT_FILE = "/cache/recovery/intent"
    
    if (send_intent != NULL) {
        FILE *fp = fopen_path(INTENT_FILE, "w");
        if (fp == NULL) {
            LOGE("Can't open %s\n", INTENT_FILE);
        } else {
            fputs(send_intent, fp);
            check_and_fclose(fp, INTENT_FILE);
        }
    }

    // Save the locale to cache, so if recovery is next started up
    
    // without a --locale argument (eg, directly from the bootloader)
    
    // it will use the last-known locale.
    
    if (locale != NULL) {
        LOGI("Saving locale \"%s\"\n", locale);
        // LOCALE_FILE = "/cache/recovery/last_locale"
        
        // 用来记录这一次的语言，在main里面有从启动参数里读语言的，如果读不到就从这里读上一次的语言
        
        FILE* fp = fopen_path(LOCALE_FILE, "w");
        fwrite(locale, 1, strlen(locale), fp);
        fflush(fp);
        fsync(fileno(fp));
        check_and_fclose(fp, LOCALE_FILE);
    }

	// 日志拷贝，见5.2
	
    copy_logs();

    // Reset to normal system boot so recovery won't cycle indefinitely.
    
    // 将boot message设置为空，下次可以正常启动到主系统
    
    struct bootloader_message boot;
    memset(&boot, 0, sizeof(boot));
    set_bootloader_message(&boot);

    // Remove the command file, so recovery won't repeat indefinitely.
    
    // 删除/cache/recovery/command
    
    if (ensure_path_mounted(COMMAND_FILE) != 0 ||
        (unlink(COMMAND_FILE) && errno != ENOENT)) {
        LOGW("Can't unlink %s\n", COMMAND_FILE);
    }

	// 卸载cache
	
    ensure_path_unmounted(CACHE_ROOT);
    sync();  // For good measure.
    
}
```

#### 5.2 copy_log

这里区分下几种日志，首先是拷贝的日志来源，有两种，TEMPORARY_LOG_FILE是记录了升级的过程日志，TEMPORARY_INSTALL_FILE是记录了升级包路径和升级结果，在4.1中介绍过；然后是copy_log_file的第三个参数true表示追加，false表示不追加，也就是打开文件时用'a'还是'w'的区别；所以可以理解拷贝生成的日志文件的差别在于，LOG_FILE是一个完整的，记录了所以升级的日志，LAST_LOG_FILE记录了上一次升级的过程，LAST_INSTALL_FILE记录了上一次升级包和升级结果。  

```cpp
static void
copy_logs() {
    // Copy logs to cache so the system can find out what happened.
    
    // TEMPORARY_LOG_FILE = "/tmp/recovery.log"
    
    // TEMPORARY_INSTALL_FILE = "/tmp/last_install"
    
    // LOG_FILE = "/cache/recovery/log"
    
    // #define LAST_LOG_FILE "/cache/recovery/last_log"
    
    // LAST_INSTALL_FILE = "/cache/recovery/last_install"
    
    copy_log_file(TEMPORARY_LOG_FILE, LOG_FILE, true);
    copy_log_file(TEMPORARY_LOG_FILE, LAST_LOG_FILE, false);
    copy_log_file(TEMPORARY_INSTALL_FILE, LAST_INSTALL_FILE, false);
    chmod(LOG_FILE, 0600);
    chown(LOG_FILE, 1000, 1000);   // system user
    
    chmod(LAST_LOG_FILE, 0640);
    chmod(LAST_INSTALL_FILE, 0644);
    sync();
}
```

### 6 手动模式  

在1.1中提到了如果启动参数中没有其他命令的时候，会调用prompt_and_wait来等待用户操作，这个时候就能看到recovery的菜单界面，可以通过上下键来选择和home/power键来确认。这里大概介绍下这里的实现。    

#### 6.1 prompt_and_wait

```cpp
static void
prompt_and_wait(Device* device, int status) {
    const char* const* headers = prepend_title(device->GetMenuHeaders());

    for (;;) {
        finish_recovery(NULL);
        switch (status) {
            case INSTALL_SUCCESS:
            case INSTALL_NONE:
                ui->SetBackground(RecoveryUI::NO_COMMAND);
                break;

            case INSTALL_ERROR:
            case INSTALL_CORRUPT:
                ui->SetBackground(RecoveryUI::ERROR);
                break;
        }
        ui->SetProgressType(RecoveryUI::EMPTY);

		// 取到选择的菜单位置
		
        int chosen_item = get_menu_selection(headers, device->GetMenuItems(), 0, 0, device);

        // device-specific code may take some action here.  It may
        
        // return one of the core actions handled in the switch
        
        // statement below.
        
        
        // 对应成菜单项，这里是一个枚举类型
        
        chosen_item = device->InvokeMenuItem(chosen_item);

        int wipe_cache;
        switch (chosen_item) {
            case Device::REBOOT:
                return;

            case Device::WIPE_DATA:
                wipe_data(ui->IsTextVisible(), device);
                if (!ui->IsTextVisible()) return;
                break;

            case Device::WIPE_CACHE:
                ui->Print("\n-- Wiping cache...\n");
                erase_volume("/cache");
                ui->Print("Cache wipe complete.\n");
                if (!ui->IsTextVisible()) return;
                break;

            case Device::APPLY_EXT:
                status = update_directory(SDCARD_ROOT, SDCARD_ROOT, &wipe_cache, device);
                if (status == INSTALL_SUCCESS && wipe_cache) {
                    ui->Print("\n-- Wiping cache (at package request)...\n");
                    if (erase_volume("/cache")) {
                        ui->Print("Cache wipe failed.\n");
                    } else {
                        ui->Print("Cache wipe complete.\n");
                    }
                }
                if (status >= 0) {
                    if (status != INSTALL_SUCCESS) {
                        ui->SetBackground(RecoveryUI::ERROR);
                        ui->Print("Installation aborted.\n");
                    } else if (!ui->IsTextVisible()) {
                        return;  // reboot if logs aren't visible
                        
                    } else {
                        ui->Print("\nInstall from sdcard complete.\n");
                    }
                }
                break;

            case Device::APPLY_CACHE:
                // Don't unmount cache at the end of this.
                
                status = update_directory(CACHE_ROOT, NULL, &wipe_cache, device);
                if (status == INSTALL_SUCCESS && wipe_cache) {
                    ui->Print("\n-- Wiping cache (at package request)...\n");
                    if (erase_volume("/cache")) {
                        ui->Print("Cache wipe failed.\n");
                    } else {
                        ui->Print("Cache wipe complete.\n");
                    }
                }
                if (status >= 0) {
                    if (status != INSTALL_SUCCESS) {
                        ui->SetBackground(RecoveryUI::ERROR);
                        ui->Print("Installation aborted.\n");
                    } else if (!ui->IsTextVisible()) {
                        return;  // reboot if logs aren't visible
                        
                    } else {
                        ui->Print("\nInstall from cache complete.\n");
                    }
                }
                break;

            case Device::APPLY_ADB_SIDELOAD:
                status = apply_from_adb(ui, &wipe_cache, TEMPORARY_INSTALL_FILE);
                if (status >= 0) {
                    if (status != INSTALL_SUCCESS) {
                        ui->SetBackground(RecoveryUI::ERROR);
                        ui->Print("Installation aborted.\n");
                        copy_logs();
                    } else if (!ui->IsTextVisible()) {
                        return;  // reboot if logs aren't visible
                        
                    } else {
                        ui->Print("\nInstall from ADB complete.\n");
                    }
                }
                break;
        }
    }
}
```

