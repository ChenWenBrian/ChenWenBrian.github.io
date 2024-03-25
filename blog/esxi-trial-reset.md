# 如何重置ESXI的试用license

> 2023年11月，博通以610亿美元收购VMware。
> 
> 2024年2月，VMware的免费虚拟机监控器软件vSphere Hypervisor（亦称为 ESXi）将不再提供。 

VMware虽然停止了免费的午餐，但是之前已经申请过免费个人license的用户依然不受新政策的影响。但是其他未申请过，又想在家里打架个人home lab的小伙伴可能就难受了。不过好在VMware的60天试用版依然是开放的，且相对个人免费license，其功能更加完整，例如可以运行超过8个CPU核心的虚拟机，多物理CPU支持，负载均衡，vCenter等。

不过试用版有60天的时间限制，时间到了之后就没法建新的虚拟机，断电后也不能启动虚拟机，使用上还是相当难受的。网上有很多方法教大家如何重置试用时间，有些甚至需要重启esxi，且都需要人工操作。为简化操作，我将收集到的信息整理成了一个自动的脚本。该脚本可以解决以下问题：
- 试用版到期后，无需重新安装ESXI
- 重置试用时间时，无需重启ESXI
- 快到期或已经过期时，自动重置试用时间

## 怎么使用?

脚本的仓库在 [esxi_trial_reset.sh](https://github.com/ChenWenBrian/esxi_trial_reset) ,下载方法请参考仓库里的[readme.md](https://github.com/ChenWenBrian/esxi_trial_reset/blob/main/README.md)。

按照readme的说明，下载[esxi_trial_reset.sh](https://raw.githubusercontent.com/ChenWenBrian/esxi_trial_reset/main/esxi_trial_reset.sh)到服务器磁盘里后，

1. 执行`sh esxi_trial_reset.sh -r`即可重置试用时间，前提是当前的试用期即将结束（小于3天），或者已经过期。
2. 执行`sh esxi_trial_reset.sh -a`会创建定时任务，自动检测试用期，并在到期前3天重置试用期。

## 注意事项

1. 服务器停机后，定时任务会消失。这是应为ESXI的安全机制，会清除所有未备份的数据。
2. 该脚本仅支持小于等于`ESXI 7`的版本，`ESXI 8`以上版本未经测试。
3. 该脚本仅限学习使用，请勿将其用于生产环境!!!