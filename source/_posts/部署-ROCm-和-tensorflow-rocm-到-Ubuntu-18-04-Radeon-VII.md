---
title: 部署 ROCm 和 tensorflow-rocm 到 Ubuntu 18.04 (Radeon VII)
date: 2020-04-22 04:58:53
tags:
---

ROCm 实在是太多坑了 …… 用剩下的最后一口气吐槽+写文记录。

### 前因

（这些都是废话，技术目的阅读请直接跳过）

因为一开始就计划要 Hackintosh 然后给 itx (机器名 madoka) 选装了 Radeon VII 。 HIS 公版，方方铁盒子信仰小红灯外加三个风扇，看起来就很生猛。这个价位可以买到的性价比最高的算力。更重要的是 macOS 可以免驱直插。 Mac Pro 同款。 7nm 制程（并没有什么卵用）， 16GB 显存富得流油（相对真正的实力来说根本不算什么）。隔壁 NIVIDA ，就算是 MacBook 官方提供的型号，更新 Mojave 之后一样驱动凉凉， N 卡用户没人权。再说我也没什么游戏需求， 3DMark 跑分看起来也七七八八，除了不限制电压的话风扇火力全开 4k rpm 噪音直接航天飞机起飞，其他一切都看起来很完美。

……直到我想搞深度学习（暴露了这家伙以前都没好好跑过几个网络的事实）。

人类回想起了被老黄 CUDA 统治的恐惧和你 AI 届 A 卡用户没人权的事实。 AMD 一点也不 Yes 。 ROCm 支持框架屈指可数，支持的还有一半在 in development ，完全不支持 Windows ，嗯 …… （科研人员：我们都是工费拨款，买来一块 Titan XP 插上就好了！

我烦恼了很久很久因为我真的很想一边跑 Tensorflow 一边开着 DAW 或者非编软件尤其是可爱的 Touch Designer 折腾可视化啊！事实已经证明了我的灵魂是媒体工作者，虽然我想搞 Deep Learning 但我很确信我大部分时间会拿现成网络倒腾倒腾跑应用，并且输入输出大概率全是媒体文件。不知道，其实我是瞎说的。所以我非常希望能够在本地跑，第一时间预览加调试，外加也许会有大量尺寸不小的数据集（比如自己整的动画或者一时兴起爬了个 exhentai ），实在不想必须提交云算力、或者把调试网络的系统环境与其他工作完全分开，每次想跑 AI 就必须停下手头全部其他事情然后重启进系统专注调参一发入魂 ……

以上这些毫无根据的执著以及自己钱太多的幻觉（信用卡警告）以至于此人花费超过半天时间研究雷蛇和技嘉的显卡盒以及DIY显卡核的可操作性和性能影响、发现itx上双显卡主板是完全不存在的伪需求、发现SLI双卡交火完全没什么意义更不要说A/N双卡打架以及自己的 MacBook Pro mid 2014 是雷电2会让外接速率雪上加霜。甚至还考虑相信 12 核 24 线程的 3900x 并使用 CPU 跑了一下 tensorflow benchmark ，结果 …… 速度是 VII 参考性能的 1/40 （ [http://ai-benchmark.com/ranking_deeplearning.html](http://ai-benchmark.com/ranking_deeplearning.html) > Desktop GPUs and CPUs）。最后认真考虑了乖乖接受 NAS + wireguard 虚拟 NAT 使用云算力的方案之后，打算随便本地再开个系统装个 ROCm 跑通跑个分备用了事了。

然后， Elementary OS 拯救了我。它太美了。

（以下正文）

### ROCm 安装

官方文档：https://rocm-documentation.readthedocs.io/en/latest/Installation_Guide/Installation-Guide.html

这里是本机直接安装， ROCm 驱动并不会影响正常显示使用，如果是 docker 安装则需要屏蔽原有驱动并导致分辨率降低，若需要 docker 教程请另行寻找。

首先更新系统然后安装基础依赖：

```bash
sudo apt update
sudo apt dist-upgrade
sudo apt install libnuma-dev
sudo reboot
```

重启了吗？然后添加 ROCm 的 ppa repo ：

```bash
wget -q -O - http://repo.radeon.com/rocm/apt/debian/rocm.gpg.key | sudo apt-key add -
echo 'deb [arch=amd64] http://repo.radeon.com/rocm/apt/debian/ xenial main' | sudo tee /etc/apt/sources.list.d/rocm.list
```

这里有个很微妙的点，看到那里写的 `xenial` 了吗？千万不要像我一样自作聪明觉得可以改成 `bionic` 。这个 repo 里面根本没有 `bionic` 目录。至少现在还没有。

接下来更新源并安装：

<!-- more -->

```bash
yarn global add hexo # npm install -g hexo-cli
cd dimpurr.github.io
git checkout -b hexo
hexo init tmp
mv ./tmp/* .
rm -rf tmp # rmdir tmp on windows
```

```bash
sudo apt update
sudo apt install rocm-dkms
```

由于 Radeon 自己的源不能被 USTC / TUNA 等 Ubuntu deb 源加速，所以国内速度往往很慢（ 40MB 40kb/s ~1h），建议临时给 apt 增加全局代理：

```bash
sudo vi /etc/apt/apt.conf.d/proxy.conf
Acquire {
  HTTP::proxy "http://127.0.0.1:1080";
  HTTPS::proxy "http://127.0.0.1:1080";
}
```

用完记得删掉。

安装完了吗？给你的用户改一下用户组，只有 video 组才可以访问 ROCm 资源。如下：

```bash
groups # 查看当前用户组
sudo usermod -a -G video $LOGNAME # 给当前用户添加用户组
# 以后新增用户自动进入 video 用户组
echo 'ADD_EXTRA_GROUPS=1' | sudo tee -a /etc/adduser.conf
echo 'EXTRA_GROUPS=video' | sudo tee -a /etc/adduser.conf
# 将 ROCm 环境变量添加到用户
echo 'export PATH=$PATH:/opt/rocm/bin:/opt/rocm/profiler/bin:/opt/rocm/opencl/bin/x86_64' |
sudo tee -a /etc/profile.d/rocm.sh
```

重启（甚至其实根本不用重启），接下来执行：

```
/opt/rocm/bin/rocminfo
/opt/rocm/opencl/bin/x86_64/clinfo
```

正确输出了吗？恭喜你，成功啦！到此结束。

个P。如果你到这里还没踩到坑，那你重启两遍。如果还没有坑，那么你用的不是 Ubuntu / Debian ，而且还拥有一个完美的内核版本，当我什么都没说。请直接跳到 `tensorflow-rocm` 部分继续阅读。

#### 重启出现 HSA_STATUS_ERROR_OUT_OF_RESOURCES

基本上如果是比较干净的系统的话，基本都能一口气跑到尾，甚至可以正常输出 `rocminfo` 等信息。然后重启一次，一切就都……变化了模样：

```bash
Failed to get user name to check for video group membership
hsa api call failure at: /data/jenkins_workspace/compute-rocm-rel-2.9/rocminfo/rocminfo.cc:1102
Call returned HSA_STATUS_ERROR_OUT_OF_RESOURCES: The runtime failed to allocate the necessary resources. This error may also occur when the core runtime library needs to spawn threads or create internal OS-specific events.
```

相关 ISSUE ： https://github.com/RadeonOpenCompute/ROCm/pull/1005

血红血红的。而且第一句特别蠢，怎么连用户名也没拿到啊（实际上这句话没什么影响）？这种情况下 `clinfo` 也无法执行，但是 `rocm-smi` 仍旧完全正常。根据上面那个 ISSUE 的说法，是用户组问题。我看也是。这个 ISSUE 关联的 Pull Request 中修改了 README.md ，提供了一套解决方法： https://github.com/RadeonOpenCompute/ROCm/pull/1005/files 但我测试并没有效果。

我是怎么解决的呢？我翻遍了 Google 前三页都没解决，然后在 [CSDN](https://blog.csdn.net/zaq15csdn/article/details/104743655) 上看到了一句话： sudo 一下试试。

`sudo rocminfo` （其实这样是没有环境变量的，所以一开始应该是 `sudo /opt/rocm/bin/rocminfo`）

……成了。

天知道为什么可以，我的 root 肯定不在 video 组，更不要说上面 ISSUE 认为需要的 render 组了。反正今后 `sudo su` 就能用。何乐而不为啊！

#### 在 `apt install rocm-dkms` 过程中出现 `Error! Bad return status for module build on kernel`

```bash
Building initial module for 5.3.0-46-generic
Error! Bad return status for module build on kernel: 5.3.0-46-generic (x86_64)
Consult /var/lib/dkms/amdgpu/3.3-19/build/make.log for more information.
```

啥事都没有，放心好了。这个 log 又长又什么都看不出来。 ROCm 也安装好了也能用。忽略就行。至少我这行。

### 官方 `amdgpu` / `amdgpu-pro` 驱动安装卸载相关问题

这其实是一个根本不存在的问题，尤其是如果你从全新系统开始。 ROCm 是不需要提前手动装其他驱动的。

#### 不必要提前安装驱动。不是 docker 版本不需要提前屏蔽驱动。

但是！也不能没有驱动。 ROCm 需要有一个 amdgpu 驱动在运行。屏蔽驱动的方法是：

```bash
sudo vim /etc/modprobe.d/blacklist.conf
#: blacklist amdgpu

sudo vim /etc/modprobe.d/blacklist-radeon.conf
#: blacklist radeon
# 这一行也可以手动直接写在上面的文件，但是如果安装过官方驱动，就会自动创建这个文件
```

告诉你这个是为了让你明白……不要屏蔽驱动。如果不知道什么情况下屏蔽了就去注释掉。

不过如果你不知道为什么装过其中一个驱动，你就会在后面 `sudo apt install rocm-dkms` 的时候碰到：

```bash
dpkg: error processing archive /var/cache/apt/archives/rock-dkms_1.8-151_all.deb (--unpack):
trying to overwrite '/usr/share/dkms/modules_to_force_install/amdgpu', which is also in package amdgpu-dkms 18.10-572953
dpkg-deb: error: subprocess paste was killed by signal (Broken pipe)
```

去网上搜一圈这里会有教你强制 overwrite 的。但恕我对 dpkg 一窍不通，我觉得并没有什么用。还是建议直接把驱动卸载掉。

### 官方驱动安装方法

以 Radeon VII 为例（应该都大同小异），[VII 驱动下载地址](https://www.amd.com/en/support/graphics/amd-radeon-2nd-generation-vega/amd-radeon-2nd-generation-vega/amd-radeon-vii) ，下载后 `tar xf amdgpu-*-ubuntu-18.04.tar.xz` 解压然后 `cd amdgpu*` 进入文件夹。

如果你要安装普通版驱动，那么 `./amdgpu-install` 。不需要提前 sudo ，后面会叫你输入密码的。如果是 Pro 驱动，那么 `./amdgpu-pro-install` 就好了。就是这么简单，快捷，以及人性化。 AMD 万岁。

……要是真的话就好了。

#### 官方驱动强制卸载方法

这个脚本有一万种情况可以出错然后喊你用 `amdgpu-uninstall` / `amdgpu-pro-uninstall` 命令或者直接在安装脚本后面加 `--uninstall` 命令来清理不正确的安装否则就完全不能继续，包括但不限于你挂了代理（是的，以我个人经验，建议请务必在下载 rocm 的时候打开 apt 代理，而同时务必在安装驱动（不，你根本就不应该装，除非……你是为了完整的安装以便卸载干净它……）的时候关掉代理）， apt 缓存的包有问题，编译哪里掉了链子以及其他一切。虽然几率其实不高，但是一旦发生了 ……

你会遇到这种错误：

```bash
You might want to run 'apt --fix-broken install' to correct these.
The following packages have unmet dependencies:
amdgpu-dkms : Depends: amdgpu-core but it is not going to be installed
amdgpu-lib-hwe : Depends: amdgpu-core (= 19.30-934563) but it is not going to be installed
amdgpu-pro-core : Depends: amdgpu-core but it is not going to be installed
gst-omx-amdgpu : Depends: amdgpu-core but it is not going to be installed
libdrm-amdgpu-common : Depends: amdgpu-core but it is not going to be installed
libdrm2-amdgpu:i386 : Depends: amdgpu-core:i386
libdrm2-amdgpu : Depends: amdgpu-core but it is not going to be installed
libegl1-amdgpu-mesa:i386 : Depends: amdgpu-core:i386
libegl1-amdgpu-mesa : Depends: amdgpu-core but it is not going to be installed
libgbm1-amdgpu:i386 : Depends: amdgpu-core:i386
libgbm1-amdgpu : Depends: amdgpu-core but it is not going to be installed
...
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).
```

（有人会叫你 `sudo dpkg -i --force-overwrite /var/cache/apt/archives/rock-dkms_2.10-14_all.deb` ）

还有这种错误：

```bash
dpkg: error processing package amdgpu (--remove):
 dependency problems - not removing
Errors were encountered while processing:
 amdgpu
```

还有前面提过的 overwrite 错误：

```bash
Preparing to unpack .../libdrm-amdgpu-common_1.0.0-633530_all.deb ...
Unpacking libdrm-amdgpu-common (1.0.0-633530) ...
dpkg: error processing archive /var/opt/amdgpu-pro-local/./libdrm-amdgpu-common_1.0.0-633530_all.deb (--unpack):
 trying to overwrite '/opt/amdgpu/share/libdrm/amdgpu.ids', which is also in package ids-amdgpu 1.0.0-606296
Errors were encountered while processing:
/var/opt/amdgpu-pro-local/./libdrm-amdgpu-common_1.0.0-633530_all.deb
```

（参考 [askubuntu post](https://askubuntu.com/questions/1068344/help-installing-amd-gpu-driver-in-ubuntu-18-04#comment1751443_1068344) ）

…… 还有长成各种各样的。这不重要，总而言之，最为行之有效的解决方案是 StackExchange 的 [某个帖子](https://askubuntu.com/questions/363200/e-unable-to-correct-problems-you-have-held-broken-packages) 藏在评论区的某个角落里的方法：

```bash
sudo apt-get install synaptic
```

是的你没看错，就是传说中的 新 立 德 。GUI，打开！ Edit > Fix Broken Packages ！然后在 Package 列表中用 Broken 过滤器，全部 Mark Completion Removal ！这还没完，去列表里定位 `amdgpu` 或者 `amdgpu-pro` ，刚才出错的是那个就右键 Mark for Installation 哪个，记得点顶上的 Apply ，安装！

这可比 amdgpu-install 那脚本好使多了。吭哧吭哧安装上了，很好。那么，可以愉快的 `amdgpu-install --uninstall` 了…… （是的，这个安装不会在命令行添加 `amdgpu-uninstall` 命令）

报 apt lock 冲突的话别忘记把 synaptic gui 关掉啊。

#### elementrary 或者其他野鸡 OS 适配

一个无足轻重的小问题。稍有常识的人都能自己想到的凑字数段落。

如果安装脚本报告 `Unsupported DEB-based OS: elementary` ，那么用一个你趁手的编辑器打开脚本，搜索 `ubuntu|linuxmint|debian` 并改成 `ubuntu|linuxmint|debian|elementary` 。其他系统同理。

### tensorflow-rocm 安装

如果你的 rocm 就位了，你可以正常输出 `rocminfo` ，就可以来安装 tensorflow-rocm 了（ [官方文档](https://rocm-documentation.readthedocs.io/en/latest/Deep_learning/Deep-learning.html#tensorflow-installation) ）。

首先去 Python 官网下载 Python 3.7 ，太高不行哦，注意目前 tensorflow 2.0.0 官方支持的版本是 3.5-3.7 。然后记得安装 pip 。事实上可以 `sudo apt install python3-pip` 这样搞定的。可以提前搞个国内源。

然后记得先把原版（CUDA 版本）的 tensorflow 卸载干净哦。 `pip list | grep tensorflow` 看一下，有残留的就 `pip uninstall` 掉。

然后按官方教程操作如下：

```bash
sudo apt update
sudo apt install rocm-libs miopen-hip cxlactivitylogger rccl
pip3 install --user tensorflow-rocm
# 或者
python3.7 -m pip install --user tensorflow-rocm --upgrade
```

大功告成！虽然 `E: Unable to locate package cxlactivitylogger` 了，但是你打算假装没看到（而且确实没有什么后果）。你愉快的打开 Python 交互式命令行，并打算来一个伟大的 Hello World ：

```python
import tensorflow as tf
# 等等，我还没来得及输入下一句 ……
print(tf.reduce_sum(tf.random.normal([1000, 1000])))
```

然后：

```bash
ImportError: librccl.so.1: cannot open shared object file: No such file or directory
Failed to load the native TensorFlow runtime.
ModuleNotFoundError: No module named 'apt_pkg'
```

眼尖的你一样就看出了问题所在，手疾眼快的打出：

```bash
sudo apt install rccl
```

或者，沉稳而又喜欢保险的你，快速的 Ctrl + V 出：

```bash
sudo apt-get install -y --allow-unauthenticated rocm-dkms rocm-dev rocm-libs rccl rocm-device-libs hsa-ext-rocr-dev hsakmt-roct-dev hsa-rocr-dev rocm-opencl rocm-opencl-dev rocm-utils rocm-profiler cxlactivitylogger miopen-hip miopengemm
```

`rocm-profiler` 和 `cxlactivitylogger` 又找不到，这怎么能难住你，反正本来其他的包装了也不知道有什么用，那就去掉这俩呗：

```bash
sudo apt-get install -y --allow-unauthenticated rocm-dkms rocm-dev rocm-libs rccl rocm-device-libs hsa-ext-rocr-dev hsakmt-roct-dev hsa-rocr-dev rocm-opencl rocm-opencl-dev rocm-utils miopen-hip miopengemm
```

让我们再来一次 `import tensorflow as tf` ，大功告成，欢呼雀跃，四海奔腾，普天同庆！

###  [tensorflow/benchmarks](https://github.com/tensorflow/benchmarks/) 跑分

点进去 repo 然后拉下文件 （你竟然懒到点击一个链接，于是贴心的我给你准备好了： `git clone https://github.com/tensorflow/benchmarks.git` ），然后 `cd scripts/tf_cnn_benchmarks` ，有用户组问题的话记得 `sudo su` 一下，然后用以下命令（或者 everything you want）：

```bash
python3.7 tf_cnn_benchmarks.py --num_gpus=1 --batch_size=64 --model=resnet50
```

土豪请按需改 `num_gpus` 。如果想知道你买 GPU 有多值并想虐待你任劳任怨的 CPU 的话，请去掉这一项（不能填0！不要问我为什么知道）[并加上必要的参数](https://github.com/tensorflow/benchmarks/issues/82#issuecomment-342625947) `--data_format=NHWC --device=cpu` 。

运行会快速滚屏初始化，然后这里会忽然报错并卡住：

```bash
MIOpen(HIP): Warning [ForwardBackwardGetWorkSpaceSizeImplicitGemm] /root/driver/MLOpen/src/lock_file.cpp:75: Error creating file </root/.config/miopen//miopen.udb.lock> for locking.
```

貌似这个报错并不重要，而且有时候不会出现。此后会一段时间没有显示，是在 `warm up` 和实时编译（ROCm 这里还要对 CUDA 做一次转换，所以速度会比 N 卡更慢一些，我这里花了足足 5 分钟），可以用 `htop` 监视 CPU 占用（基本只会跑满两三个单核），还可以看到不断刷新的 `clang` 进程。

也可以用 `watch -n 1 rocm-smi` 实时监控显卡资源情况：

```
========================ROCm System Management Interface========================
================================================================================
GPU  Temp   AvgPwr  SCLK     MCLK    Fan    Perf  PwrCap  VRAM%  GPU%  
0    55.0c  25.0W   1546Mhz  800Mhz  54.9%  auto  250.0W   99%   0%    
================================================================================
==============================End of ROCm SMI Log ==============================
```

会发现 VRAM 很快就满了。恩 …… 16G 看来也不怎么够啊（逃

### 参考成绩

这个 ISSUE 里有大量相同参数的跑分可供对比： https://github.com/ROCmSoftwarePlatform/tensorflow-upstream/issues/173

以下是我个人的记录，跑的同时系统日常后台负载（浏览器，播放器，IM等），没有什么特别性能处理。

```
OS: elementary OS 5.1.3 Hera x86_64
Kernel: 5.3.0-46-generic 
```

##### GPU: AMD Radeon VII 16GB

```
Step  Img/sec  total_loss
1  images/sec: 241.7 +/- 0.0 (jitter = 0.0)  8.220
10  images/sec: 241.6 +/- 0.2 (jitter = 0.5)  7.880
20  images/sec: 241.3 +/- 0.2 (jitter = 0.7)  7.910
30  images/sec: 241.3 +/- 0.2 (jitter = 0.9)  7.820
40  images/sec: 236.9 +/- 2.0 (jitter = 1.1)  8.004
50  images/sec: 237.3 +/- 1.6 (jitter = 1.3)  7.768
60  images/sec: 237.8 +/- 1.3 (jitter = 1.2)  8.115
70  images/sec: 237.7 +/- 1.2 (jitter = 1.5)  7.815
80  images/sec: 238.0 +/- 1.0 (jitter = 1.4)  7.975
90  images/sec: 238.4 +/- 0.9 (jitter = 1.3)  8.101
100  images/sec: 238.7 +/- 0.8 (jitter = 1.2)  8.042
-
total images/sec: 238.68
```

##### CPU: AMD Ryzen 9 3900X 12- (24) @ 3.800GHz 

```
Step    Img/sec total_loss
1       images/sec: 5.7 +/- 0.0 (jitter = 0.0)  8.108
10      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  8.122
20      images/sec: 5.7 +/- 0.0 (jitter = 0.1)  7.983
30      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.780
40      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.849
50      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.779
60      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.825
70      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.840
80      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.818
90      images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.646
100     images/sec: 5.8 +/- 0.0 (jitter = 0.1)  7.913
----------------------------------------------------------------
total images/sec: 5.26
----------------------------------------------------------------
```

这个是来搞笑的。不得不承认人类文明一直在进步。

![200422-3900x-benchmark](200422-3900x-benchmark.jpg)

![200422-3900x-monitor](200422-3900x-monitor.jpg)

![200422-gpu-drinking-tea](200422-gpu-drinking-tea.jpg)

真·12核有难GPU围观。

### 结束

有一个槽一直忘记吐：

![200422-elementary](200422-elementary.png)

在 Linux 下 Radeon VII 无论在 **任何地方** 都会被认做 Vega 20 。这两张卡到底有什么关系啊 ……

以及， Elementary 真是太美了。上一次用还是如此久远的 [Elementary OS Luna 第一天 | 钉子の次元](http://blog.dimpurr.com/elementary-first/) (2018-08-13) ，要不是 [Menci](http://men.ci/) 提到我都完全忘记了这个发行版的存在 …… 然而本来随便划分了 32G 打算献祭给 ROCm 的 eOS 就这样让我愉快的逃离了 Kubuntu 的魔爪，成为了一个我完全不会想要急着换回 Windows 或者 macOS 的主役日用 Linux 系统。请容我说， eOS 的开箱即用&界面&窗口管理&日常体验实在是太优雅太舒服了！！

…… 唯一的缺点是基于 GTK 的 DE 在我 32‘ 2160p 的屏幕上没法 150% HiDPI 缩放，只能 2x 凑合用，好看但是牺牲了不少屏幕空间。而 KDE 虽然各种错乱，但也有一个聊胜于无的 1.5x ……

嘛总之，设计有 Figma ，画画有 Krita ， DAW 还有 BitWig 和 REAPER ，非编可以 DaVinci Resolve …… 这些都能解决，别的还叫事吗！ QQ WeChat 哪个不能 Wine ，日常笔记 SimpleNote Evernote 哪个不是全平台，更别说听歌 Apple Music Youtube Music 到哪都能 last.fm Scrobbler ，根本懒得重启的我直接安装 Android Studio 和 WPS 开始准备写学校实验报告了。操作系统，于我如浮云哉～

（这个人选择性忽视了还有 16 个小时就是移动开发课程作业 DDL 然而这个人还彻夜折腾 ROCm 至今项目一点没动的事实