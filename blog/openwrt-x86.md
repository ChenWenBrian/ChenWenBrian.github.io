# 定制原生X86版OpenWRT

## 准备工作：
- 下载OpenWRT官方镜像：https://firmware-selector.openwrt.org/?version=23.05.4&target=x86%2F64&id=generic
- 下载starwind v2vconverter，用于将官方镜像转换为虚拟机格式：https://www.starwindsoftware.com/v2vconverter

## 转写hyper-v硬盘文件

将下载下来的`openwrt-23.05.4-x86-64-generic-ext4-combined-efi.img.gz`文件，用7-zip解压，得到`openwrt-23.05.4-x86-64-generic-ext4-combined-efi.img`。然后启动StarWind V2V Converter，按如下步骤操作完成镜像转换：

- 双击打开StarWind V2V Converter
- 在Select location of image to convert界面，选择Local file
- 在Source image界面，选择你解压出来的`openwrt-23.05.4-x86-64-generic-ext4-combined-efi.img`文件
- 在Select location of destination image界面，选择Local file
- 在Select destination image format界面，选择VHD/VHDX
- 在Select option for VHD/VHDX image format界面，选择VHDX growable image
- 在Set destination file name界面，选择文件保存地址，这里默认即可
- 完成转换，得到`openwrt-23.05.4-x86-64-generic-ext4-combined-efi.vhdx`文件

## 配置网络和虚拟机

首先确保你已经安装了Hyper-V组件。在开始按钮右边的搜索栏里输入`hyper-v`，在弹出的搜索结果里点击`Hyper-V管理器`。

### 1.创建虚拟交换机
在弹出的`Hyper-V管理器`的右边，点击`虚拟交换机管理器`，按如下步骤依次创建桥接模式的虚拟交换机：
- 左侧选择`新建虚拟交换机`，右侧选择`外部`，点击`创建虚拟交换机`
- 在新出现的虚拟交换机的右侧名称处，填写`Bridge Switch`，连接类型保持`外部网络`，网卡下拉列表里务必选择物理机的真实物理网卡。这里是以单网卡为例讲解，一般默认的网卡即你的外部网络访问网卡，如WiFi网卡。如果你有多个物理网卡，请选择你需要桥接的网卡。
- 保持勾选`允许管理操作系统共享此网络适配器`
- 点击确认完成交换机的创建

### 2.创建虚拟机
在`Hyper-V管理器`的右边，点击`新建`->`虚拟机`，按如下步骤依次创建虚拟机：
- 指定名称和位置：名称填Openwrt
- 指定代数：这里我们选择第二代，因为前面我们下载的是支持uefi的镜像，所以可以选择第二代，否则请使用第一代
- 分配内存：这里填512MB足够了，如果你的内存够多，分个1GB的内存也没关系
- 配置网络：这里选择刚刚创建的`Bridge Switch`
- 连接虚拟硬盘：这里选择使用现有的虚拟硬盘，文件即前面我们转换出来的`openwrt-23.05.4-x86-64-generic-ext4-combined-efi.vhdx`文件
- 摘要：最后确认下没问题就点完成，开始创建虚拟机

### 3.配置虚拟机
在`Hyper-V管理器`的中间，我们可以看到刚刚创建的名为`Openwrt`的虚拟机，右键选择设置：
- 在设置窗口的左侧，找到安全选项卡：去掉右侧的`启用安全启动`勾选
- 内存：512MB足够，如果你的内存不多的话，还可以启动动态内存，并将最小RAM设置为256MB
- 处理器：1个虚拟处理器足够
- 自动启动操作：如果想Windows启动就让软路由提供服务，可以选择
- 自动停止操作：选择强行关闭虚拟机

完成以上配置后，即可启动虚拟机，并连接上去看控制台的输出。待控制台输出稳定后，回车输入`ifconfig`可以查看到当前的网卡配置。其中，br-lan网卡的IP地址即我们桥接上去的网卡地址，如果默认的IP地址的地址段与物理网卡的地址段不一致，我们可以输入`vim /etc/config/networt`打开网络配置的编辑窗口，找到第三块的`config interface 'lan'`设置并修改响应的IP地址与网关地址。参考配置如下：

```ini
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.1.10'    #修改为可用的固定IP地址
        option netmask '255.255.255.0'    #子网掩码与你的物理网卡保持一致
        option ip6assign '60'
        option multipath 'off'
        option gateway '192.168.1.1'    #网关地址与你的物理网卡保持一致
        option dns '192.168.1.1 223.6.6.6'    #DNS地址与你的物理网卡保持一致

```

如果你是通过DHCP动态获取IP地址，无法确定可用的固定IP地址，也可以使用DHCP动态获取openwrt的IP地址。参考配置如下：
```ini
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'dhcp'
        option multipath 'off'
        option delegate '0'
        option peerdns '0'
        option dns '192.168.1.1 223.6.6.6'   #DNS地址与你的物理网卡保持一致
        option clientid 'a1'    #clientID在某些DHCP服务器上需要，防止通过WiFi网卡桥接时，获取到的IP地址与物理网卡重复，可选

```

完成以上修改后，保存并重启虚拟机。虚拟机重启后，在cmd里通过`ping openwrt`能顺利ping通则表示网卡配置正确了。在浏览器里输入[http://openwrt](http://openwrt)打开软路由的配置页，在`网络`->`接口`中，即可看到刚才我们配置的网卡，在这里也可以通过图形界面修改了。

## 定制OpenWRT

### 更换软件源

OpenWRT的官方源有很多三方软件集成的不够好，如果不介意安全问题，可以考虑使用三方软件源。
- 打开浏览器，输入[http://openwrt/cgi-bin/luci/admin/system/opkg](http://openwrt/cgi-bin/luci/admin/system/opkg)进入软件包管理页面
- 点击`配置opkg`，选择`/etc/opkg/distfeeds.conf`，注释掉官方源，并添加新的源如下：

``` ini
# src/gz openwrt_core https://downloads.openwrt.org/releases/23.05.4/targets/x86/64/packages
# src/gz openwrt_base https://downloads.openwrt.org/releases/23.05.4/packages/x86_64/base
# src/gz openwrt_luci https://downloads.openwrt.org/releases/23.05.4/packages/x86_64/luci
# src/gz openwrt_packages https://downloads.openwrt.org/releases/23.05.4/packages/x86_64/packages
# src/gz openwrt_routing https://downloads.openwrt.org/releases/23.05.4/packages/x86_64/routing
# src/gz openwrt_telephony https://downloads.openwrt.org/releases/23.05.4/packages/x86_64/telephony

src/gz openwrt_core https://dl.openwrt.ai/23.05/targets/x86/64/5.15.162
src/gz openwrt_base https://dl.openwrt.ai/23.05/packages/x86_64/base
src/gz openwrt_packages https://dl.openwrt.ai/23.05/packages/x86_64/packages
src/gz openwrt_luci https://dl.openwrt.ai/23.05/packages/x86_64/luci
src/gz openwrt_routing https://dl.openwrt.ai/23.05/packages/x86_64/routing
src/gz openwrt_kiddin9 https://dl.openwrt.ai/23.05/packages/x86_64/kiddin9

```

- 点击`保存`
- 点击`更新列表`，当软件列表更新出来后，即可通过过滤器刷选软件包进行安装

### 更换主题

OpenWRT默认主题是OpenWrt，我们可以更换为其他主题，如luci-theme-argon。

- 在`软件包`页面，过滤器里输入`luci-theme-argon`，找到`luci-theme-argon`，点击`安装`，等待安装完成。
- 点击`主题`，选择`Argon`，点击`应用`。主题设置可以在`系统`->`语言和界面`中进行配置。
- 在`系统`的`常规设置`下，记得将时区修改为`Asia/Shanghai`。
- 退出登录，然后重新登录即可看到新主题。
- 如果想要定制该主题，还可以继续安装`luci-app-argon-config`。安装完成后`系统`->`语言和界面`

### 安装虚拟机驱动
如果你的硬件支持虚拟机，可以安装相应的驱动，以便在虚拟机中使用。
例如：
- 对于vmware虚拟机，可以安装`open-vm-tools`，安装完成后重启虚拟机即可。
- 对于VirtualBox虚拟机，可以安装`Oracle VM VirtualBox Extension Pack`，安装完成后重启虚拟机即可。
- 对于KVM虚拟机，可以安装`qemu-guest-agent`，安装完成后重启虚拟机即可。

### 安装其它软件

OpenWRT的插件很多，我们可以根据自己的需求安装。
参考软件如下：
- shell：`bash`
- 挂载点/分区扩容：`luci-app-partexp`
- 文件管理：`luci-app-fileassistant`
- 端口映射：`luci-app-socat`
- 串口终端：`luci-app-ttyd` 
- 网络唤醒：`luci-app-wol`
- 网络管理：`luci-app-netdata`
- 远程管理：`luci-app-ddns`
- 系统管理：`luci-app-system`
- 性能监控：`luci-app-nlbwmon`
- 日志查看：`luci-app-logread`
- 系统信息：`luci-app-ddns`
- 定时重启：`luci-app-autoreboot`
- 

