> The note is based on Linux version 5.15.43 in OpenBMC.

## Index

- [Introduction](#introduction)
- [Device File](#device-file)
- [Device Tree](#device-tree)
- [System Startup](#system-startup)
- [Cheat Sheet](#cheat-sheet)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

(TBD)

## <a name="device-file"></a> Device File

```
struct inode {
    umode_t         i_mode;                     // file type: char or block
    dev_t           i_rdev;                     // dev#
    union {
        const struct file_operations    *i_fop; // file operations
        void (*free_inode)(struct inode *);
    };
    union {
        struct pipe_inode_info  *i_pipe;
        struct cdev     *i_cdev;
        char            *i_link;
        unsigned        i_dir_seq;
    };
}
```

```
const struct file_operations def_chr_fops = {
    .open = chrdev_open,
    .llseek = noop_llseek,
};
```

```
+-------------+                                         
| chrdev_open | :
+---|---------+                                         
    |                                                   
    |--> if inode doesn't know where cdev is            
    |                                                   
    |------> serach cdev in 'cdev_map' based on dev#    
    |                                                   
    |------> save cdev addr in inode                    
    |                                                   
    |--> get file ops from cdev                         
    |                                                   
    |    +--------------+                               
    |--> | replace_fops | replace fops of file to cdev's
    |    +--------------+                               
    |                                                   
    +--> call ->open, e.g.,                             
         +--------------+                               
         | mtdchar_open |                               
         +--------------+                               
```

```
const struct file_operations def_blk_fops = { 
    .open       = blkdev_open,
    .release    = blkdev_close,
    .llseek     = blkdev_llseek,
    .read_iter  = blkdev_read_iter,
    .write_iter = blkdev_write_iter,
    .iopoll     = blkdev_iopoll,
    .mmap       = generic_file_mmap,
    .fsync      = blkdev_fsync,
    .unlocked_ioctl = block_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl   = compat_blkdev_ioctl,
#endif
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .fallocate  = blkdev_fallocate,
};
```

```
struct cdev {
    struct kobject kobj;
    struct module *owner;               // points to module if there's any
    const struct file_operations *ops;  // file operations
    struct list_head list;              // all inodes that represent the cdev are on the list
    dev_t dev;                          // dev#
    unsigned int count;                 // range of minor
} __randomize_layout;
```

## <a name="device-tree"></a> Device Tree

Device tree source (DTS) is the text file describing the list of devices of the SoC and gets compiled into Device tree blob (DTB). 
The kernel will know where to find them in memory for parsing and sequentially registering devices on the list. 
There are also DTS files for inclusion only (DTSI), which helps avoid content duplication.

```
arch/arm/boot/dts/aspeed-g5.dtsi            : base; list of supported components in processor 'apeed'
arch/arm/boot/dts/openbmc-flash-layout.dtsi : partition layout in SPI flash
arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts: overwrite; specially tailored for machine 'romulus'
```

DTS files arrange devices in a hierarchy structure, and each parent can have a few properties and child nodes. 
Property **#address-cells** and **size-cells** in parent node specify how to interpret property **reg** of children. 
The most common node properties are the register base and hardware interrupt number, and the kernel will regard them as device resources when parsing.

```
    ahb {
        compatible = "simple-bus";
        #address-cells = <1>;   // 1 dword (4 bytes)
        #size-cells = <1>;      // 1 dword (4 bytes)
        ranges;                 // so we interpret child node's reg as < 1 dword addr + 1 dword size, and so on >

        fmc: spi@1e620000 {
            reg = < 0x1e620000 0xc4             // a. 1 dword addr (0x1e620000) + 1 dword size (0xc4)
                0x20000000 0x10000000 >;        // b. 1 dword addr (0x20000000) + 1 dword size (0x10000000)
            #address-cells = <1>;              
            #size-cells = <0>; 
            compatible = "aspeed,ast2500-fmc";
            clocks = <&syscon ASPEED_CLK_AHB>;
            status = "disabled";
            interrupts = <19>;                  // c. hardware irq
```

The kernel will add resources for the above item a, b, and c. So the driver can later access them by the usage:

```
platform_get_resource(pdev, IORESOURCE_MEM, 0); // access the MEM type resource with index = 0
platform_get_resource(pdev, IORESOURCE_MEM, 1); // access the MEM type resource with index = 1
platform_get_resource(pdev, IORESOURCE_IRQ, 0); // access the IRQ type resource with index = 0
```

If many DTS and DTSI files are involved in the DTB complication, it might be challenging to know whether a device is enabled or disabled eventually. 
Utility **dtc** can construct the DTS from DTB and give us a quick inspection without booting up the target system.

```
dtc -I dtb -O dts arch/arm/boot/dts/aspeed-bmc-opp-romulus.dtb  # construct dts from dtb
dtc -I fs -O dts /sys/firmware/devicetree/base                  # construct dts from filesystem (not that useful to me)
```

- List of devices added from DTB in function **of_platform_default_populate_init**

```
[    0.180038] device: 'ahb': device_add
[    0.198481] device: '1e620000.spi': device_add
[    0.203438] device: '1e630000.spi': device_add
[    0.204606] device: '1e6c2000.copro-interrupt-controller': device_add
[    0.205406] device: '1e660000.ethernet': device_add
[    0.206037] device: '1e6a0000.usb-vhub': device_add
[    0.206468] device: 'ahb:apb': device_add
[    0.207725] device: '1e6e2000.syscon': device_add
[    0.209195] device: '1e6e207c.silicon-id': device_add
[    0.209675] device: '1e6e2080.pinctrl': device_add
[    0.210556] device: 'platform:1e6e2080.pinctrl--platform:1e6a0000.usb-vhub': device_add
[    0.211799] device: 'platform:1e6e2080.pinctrl--platform:1e660000.ethernet': device_add
[    0.212234] device: 'platform:1e6e2080.pinctrl--platform:1e630000.spi': device_add
[    0.219286] device: '1e6e2078.hwrng': device_add
[    0.220000] device: '1e6e6000.display': device_add
[    0.220521] device: '1e6e9000.adc': device_add
[    0.221058] device: '1e700000.video': device_add
[    0.221494] device: '1e720000.sram': device_add
[    0.224925] device: '1e780000.gpio': device_add
[    0.226185] device: '1e782000.timer': device_add
[    0.226842] device: '1e783000.serial': device_add
[    0.227221] device: 'platform:1e6e2080.pinctrl--platform:1e783000.serial': device_add
[    0.228314] device: '1e784000.serial': device_add
[    0.228824] device: '1e785000.watchdog': device_add
[    0.230142] device: '1e785020.watchdog': device_add
[    0.230842] device: '1e786000.pwm-tacho-controller': device_add
[    0.231402] device: 'platform:1e6e2080.pinctrl--platform:1e786000.pwm-tacho-controller': device_add
[    0.233039] device: '1e787000.serial': device_add
[    0.233714] device: '1e789000.lpc': device_add
[    0.234579] device: '1e789080.lpc-ctrl': device_add
[    0.235052] device: '1e789098.reset-controller': device_add
[    0.235502] device: 'platform:1e789098.reset-controller--platform:1e783000.serial': device_add
[    0.236189] device: '1e7890a0.lhc': device_add
[    0.236727] device: '1e789140.ibt': device_add
[    0.237134] device: 'ahb:apb:bus@1e78a000': device_add
[    0.237557] device: 'platform:1e6e2080.pinctrl--platform:ahb:apb:bus@1e78a000': device_add
[    0.238347] device: '1e78a080.i2c-bus': device_add
[    0.238912] device: '1e78a0c0.i2c-bus': device_add
[    0.239235] device: 'platform:1e6e2080.pinctrl--platform:1e78a0c0.i2c-bus': device_add
[    0.240138] device: '1e78a100.i2c-bus': device_add
[    0.240517] device: 'platform:1e6e2080.pinctrl--platform:1e78a100.i2c-bus': device_add
[    0.241372] device: '1e78a140.i2c-bus': device_add
[    0.241701] device: 'platform:1e6e2080.pinctrl--platform:1e78a140.i2c-bus': device_add
[    0.242752] device: '1e78a180.i2c-bus': device_add
[    0.243082] device: 'platform:1e6e2080.pinctrl--platform:1e78a180.i2c-bus': device_add
[    0.244224] device: '1e78a1c0.i2c-bus': device_add
[    0.244661] device: 'platform:1e6e2080.pinctrl--platform:1e78a1c0.i2c-bus': device_add
[    0.245651] device: '1e78a300.i2c-bus': device_add
[    0.246012] device: 'platform:1e6e2080.pinctrl--platform:1e78a300.i2c-bus': device_add
[    0.246913] device: '1e78a340.i2c-bus': device_add
[    0.247237] device: 'platform:1e6e2080.pinctrl--platform:1e78a340.i2c-bus': device_add
[    0.248209] device: '1e78a380.i2c-bus': device_add
[    0.248543] device: 'platform:1e6e2080.pinctrl--platform:1e78a380.i2c-bus': device_add
[    0.249513] device: '1e78a3c0.i2c-bus': device_add
[    0.249877] device: 'platform:1e6e2080.pinctrl--platform:1e78a3c0.i2c-bus': device_add
[    0.251000] device: '1e78a400.i2c-bus': device_add
[    0.251382] device: 'platform:1e6e2080.pinctrl--platform:1e78a400.i2c-bus': device_add
[    0.252276] device: '1e78a440.i2c-bus': device_add
[    0.252613] device: 'platform:1e6e2080.pinctrl--platform:1e78a440.i2c-bus': device_add
[    0.253403] device: 'leds': device_add
[    0.254278] device: 'platform:1e780000.gpio--platform:leds': device_add
[    0.254838] device: 'gpio-fsi': device_add
[    0.256303] device: 'platform:1e780000.gpio--platform:gpio-fsi': device_add
[    0.256757] device: 'gpio-keys': device_add
[    0.257201] device: 'platform:1e780000.gpio--platform:gpio-keys': device_add
[    0.257606] device: 'iio-hwmon-battery': device_add
```

## <a name="system-startup"></a> System Startup

Before introducing driver and device, let's talk about the **bus** first. 
We can regard the **bus** as a collection of devices and drivers, and there are numerous buses in the system.

```
                                        +-------+        +-------+
                                   ┌──► | dev A | ◄────► | dev B |
                 subsys_private    │    +-------+        +-------+
bus_type        +---------------+  │
  +---+         | klist_devices ◄──┘
  | p ────────► |               |
  +---+         | klist_drivers ◄──┐
                +---------------+  │
                                   │    +-------+        +-------+        +-------+
                                   └──► | drv C | ◄────► | drv D | ◄────► | drv E |
                                        +-------+        +-------+        +-------+
```

```
root@romulus:/sys/bus# ls
clockevents   cpu           fsi           i2c           media         nvmem         serio         usb
clocksource   edac          gpio          iio           mmc           platform      soc           w1
container     event_source  hid           mdio_bus      mmc_rpmb      sdio          spi           workqueue
```

When registering a driver, it must specify the bus type so the kernel knows where to append the driver structure. 
Function **paltform_driver_register** is the helper that selects the bus **platform** automatically.


```
struct kobj_map {
    struct probe {
        struct probe *next;   // singly linked list
        dev_t dev;            // dev# = (major, minor)
        unsigned long range;  // range of consecutive minor#
        struct module *owner;
        kobj_probe_t *get;    // point to func that returns kobj
        int (*lock)(dev_t, void *);
        void *data;           // points to struct cdev of gendisk           
    } *probes[255];
    struct mutex *lock;
};
```

```
static struct char_device_struct {
    struct char_device_struct *next;    // singly linked list
    unsigned int major;                 // major#
    unsigned int baseminor;             // smallest minor#
    int minorct;                        // minor count = 'range' in kobj_map
    char name[64];                      // dev identifier
    struct cdev *cdev;                  // points to cdev
} *chrdevs[CHRDEV_MAJOR_HASH_SIZE];
```

```
+-------------------+                                                                              
| __register_chrdev |                                                                              
+----|--------------+                                                                              
     |    +--------------------------+                                                             
     |--> | __register_chrdev_region | check if specified dev# range is available, reserve it if so
     |    +--------------------------+                                                             
     |                                                                                             
     |--> prepare a 'cdev' for the dev# range                                                      
     |                                                                                             
     |    +----------+                                                                             
     +--> | cdev_add | add the 'cdev' to a table for later lookup                                  
          +----------+                                                                             
```

```
+----------+                                                                               
| add_disk | :
+--|-------+                                                                               
   |    +-----------------+                                                                
   +--> | device_add_disk | : add device, register bdi, and scan partitions
        +----|------------+                                                                
             |    +------------------+                                                     
             |--> | elevator_init_mq | (skip, there's no any elevator)                     
             |    +------------------+                                                     
             |    +------------+                                                           
             |--> | device_add |                                                           
             |    +------------+                                                           
             |    +--------------------+                                                   
             |--> | blk_register_queue | register the queue to sysfs, label it 'registered'
             |    +--------------------+                                                   
             |    +--------------+                                                         
             |--> | bdi_register | register bdi to bdi_tree and bdi_list                   
             |    +--------------+                                                         
             |    +----------------------+                                                 
             +--> | disk_scan_partitions |                                                 
                  +----------------------+                                                 
```

```
 +--------------------------+                                                        
 | platform_driver_register |                                                        
 +------|-------------------+                                                        
        |                                                                            
        |--> driver bus = platform_bus_type                                          
        |                                                                            
        |    +-----------------+                                                     
        +--> | driver_register |                                                     
             +----|------------+                                                     
                  |                                                                  
                  |--> determine which bus to register the driver                    
                  |                                                                  
                  |    +----------------+                                            
                  +--> | klist_add_tail | append the driver to the driver list of bus
                       +----------------+                                            
                  |    +---------------+                                             
                  +--> | driver_attach | probe each device in bus to find the match  
                       +---------------+                                                                                     
```

The kernel prepares the device structure for any device from DTS/DTB, parses its node properties, and then allocates resource structures accordingly. 
The rest is similar to the driver registration.
It doesn't matter whether the driver or device registers first. Both flows will trigger the probe mechanism to find the match within the bus.

<details>
  <summary> Code Trace </summary>

```
+---------------------------------+                                                                            
| of_platform_device_create_pdata |                                                                            
+--------|------------------------+                                                                            
         |    +-----------------+                                                                              
         |--> | of_device_alloc | prepare the structure of the device and resources                            
         |    +-----------------+                                                                              
         |                                                                                                     
         |--> set bus = platform_bus_type                                                                      
         |                                                                                                     
         |    +---------------+                                                                                
         +--> | of_device_add |                                                                                
              +---------------+                                                                                
                                                         
```
    
```
+------------+
| device_add | : add to bus, send 'add dev' to notifier, probe device
+--|---------+
   |    +----------------+
   |--> | bus_add_device |
   |    +---|------------+
   |        |
   |        |--> determine which bus to register the device
   |        |
   |        |    +----------------+
   |        +--> | klist_add_tail | append the device to the device list of bus
   |             +----------------+
   |
   |--> if dev->bus is set
   |
   |        +------------------------------+
   |------> | blocking_notifier_call_chain | send event 'ADD_DEVICE' to notifier, e.g.,
   |        +------------------------------+ +----------------------+
   |                                         | i2cdev_notifier_call | action handler
   |                                         +----------------------+ e.g., if 'add device': prepare and register cdev
   |    +------------------+
   +--> | bus_probe_device | traverse each driver and try to match, call driver->probe() if matched
        +------------------+
```
    
```
+-----------------+                                                          
| device_register | ： add to bus, send 'add dev' to notifier, probe device   
+----|------------+                                                          
     |    +-------------------+                                              
     |--> | device_initialize |                                              
     |    +-------------------+                                              
     |    +------------+                                                     
     +--> | device_add | add to bus, send 'add dev' to notifier, probe device
          +------------+                                                     
```
    
</details>

Both **driver_attach** and **bus_probe_device** are not directly but eventually go to **really_probe**, which triggers the driver defined probe function

```
+--------------+                                                                            
| really_probe |                                                                            
+---|----------+                                                                            
    |    +-------------------+                                                              
    |--> | pinctrl_bind_pins | switch pins to the expected state before probing if necessary
    |    +-------------------+                                                              
    |    +-------------------+                                                              
    |--> | call_driver_probe | call bus->probe or driver->probe (most cases)                
    |    +-------------------+                                                              
    |    +--------------+                                                                   
    +--> | driver_bound | attach the device to the driver, but not vice versa               
         +--------------+ (one driver might take care of multiple similar devices)          
```

## <a name="cheat-sheet"></a> Cheat Sheet

## <a name="reference"></a> Reference

- W. Mauerer, Professional Linux Kernel Architecture

<details>
  <summary> Messy Notes </summary>
    
To debug init calls, we add _initcall_debug_ to _bootargs_ in DTS.

```
    chosen {
        stdout-path = &uart5;
        bootargs = "console=ttyS4,115200 earlycon initcall_debug";
    };

```

Please note that by default the debug log won't display during boot time, and we need to utilize _dmesg_ to check it instead. 

- Related log
```
[    1.489785] aspeed-smc 1e620000.spi: Using 50 MHz SPI frequency
[    1.493323] aspeed-smc 1e620000.spi: n25q256a (32768 Kbytes)
[    1.493947] aspeed-smc 1e620000.spi: CE0 window [ 0x20000000 - 0x22000000 ] 32MB
[    1.494494] aspeed-smc 1e620000.spi: CE1 window [ 0x22000000 - 0x2a000000 ] 128MB
[    1.494842] aspeed-smc 1e620000.spi: read control register: 203b0045
[    1.728658] random: crng init done
[    1.923572] 5 fixed-partitions partitions found on MTD device bmc
[    1.923940] Creating 5 MTD partitions on "bmc":
[    1.924305] 0x000000000000-0x000000060000 : "u-boot"
[    1.927399] 0x000000060000-0x000000080000 : "u-boot-env"
[    1.930167] 0x000000080000-0x0000004c0000 : "kernel"
[    1.932337] 0x0000004c0000-0x000001c00000 : "rofs"
[    1.934627] 0x000001c00000-0x000002000000 : "rwfs"

▲ fmc: spi@1e620000/flash@0
------------------------------------------------------------------------------------------
▼ spi1: spi@1e630000/flash@0

[    1.941091] aspeed-smc 1e630000.spi: Using 100 MHz SPI frequency
[    1.942534] aspeed-smc 1e630000.spi: mx66l1g45g (131072 Kbytes)
[    1.942806] aspeed-smc 1e630000.spi: CE0 window resized to 120MB (AST2500 HW quirk)
[    1.943841] aspeed-smc 1e630000.spi: CE0 window [ 0x30000000 - 0x37800000 ] 120MB
[    1.944343] aspeed-smc 1e630000.spi: CE1 window [ 0x37800000 - 0x38000000 ] 8MB
[    1.944715] aspeed-smc 1e630000.spi: CE0 window too small for chip 128MB
[    1.945007] aspeed-smc 1e630000.spi: read control register: 203c0045
[    1.947181] aspeed-smc 1e630000.spi: Calibration area too uniform, using low speed

Code flow:
[aspeed_smc_probe]
    prepare controller
    get 1st MEM resource and map it as registers
    get 2nd MEM resource and map it as AHB base <- ?
    [aspeed_smc_setup_flash]
        for each flash connected to the controller (e.g. only 1)
            prepare chip
        
```
                                                   
```
ls -l /sys/dev/char
ls -l /sys/dev/block                                                   
```
    
</details>
