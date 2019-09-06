arch/arm/system-onesegment.ld 连接脚本中定义入口函数为_start
```
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
OUTPUT_ARCH(arm)

ENTRY(_start)
```

\_start函数定义位于bootable/bootloader/lk/arch/arm/crt0.S
```
.section ".text.boot"
.globl _start
_start:
    b   reset
    b   arm_undefined
    b   arm_syscall
    b   arm_prefetch_abort
    b   arm_data_abort
    b   arm_reserved
    b   arm_irq
    b   arm_fiq

......

最后调用kmain函数
	bl		kmain
	b		.
```

跳转的kmain函数

bootloader/lk/kernel/main.c
```
void kmain(void) __NO_RETURN __EXTERNALLY_VISIBLE;
void kmain(void)
{
	thread_t *thr;

	// get us into some sort of thread context
	thread_init_early();	// 创建各种线程

	// early arch stuff
	arch_early_init();	// 初始化MMU

	// do any super early platform initialization
	platform_early_init();	// 获取MODEM类型，设置系统时钟等

	// do any super early target initialization
	target_early_init();	// 初始化串口,用于调试

	dprintf(INFO, "welcome to lk\n\n");
	bs_set_timestamp(BS_BL_START);

	// deal with any static constructors
	dprintf(SPEW, "calling constructors\n");
	call_constructors();

	// bring up the kernel heap
	dprintf(SPEW, "initializing heap\n");
	heap_init();	// 创建栈的大小

	__stack_chk_guard_setup();

	// initialize the threading system
	dprintf(SPEW, "initializing threads\n");
	thread_init();

	// initialize the dpc system
	dprintf(SPEW, "initializing dpc\n");
	dpc_init();

	// initialize kernel timers
	dprintf(SPEW, "initializing timers\n");
	timer_init();		// 定时器初始化

#if (!ENABLE_NANDWRITE)
	// create a thread to complete system initialization
	dprintf(SPEW, "creating bootstrap completion thread\n");
	// 创建bootstrap2线程
	thr = thread_create("bootstrap2", &bootstrap2, NULL, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE);
	if (!thr)
	{
		panic("failed to create thread bootstrap2\n");
	}
	thread_resume(thr);			//启动线程


	// enable interrupts
	exit_critical_section();	// 使能中断

	// become the idle thread
	thread_become_idle();
#else
        bootstrap_nandwrite();
#endif
}
```

继续分析上面的bootstrap2线程
```
static int bootstrap2(void *arg)
{
	dprintf(SPEW, "top of bootstrap2()\n");

	arch_init();	// 啥也没干

	// XXX put this somewhere else
#if WITH_LIB_BIO
	bio_init();
#endif
#if WITH_LIB_FS
	fs_init();
#endif

	// initialize the rest of the platform
	dprintf(SPEW, "initializing platform\n");
	platform_init();	// 函数为空

	// initialize the target
	dprintf(SPEW, "initializing target\n");
	target_init();	// 设置EMMC，读取分区表

	dprintf(SPEW, "calling apps_init()\n");
	apps_init();

	return 0;
}
```

继续分析上面的apps_init函数

bootloader/lk/app/app.c
```
void apps_init(void)
{
	const struct app_descriptor *app;
	// __apps_start和__apps_end都是在链接脚本中定义的
	/* call all the init routines */
	for (app = &__apps_start; app != &__apps_end; app++) {
		if (app->init)
			app->init(app);
	}

	/* start any that want to start on boot */
	for (app = &__apps_start; app != &__apps_end; app++) {
		if (app->entry && (app->flags & APP_FLAG_DONT_START_ON_BOOT) == 0) {
			start_app(app);
		}
	}
}
```

arch/arm/system-onesegment.ld脚本中定义__app_start和__app_end：
```
		__apps_start = .;
		KEEP (*(.apps))
		__apps_end = .;
```

.app段的定义位于

bootloader/lk/include/app.h
```
#define APP_START(appname) struct app_descriptor _app_##appname __SECTION(".apps") = { .name = #appname,
#define APP_END };
```

所以只要用APP_START定义的函数都位于.app段中，

bootloader/lk/app/abort/aboot.c
```
APP_START(aboot)
	.init = aboot_init,
APP_END
```

aboot_init初始化
```
void aboot_init(const struct app_descriptor *app)
{
	unsigned reboot_mode = 0;
	int boot_err_type = 0;
	int boot_slot = INVALID;

	/* Initialise wdog to catch early lk crashes */
#if WDOG_SUPPORT
	msm_wdog_init();	//设置看门狗
#endif

	/* Setup page size information for nv storage */
	// 读取EMMC信息
	if (target_is_emmc_boot())
	{
		// EMMC存储
		page_size = mmc_page_size();
		page_mask = page_size - 1;
		mmc_blocksize = mmc_get_device_blocksize();
		mmc_blocksize_mask = mmc_blocksize - 1;
	}
	else
	{
		// flash存储
		page_size = flash_page_size();
		page_mask = page_size - 1;
	}
	ASSERT((MEMBASE + MEMSIZE) > MEMBASE);

    dprintf(INFO, "aboot_read_info_mmc: Start\n");
	aboot_read_info_mmc();
	dprintf(INFO, "aboot_read_info_mmc: Done\n");

	read_device_info(&device);	// 读取device_info
    check_device_lock_status();
	read_allow_oem_unlock(&device);

	/* Detect multi-slot support */
	if (partition_multislot_is_supported())
	{
		boot_slot = partition_find_active_slot();
		if (boot_slot == INVALID)
		{
			boot_into_fastboot = true;
			dprintf(INFO, "Active Slot: (INVALID)\n");
		}
		else
		{
			/* Setting the state of system to boot active slot */
			partition_mark_active_slot(boot_slot);
			dprintf(INFO, "Active Slot: (%s)\n", SUFFIX_SLOT(boot_slot));
		}
	}

	/* Display splash screen if enabled */
#if DISPLAY_SPLASH_SCREEN
#if NO_ALARM_DISPLAY
	if (!check_alarm_boot()) {	// 判断开机原因是否是闹钟导致的
#endif
		dprintf(SPEW, "Display Init: Start\n");
#if DISPLAY_HDMI_PRIMARY
	if (!strlen(device.display_panel))
		strlcpy(device.display_panel, DISPLAY_PANEL_HDMI,
			sizeof(device.display_panel));
#endif
#if ENABLE_WBC
		/* Wait if the display shutdown is in progress */
		while(pm_app_display_shutdown_in_prgs());
		if (!pm_appsbl_display_init_done())
			target_display_init(device.display_panel);
		else
			display_image_on_screen();
#else
		target_display_init(device.display_panel);
#endif
		dprintf(SPEW, "Display Init: Done\n");
#if NO_ALARM_DISPLAY
	}
#endif
#endif
	check_eeprom();

	/*
	 * Check power off reason if user force reset,
	 * if yes phone will do normal boot.
	 */
	if (is_user_force_reset())
		goto normal_boot;

	/* Check if we should do something other than booting up */
	// 音量上和下键按下，进入EMERGENCY_DLOAD模式
	if (keys_get_state(KEY_VOLUMEUP) && keys_get_state(KEY_VOLUMEDOWN))
	{
		dprintf(ALWAYS,"dload mode key sequence detected\n");
		reboot_device(EMERGENCY_DLOAD);
		dprintf(CRITICAL,"Failed to reboot into dload mode\n");

		boot_into_fastboot = true;
	}
	if (!boot_into_fastboot)
	{
		// home键或者音量上键按下，进入recovery
		if (keys_get_state(KEY_HOME) || keys_get_state(KEY_VOLUMEUP))
            if (mdm_boot_ctl_disable_fastboot_recovery == false)
			boot_into_recovery = 1;
		// back键或者音量下键按下，进入fastboot
		if (!boot_into_recovery &&
			(keys_get_state(KEY_BACK) || keys_get_state(KEY_VOLUMEDOWN)))
			boot_into_fastboot = true;
	}
	#if NO_KEYPAD_DRIVER
	if (fastboot_trigger())
		boot_into_fastboot = true;
	#endif

#if USE_PON_REBOOT_REG
	reboot_mode = check_hard_reboot_mode();
#else
	reboot_mode = check_reboot_mode();
#endif
	if (reboot_mode == RECOVERY_MODE)
	{
		boot_into_recovery = 1;
	}
	else if(reboot_mode == FASTBOOT_MODE)
	{
		boot_into_fastboot = true;
	}
	else if(reboot_mode == ALARM_BOOT)
	{
		boot_reason_alarm = true;
	}
#if VERIFIED_BOOT
	else if (VB_V2 == target_get_vb_version())
	{
		if (reboot_mode == DM_VERITY_ENFORCING)
		{
			device.verity_mode = 1;
			write_device_info(&device);
		}
#if ENABLE_VB_ATTEST
		else if (reboot_mode == DM_VERITY_EIO)
#else
		else if (reboot_mode == DM_VERITY_LOGGING)
#endif
		{
			device.verity_mode = 0;
			write_device_info(&device);
		}
		else if (reboot_mode == DM_VERITY_KEYSCLEAR)
		{
			if(send_delete_keys_to_tz())
				ASSERT(0);
		}
	}
#endif

normal_boot:
    if (mdm_boot_ctl_disable_fastboot_recovery == true) {
		boot_into_fastboot = false;
	}
	if (!boot_into_fastboot)
	{
		if (target_is_emmc_boot())
		{
			if(emmc_recovery_init())
				dprintf(ALWAYS,"error in emmc_recovery_init\n");
			if(target_use_signed_kernel())
			{
				if((device.is_unlocked) || (device.is_tampered))
				{
				#ifdef TZ_TAMPER_FUSE
					set_tamper_fuse_cmd();
				#endif
				#if USE_PCOM_SECBOOT
					set_tamper_flag(device.is_tampered);
				#endif
				}
			}

retry_boot:
			/* Trying to boot active partition */
			if (partition_multislot_is_supported())
			{
				boot_slot = partition_find_boot_slot();
				partition_mark_active_slot(boot_slot);
				if (boot_slot == INVALID)
					goto fastboot;
			}

			boot_err_type = boot_linux_from_mmc();		// 从MMC启动
			switch (boot_err_type)
			{
				case ERR_INVALID_PAGE_SIZE:
				case ERR_DT_PARSE:
				case ERR_ABOOT_ADDR_OVERLAP:
				case ERR_INVALID_BOOT_MAGIC:
					if(partition_multislot_is_supported())
					{
						/*
						 * Deactivate current slot, as it failed to
						 * boot, and retry next slot.
						 */
						partition_deactivate_slot(boot_slot);
						goto retry_boot;
					}
					else
						break;
				default:
					break;
				/* going to fastboot menu */
			}
		}
		else
		{
			recovery_init();
	#if USE_PCOM_SECBOOT
		if((device.is_unlocked) || (device.is_tampered))
			set_tamper_flag(device.is_tampered);
	#endif
			boot_linux_from_flash();		// 从FLASH启动
		}
		dprintf(CRITICAL, "ERROR: Could not do normal boot. Reverting "
			"to fastboot mode.\n");
	}

fastboot:
	/* We are here means regular boot did not happen. Start fastboot. */
	/* register aboot specific fastboot commands */
	aboot_fastboot_register_commands();

	/* dump partition table for debug info */
	partition_dump();

	/* initialize and start fastboot */
	fastboot_init(target_get_scratch_address(), target_get_max_flash_size());

#if DISPLAY_SPLASH_SCREEN
        display_fastboot_on_screen();	// 显示fastboot界面
#endif

#if FBCON_DISPLAY_MSG
	display_fastboot_menu();	// 显示fastboot选项
#endif
}
```