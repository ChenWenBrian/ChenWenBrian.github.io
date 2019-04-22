# 单网卡玩转智能网关——把OpenWRT塞进虚拟机

玩过智能路由器的同学可能都知道，国内比较火的koolshare论坛提供的梅林固件的那个强大：科学上网、游戏加速、自动签到、VPN服务器、FTP服务器、智能去广告、动态DNS、内网穿透、BT下载、私有云、KMS等众多扩展软件，让你的路由器从此变的强大无比。但是呢，梅林固件支持的路由器毕竟有限，我们不一定刚好拥有，另外如果我们想在工作场所一样拥有强大且不受限的网络访问能力怎么办呢？这里我们就可以考虑软路由了。

软路由是指利用台式机或服务器配合软件形成路由解决方案，主要靠软件的设置，达成路由器的功能；而硬路由则是以特有的硬设备，包括处理器、电源供应、嵌入式软件，提供设定的路由器功能。常见的有海蜘蛛、OpenWRT、DDWRT、Tomato等，这些系统共有的特点是一般对硬件要求较低，甚至只需要一台486电脑，一张软盘，两块网卡就可以安装出一台非常专业的软件防火墙。

## 系统需求：
- Windows 7/8/10 64位：专业版，企业版，教育版
- BIOS开启虚拟化（因为需要使用hyper-v或者vmware跑OpenWRT）
- 最少4GB内存（要开虚拟机，低于4GB也能玩，但不建议长时间使用）
- Hyper-V或者Vmware Player等虚拟机软件

>PS：如果系统版本不够，无法使用hyper-v，可以考虑将系统升级到Windows 10 pro，譬如我的工作电脑原本是Windows 10 Home版，换个pro的key顺利升级为专业版。如果不具备升级条件，那么就考虑vmware player吧，效果是完全一样的。

## 下载OpenWRT镜像与转盘工具

第一步是系统开启hyper-v（本文全文以hyper-v为例，vmware操作基本类似），这个网上教程还是很多的，这里我就不赘述了。首先去[koolshare](http://firmware.koolshare.cn/LEDE_X64_fw867/)下载[OpenWRT](http://firmware.koolshare.cn/LEDE_X64_fw867/%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BD%AC%E7%9B%98%E6%88%96PE%E4%B8%8B%E5%86%99%E7%9B%98%E4%B8%93%E7%94%A8/openwrt-koolshare-mod-v2.30-r10402-51ad900e2c-x86-64-uefi-gpt-squashfs.img.gz)的镜像文件。

>需要注意的是，koolshare官网的[下载目录](http://firmware.koolshare.cn/LEDE_X64_fw867/)里，有一个目录是`虚拟机转盘或PE下写盘专用`，点击进来有两个镜像文件，一个是非UEFI的兼容镜像，支持hyper-v一代机；另一个uefi-gpt镜像可以使用hyper-v的二代机。本文以第二个uefi-gpt的镜像为例展开。

文件不大，40MB+多点儿，下载下来后由于是img.gz结尾，我们需要先用7-zip等压缩软件解压得到img文件，由于hyper-v的硬盘文件为vhdx，所以我们还需要虚拟机磁盘文件转换的工具[StarWind V2V Converter](https://www.starwindsoftware.com/starwind-v2v-converter)，该工具官网可以免费下载使用。

## 转写hyper-v硬盘文件

将下载下来的`openwrt-koolshare-mod-v2.30-r10402-51ad900e2c-x86-64-uefi-gpt-squashfs.img.gz`文件，用7-zip解压，的到`openwrt-koolshare-mod-v2.30-r10402-51ad900e2c-x86-64-uefi-gpt-squashfs.img`。然后启动StarWind V2V Converter，按如下步骤操作完成镜像转换：

- 双击打开StarWind V2V Converter
- 在Select location of image to convert界面，选择Local file
- 在Source image界面，选择你解压出来的`openwrt-koolshare-mod-v2.30-r10402-51ad900e2c-x86-64-uefi-gpt-squashfs.img`文件
- 在Select location of destination image界面，选择Local file
- 在Select destination image format界面，选择VHD/VHDX
- 在Select option for VHD/VHDX image format界面，选择VHDX growable image
- 在Set destination file name界面，选择文件保存地址，这里默认即可
- 完成转换，得到`openwrt-koolshare-mod-v2.30-r10402-51ad900e2c-x86-64-uefi-gpt-squashfs.vhdx`文件

## 配置虚拟机

