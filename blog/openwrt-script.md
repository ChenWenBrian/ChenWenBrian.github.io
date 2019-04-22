# 单网卡玩转智能网关——一键自动启停脚本

接前文[单网卡玩转智能网关——把OpenWRT塞进虚拟机](blog/openwrt-for-vm.md)，我们纯手工完成了全局软路由的配置，但是有时候我们并不希望一直开启软路由，于是又得手工改回各项配置；然而有时候又需要软路由了，又要改配置……改来改去的过程那个麻烦那个痛苦啊，还是弄个自动化配置脚本吧。我们的目标是：

- 自动判断当前是否已启用软路由
- 自动启动软路由并完成所有参数配置
- 自动关闭软路由并恢复所有参数配置

于是我们有了以下自动化脚本，基于微软家的powershell，使用需要管理员权限。其中脚本的前面几行是定义的基本变量，需要跟自己的本地环境保持一致。例如我的WiFi网卡在`控制面板`->`网络和Internet`->`网络连接`中，显示的名称为`WLAN`，那么就有定义的变量`$wifi_name='WLAN'`，其他几个变量参考前文中网卡的命名。

```powershell
#variables
$switch_name='Bridge Switch'
$wifi_name='WLAN'
$vm_name='openwrt'
$net_alias_name='vEthernet (Bridge Switch)'
$destination = '0.0.0.0/0'

function Write-Log {
    Param(
        $Message,
        $Path = "$env:USERPROFILE\gateway_change.log"
    )

    function TS {Get-Date -Format 'hh:mm:ss'}
    "[$(TS)]$Message" | Tee-Object -FilePath $Path -Append | Write-Host
}

#Start VM and changing the Gateway
Function Start-Gateway() {
    $cur_index = get-netroute -DestinationPrefix $destination | select -ExpandProperty ifIndex
    $cur_ip = get-netipaddress | where-object {$_.InterfaceIndex -eq $cur_index} | select -ExpandProperty IPAddress
    $prefixLength = Get-NetIPAddress -InterfaceAlias $wifi_name | Select-Object -ExpandProperty PrefixLength

    Write-Log -Message "==============Start Gateway at $(Get-Date)=============="
    Write-Log -Message "Gateway is $cur_gateway and will be changed to $vm_name IpAddress"

    #修改桥接器
    Write-Log "Enable bridged virtual switch: $switch_name ..."
    Set-VMSwitch $switch_name -NetAdapterName $wifi_name

    #取消VMQ
    #Get-VMNetworkAdapter -ManagementOS
    Write-Log "Cancel VMQ on bridged virtual switch: $switch_name ..."
    Set-VMNetworkAdapter –ManagementOS -Name $switch_name -VmqWeight 0

    #启动vm
    Write-Log "Start virtual mathine: $vm_name ..."
    Start-VM -Name $vm_name
    Write-Log "Trying to get IP address, this might take several minutes..."
    sleep 10
    $vm_ip = (Test-Connection -ComputerName $vm_name -Count 20 | Where-Object {$_.IPV4Address -ne $null} | Select-Object -ExpandProperty IPV4Address -First 1).IPAddressToString
    Write-Log "Virtual mathine IP is $vm_ip"

    #修改接口路由与默认dns
    Write-Log "Set static IP address to $cur_ip, prefix length to $prefixLength, and gateway to $vm_ip ..."
    Remove-NetIPAddress -InterfaceAlias $net_alias_name -confirm:$false
    New-NetIPAddress -InterfaceAlias $net_alias_name -IPAddress $cur_ip -AddressFamily IPv4 -PrefixLength $prefixLength -DefaultGateway $vm_ip


    Write-Log "Set static DNS Server: $vm_ip ..."
    Set-DnsClientServerAddress -InterfaceAlias $net_alias_name -ServerAddresses @($vm_ip)

    #重启WLAN网卡使VMQ生效
    Write-Log  "Restart phisical network adapter: $wifi_name ..."
    Restart-NetAdapter -Name $wifi_name
}

#Stop VM and changing the Gateway to DHCP
Function Stop-Gateway() {
    Write-Log -Message "==============Stop Gateway at $(Get-Date)=============="
    Write-Log -Message "Gateway is $cur_gateway and will be changed to DHCP default settings"

    #关闭vm
    Write-Log "Shutdown virtual mathine: $vm_name ..."
    Stop-VM -Name $vm_name
    sleep 3

    #取消桥接器
    Write-Log "Remove bridged virtual switch: $switch_name ..."
    Set-VMSwitch $switch_name  -SwitchType Private

    #修改接口路由与默认dns
    Write-Log "Reset phisical network adapter: $wifi_name to DHCP default settings ..."
    Set-NetIPInterface -InterfaceAlias $wifi_name -Dhcp Enabled
    Set-DnsClientServerAddress -InterfaceAlias $wifi_name -ResetServerAddresses

    Write-Log "Restart phisical network adapter: $wifi_name ..."
    Restart-NetAdapter -Name $wifi_name
}


#region 强制以管理员权限运行
If (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator"))
{   
    $arguments = "& '" + $myinvocation.mycommand.definition + "'"
    Start-Process powershell -Verb runAs -ArgumentList $arguments
    Break
}
#endregion


$state = Get-VMSwitch $switch_name | select -ExpandProperty SwitchType
$cur_gateway = get-netroute -DestinationPrefix $destination | select -ExpandProperty NextHop
if ($state -eq "External") {
    Stop-Gateway 
}
else {
    Start-Gateway
}
```
