需要在fastboot里面添加功能用于保存，记录一下fastboot显示的过程。

android O新添加了选项，如下
`platform/msm_shared/rules.mk`
```
ifeq ($(ENABLE_FBCON_DISPLAY_MSG),1)
OBJS += \
    $(LOCAL_DIR)/menu_keys_detect.o \
    $(LOCAL_DIR)/display_menu.o
endif
```
要显示fastboot的选项，需要打开ENABLE_FBCON_DISPLAY_MSG， FBCON_DISPLAY_MSG。
`project/msm8953.mk`
```
ifeq ($(VERIFIED_BOOT),1)
ENABLE_SECAPP_LOADER := 1
ENABLE_RPMB_SUPPORT := 1
ifneq (,$(findstring DISPLAY_SPLASH_SCREEN,$(DEFINES)))
#enable fbcon display menu
ENABLE_FBCON_DISPLAY_MSG := 1				## 这个打开了才会显示fastboot的选项框
endif
endif

ifeq ($(ENABLE_FBCON_DISPLAY_MSG),1)
DEFINES += FBCON_DISPLAY_MSG=1                    ## 代码中会用到
endif
```


app/aboot/aboot.c
```
void aboot_init()
{

fastboot:
    /* We are here means regular boot did not happen. Start fastboot. */

    /* register aboot specific fastboot commands */
    aboot_fastboot_register_commands();

    /* dump partition table for debug info */
    partition_dump();

    /* initialize and start fastboot */
    fastboot_init(target_get_scratch_address(), target_get_max_flash_size());

#if DISPLAY_SPLASH_SCREEN
        display_fastboot_on_screen();
#endif

#if FBCON_DISPLAY_MSG
    display_fastboot_menu();
#endif

}
```

显示有两种情况，display_fastboot_on_screen()显示界面就是一个空白的界面，中间显示一个很小的"Fastboot Mode".

dev/fbcon/fbcon.c
```
void display_fastboot_on_screen(void)
{
       unsigned count = config->width * config->height;
    char *pixels;
    char *draw_text = "Fastboot Mode";
    char *c; 
    unsigned char_count, loop = 0, bpp;

    bpp = (config->bpp) / 8;
    // Draw White Background
       memset(config->base, RGB565_WHITE, count * bpp);

    // Draw
    char_count = strlen(draw_text);

    pixels = config->base;
       pixels += ((config->height - FONT_HEIGHT) / 2) * config->width * bpp;
       pixels += ((config->width - (FONT_WIDTH + 1)*char_count) / 2) * bpp;

    c = draw_text;
    while(loop < char_count)
    {   
        fbcon_drawglyph_fastboot(pixels, config->width, bpp, font5x12 + (*c - 32) * 2); 
        loop++;
        c++;
        pixels += (FONT_WIDTH + 1) * bpp;
    }   
    arch_clean_invalidate_cache_range((addr_t) config->base, (count * bpp));

    fbcon_flush();

}
```

platform/msm_shared/display_menu.c 
```
//fastboot显示出来的选项
static char *fastboot_option_menu[] = {
        [0] = "START\n",
        [1] = "Restart bootloader\n",
        [2] = "Recovery mode\n",
        [3] = "Power off\n",
        [4] = "Boot to FFBM\n"
		}; 

void display_fastboot_menu()
{
    struct select_msg_info *fastboot_menu_msg_info;
    fastboot_menu_msg_info = &msg_info;

    set_message_factor();

    msg_lock_init();
    mutex_acquire(&fastboot_menu_msg_info->msg_lock);

    /* There are 4 pages for fastboot menu:
     * Page: Start/Fastboot/Recovery/Poweroff
     * The menu is switched base on the option index
     * Initialize the option index and last_msg_type
     */
    fastboot_menu_msg_info->info.option_index = 0;
    fastboot_menu_msg_info->last_msg_type =
        fastboot_menu_msg_info->info.msg_type;

    display_fastboot_menu_renew(fastboot_menu_msg_info);		// bootloader显示
    mutex_release(&fastboot_menu_msg_info->msg_lock);

    dprintf(INFO, "creating fastboot menu keys detect thread\n");
    display_menu_thread_start(fastboot_menu_msg_info);			// 显示线程，检测按键按下，对应的显示
}

// fastboot显示的文本，每次都会更新
void display_fastboot_menu_renew(struct select_msg_info *fastboot_msg_info)
{
    int len;
    int msg_type = FBCON_COMMON_MSG;
    char msg_buf[64];
    char msg[128];
    device_info device;
    /* The fastboot menu is switched base on the option index
     * So it's need to store the index for the menu switching
     */
    uint32_t option_index = fastboot_msg_info->info.option_index;

    fbcon_clear();
    memset(&fastboot_msg_info->info, 0, sizeof(struct menu_info));

    len = ARRAY_SIZE(fastboot_option_menu);
	// 不同选项对应的字体颜色
    switch(option_index) {
        case 0:
            msg_type = FBCON_GREEN_MSG;					
            break;
        case 1:
        case 2:
            msg_type = FBCON_RED_MSG;
            break;
        case 3:
        case 4:
            msg_type = FBCON_COMMON_MSG;
            break;
    }
	
	fbcon_draw_line(msg_type);
    display_fbcon_menu_message(fastboot_option_menu[option_index],
        msg_type, big_factor);
    fbcon_draw_line(msg_type);				// 显示文字
    display_fbcon_menu_message("\n\nPress volume key to select, and "\
        "press power key to select\n\n", FBCON_COMMON_MSG, common_factor);

    display_fbcon_menu_message("FASTBOOT MODE\n", FBCON_RED_MSG, common_factor);

    get_product_name((unsigned char *) msg_buf);
    snprintf(msg, sizeof(msg), "PRODUCT_NAME - %s\n", msg_buf);
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);

    memset(msg_buf, 0, sizeof(msg_buf));
    smem_get_hw_platform_name((unsigned char *) msg_buf, sizeof(msg_buf));
    snprintf(msg, sizeof(msg), "VARIANT - %s %s\n",
        msg_buf, target_is_emmc_boot()? "eMMC":"UFS");
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);

    memset(msg_buf, 0, sizeof(msg_buf));
    get_bootloader_version((unsigned char *) msg_buf);
    snprintf(msg, sizeof(msg), "BOOTLOADER VERSION - %s\n",
        msg_buf);
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);

    memset(msg_buf, 0, sizeof(msg_buf));
    get_baseband_version((unsigned char *) msg_buf);
    snprintf(msg, sizeof(msg), "BASEBAND VERSION - %s\n",
        msg_buf);
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);

    memset(msg_buf, 0, sizeof(msg_buf));
    target_serialno((unsigned char *) msg_buf);
    snprintf(msg, sizeof(msg), "SERIAL NUMBER - %s\n", msg_buf);
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);
	
	    snprintf(msg, sizeof(msg), "SECURE BOOT - %s\n",
        is_secure_boot_enable()? "enabled":"disabled");
    display_fbcon_menu_message(msg, FBCON_COMMON_MSG, common_factor);

    snprintf(msg, sizeof(msg), "DEVICE STATE - %s\n",
        is_device_locked()? "locked":"unlocked");
    display_fbcon_menu_message(msg, FBCON_RED_MSG, common_factor);
 
    fastboot_msg_info->info.msg_type = DISPLAY_MENU_FASTBOOT;
    fastboot_msg_info->info.option_num = len;
    fastboot_msg_info->info.option_index = option_index;
}
```

按键线程
platform/msm_shared/menu_keys_detect.c
```
int select_msg_keys_detect(void *param) {
    struct select_msg_info *msg_info = (struct select_msg_info*)param;

    msg_lock_init();
    keys_detect_init();
    while(1) {
        /* 1: update select option's index, default it is the total option number
         *  volume up: index decrease, the option will scroll up from
         *  the bottom to top if the key is pressed firstly.
         *  eg: 5->4->3->2->1->0
         *  volume down: index increase, the option will scroll down from
         *  the bottom to top if the key is pressed firstly.
         *  eg: 5->0
         * 2: update device's status via select option's index
         */
        if (is_key_pressed(VOLUME_UP)) {			
            mutex_acquire(&msg_info->msg_lock);
            menu_pages_action[msg_info->info.msg_type].up_action_func(msg_info);			
            mutex_release(&msg_info->msg_lock);
        } else if (is_key_pressed(VOLUME_DOWN)) {	// VOLUME_DOWN
            mutex_acquire(&msg_info->msg_lock);
            menu_pages_action[msg_info->info.msg_type].down_action_func(msg_info);
            mutex_release(&msg_info->msg_lock);
        } else if (is_key_pressed(POWER_KEY)) {
            mutex_acquire(&msg_info->msg_lock);
            menu_pages_action[msg_info->info.msg_type].enter_action_func(msg_info);
            mutex_release(&msg_info->msg_lock);
        }   
		        mutex_acquire(&msg_info->msg_lock);
        /* Never time out if the timeout_time is 0 */
        if(msg_info->info.timeout_time) {
            if ((current_time() - before_time) > msg_info->info.timeout_time)
                msg_info->info.is_exit = true;
        }

        if (msg_info->info.is_exit) {
            msg_info->info.rel_exit = true;
            mutex_release(&msg_info->msg_lock);
            break;
        }
        mutex_release(&msg_info->msg_lock);
        thread_sleep(KEY_DETECT_FREQUENCY);
    }

    return 0;
}
``` 

对应的结构体中的函数指针如下：	
```
static struct pages_action menu_pages_action[] = {
    [DISPLAY_MENU_UNLOCK] = {
        menu_volume_up_func,
        menu_volume_down_func,
        power_key_func,
    },
    [DISPLAY_MENU_UNLOCK_CRITICAL] = {
        menu_volume_up_func,
        menu_volume_down_func,
        power_key_func,
    },
    [DISPLAY_MENU_YELLOW] = {
        boot_warning_volume_keys_func,
        boot_warning_volume_keys_func,
        power_key_func,
    },
    [DISPLAY_MENU_ORANGE] = {
        boot_warning_volume_keys_func,
        boot_warning_volume_keys_func,
        power_key_func,
    },
    [DISPLAY_MENU_RED] = {
        boot_warning_volume_keys_func,
        boot_warning_volume_keys_func,
        power_key_func,
    },
    [DISPLAY_MENU_LOGGING] = {
        boot_warning_volume_keys_func,
        boot_warning_volume_keys_func,
        power_key_func,
    },
    [DISPLAY_MENU_EIO] = {
        boot_warning_volume_keys_func,
        boot_warning_volume_keys_func,
        power_key_func,
}
```		
		
分析其中的power按键功能	
```	
static void power_key_func(struct select_msg_info* msg_info)
{
    int reason = -1;
    
    switch (msg_info->info.msg_type) {
        case DISPLAY_MENU_YELLOW:
        case DISPLAY_MENU_ORANGE:
        case DISPLAY_MENU_RED:
        case DISPLAY_MENU_LOGGING:
            reason = CONTINUE;
            break; 
        case DISPLAY_MENU_EIO:
            pwr_key_is_pressed = true;
            reason = CONTINUE; 
            break; 
        case DISPLAY_MENU_MORE_OPTION:
            if(msg_info->info.option_index < ARRAY_SIZE(verify_index_action))
                reason = verify_index_action[msg_info->info.option_index];
            break;
        case DISPLAY_MENU_UNLOCK:
        case DISPLAY_MENU_UNLOCK_CRITICAL:
            if(msg_info->info.option_index < ARRAY_SIZE(unlock_index_action))
                reason = unlock_index_action[msg_info->info.option_index];
            break;
        case DISPLAY_MENU_FASTBOOT:
            if(msg_info->info.option_index < ARRAY_SIZE(fastboot_index_action))
                reason = fastboot_index_action[msg_info->info.option_index];
            break;
        default:
            dprintf(CRITICAL,"Unsupported menu type\n");
            break;
    }       
    
    if (reason != -1) {
        update_device_status(msg_info, reason);					// 更新显示状态
		}
		    }   
} 


static void update_device_status(struct select_msg_info* msg_info, int reason)
{
    char ffbm_page_buffer[FFBM_MODE_BUF_SIZE];
    fbcon_clear();
    struct device_info device;
    switch (reason) {
        case RECOVER:			// 根据选项进行设置
            if (msg_info->info.msg_type == DISPLAY_MENU_UNLOCK) {
                set_device_unlock_value(UNLOCK, TRUE);
            } else if (msg_info->info.msg_type == DISPLAY_MENU_UNLOCK_CRITICAL) {
                set_device_unlock_value(UNLOCK_CRITICAL, TRUE);
            }

            if (msg_info->info.msg_type == DISPLAY_MENU_UNLOCK ||
                msg_info->info.msg_type == DISPLAY_MENU_UNLOCK_CRITICAL) {
                /* wipe data */
                struct recovery_message msg;

                memset(&msg, 0, sizeof(msg));
                snprintf(msg.recovery, sizeof(msg.recovery), "recovery\n--wipe_data");
                write_misc(0, &msg, sizeof(msg));
            }
            reboot_device(RECOVERY_MODE);
            break;
        case RESTART:
            reboot_device(0);
            break;
        case POWEROFF:
            shutdown_device();
            break;
        case FASTBOOT:
            reboot_device(FASTBOOT_MODE);
            break;
        case CONTINUE:
            display_image_on_screen();
			           /* Continue boot, no need to detect the keys'status */
            msg_info->info.is_exit = true;
            break;
        case BACK:					
            display_bootverify_menu_renew(msg_info, msg_info->last_msg_type);
            before_time = current_time();

            break;
        case FFBM:
            memset(&ffbm_page_buffer, 0, sizeof(ffbm_page_buffer));
            snprintf(ffbm_page_buffer, sizeof(ffbm_page_buffer), "ffbm-00");
            write_misc(0, ffbm_page_buffer, sizeof(ffbm_page_buffer));

            reboot_device(0);
            break;
}

enum device_select_option {
    POWEROFF = 0,
    RESTART,
    RECOVER,
    FASTBOOT,
    BACK,

    CONTINUE,
    FFBM,
};
```