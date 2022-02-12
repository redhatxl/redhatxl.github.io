### Linux 内核引导

## 一 概述

内核引导选项大体上可以分为两类：一类与设备无关、另一类与设备有关。与设备有关的引导选项多如牛毛，需要你自己阅读内核中的相应驱动程序源码以获取其能够接受的引导选项。比如，如果你想知道可以向 AHA1542 SCSI 驱动程序传递哪些引导选项，那么就查看 drivers/scsi/aha1542.c 文件，一般在前面 100 行注释里就可以找到所接受的引导选项说明。大多数选项是通过"__setup()"函数设置的，少部分是通过"early_param()"或"module_param()"或"module_param_named()"之类的函数设置的，逗号前的部分就是引导选项的名称，后面的部分就是处理这些选项的函数名。

[提示]你可以在源码树的根目录下试一试下面几个命令：

```shell
grep -r '\b__setup *(' *
grep -r '\bearly_param *(' *
grep -r '\bmodule_param *(' *
grep -r '\bmodule_param_named *(' *
```

格式上，多个选项之间用空格分割，选项值是一个逗号分割的列表，并且选项值中不能包含空白。

```shell
正确：ether=9,0x300,0xd0000,0xd4000,eth0  root=/dev/sda2
错误：ether = 9, 0x300, 0xd0000, 0xd4000, eth0  root = /dev/sda2
```

注意，所有引导选项都是大小写敏感的！

在内核运行起来之后，可以通过 cat /proc/cmdline 命令查看当初使用的引导选项以及相应的值。

## 二 内核模块

对于模块而言，引导选项只能用于直接编译到核心中的模块，格式是"模块名.选项=值"，比如"usbcore.blinkenlights=1"。动态加载的模块则可以在 modprobe 命令行上指定相应的选项值，比如"modprobe usbcore blinkenlights=1"。

可以使用"modinfo -p \${modulename}"命令显示可加载模块的所有可用选项。已经加载到内核中的模块会在 /sys/module/\${modulename}/parameters/ 中显示出其选项，并且某些选项的值还可以在运行时通过"echo -n ​\${value} > /sys/module/​\${modulename}/parameters/${parm}"进行修改。

## 三 内核如何处理引导项

绝大部分的内核引导选项的格式如下(每个选项的值列表中最多只能有十项)：

```
name[=value_1][,value_2]...[,value_10]
```

如果"name"不能被识别并且满足"name=value"的格式，那么将被解译为一个环境变量(比如"TERM=linux"或"BOOT_IMAGE=vmlinuz.bak")，否则将被原封不动的传递给 init 程序(比如"single")。

内核可以接受的选项个数没有限制，但是整个命令行的总长度(选项/值/空格全部包含在内)却是有限制的，定义在 include/asm/setup.h 中的 COMMAND_LINE_SIZE 宏中(对于X86_64而言是2048)。

## 四 内核引导选项精选

由于引导选项多如牛毛，本文不可能涉及全部，因此本文只基于 X86_64 平台以及 Linux-3.13.2 精选了一些与设备无关的引导选项以及少部分与设备有关的引导选项，过时的选项、非x86平台选项、与设备有关的选项，基本上都被忽略了。

[提示]内核源码树下的 [Documentation/admin-guide/kernel-parameters.txt](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html) 和 [Documentation/x86/x86_64/boot-options.rst](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/boot-options.rst) 文件列出了所有可用的引导选项，并作了简要说明。

由于引导选项多如牛毛，本文不可能涉及全部，因此本文只基于 X86_64 平台以及 Linux-3.13.2 精选了一些与设备无关的引导选项以及少部分与设备有关的引导选项，过时的选项、非x86平台选项、与设备有关的选项，基本上都被忽略了。

[提示]内核源码树下的 [Documentation/admin-guide/kernel-parameters.txt](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html) 和 [Documentation/x86/x86_64/boot-options.rst](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/boot-options.rst) 文件列出了所有可用的引导选项，并作了简要说明。

### 4.1 标记说明

并不是所有的选项都是永远可用的，只有在特定的模块存在并且相应的硬件也存在的情况下才可用。引导选项上面的方括号说明了其依赖关系，其中使用的标记解释如下：

```shell
ACPI     开启了高级配置与电源接口(CONFIG_ACPI)支持
AGP      开启了AGP(CONFIG_AGP)支持
APIC     开启了高级可编程中断控制器支持(2000年以后的CPU都支持)
APPARMOR 开启了AppArmor(CONFIG_SECURITY_APPARMOR)支持
DRM      开启了Direct Rendering Manager(CONFIG_DRM)支持
EFI      开启了EFI分区(CONFIG_EFI_PARTITION)支持
EVM      开启了Extended Verification Module(CONFIG_EVM)支持
FB       开启了帧缓冲设备(CONFIG_FB)支持
HIBERNATION  开启了"休眠到硬盘"(CONFIG_HIBERNATION)支持
HPET_MMAP    允许对HPET寄存器进行映射(CONFIG_HPET_MMAP)
HW       存在相应的硬件设备
IOMMU    开启了IOMMU(CONFIG_IOMMU_SUPPORT)支持
IOSCHED  开启了多个不同的I/O调度程序(CONFIG_IOSCHED_*)
IPV6     开启了IPv6(CONFIG_IPV6)支持
IP_PNP   开启了自动获取IP的协议(DHCP,BOOTP,RARP)支持
IP_VS_FTP    开启了IPVS FTP协议连接追踪(CONFIG_IP_VS_FTP)支持
KVM      开启了KVM(CONFIG_KVM_*)支持
LIBATA   开启了libata(CONFIG_ATA)驱动支持
LOOP     开启了回环设备(CONFIG_BLK_DEV_LOOP)支持
MCE      开启了Machine Check Exception(CONFIG_X86_MCE)支持
MOUSE    开启了鼠标(CONFIG_INPUT_MOUSEDEV)支持
MSI      开启了PCI MSI(CONFIG_PCI_MSI)支持
NET      开启了网络支持
NETFILTER    开启了Netfilter(CONFIG_NETFILTER)支持
NFS      开启了NFS(网络文件系统)支持
NUMA     开启了NUMA(CONFIG_NUMA)支持
PCI      开启了PCI总线(CONFIG_PCI)支持
PCIE     开启了PCI-Express(CONFIG_PCIEPORTBUS)支持
PNP      开启了即插即用(CONFIG_PNP)支持
PV_OPS   内核本身是半虚拟化的(paravirtualized)
RAID     开启了软RAID(CONFIG_BLK_DEV_MD)支持
SECURITY 开启了多个不同的安全模型(CONFIG_SECURITY)
SELINUX  开启了SELinux(CONFIG_SECURITY_SELINUX)支持
SLUB     开启了SLUB内存分配管理器(CONFIG_SLUB)
SMP      开启了对称多处理器(CONFIG_SMP)支持
TPM      开启了可信赖平台模块(CONFIG_TCG_TPM)支持
UMS      开启了USB大容量存储设备(CONFIG_USB_STORAGE)支持
USB      开启了USB(CONFIG_USB_SUPPORT)支持
USBHID   开启了USB HID(CONFIG_USB_HID)支持
VMMIO    开启了使用内存映射机制的virtio设备驱动(CONFIG_VIRTIO_MMIO)
VT       开启了虚拟终端(CONFIG_VT)支持
```

此外，下面的标记在含义上与上面的有所不同：

```
BUGS    用于解决某些特定硬件的缺陷
KNL     是一个内核启动选项
BOOT    是一个引导程序选项
```

标记为"BOOT"的选项实际上由引导程序(例如GRUB)使用，对内核本身没有直接的意义。如果没有特别的需求，请不要修改此类选项的语法，更多信息请阅读 [Documentation/x86/boot.txt](http://lxr.linux.no/linux/Documentation/x86/boot.txt) 文档。

说明：下文中的 [KMG] 后缀表示 210, 220, 230 的含义。

### 4.2 控制台与终端

* **[KNL]**
  console=设备及选项

设置输出控制台使用的设备及选项。例如：ttyN 表示使用第N个虚拟控制台。其它用法主要针对嵌入式环境([Documentation/serial-console.txt](http://lxr.linux.no/linux/Documentation/serial-console.txt))。

* **[KNL]**
  consoleblank=秒数

控制台多长时间无操作后黑屏，默认值是600秒，设为0表示禁止黑屏。

* **[HW]**
  no_console_suspend

永远也不要将控制台进入休眠状态。因为当控制台进入休眠之后，所有内核的消息就都看不见了(包括串口与VGA)。开启此选项有助于调试系统在休眠/唤醒中发生的故障。

* **[VT]**
  vt.default_utf8={0|1}

是否将所有TTY都默认设置为UTF-8模式。默认值"1"表示将所有新打开的终端都设置为UTF-8模式。

### 4.3 日志与调试

* **earlyprintk=设备[,keep]**

使用哪个设备显示早期的引导信息，主要用于调试硬件故障。此选项默认并未开启，因为在某些情况下并不能正常工作。
在传统的控制台初始化之前，在哪个设备上显示内核日志信息。不使用此选项，那么你将永远没机会看见这些信息。
在尾部加上",keep"选项表示在真正的内核控制台初始化并接管系统后，不会抹掉本选项消息的显示。
earlyprintk=vga 表示在VGA上显示内核日志信息，这是最常用的选项，但不能用于EFI环境。
earlyprintk=efi v3.13新增，表示将错误日志写入EFI framebuffer，专用于EFI环境。
earlyprintk=xen 仅可用于XEN的半虚拟化客户机。

* **loglevel={0|1|2|3|4|5|6|7}**

设置内核日志的级别，所有小于该数字的内核信息(具有更高优先级的信息)都将在控制台上显示出来。这个级别可以使用 klogd 程序或者修改 /proc/sys/kernel/printk 文件进行调整。取值范围是"0"(不显示任何信息)到"7"(显示所有级别的信息)。建议至少设为"4"(WARNING)。[提示]级别"7"要求编译时加入了调试支持。

* **[KNL]**
  ignore_loglevel

忽略内核日志等级的设置，向控制台输出所有内核消息。仅用于调试目的。

* **[KNL]**
  debug

将引导过程中的所有调试信息都显示在控制台上。相当于设置"loglevel=7"(DEBUG)。

* **[KNL]**
  quiet

静默模式。相当于设置"loglevel=4"(WARNING)。

* **log_buf_len=n[KMG]**

内核日志缓冲区的大小。"n"必须是2的整数倍(否则会被自动上调到最接近的2的整数倍)。该值也可以通过内核配置选项CONFIG_LOG_BUF_SHIFT来设置。

* **[KNL]**
  initcall_debug

跟踪所有内核初始化过程中调用的函数。有助于诊断内核在启动过程中死在了那个函数上面。

* **kstack=N**

在内核异常(oops)时，应该打印出内核栈中多少个字(word)到异常转储中。仅供调试使用。

* **[KNL]**
  kmemleak={on|off}

是否开启检测内核内存泄漏的功能(CONFIG_DEBUG_KMEMLEAK)，默认为"on"，仅供调试使用。
检测方法类似于跟踪内存收集器，一个内核线程每10分钟(默认值)扫描内存，并打印发现新的未引用的对象的数量。

* **[KNL]**
  memtest=整数

设置内存测试(CONFIG_MEMTEST)的轮数。"0"表示禁止测试。仅在你确实知道这是什么东西并且确实需要的时候再开启。

* **norandmaps**

默认情况下，内核会随机化程序的启动地址，也就是每一次分配给程序的虚拟地址空间都不一样，主要目的是为了防止缓冲区溢出攻击。但是这也给程序调试增加了麻烦，此选项(相当于"echo 0 > /proc/sys/kernel/randomize_va_space")的目的就是禁用该功能以方便调试。

* **[PNP]**
  pnp.debug=1

开启PNP调试信息(需要内核已开启CONFIG_PNP_DEBUG_MESSAGES选项)，仅用于调试目的。也可在运行时通过 /sys/module/pnp/parameters/debug 来控制。

* **show_msr=CPU数**

显示启动时由BIOS初始化的MSR(Model-Specific Register)寄存器设置。CPU数设为"1"表示仅显示"boot CPU"的设置。

* **printk.time={0|1}**

是否在每一行printk输出前都加上时间戳，仅供调试使用。默认值是"0"(不添加)

* **boot_delay=毫秒数**

在启动过程中，为每一个printk动作延迟指定的毫秒数，取值范围是[0-10000](最大10秒)，超出这个范围将等价于"0"(无延迟)。仅用于调试目的。

* **pause_on_oops=秒数**

当内核发生异常时，挂起所有CPU的时间。当异常信息太多，屏幕持续滚动时，这个选项就很有用处了。主要用于调试目的。

### 4.4 异常检测与处理

- **[MCE] mce=off**

  彻底禁用MCE(CONFIG_X86_MCE)

- **[MCE] mce=dont_log_ce**

  不为已纠正错误(corrected error)记录日志。

- **[MCE] mce=容错级别[,超时]**

  容错级别(还可通过sysfs设置)： 0 在出现未能纠正的错误时panic，记录所有已纠正的错误 1(默认值) 在出现未能纠正的错误时panic或SIGBUS，记录所有已纠正的错误 2 在出现未能纠正的错误时SIGBUS或记录日志，记录所有已纠正的错误 3 从不panic或SIGBUS，记录所有日志。仅用于调试目的。 超时(单位是微秒[百万分之一秒])：在machine check时等待其它CPU的时长，"0"表示不等待。

- **[ACPI] erst_disable**

  禁用[ERST](http://www.ibm.com/developerworks/cn/linux/l-cn-apei/)(Error Record Serialization Table)支持。主要用于解决某些有缺陷的BIOS导致的ERST故障。

- **[ACPI] hest_disable**

  禁用[HEST](http://www.ibm.com/developerworks/cn/linux/l-cn-apei/)(Hardware Error Source Table)支持。主要用于解决某些有缺陷的BIOS导致的HEST故障。

- **[KNL] nosoftlockup**

  禁止内核进行软[死锁检测](http://blog.chinaunix.net/uid-25942458-id-3823545.html)

- **[KNL] softlockup_panic={0|1}**

  是否在检测到软死锁(soft-lockup)的时候让内核panic，其默认值由 CONFIG_BOOTPARAM_SOFTLOCKUP_PANIC_VALUE 确定

- **[KNL] nowatchdog**

  禁止硬死锁检测([NMI watchdog](http://www.cnblogs.com/sdphome/archive/2011/11/16/2251042.html))

- **[KNL,BUGS] nmi_watchdog={0|panic|nopanic}**

  配置[nmi_watchdog](http://bbs.chinaunix.net/thread-4095844-1-1.html)(不可屏蔽中断看门狗)。更多信息可查看"lockup-watchdogs.txt"文档。 0 表示关闭看门狗； panic 表示出现看门狗超时(长时间没喂狗)的时候触发[内核错误](http://baike.baidu.com/view/5538970.htm)，通常和"panic="配合使用，以实现在系统出现锁死的时候自动重启。 nopanic 正好相反，表示即使出现看门狗超时(长时间没喂狗)，也不触发内核错误。

- **unknown_nmi_panic**

  在收到未知的NMI(不可屏蔽中断)时直接panic

- **oops=panic**

  在内核oops时直接panic(而默认是仅仅杀死oops进程[这样做会有很小的概率导致死锁])，而且这同样也会导致在发生MCE(CONFIG_X86_MCE)时直接panic。主要目的是和"panic="选项连用以实现自动重启。

- **[KNL] panic=秒数**

  内核在遇到panic时等待重启的行为：

   秒数>0 等待指定的秒数后重启 

  秒数=0(默认值) 只是简单的挂起，而永不重启 

  秒数<0 立即重启

  ## 4.5 时钟(Timer)

  [时钟(Timer)](http://msdn.microsoft.com/zh-cn/windows/hardware/gg463347)的功能有两个：(1)定时触发中断；(2)维护和读取当前时间。x86_64平台常见的[时钟硬件](http://www.ibm.com/developerworks/cn/linux/1307_liuming_linuxtime2/)有以下这些：
  RTC(Real Time Clock) 实时时钟的独特之处在于，RTC是主板上一块电池供电的CMOS芯片(精度一般只到秒级)，RTC(Clock)吐出来的是"时刻"(例如"2014-2-22 23:38:44")，而其他硬件时钟(Timer)吐出来的是"时长"(我走过了XX个周期，按照我的频率，应该是10秒钟)。
  PIT(Programmable Interval Timer) PIT是最古老的时钟源，产生周期性的时钟中断(IRQ0)，精度在100-1000Hz，现在基本已经被HPET取代。
  APIC Timer 这是PIT针对多CPU环境的升级，每个CPU上都有一个APIC Timer(而PIT则是所有CPU共享的)，但是它经常有BUG且精度也不高(3MHz左右)，所实际很少使用。
  ACPI Timer(Power Management Timer) 它唯一的功能就是为每个时钟周期提供一个时间戳，用于提供与处理器速度无关的可靠时间戳。但其精度并不高(3.579545MHz)。
  HPET(High Precision Event Timer) HPET提供了更高的精度(14.31818MHz)以及更宽的计数器(64位)。HPET可以替代前述除RTC之外的所有时钟硬件(Timer)，因为它既能定时触发中断，又能维护和读取当前时间。一个HPET包含了一个固定频率的数值递增的计数器以及3-32个独立计数器，每个计数器又包含了一个比较器和一个寄存器，当两者数值相等时就会触发中断。HPET的出现将允许删除芯片组中的一些冗余的旧式硬件。2006年之后的主板基本都已支持HPET。
  TSC(Time Stamp Counter) TSC是位于CPU里面的一个64位寄存器，与传统的周期性时钟不同，TSC并不触发中断，它是以计数器形式存在的单步递增性时钟。也就是说，周期性时钟是通过周期性触发中断达到计时目的，如心跳一般。而单步递增时钟则不发送中断，取而代之的是由软件自己在需要的时候去主动读取TSC寄存器的值来获得时间。TSC的精度(纳秒级)远超HPET并且速度更快，但仅能在较新的CPU(Sandy Bridge之后)上使用。

* **[HW,ACPI]**
  acpi_skip_timer_override

用于解决某些有缺陷的Nvidia nForce2 BIOS中的计时器覆盖问题(例如开启ACPI后频繁死机或时钟故障)。

* **[HW,ACPI]**
  acpi_use_timer_override

用于解决某些有缺陷的Nvidia nForce5 BIOS中的计时器覆盖问题(例如开启ACPI后频繁死机或时钟故障)。

* **[APIC]**
  no_timer_check

禁止运行内核中时钟IRQ源缺陷检测代码。主要用于解决某些AMD平台的[CPU占用过高以及时钟过快](http://www.ensode.net/no_timer_check.html)的故障。

* **pmtmr=十六进制端口号**

手动指定pmtimer(CONFIG_X86_PM_TIMER)的I/O端口(16进制值)，例如：pmtmr=0x508

* **acpi_pm_good**

跳过pmtimer(CONFIG_X86_PM_TIMER)的bug检测，强制内核认为这台机器的pmtimer没有毛病。用于解决某些有缺陷的BIOS导致的故障。

* **[APIC]**
  apicpmtimer

使用pmtimer(CONFIG_X86_PM_TIMER)来校准APIC timer。此选项隐含了"apicmaintimer"。用于[PIT timer](http://www.ibm.com/developerworks/cn/linux/l-cn-timerm/)彻底坏掉的场合。

* **[APIC]**
  apicmaintimer
  noapicmaintimer

apicmaintimer 将APIC timer用于计时(而不是PIT/HPET中断)。这主要用于PIT/HPET中断不可靠的场合。
noapicmaintimer 不将APIC timer用于计时(而是使用PIT/HPET中断)。这是默认值。但有时候依然需要明确指定。

* **[APIC]**
  lapic_timer_c2_ok

按照ACPI规范的要求，local APIC Timer 不能在C2休眠状态下关闭，但可以在C3休眠状态下关闭。但某些BIOS(主要是AMD平台)会在向操作系统报告CPU进入C2休眠状态时，实际进入C3休眠状态。因此，内核默认采取了保守的假定：认为 local APIC Timer 在C2/C3状态时皆处于关闭状态。如果你确定你的BIOS没有这个问题，那么可以使用此选项明确告诉内核，即使CPU在C2休眠状态，local APIC Timer 也依然可用。

* **[APIC]**
  noapictimer

禁用CPU [Local APIC Timer](http://www.giraffetea.com/?p=61)

**enable_timer_pin_1**
**disable_timer_pin_1**

开启/关闭APIC定时器的PIN1，内核将尽可能自动探测正确的值。但有时需要手动指定以解决某些有缺陷的ATI芯片组故障。

* **clocksource={jiffies|acpi_pm|hpet|tsc}**

强制使用指定的时钟源，以代替内核默认的时钟源。
jiffies 最差的时钟源，只能作为最后的选择。
acpi_pm [ACPI]符合ACPI规范的主板都提供的硬件时钟源(CONFIG_X86_PM_TIMER)，提供3.579545MHz固定频率，这是传统的硬件时钟发生器。
hpet 一种取代传统"acpi_pm"的高精度硬件时钟源(CONFIG_HPET)，提供14.31818MHz固定频率。2007年以后的芯片组一般都支持。
tsc TSC(Time Stamp Counter)的主体是位于CPU里面的一个64位TSC寄存器，与传统的以中断形式存在的周期性时钟不同，TSC是以计数器形式存在的单步递增性时钟，两者的区别在于，周期性时钟是通过周期性触发中断达到计时目的，如心跳一般。而单步递增时钟则不发送中断，取而代之的是由软件自己在需要的时候去主动读取TSC寄存器的值来获得时间。TSC的精度更高并且速度更快，但仅能在较新的CPU(Sandy Bridge之后)上使用。

* **[KNL]**
  highres={"on"|"off"}

启用(默认值)还是禁用高精度定时器模式。主要用于关闭主板上有故障的高精度时钟源。

* **nohpet**

禁用HPET timer(CONFIG_HPET)

* **[HPET_MMAP]**
  hpet_mmap

v3.13新增，默认允许对HPET寄存器进行映射，相当于开启了内核CONFIG_HPET_MMAP_DEFAULT选项。需要注意的是，某些包含HPET硬件寄存器的页中同时还含有其他不该暴露给用户的信息。

* **notsc**
  tsc=reliable
  tsc=noirqtime

设置TSC时钟源的属性。
notsc 表示不将TSC用作"wall time"时钟源，主要用于不能在多个CPU之间保持正确同步的SMP系统。
tsc=reliable 表示TSC时钟源是绝对稳定的，关闭启动时和运行时的稳定性检查。用于在某些老旧硬件/虚拟化环境使用TSC时钟源。
tsc=noirqtime 不将TSC用于统计进程IRQ时间。主要用于在RDTSC速度较慢的CPU上禁止内核的CONFIG_IRQ_TIME_ACCOUNTING功能。
关于"TSC时钟源"，详见"clocksource="选项的说明。

### 4.6 中断

常见的中断控制器有两种：传统的8259A和新式的APIC，前者也被称为"PIC"。8259A只适合单CPU的场合，而APIC则能够把中断传递给系统中的每个CPU，从而充分挖掘SMP体系结构的并行性。所以8259A已经被淘汰了。[APIC](http://blog.csdn.net/ustc_dylan/article/details/4132046)系统由3部分组成：APIC总线(前端总线)、IO-APIC(南桥)、本地APIC(CPU)。每个CPU中集成了一个本地APIC，负责传递中断信号到处理器。而IO-APIC是系统芯片组中一部分，负责收集来自I/O设备的中断信号并发送到本地APIC。APIC总线则是连接IO-APIC和各个本地APIC的桥梁。

常见的中断控制器有两种：传统的8259A和新式的APIC，前者也被称为"PIC"。8259A只适合单CPU的场合，而APIC则能够把中断传递给系统中的每个CPU，从而充分挖掘SMP体系结构的并行性。所以8259A已经被淘汰了。[APIC](http://blog.csdn.net/ustc_dylan/article/details/4132046)系统由3部分组成：APIC总线(前端总线)、IO-APIC(南桥)、本地APIC(CPU)。每个CPU中集成了一个本地APIC，负责传递中断信号到处理器。而IO-APIC是系统芯片组中一部分，负责收集来自I/O设备的中断信号并发送到本地APIC。APIC总线则是连接IO-APIC和各个本地APIC的桥梁。

- **[SMP,APIC] noapic**

  禁止使用IO-APIC(输入输出高级可编程输入控制器)，主要用于解决某些有缺陷的BIOS导致的APIC故障。

- **[APIC] nolapic disableapic**

  禁止使用local APIC。主要用于解决某些有缺陷的BIOS导致的APIC故障。"nolapic"是为了保持传统习惯的兼容写法，与"disableapic"的含义相同。

- **[APIC] nox2apic**

  关闭x2APIC支持(CONFIG_X86_X2APIC)

- **[APIC] x2apic_phys**

  在支持x2apic的平台上使用physical模式代替默认的cluster模式。

- **[KNL] threadirqs**

  强制线程化所有的中断处理器(明确标记为IRQF_NO_THREAD的除外)

- **[SMP,APIC] pirq=**

  手动指定mp-table的设置。此选项仅对某些有缺陷的、具备多个IO-APIC的高端主板有意义。详见[Documentation/x86/i386/IO-APIC.txt](http://lxr.linux.no/linux/Documentation/x86/i386/IO-APIC.txt)文档

- **[HW] irqfixup**

  用于修复简单的中断问题：当一个中断没有被处理时搜索所有可用的中断处理器。用于解决某些简单的固件缺陷。

- **[HW] irqpoll**

  用于修复高级的中断问题：当一个中断没有被处理时搜索所有可用的中断处理器，并且对每个时钟中断都进行搜索。用于解决某些严重的固件缺陷。

### 4.7 ACPI

高级配置与电源管理接口(Advanced Configuration and Power Interface)是提供操作系统与应用程序管理所有电源管理接口，包括了各种软件和硬件方面的规范。2004年推出3.0规范；2009年推出4.0规范；2011年推出5.0规范。2013年之后新的ACPI规格将由UEFI论坛制定。ACPI可以实现的功能包括：电源管理；性能管理；配置与即插即用；系统事件；温度管理；电池管理；SMBus控制器；嵌入式控制器。

- [HW,ACPI] acpi={force|off|noirq|strict|rsdt|nocmcff|copy_dsdt}

  ACPI的总开关。 force 表示强制启用ACPI(即使BIOS中已关闭)； off 表示强制禁用ACPI(即使BIOS中已开启)； noirq 表示不要将ACPI用于IRQ路由； strict 表示严格要求系统遵循ACPI规格(降低兼容性)； rsdt 表示使用老旧的RSDT(Root System Description Table)代替较新的XSDT(Extended System Description Table)； copy_dsdt 表示将DSDT(Differentiated System Description Table)复制到内存中。 更多信息可参考[Documentation/power/runtime_pm.txt](http://lxr.linux.no/linux/Documentation/power/runtime_pm.txt)以及"pci=noacpi"。

- [HW,ACPI] acpi_backlight={vendor|video}

  选择[屏幕背光亮度调节](https://wiki.archlinux.org/index.php/Backlight)驱动。 video(默认值)表示使用通用的ACPI video.ko驱动(CONFIG_ACPI_VIDEO)，该驱动仅可用于集成显卡。 vendor表示使用厂商特定的ACPI驱动(thinkpad_acpi,sony_acpi等)。 详见[Documentation/acpi/video_extension.txt](http://lxr.linux.no/linux/Documentation/acpi/video_extension.txt)文档。

- [HW,ACPI] acpi_os_name="字符串"

  告诉ACPI BIOS操作系统的名称。 常用于哄骗有缺陷的BIOS，让其以为运行的是Windows系统而不是Linux系统。 "Linux" = Linux "Microsoft Windows" = Windows 98 "Windows 2000" = Windows 2000 "Windows 2001" = Windows XP "Windows 2001 SP2" = Windows XP SP2 "Windows 2001.1" = Windows Server 2003 "Windows 2001.1 SP1" = Windows Server 2003 SP1 "Windows 2006" = Windows Vista "Windows 2006 SP1" = Windows Vista SP1 "Windows 2006.1" = Windows Server 2008 "Windows 2009" = Windows 7 / Windows Server 2008 R2 "Windows 2012" = Windows 8 / Windows Server 2012 "Windows 2013" = Windows 8.1 / Windows Server 2012 R2

- [HW,ACPI] acpi_osi="字符串"

  对于较新的内核(Linux-2.6.23之后)而言，当BIOS询问内核："你是Linux吗?"，内核都会回答"No"，但历史上(Linux-2.6.22及更早版本)内核会如实回答"Yes"，结果造成很多BIOS兼容性问题(主要是电源管理方面)。具体故事的细节请到内核源码文件[drivers/acpi/osl.c](http://lxr.linux.no/linux/drivers/acpi/osl.c)中搜索"The story of _OSI(Linux)"注释。 此选项用于修改内核中的操作系统接口字符串([_OSI string](http://msdn.microsoft.com/zh-cn/library/windows/hardware/gg463275.aspx))列表默认值，这样当BIOS向内核询问："你是xxx吗?"的时候，内核就可以根据修改后的列表中是否存在"xxx"回答"Yes"或"No"了，主要用于解决BIOS兼容性问题导致的故障(例如[屏幕亮度调整](https://wiki.archlinux.org/index.php/Backlight))。 acpi_osi="Linux"表示添加"Linux"； acpi_osi="!Linux"表示删除"Linux"； acpi_osi=!* 表示删除所有字符串(v3.13新增)，可以和多个acpi_osi="Linux"格式联合使用； acpi_osi=! 表示删除所有内置的字符串(v3.13新增)，可以和多个acpi_osi="Linux"格式联合使用； acpi_osi= 表示禁用所有字符串，仅可单独使用(不能联合使用)。

- [HW,ACPI] acpi_serialize

  强制内核以串行方式执行AML(ACPI Machine Language)字节码。用于解决某些有缺陷的BIOS导致的故障。

- [ACPI] acpi_enforce_resources={strict|lax|no}

  检查驱动程序和ACPI操作区域(SystemIO,SystemMemory)之间资源冲突的方式。 strict(默认值)禁止任何驱动程序访问已被ACPI声明为"受保护"的操作区域，这是最安全的方式，可以从根本上避免冲突。 lax允许驱动程序访问已被ACPI声明的保护区域(但会显示一个警告)。这可能会造成冲突，但是可以兼容某些老旧且脑残的驱动程序(例如某些硬件监控驱动)。 no表示根本不声明任何ACPI保护区域，也就是完全允许任意驱动程序访问ACPI操作区域。

- [ACPI] pnpacpi=off

  禁用ACPI的即插即用功能，转而使用古董的PNPBIOS来代替。

### 4.8 休眠与唤醒

* **[HW,ACPI]**
  acpi_sleep={s3_bios,s3_mode,s3_beep,s4_nohwsig,old_ordering,nonvs,sci_force_enable}

ACPI休眠选项。
(1)s3_bios和s3_mode与显卡有关。计算机从S3状态(挂起到内存)恢复时，硬件需要被正确的初始化。这对大多数硬件都不是问题，但因为显卡是由BIOS初始化的，内核无法获取必要的恢复信息(仅存在于BIOS中，内核无法读取)，所以这里就提供了两个选项，以允许内核通过两种不同的方式来恢复显卡，更多细节请参考[Documentation/power/video.txt](http://lxr.linux.no/linux/Documentation/power/video.txt)文档。
(2)s3_beep主要用于调试，它让PC喇叭在内核的实模式入口点被调用时发出响声。
(3)s4_nohwsig用于关闭ACPI硬件签名功能，某些有缺陷的BIOS会因为这个原因导致从S4状态(挂起到硬盘)唤醒时失败。
(4)old_ordering用于兼容古董级的ACPI 1.0 BIOS
(5)nonvs表示阻止内核在挂起/唤醒过程中保存/恢复ACPI NVS内存信息，主要用于解决某些有缺陷的BIOS导致的挂起/唤醒故障。
(6)sci_force_enable表示由内核直接设置SCI_EN(ACPI模式开关)的状态，主要用于解决某些有缺陷的BIOS导致的从S1/S3状态唤醒时的故障。

* **[HIBERNATION]**
  noresume

禁用内核的休眠到硬盘功能(CONFIG_HIBERNATION)，也就是不从先前的休眠状态中恢复(即使该状态已经被保存在了硬盘的swap分区上)，并且清楚先前已经保存的休眠状态(如果有的话)。

* **[HIBERNATION]**
  hibernate={noresume|nocompress}

设置休眠/唤醒属性：
noresume 表示禁用唤醒，也就是在启动过程中无视任何已经存在的休眠镜像，完全重新启动。
nocompress 表示禁止对休眠镜像进行压缩/解压。

* **[HIBERNATION]**
  resume={ /dev/swap | PARTUUID=uuid | major:minor | hex }

告诉内核被挂起的内存镜像存放在那个磁盘分区(默认值是CONFIG_PM_STD_PARTITION)。
假定内存镜像存放在"/dev/sdc15"分区上，该分区的 UUID=0123456789ABCDEF ，其主设备号是"8"，次设备号是"47"，那么这4种表示法应该分别这样表示：
resume=/dev/sdc15 (这是内核设备名称，有可能与用户空间的设备名称不同)
resume=PARTUUID=0123456789ABCDEF
resume=08:47
resume=082F

* **[HIBERNATION]**
  resume_offset=整数

指定swap header所在位置的偏移量(单位是PAGE_SIZE)，偏移量的计算基准点是"resume="分区的起点。
仅在使用swap文件(而不是分区)的时候才需要此选项。详见[Documentation/power/swsusp-and-swap-files.txt](http://lxr.linux.no/linux/Documentation/power/swsusp-and-swap-files.txt)文档

* **[HIBERNATION]**
  resumedelay=秒数

在读取resume文件(设备)之前延迟的秒数，主要用于等待那些反应速度较慢的异步检测的设备就绪(例如USB/MMC)。

* **[HIBERNATION]**
  resumewait

在resume设备没有就绪之前无限等待，主要用于等待那些反应速度较慢的异步检测的设备就绪(例如USB/MMC)。

### 4.9 温度控制

* **[HW,ACPI]**
  thermal.act=摄氏度

-1 禁用所有"主动散热"标志点(active trip point)
正整数 强制设置所有的最低"主动散热"标志点的温度值，单位是摄氏度。
详见[Documentation/thermal/sysfs-api.txt](http://lxr.linux.no/linux/Documentation/thermal/sysfs-api.txt)文档。

* **[HW,ACPI]**
  thermal.psv=摄氏度

-1 禁用所有"被动散热"标志点(passive trip point)
正整数 强制设置所有的"被动散热"标志点的温度值，单位是摄氏度。
详见[Documentation/thermal/sysfs-api.txt](http://lxr.linux.no/linux/Documentation/thermal/sysfs-api.txt)文档。

* **[HW,ACPI]**
  thermal.crt=摄氏度

-1 禁用所有"紧急"标志点(critical trip point)
正整数 强制设置所有的"紧急"标志点的温度值，单位是摄氏度。
详见[Documentation/thermal/sysfs-api.txt](http://lxr.linux.no/linux/Documentation/thermal/sysfs-api.txt)文档。

* **[HW,ACPI]**
  thermal.nocrt=1

禁止在ACPI热区(thermal zone)温度达到"紧急"标志点时采取任何动作。

* **[HW,ACPI]**
  thermal.off=1

彻底关闭ACPI热量控制(CONFIG_ACPI_THERMAL)

* **[HW,ACPI]**
  thermal.tzp=整数

设置ACPI热区(thermal zone)的轮询速度：
0(默认值) 不轮询
正整数 轮询间隔，单位是十分之一秒。

### 4.10 CPU节能

- **[KNL] nohz={on|off}**

  启用/禁用内核的dynamic ticks特性。默认值是"on"。

- **[KNL,BOOT] nohz_full=CPU列表**

  在内核"CONFIG_NO_HZ_FULL=y"的前提下，指定哪些CPU核心可以进入完全无滴答状态。 "CPU列表"是一个逗号分隔的CPU编号(从0开始计数)，也可以使用"-"界定一个范围。例如"0,2,4-7"等价于"0,2,4,5,6,7" [注意](1)"boot CPU"(通常都是"0"号CPU)会无条件的从列表中剔除。(2)这里列出的CPU编号必须也要同时列进"rcu_nocbs=..."选项中。

- **[HW,ACPI] processor.nocst**

  不使用_CST方法检测C-states，而是用老旧的FADT方法检测。

- **[HW,ACPI] processor.max_cstate={0|1|2|3|4|5|6|7|8|9}**

  无视ACPI表报告的值，强制指定CPU的最大[C-state](http://www.expreview.com/25426.html)值(必须是一个有效值)：C0为正常状态，其他则为不同的省电模式(数字越大表示CPU休眠的程度越深/越省电)。"9"表示无视所有的DMI黑名单限制。

- **[KNL,HW,ACPI] intel_idle.max_cstate=[0|正整数]**

  设置intel_idle驱动(CONFIG_INTEL_IDLE)允许使用的最大[C-state](http://www.expreview.com/25426.html)深度。"0"表示禁用intel_idle驱动，转而使用通用的acpi_idle驱动(CONFIG_CPU_IDLE)

- **idle=poll idle=halt idle=nomwait**

  对CPU进入[休眠状态](http://www.expreview.com/25426.html)的额外设置。 poll 从根本上禁用休眠功能(也就是禁止进入C-states状态)，可以略微提升一些CPU性能，但是却需要多消耗许多电力，得不偿失。不推荐使用。 halt 表示直接使用HALT指令让CPU进入C1/C1E休眠状态，但是不再继续进入C2/C3以及更深的休眠状态。此选项兼容性最好，唤醒速度也最快。但是电力消耗并不最低。 nomwait 表示进入休眠状态时禁止使用CPU的MWAIT指令。MWAIT是专用于Intel超线程技术的线程同步指令，有助于提升CPU的超线程效能，但对于不具备超线程技术的CPU没有意义。 [提示]可以同时使用halt和nomwait，也就是"idle=halt idle=nomwait"(但不是：idle=halt,nomwait)

- **intel_pstate=disable**

  禁用 Intel CPU 的 P-state 驱动(CONFIG_X86_INTEL_PSTATE)，也就是Intel CPU专用的频率调节器驱动

## 参考链接

* http://www.jinbuguo.com/kernel/boot_parameters.html
* https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html