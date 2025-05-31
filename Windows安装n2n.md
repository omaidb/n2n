[http://nton.eu.org/](http://nton.eu.org/)

三方网友搞的版本，自带Windows客户端，持续活跃。很厉害。

[https://github.com/n42n/n3n](https://github.com/n42n/n3n)



Windows客户端 [https://github.com/lucktu/n2n/tree/master/Windows](https://github.com/lucktu/n2n/tree/master/Windows)

```shell
# 下载三方编译包
wget -c https://raw.githubusercontent.com/lucktu/n2n/refs/heads/master/Windows/n2n_v3_windows_x64_v3.1.1_r1255_static_by_heiye.zip
```





## 开启IP路由


### 方法1: 用注册表开启IP路由
注册表定位到 

```shell
# HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Servicess\\Tcpip\\Parameters
计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
```

修改 `IPEnableRouter` 的值为 `1`

![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677507346837-b211f882-1780-4f93-8b65-cdc9db47a850.png)

命令行执行

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 1 /f
```



### 方法2: powershell开启ip路由
```shell
# 开启IP转发功能
Set-NetIPInterface -Forwarding Enabled
```



### 查看是否开启IP路由
```shell
# 查看是否开启IP路由
ipconfig /all
```

![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677507250656-64f9cd50-d67a-47ab-b247-5b084f0753d3.png)

```powershell
# 检查是否开启IP转发功能
Get-NetIPInterface | Select-Object ifIndex,InterfaceAlias,AddressFamily,ConnectionState,Forwarding | Sort-Object -Property IfIndex | Format-Table
```



![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677507494506-80b3e868-cf09-412e-a689-f2efd719cbb1.png)



### 开启icmp入站规则
```shell
# 开启IPV4入站规则
netsh advfirewall firewall add rule name= "All ICMP V4" protocol=icmpv4:any,any dir=in action=allow

# 开启IPV6入站规则
netsh advfirewall firewall add rule name= "All ICMP V6" protocol=icmpv6:any,any dir=in action=allow
```





## 创建`tap`虚拟网卡
`开始`--》`运行`--》输入：`<font style="color:rgb(77, 77, 77);">hdwwiz</font>`<font style="color:rgb(77, 77, 77);">；添加</font>`<font style="color:rgb(77, 77, 77);">过时硬件</font>`<font style="color:rgb(77, 77, 77);">。</font>

![](https://cdn.nlark.com/yuque/0/2025/png/300262/1748697362653-ff1b727a-6cce-4779-aaec-fefd75122a14.png)

<font style="color:rgb(77, 77, 77);"></font>

![](https://cdn.nlark.com/yuque/0/2025/png/300262/1748697386141-c8daa857-178d-4afb-83a9-5597c269f697.png)

![](https://cdn.nlark.com/yuque/0/2025/png/300262/1748697404399-538c69ea-2730-41c0-bffe-71682fe296bc.png)



### 命令行添加tap网卡
[https://community.openvpn.net/openvpn/wiki/ManagingWindowsTAPDrivers](https://community.openvpn.net/openvpn/wiki/ManagingWindowsTAPDrivers)

[https://www.twblogs.net/a/5b7c19842b71770a43d95219/?lang=zh-cn](https://www.twblogs.net/a/5b7c19842b71770a43d95219/?lang=zh-cn)

```shell
# 驱动未签名时需关闭驱动签名验证
bcdedit /set testsigning on

# 查看tap命令帮助
tapinstall.exe help

# 查看install参数帮助
tapinstall.exe help install

#  创建虚拟网卡
## tapinstall.exe install <something.inf驱动文件> <网卡类型>
## 最后的tap0901对应于driver目录下的文件名，不一致自行修改。
cd "C:\Program Files\TAP-Windows\"
```

创建`tap网卡`

```powershell
# 创建tap虚拟网卡
## .\driver\OemVista.inf  - 驱动程序信息文件（兼容Vista及更新系统）
##  tap0901        - 设备实例名称（对应驱动版本号，此处为TAP-Windows 9.0.1）
.\bin\tapinstall.exe install .\driver\OemVista.inf tap0901

# 替代命令
devcon update .\driver\OemVista.inf TAP0901

# 📁 目录结构示例：
  ├─bin
  │   └─tapinstall.exe  (驱动安装工具)
  └─driver
      ├─OemVista.inf    (驱动配置文件)
      ├─tap0901.cat    (驱动签名文件) 
      └─tap0901.sys    (驱动核心文件)
```

安装后可在`设备管理器`查看"`网络适配器`"中的"`TAP-Windows Adapter V9`"

![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677513645756-1d189124-8628-49a8-995e-4f47e73e5c78.png)



### 重命名tap网卡为n2n0
选中tap网卡，按下`F2`，重命名为`n2n0`

`配置文件`中指定的`tun网卡`名称为`n2n0`,就将本地的`tap网卡`重命名为`n2n0`.

![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677508570931-3a905bdb-9b54-43bb-917c-89dfce460a0d.png)

```powershell
# 使用 PowerShell 重命名适配器（需管理员权限）
Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "*TAP-Windows*" } | Rename-NetAdapter -NewName "n2n0"

# 或使用 devcon.exe 工具（微软官方工具）
devcon.exe findall =net | find "TAP-Windows" > tmp.txt
for /f "tokens=3" %i in ('type tmp.txt') do devcon.exe rename "%i" "n2n0"
```





## 启动supernode
```shell
# -f 前台运行
supernode /etc/n2n/supernode.conf -f
```





## 启动edge
创建edge配置文件

在`edge.exe`同目录创建`edge.conf`配置文件



以`<font style="color:#DF2A3F;">管理员</font>`执行.

```shell
# 启动edge
edge.exe

# -f 前台运行
edge /etc/n2n/edge.conf -f
```

![](https://cdn.nlark.com/yuque/0/2023/png/300262/1677509081351-c28a95e2-6214-4639-bec7-6a0dbafd2da9.png)



## 使用winsw安装edge为Windows服务
`WinSW.xml`配置内容如下：

```xml
<service>
    <!-- id：安装windows服务后的服务ID，必须是唯一的。 -->
    <id>edge</id>
    <!-- name：服务名称，也必须是唯一的。一般和id一致即可。 -->
    <name>edge</name>
    <!-- description：服务说明，可以使用中文，可做备注使用。 -->
    <description>edge Service</description>
    <!-- 工作目录 -->
    <directory>C:\opt\n2n\edge</directory>
    <!-- executable：执行的命令，比如启动springboot应用的命令java。 -->
    <executable>"C:\opt\n2n\edge\edge.exe"</executable>
    <!-- arguments：命令执行参数，比如 包路径，类路径等。 -->
    <arguments>C:\opt\n2n\etc\edge.conf -v </arguments>
    <!-- 日志模式  reset|roll -->
    <log mode="reset"></log>
    <!-- 故障恢复策略：失败后 60 秒重启 -->
    <onfailure action="restart" delay="5 sec" />
    <!-- 优先级 -->
    <priority>high</priority>
</service>
```



安装为Windows服务

```shell
# 使用winsw安装edge为服务
winSW.exe install
```



## 卸载TAP网卡
在`设备管理器`中`删除`即可。

```shell
# 列出所有 TAP 适配器的永久 ID（Permanent ID）
Get-NetAdapter | Where-Object { $_.InterfaceDescription -like "*TAP-Windows*" } | ForEach-Object {
    Write-Host "设备名: $($_.Name), 永久ID: $((Get-PnpDevice -InstanceId $_.PermanentID).InstanceId)"
}

# 根据永久ID卸载设备（需管理员权限）
pnputil /remove-device "USB\VID_ABCD&PID_1234\000000000001"
```

