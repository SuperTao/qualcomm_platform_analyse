最近需要设置一个只读的属性值，采用的方法是在cmdline中添加，然后在init进程中解读。

记录一下代码跟踪过程。

lk/app/aboot/aboot.c
```
static const char *usb_sn_cmdline = " androidboot.serialno=";
......
unsigned char *update_cmdline(const char * cmdline)
{
    int cmdline_len = 0;
    int have_cmdline = 0;
    unsigned char *cmdline_final = NULL;
    int pause_at_bootup = 0;
    bool warm_boot = false;
    bool gpt_exists = partition_gpt_exists();
    int have_target_boot_params = 0;
    char *boot_dev_buf = NULL;
        bool is_mdtp_activated = 0;
    int current_active_slot = INVALID;
    int system_ptn_index = -1;
    unsigned int lun = 0;
    char lun_char_base = 'a';
    int syspath_buflen = strlen(sys_path) + sizeof(int) + 1; /*allocate buflen for largest possible string*/
    char syspath_buf[syspath_buflen];
#if VERIFIED_BOOT
    uint32_t boot_state = RED;
#endif

#if USE_LE_SYSTEMD
    is_systemd_present=true;
#endif

#if VERIFIED_BOOT
    if (VB_V2 == target_get_vb_version())
    {
            boot_state = boot_verify_get_state();
    }
#endif

#ifdef MDTP_SUPPORT
    mdtp_activated(&is_mdtp_activated);
#endif /* MDTP_SUPPORT */

    if (cmdline && cmdline[0]) {
        cmdline_len = strlen(cmdline);
        have_cmdline = 1;
    }
    if (target_is_emmc_boot()) {
        cmdline_len += strlen(emmc_cmdline);
#if USE_BOOTDEV_CMDLINE
        boot_dev_buf = (char *) malloc(sizeof(char) * BOOT_DEV_MAX_LEN);
        ASSERT(boot_dev_buf);
        platform_boot_dev_cmdline(boot_dev_buf);
        cmdline_len += strlen(boot_dev_buf);
#endif
    }

    cmdline_len += strlen(usb_sn_cmdline);			// 计算cmdline_len长度
    cmdline_len += strlen(sn_buf);					// 将usb_sn的值也添加到usb_sn_cmdline后面
	
	
	......
	
	
	    if (cmdline_len > 0) {
        const char *src;
        unsigned char *dst;

        cmdline_final = (unsigned char*) malloc((cmdline_len + 4) & (~3));			// 申请空间保存cmdline
        ASSERT(cmdline_final != NULL);
        memset((void *)cmdline_final, 0, sizeof(*cmdline_final));
        dst = cmdline_final;

        /* Save start ptr for debug print */
        if (have_cmdline) {
            src = cmdline;
            while ((*dst++ = *src++));
        }
		
		······
		src = usb_sn_cmdline;				//将usb_sn_cmdline保存到cmdline
        if (have_cmdline) --dst;
        have_cmdline = 1;
        while ((*dst++ = *src++));
        src = sn_buf;
		
        if (target_is_emmc_boot()) {
            src = emmc_cmdline;
            if (have_cmdline) --dst;
            have_cmdline = 1;
            while ((*dst++ = *src++));
#if USE_BOOTDEV_CMDLINE
            src = boot_dev_buf;
            if (have_cmdline) --dst;
            while ((*dst++ = *src++));
#endif
        }
		
		    if (boot_dev_buf)
        free(boot_dev_buf);

    if (cmdline_final)
        dprintf(INFO, "cmdline: %s\n", cmdline_final);
    else
        dprintf(INFO, "cmdline is NULL\n");
    return cmdline_final;
}
```

系统起来后自动解读设置的属性值，以上面的属性值为例“androidboot.serialno”

init进程解读：

system/core/init/init.cpp

int main() ---> export_kernel_boot_props() 
```
static void export_kernel_boot_props() {
    struct {
        const char *src_prop;
        const char *dst_prop;
        const char *default_value;
    } prop_map[] = {
        { "ro.boot.serialno",   "ro.serialno",   "", },
        { "ro.boot.mode",       "ro.bootmode",   "unknown", },
        { "ro.boot.baseband",   "ro.baseband",   "unknown", },
        { "ro.boot.bootloader", "ro.bootloader", "unknown", },
        { "ro.boot.hardware",   "ro.hardware",   "unknown", },
        { "ro.boot.revision",   "ro.revision",   "0", },
    };   
    for (size_t i = 0; i < arraysize(prop_map); i++) {
        std::string value = GetProperty(prop_map[i].src_prop, ""); 			// 读取属性值
        property_set(prop_map[i].dst_prop, (!value.empty()) ? value : prop_map[i].default_value);		// 设置属性值
    }    
}
```

默认cmdline位于`device/qcom/msm8953_64/BoardConfig.mk`,如下：
```
BOARD_KERNEL_CMDLINE := console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom msm_rtb.filter=0x237 ehci-hcd.park=3 lpm_levels.sleep_disabled=1 androidboot.bootdevice=7824900.sdhci earlycon=msm_hsl_uart,0x78af000
```