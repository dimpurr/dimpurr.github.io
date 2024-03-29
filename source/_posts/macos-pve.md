---
title: 0202 年，在单分区 Proxmox 中安装 macOS Catalina (Hackintosh) 并直通硬件实现多系统
date: 2020-06-23 10:14:01
tags: 
- Virtual Machine
slug: macos-pve
---

{% asset_img "mac-info.png" "Hackintosh macOS info with Radeon VII" %}

促使我现在坐在这里生无可恋并心如死灰的撰写这篇文章的唯一原因，是我又手贱炸掉了 Proxmox 的 grub 引导。在如此接近胜利的黎明曙光，以至于我都用 GeekBench 在虚拟的 Hackintosh macOS 中都已经跑完了 CPU 多核和 GPU 直通的分数之后，就在我正愉快的准备修好 iService 以便享受 Apple Store 时，一次小小的崩溃和重启，配合遭天杀的 os-prober ，一切努力变成了屏幕上白花花的 GRUB > 。

人不作死，就不会死。

## 前戏

今晚恰好是 WWDC 的发布会，变得更像 iPadOS 的 ARM 版 macOS 的发布似乎正提前宣告着 AMD Mac 末日以及 Hackintosh 的死刑。尽管 Telegram 上难得的又一次几乎每个群都在因为同一个话题而热火朝天，而我却在因为 Sketch 和 MarginNote 和 OmniFocus 和 Logic X Pro 和下略的各种无关紧要的独占 App 开始严重的怀念起 macOS ，并决定把 Hackintosh 付诸实践以对得起当初下决心买 Radeon VII 并含泪折腾 ROCm 的日日夜夜。当我辛辛苦苦完成了 OpenCore 的 Gathering Files 并用 ProperTree 调教好了 config.plist ，瑟瑟发抖并战战兢兢的尝试从安装盘启动的时候，满怀期待的我看到了一行 ……

```
nvme: Fatal error occurred
```

接下来的对话是这样的：

```
> NVMEFix有了吗
- 有啊
> 忘了一件很重要的事 中途C2000Pro换成了Intel Intel不行 装MX500？
- 不是 我这 我难道得把 intel 拔掉？
> 不，屏蔽掉
- 。。。那我的 macos 永远不能访问这个 intel ssd ？
> 不能
- 东西都在里边诶。。。 我现在换一块硬盘来得及吗（
> 你可以换sn750 2t，不过一块要3000+
- 我这块多少钱来着。。
> 1700好像
- 啥原理，死刑？么有什么 patch ？
> 死刑 Intel家的基本都不行 mac pro那个群有阵子提到过
```

是的，我想起来了， P4510 不被 macOS 原生支持 …… 虽然我还有一块 1TB 的 Curcial MX550 SSD 从空间上来说是足够的，但是显然我没法接受在使用 macOS 的时候完全无法访问另外 2TB 的常用数据，更不要说这块硬盘中间有 512GB 的未使用分区已经为 macOS 预留了很久很久了。于是，诞生了一个非常奇葩的解决方案：

在这个分区安装 debian 。然后把 debian 升级成 Proxmox 。然后，在里面创建一个占满全盘（当然，还有全部内存）的 Hackintosh 虚拟机。然后，再依次进行各种硬件的直通。

那么下文的记录献给和我一样在错误的时间（指发布 ARM macOS 的 WWDC 深夜）用奇怪的组合（指单分区 debian 升级 Proxmox）进行迷惑操作（指在双 AMD 平台进行 OpenCore 硬件直通 Hackintosh）的你。

<!-- more -->

### Debian & Proxmox

刻一个 Debian 安装盘。注意下载的时候选右侧的 Live CD 而不是左侧的 Installer 。引导进去选 Graphical Installer 。

安装过程中选择 manual 分区，选中目标分区分配给 / 。不需要 swap ，毕竟内存有 64G 呢。强烈建议在这之前先手动把目标分区格式化成 FAT32 或者 exFAT 或者你乐意的话 ext4 都行，因为 Debian 的图形安装向导的分区工具实在是……屑。根本看不到 LABEL 也看不到 Data Used ，三个一模一样 512G 的分区根本分不清哪个是哪个。如果记错顺序或者手贱点错，凉凉。确认分区更改。

最困难的一步是……起计算机名。系统太多了名字不够用了。因为 Debian/Proxmox 宿主和里面的 macOS 需要分开的计算机名，所以最后决定分别是 dim-archer-kvm-madoka 和 DimMiyuMacMadoka 。（红 A 明明是小黑……但是 Chole 给 Kubuntu 用了，顺便 Sapphire 是 Elementary OS ，就差哪天再装个名为 Ruby 的 Manjaro Linux 了（逃））

其他时间用户名等就无关紧要了，如果有 http 代理可以适当设置，会很省心。重启进入系统，一个漂亮干净的 debian 。有事没事先 `su root` ，管理员滥权时间。

```bash
apt update
apt install htop vim
apt install openssh-server
```

没有什么比把 ssh 开起来更重要。这样你就可以在别的设备上复制粘贴代码而不是靠手敲了。因为 debian 的安装向导就算手动选了 ustc 等国内源也不会修改 secruity 源，所以你也许需要：

```bash
sudo sed -i 's|security.debian.org/debian-security|mirrors.ustc.edu.cn/debian-security|g' /etc/apt/sources.list
```

接下来 [Install Proxmox VE on Debian Buster](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Buster) 。官方教程很清晰，首先改 `vim /etc/hosts` 以免永远可以定位静态 ip 地址：

```bash
127.0.0.1	localhost
192.168.1.228	dim-archer-kvm-madoka.dimp.cloud	dim-archer-kvm-madoka
```

确认有效：

```bash
root@dim-archer-kvm-madoka:/home/dimpurr# hostname --ip-address
192.168.1.228
```

接下来就是一把梭安装了， Proxmox 的 USTC 源：

```bash
wget http://download.proxmox.com/debian/proxmox-ve-release-5.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-5.x.gpg
wget http://download.proxmox.com/debian/proxmox-ve-release-6.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg
chmod +r /etc/apt/trusted.gpg.d/proxmox-ve-release-6.x.gpg  # optional, if you have a non-default umask
echo "deb https://mirrors.ustc.edu.cn/proxmox/debian/pve buster pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
# 当然如果你有透明代理或者对网络很自信，这里是官方源:
# echo "deb http://download.proxmox.com/debian/pve buster pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
```

然后更新：

```bash
apt update && apt full-upgrade
```

并安装：

```bash
apt install proxmox-ve postfix open-iscsi
```

这里大概有 1.7G 的 apt 要 get 。透明代理真是太爽了！配合 IPLC 线路任何国外源都可以论秒拖完。我也许可以找个时间写个十分钟 PVE 创建 koolclash 软路由代理的记录文。

安装完触发器会弹出提示问你要不要配置 SAMBA （确认就好）和邮件服务（local only 就好）。

结束后，官方推荐你 `apt remove os-prober` 。它们就没考虑假如你还有其他系统要引导要怎么办。

安装完了重启，然后进 https://192.168.1.228:8006/ 类似的地址进管理页面，用 root 及其密码登陆。注意，是 `https` 。

### 网络桥接配置

也可以虚拟机创建完再做，但是虚拟机创建时就需要选网卡，所以提前做也可以。因为 proxmox 默认全新安装的网卡是桥接的，但是基于 debian 安装则不是，所以需要手动改一哈子。

在 Web 面板的 System > Network 中找到现有的网卡，编辑，网卡本身不要删掉，但是里面的 Field 内容全部删光否则会和网桥冲突，删除前记一下里面的内容。

新建一个 Linux Bridge ，地址填之前的静态地址以及斜杠掩码长度（例如 `192.168.1.228/24` ），网关也记得填对。Bridge Ports 写原来的网卡名比如 `enp38s0` 。

在 Apply config 之前记得先 `apt install ifupdown2` 否则会报错。然后点击 Apply Configuration ，上天保佑你的 Proxmox 不会忽然从网络上消失。

### 虚拟机的创建

主要参考 [Installing macOS Catalina 10.15 on Proxmox 6.1 or 6.2 using OpenCore](https://www.nicksherlock.com/2020/04/installing-macos-catalina-on-proxmox-with-opencore/) 这篇。

下载 [fetch-macOS.py](https://raw.githubusercontent.com/thenickdude/OSX-KVM/master/fetch-macOS.py) 然后选择最新的系统，获得 BaseSystem.dmg 。 macOS 下将其转换成硬盘镜像：

```bash
hdiutil convert BaseSystem.dmg -format RdWr -o Catalina-installer.iso
mv Catalina-installer.iso.img Catalina-installer.iso
```

原文还有 Linux 下的对应操作，这意味着你可以直接在 pve 中完成这一步。当然我选择 scp 传上去：

```bash
# on mac
➜  Hackintosh scp Catalina-installer.iso dimpurr@192.168.1.228:~
# on pve
root@dim-archer-kvm-madoka:/home/dimpurr# mv ~dimpurr/Catalina-installer.iso /var/lib/vz/template/iso
```

Win 怎么办我不知道，别问我。同理下载 [作者整好的 OpenCore 一键包](https://github.com/thenickdude/KVM-Opencore/releases/) （不久前才手动 gathering files 完毕的人表示这实在是太爽了！）。老办法，扔进上述 iso 目录。

然后整一个 `smc_read.c` 文件，原文链接已经被和谐了，实际内容如下（不要在意那个 10.5）：

```c
/*
 * smc_read.c: Written for Mac OS X 10.5. Compile as follows:
 *
 * gcc -Wall -o smc_read smc_read.c -framework IOKit
 */

#include <stdio.h>
#include <IOKit/IOKitLib.h>

typedef struct {
    uint32_t key;
    uint8_t  __d0[22];
    uint32_t datasize;
    uint8_t  __d1[10];
    uint8_t  cmd;
    uint32_t __d2;
    uint8_t  data[32];
} AppleSMCBuffer_t;

int
main(void)
{
    io_service_t service = IOServiceGetMatchingService(kIOMasterPortDefault,
                               IOServiceMatching("AppleSMC"));
    if (!service)
        return -1;

    io_connect_t port = (io_connect_t)0;
    kern_return_t kr = IOServiceOpen(service, mach_task_self(), 0, &port);
    IOObjectRelease(service);
    if (kr != kIOReturnSuccess)
        return kr;

    AppleSMCBuffer_t inputStruct = { 'OSK0', {0}, 32, {0}, 5, }, outputStruct;
    size_t outputStructCnt = sizeof(outputStruct);

    kr = IOConnectCallStructMethod((mach_port_t)port, (uint32_t)2,
             (const void*)&inputStruct, sizeof(inputStruct),
             (void*)&outputStruct, &outputStructCnt);
    if (kr != kIOReturnSuccess)
        return kr;

    int i = 0;
    for (i = 0; i < 32; i++)
        printf("%c", outputStruct.data[i]);

    inputStruct.key = 'OSK1';
    kr = IOConnectCallStructMethod((mach_port_t)port, (uint32_t)2,
             (const void*)&inputStruct, sizeof(inputStruct),
             (void*)&outputStruct, &outputStructCnt);
    if (kr == kIOReturnSuccess)
        for (i = 0; i < 32; i++)
            printf("%c", outputStruct.data[i]);

    printf("\n");

    return IOServiceClose(port);
}
```

编译并运行：

```bash
xcode-select --install # If you don't already have gcc
gcc -o smc_read smc_read.c -framework IOKit
./smc_read
```

如果版本相同，你大概会看到类似 `ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc` 这样的输出。（一开始还以为是假的（笑））

接下来是虚拟机的创建，在 Web 面板完成。原文图文并茂讲的贼详细，我就不赘述了。要点：

- Guest OS 选择 other 然后选中 OpenCore.iso
- Graphic card 选 VMware compitable ， SCSI 控制器选 VirtIO ， BIOS 选 OVMF ，机器 q35 ，给 Add EFI 选一个 storage
- Disk 选 Write Back (unsafe) ， Discard ， SSD Emulation 都开起来，自己定大小，扔到 SATA 0
- CPU 选 Penryn ，酌情给个 8 cores 之类的
- Memory 不要 Balloning ，空间酌情，我给了 61,440 （60G）
- Network 用 VMware vmxnet3 选刚做的 vmbr0

不要着急启动，去 VM 的 Options 页面确保 use tablet for pointer 打开。去 Hardware 新建一个 CD/DVD ，把 Catalina-installer.iso 挂到 IDE 上。

然后编辑 `vim /etc/pve/qemu-server/100.conf` （记得换成自己的 vm id ），增加一行（记得替换对应的 osk）：

```bash
# intel
args: -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -cpu host,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc
# amd
args: -device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc" -smbios type=2 -device usb-kbd,bus=ehci.0,port=2 -cpu Penryn,kvm=on,vendor=GenuineIntel,+kvm_pv_unhalt,+kvm_pv_eoi,+hypervisor,+invtsc,+pcid,+ssse3,+sse4.2,+popcnt,+avx,+avx2,+aes,+fma,+fma4,+bmi1,+bmi2,+xsave,+xsaveopt,check
```

然后找到两个 iso ，删掉 `,media=cdrom` 同时增加 `cache=unsafe` 以便让他们被识别为硬盘，最后看起来像这样：

```bash
ide1: local:iso/Catalina-installer.iso,cache=unsafe,size=2095868K
ide2: local:iso/OpenCore.iso,cache=unsafe
```

最后配置 Proxmox 以避免 macOS 无限循环引导：

```bash
echo 1 > /sys/module/kvm/parameters/ignore_msrs # 本次生效
echo "options kvm ignore_msrs=Y" >> /etc/modprobe.d/kvm.conf && update-initramfs -k all -u # 持久设置
```

Debian 在 root 下可能会告诉你 `bash: update-initramfs: command not found` ，不怕， `export PATH=/sbin:$PATH` 再来就好。

如果配置虚拟机的时候 `vmgenid` 报错，删掉这一行就好。

## 安装

可以启动虚拟机了。去 Option 下启动顺序改成 IDE 2 第一个。

选择 macOS Base System ，用 Disk Utilty 把全盘格成 APFS 。然后正常安装就行。中间会重启一次，第二次不要再选这个，选 macOS Installer 以便继续安装。再下一次重启就选择你命名的主硬盘。

由于我这里是偷懒用的非离线镜像，中间安装程序自动 DHCP 出了点问题导致连不上网提示 `The recovery server could not be contacted` 或 An Internet connection is required to install macOS` ，这里可以打开 Terminal 手动设置网关：

```bash
route delete default
route add default 192.168.1.1
```

安装完重启进入配置向导，记得前往不要配置 Apple ID ！否则坐等被 Ban 号。然后就可以欣赏美丽的 macOS 欢迎界面了。

## 硬件直通

（待续）
