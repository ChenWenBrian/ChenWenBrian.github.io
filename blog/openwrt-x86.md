---
layout: post
title: 定制原生X86版OpenWRT
categories: [路由器]
description: 介绍如何将官方原生X86版OpenWRT镜像转换为Hyper-V虚拟机格式，并配置网络和虚拟机。
keywords: 路由器, OpenWRT, Hyper-V
---

# 定制原生X86版OpenWRT

本文介绍如何将官方原生X86版OpenWRT镜像转换为Hyper-V虚拟机格式，并配置网络和虚拟机。
以及如何基于官方原生X86版OpenWRT，定制适合自己需要的主题皮肤、扩容、第三方源、插件、功能等。  

# 一、虚拟机安装OpenWRT

## 准备工作：
- 下载OpenWRT官方镜像：https://firmware-selector.openwrt.org/?version=23.05.4&target=x86%2F64&id=generic
- 或者下载ImmortalWrt镜像：https://firmware-selector.immortalwrt.org/?version=23.05.3&target=x86%2F64&id=generic
- 下载starwind v2vconverter，用于将官方镜像转换为虚拟机格式：https://www.starwindsoftware.com/v2vconverter

> 注：ImmortalWrt镜像相比官方镜像，多了一些插件、语言包和主题，也集成了国内源，适合新手入门。 
> 使用方法上与官方镜像差异不大，文中的部分操作ImmortalWrt镜像已经做好了的就不必重复操作了。

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

# 二、定制OpenWRT

## 更换软件源

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
- 如果更新出错，例如找不到sig文件，请尝试注释掉`option check_signature`，然后再次尝试更新。

## 更换主题和语言

OpenWRT默认语言是英语，我们可以更换为中文。

- 在`软件包`页面，过滤器里输入`luci-i18n-base-zh-cn`，找到`luci-i18n-base-zh-cn`，点击`安装`，等待安装完成。
- 语言设置可以在`系统`->`语言和界面`中进行配置。

OpenWRT默认主题是OpenWrt，我们可以更换为其他主题，如luci-theme-argon。

- 在`软件包`页面，过滤器里输入`luci-theme-argon`，找到`luci-theme-argon`，点击`安装`，等待安装完成。
- 点击`主题`，选择`Argon`，点击`应用`。主题设置可以在`系统`->`语言和界面`中进行配置。
- 在`系统`的`常规设置`下，记得将时区修改为`Asia/Shanghai`。
- 退出登录，然后重新登录即可看到新主题。
- 如果想要定制该主题，还可以继续安装`luci-app-argon-config`。安装完成后`系统`->`语言和界面`

## 安装虚拟机驱动
如果你的硬件支持虚拟机，可以安装相应的驱动，以便在虚拟机中使用。
例如：
- 对于vmware虚拟机，可以安装`open-vm-tools`，安装完成后重启虚拟机即可。
- 对于VirtualBox虚拟机，可以安装`Oracle VM VirtualBox Extension Pack`，安装完成后重启虚拟机即可。
- 对于KVM虚拟机，可以安装`qemu-guest-agent`，安装完成后重启虚拟机即可。

## 更换默认的shell

OpenWRT默认的shell是ash，我们可以更换为bash。

- 打开`终端`，输入`opkg update`，更新软件源。
- 输入`opkg install bash`，安装bash。
- 输入`vi /etc/passwd`，找到`root`的行，将`ash`改为`bash`。
- 输入`reboot`，重启系统。
- 登录系统，输入`echo $SHELL`，返回的若是`/bin/bash`则表示bash安装成功。

## 增加shell登录信息
OpenWRT默认的shell登录信息很简陋，我们可以增加一些信息。

例如，在/etc/profile.d/下新建一个文件，如`30_sysinfo.sh`，并赋予执行权限。

内容如下：

```bash
#!/bin/bash

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LANG=zh_CN.UTF-8

THIS_SCRIPT="sysinfo"
MOTD_DISABLE=""

SHOW_IP_PATTERN="^[ewr].*|^br.*|^lt.*|^umts.*"

DATA_STORAGE=/userdisk/data
MEDIA_STORAGE=/userdisk/snail


# don't edit below here
function display()
{
	# $1=name $2=value $3=red_limit $4=minimal_show_limit $5=unit $6=after $7=acs/desc{
	# battery red color is opposite, lower number
	if [[ "$1" == "Battery" ]]; then
		local great="<";
	else
		local great=">";
	fi
	if [[ -n "$2" && "$2" > "0" && (( "${2%.*}" -ge "$4" )) ]]; then
		printf "%-14s%s" "$1:"
		if awk "BEGIN{exit ! ($2 $great $3)}"; then
			echo -ne "\e[0;91m $2";
		else
			echo -ne "\e[0;92m $2";
		fi
		printf "%-1s%s\x1B[0m" "$5"
		printf "%-11s%s\t" "$6"
		return 1
	fi
} # display


function get_ip_addresses()
{
	local ips=()
	for f in /sys/class/net/*; do
		local intf=$(basename $f)
		# match only interface names starting with e (Ethernet), br (bridge), w (wireless), r (some Ralink drivers use ra<number> format)
		if [[ $intf =~ $SHOW_IP_PATTERN ]]; then
			local tmp=$(ip -4 addr show dev $intf | awk '/inet/ {print $2}' | cut -d'/' -f1)
			# add both name and IP - can be informative but becomes ugly with long persistent/predictable device names
			#[[ -n $tmp ]] && ips+=("$intf: $tmp")
			# add IP only
			[[ -n $tmp ]] && ips+=("$tmp")
		fi
	done
	echo "${ips[@]}"
} # get_ip_addresses


function storage_info()
{
	# storage info
	RootInfo=$(df -h /)
	root_usage=$(awk '/\// {print $(NF-1)}' <<<${RootInfo} | sed 's/%//g')
	root_total=$(awk '/\// {print $(NF-4)}' <<<${RootInfo})
} # storage_info


# query various systems and send some stuff to the background for overall faster execution.
# Works only with ambienttemp and batteryinfo since A20 is slow enough :)
storage_info
critical_load=$(( 1 + $(grep -c processor /proc/cpuinfo) / 2 ))

# get uptime, logged in users and load in one take
UptimeString=$(uptime | tr -d ',')
time=$(awk -F" " '{print $3" "$4}' <<<"${UptimeString}")
load="$(awk -F"average: " '{print $2}'<<<"${UptimeString}")"
case ${time} in
	1:*) # 1-2 hours
		time=$(awk -F" " '{print $3" 小时"}' <<<"${UptimeString}")
		;;
	*:*) # 2-24 hours
		time=$(awk -F" " '{print $3" 小时"}' <<<"${UptimeString}")
		;;
	*day) # days
		days=$(awk -F" " '{print $3"天"}' <<<"${UptimeString}")
		time=$(awk -F" " '{print $5}' <<<"${UptimeString}")
		time="$days "$(awk -F":" '{print $1"小时 "$2"分钟"}' <<<"${time}")
		;;
esac


# memory and swap
mem_info=$(LC_ALL=C free -w 2>/dev/null | grep "^Mem" || LC_ALL=C free | grep "^Mem")
memory_usage=$(awk '{printf("%.0f",(($2-($4+$6))/$2) * 100)}' <<<${mem_info})
memory_total=$(awk '{printf("%d",$2/1024)}' <<<${mem_info})
swap_info=$(LC_ALL=C free -m | grep "^Swap")
swap_usage=$( (awk '/Swap/ { printf("%3.0f", $3/$2*100) }' <<<${swap_info} 2>/dev/null || echo 0) | tr -c -d '[:digit:]')
swap_total=$(awk '{print $(2)}' <<<${swap_info})

c=0
while [ ! -n "$(get_ip_addresses)" ];do
[ $c -eq 3 ] && break || let c++
sleep 1
done
ip_address="$(get_ip_addresses)"

# display info
display "系统负载" "${load%% *}" "${critical_load}" "0" "" "${load#* }"
printf "运行时间:  \x1B[92m%s\x1B[0m\t\t" "$time"
echo "" # fixed newline


display "内存已用" "$memory_usage" "70" "0" " %" " of ${memory_total}MB"
display "交换内存" "$swap_usage" "10" "0" " %" " of $swap_total""Mb"
printf "IP  地址:  \x1B[92m%s\x1B[0m" "$ip_address"
echo "" # fixed newline

display "系统存储" "$root_usage" "90" "1" "%" " of $root_total"
if [ -x /sbin/cpuinfo ]; then
printf "CPU 信息: \x1B[92m%s\x1B[0m\t" "$(echo `/sbin/cpuinfo | cut -d ' ' -f -4`)"
fi
echo ""
echo ""

```

## 扩容硬盘

官方镜像默认之有120MB的硬盘空间，如果需要更多的硬盘空间，可以扩容。可以用插件自动扩容，也可以手动扩容。

### 1.自动扩容

可以安装挂载点/分区扩容插件，自动扩容硬盘。插件名称：`luci-app-partexp`，安装完成后在`系统`->`分区扩容`中进行配置即可。
该插件简单粗暴，不能单独设置分区，只能扩容整个硬盘。

### 2.手动扩容（仅限ext4固件）

如果不想安装插件，或者想自定义分区，可以手动扩容硬盘。

需要用到的软件有：

- fdisk：分区插件，用于分区
- block-mount：挂载点插件，用于挂载分区，需要重启后UI界面上才会出现挂载点。

扩容步骤如下：

- 打开`终端`，输入`fdisk -l`，查看当前硬盘分区情况。
- 找到当前硬盘的分区，如sda1，sda2等，以及对应的磁盘，如`/dev/sda`。
- 输入`fdisk /dev/sda`，按提示操作，扩容硬盘。务必注意分区的起始值和结束值，不能超出磁盘大小。
- 输入`mkfs.ext4 /dev/sda3`，格式化扩容后的分区。
- 打开web界面，点击`系统`->`挂载点`，找到下方的`挂载点`，点击添加
- 勾选启用，UUID选择刚才扩容的分区，挂载点选择根目录/，并复制根目录准备的脚本到记事本里备用，点击保存。
- 回到父页面，保存并应用。
- 修改刚刚复制的脚本里的分区路径为刚才扩容的分区路径，如`/dev/sda1`改为`/dev/sda3`。
- 把修改完成后的脚步全部复制到终端并执行，执行完成后，输入`reboot`，重启系统。

```bash
# 扩容脚本sample
mkdir -p /tmp/introot
mkdir -p /tmp/extroot
mount --bind / /tmp/introot
mount /dev/sda3 /tmp/extroot
tar -C /tmp/introot -cvf - . | tar -C /tmp/extroot -xf -
umount /tmp/introot
umount /tmp/extroot
```

## 安装其它软件

OpenWRT的插件很多，我们可以根据自己的需求安装。

luci兼容基础包：
- luci-base
- luci-compat

参考软件如下：
- shell：`bash`
- 挂载点/分区扩容：`luci-app-partexp`
- 文件管理：`luci-app-fileassistant`
- 端口映射：`luci-app-socat`，依赖`iptables-nft`、`iptables-mod-extra`和`ip6tables-zz-legacy`
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

