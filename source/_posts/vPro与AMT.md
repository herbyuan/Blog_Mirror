---
title: vPro与AMT
date: 2023-10-05 22:51:33
tags: vPro
---

# vPro与AMT

## 概述

`vPro` 是英特尔为企业用途打造的商用PC平台, 具体介绍在官网以及售卖PC的网站上都可以看到. 总体来说想要粗略理解 `vPro`, 只需要掌握一下概念即可:
- `vPro` 是一个完整的平台技术, 其实现需要芯片组, 处理器的共同支持, 很多功能还需要人为设置;
- `vPro` 包括众多功能, 在较新的 `vPro` 认证模式下, 不仅需要基础设施(CPU,芯片组)的支持, 还需要 PC 满足一些条件(例如笔记本的续航, 台式机的运行稳定性等).


我们常常听说 `vPro` 最重要的一个特性就是带外管理 (Out of Band Management), 也就是即使设备不在局域网内, 甚至不开机, 管理者依然可以对电脑进行控制. 简单理解就是管理者可以在电脑关机, 死机或者BIOS状态下进行管理, 实现普通远程控制做不到的事情. 这其实主要依赖 `vPro` 中的 `AMT(Active Management Technology)` 功能. 如果有过企业级服务器的管理经验, 应该会对这种功能比较熟悉, 例如惠普的 `iLO`, 戴尔的 `IDRAC` 都属于带外管理. 这些带外管理往往使用的是 `IPMI`, 即有一个单独的芯片负责此类事务. 事实上, `IPMI` 能够提供的管理功能往往比 `AMT` 要多, 厂商也有更大的自由度(比如增加或者阉割功能), 但是相比 `vPro` 技术需要耗费更大的电能, 实现方式也不完全统一.

## vPro 的使用条件

对于一般的家用主板, 根本不用想, 基本上都不会支持 `vPro`, 哪怕是尊贵的 `Z-X90` 芯片组依然不支持. 但是如果你用的是商用机, 惠普戴尔联想之类的或者英特尔自己推出的 `NUC`, 这个功能倒是有可能附带. 总体来说, `vPro` 的运行需要芯片组和处理器的共同支持. 在英特尔官网查找对应产品, 如果显示 `vPro` 对应条目, 则硬件上支持 `vPro`. 例如 `Q-X70` 和 `W-X80` 芯片组, 以及酷睿的 `i5` 以上型号处理器. 同时注意, 对于戴尔的产品, `vPro` 是需要在购买时加钱的, 否则打开机器, 会看到一个标签, 上面有数字 `3`, 以及 `ME DISABLED` 字样, 表明出厂时禁用了此类功能(据说可以强刷BIOS破解, 并没有仔细研究). 

## AMT功能启用

`AMT` 功能出厂时是默认禁用的, 需要手动开启并配置后才能使用. 在 BIOS 界面中, 找到类似选项并开启, 然后设法进入 MEBx 设置界面.

默认的密码是 `admin`, 第一次使用时需要设置强密码. 

> *Minimum 8 characters with upper, lowercase, 0-9, and one of !@#$%^&*()+-

在有线连接的情况下, 需要设置 `IPv4` 地址, 注意这里的 `MAC` 地址是和系统中共享的, 所以如果使用 `DHCP` 方式获取地址, 则进入系统后二者会共享同一个地址. 要想完全保留 `AMT` 的独立性, 可以选择关闭 `DHCP` 手动配置信息, 这样可以更好地指定 `IP` 地址.

在配置完成后, 一定要激活当前的网络设置, 这一步通常在设置界面中有对应选项. 下次开机前, `AMT` 会在 `BIOS` 自检后进系统前验证网络配置信息.

## AMT的管理方式

注意, `Windows` 系统中如果安装了 `Management Engine Interface (MEI)` 相关的驱动, 则 `LMS` 服务会被自动安装, 以负责本地的信息转发. 也就是说, 在运行 `AMT` 的系统中访问 `<https://IP:16992>`, 其实是 `Windows` 的服务在提供转发. 这里的 `IP` 可以是本机的任意 `IP`. `LMS` 服务被设置为仅能通过本地访问, 局域网中的其他电脑都不能连接到 `LMS`. 其他的电脑想要连接, 需要使用 `AMT` 对应的 `IP` 以及对应端口 (16992 for Non-TLS, 16993 for TLS). 注意, 在 `16.1` 版本之后, 外部将限制为仅能通过 `TLS` 连接, 也就是说必须使用 `HTTPS` 协议访问 `16993` 端口. 使用 `TLS` 需要证书, 如果是外部权威机构提供的签名需要自己的域名, 且限制三个机构, 购买证书非常昂贵, 而英特尔也可以提供自签名的证书, 但是也需要自己设置. 所以初始的设置建议在本机上进行.

`AMT` 的设置叫做 `provision`, 即向 `AMT` 提供相应的信息. 安装完成本地驱动之后, 系统会自动完成相关的设置, 在 `Intel Management and Security Status` (可以在 `Microsoft Store` 下载) 看到相关的信息.

<img src="IMSS.png">

> 注意, 设置完成后 `AMT` 的状态为 `Post-Provision`, 可以在 `BIOS` 中回退到 `Unprovion` 状态. 这里会提供 `Partial Unprovion` 和 `Full Unprovion` 两个选项, 大致所做的事情就是清空 `TLS` 证书, 并重置网络连接信息. 但是两个选项都不会重置 `admin` 的密码. 所以重置之后密码没变是正常的, 想要彻底重置只能扣电池清空 `CMOS`.

确认连接状态为 `已连接`, 说明已经可以访问网页进行配置了. 如果还是 `未配置` 请重新进行激活网络操作, 如果根本没有这几个选项卡说明 `MEI` 相关驱动没有完整安装. 初次配置建议在本地进行, 这里提供三种方法:

1. 访问 [127.0.0.1:16992](http://127.0.0.1:16992), 用户名 `admin`;
2. 下载 `Intel Manageability Commander`, 添加计算机 `127.0.0.1`, 连接方式 `Digest`, 取消勾选下方的 `Use TLS`;
3. 下载 `MeshCommander`, 添加本地计算机后进行访问.

> 安装 `Intel Manageability Commander` 之后, 会提示下载一个组件, 下载完成后完整解压到 `IMC` 的安装目录即可运行.
> 
> 此处 `MeshCommander` 软件英特尔已经停止支持, 因此官网上的下载链接也失效了, 在文章的最后会提供相关的文件.

进入之后可以看到的信息都差不多, 总的来说原版的网页功能是最少的, 但是之后外网使用起来也最方便, 因为不用安装任何本地软件. 想要扩展功能 (例如远程KVM桌面) 可以替换原先的网页, 方法很简单, 直接运行 `MeshCommanderFirmware`, 根据提示上传新的网页就好. 但是这个网页好像有一些奇怪的 Bug, 建议在完全配置好之后再上传.

## 自签名 TLS 的启用

本地基本上不太需要配置, 毕竟这是一个管理工具, 不是配置工具. 但是从 `16.1` 版本开始, 不支持 `Non-TLS` 的连接了, 所以不配置 `TLS` 就没法远程控制.

最好的方式当然是去买一张签名的证书, 并且机构一定要是英特尔指定的, 因为 `AMT` 内置的密钥就那么几个, 价格大概在 100 刀一年, 简直是抢劫. 有没有办法不买但是仍然启用呢? 那就是英特尔的自签名证书. 英特尔在 `AMT` 也内置了自己的私钥, 可以签发证书. 这样的证书是没法被机构验证的, 所以在访问时会跳出警告. 

这里导出英特尔的自签名证书比较奇怪, 我也是始终不能实现最后莫名其妙又搞定了. 那就大致分享一下我的操作思路.

直接本地访问 `16993` 端口或者 `IMC` 使用 `TLS` 都是不行的, 我的理解是 `LMS` 做了转发, 但是没配置 `TLS` 的时候底层的访问就出问题了.

曲线救国的方式是使用 `MeshCommander`, 通过非 `TLS` 连接后, 左上角会提示没有启用 `TLS`, 点击后会提示是否转为使用 `TLS`. 确认之后可以看到连接已经转为 `TLS` 了. 此时下载下来对应的证书, 再在 `Security Settings` 界面上传, 如果不行尝试使用 `IMC` 进行上传. 最后在 `Remote TLS security` 选择对应上传的证书. 重启电脑, 重新插拔网线, 尝试使用别的电脑访问 `https://<AMT-IP>:16993`, 理论上此时应该可以连接上了.

<img src="Mesh-TLS.png">

## 开启 `IPv6` 以及获取地址

在 `Network Setting` 界面中可以手动开启 `IPv6`, 这里建议选择自动, 此时地址不会与宿主系统相同.

在 `Windows` 系统中获取 `AMT` 的 `IP` 最简单的方法就是打开 `IMMS`, 但是如果有神奇的需要想通过 `Powershell` 获取地址, 则要使用 [`IntelvProModule`](IntelvProModule_16.0.7.1.zip). 我根据说明安装模块之后, 引用还是提示找不到. 我的方法是直接使用绝对地址导入模块, 引用的是 `IntelvPro.psd1` 文件, 这样使用相关功能好像是没有问题的.

[这个网页](https://software.intel.com/sites/manageability/AMT_Implementation_and_Reference_Guide/default.htm?turl=WordDocuments%2Fconfiguringtheintelamtipaddress.htm#:~:text=The%20Intel%20AMT%20IP%20address%20can%20be%20either,a%20DHCP%20server%20on%20request%20by%20the%20host.)提供了使用脚本的详细说明, 但是目前直接把 `password` 当 `string` 会引发类型错误, 需要显式转换一下.

最后以获取公网 `IPv6` 地址为例, 我给出一段脚本:

```Bash
Import-Module 'PATH_TO_\IntelvProModule\Bin\IntelvPro\IntelvPro.psd1'

$hostname = "127.0.0.1"
$user = "admin"
$password = "YOURPASSWD" | ConvertTo-SecureString -AsPlainText -Force
$wsmanConnectionObject = new-object 'Intel.Management.Wsman.WsmanConnection'
$wsmanConnectionObject.Username = $user
$wsmanConnectionObject.Password = $password
$ipv6PortSettingsRef =$wsmanConnectionObject.NewReference("SELECT * FROM IPS_IPv6PortSettings WHERE InstanceID='Intel(r) IPS IPv6 Settings 0'")
$ipv6PortSettingsInstance =$ipv6PortSettingsRef.Get()

$xml = [xml]$ipv6PortSettingsInstance.Xml
$namespaceManager = New-Object System.Xml.XmlNamespaceManager($xml.NameTable)
$namespaceManager.AddNamespace("g", "http://intel.com/wbem/wscim/1/ips-schema/1/IPS_IPv6PortSettings")

$currentAddressInfos = $xml.SelectNodes("//g:CurrentAddressInfo", $namespaceManager)

if ($currentAddressInfos.Count -ge 2) {
    $secondCurrentAddressInfo = $currentAddressInfos[$currentAddressInfos.Count-1].InnerText
    # Write-Output "Second Current Address Info: $secondCurrentAddressInfo"
    Write-Output $secondCurrentAddressInfo.Split(',')[0]
} else {
    Write-Output "There is no second Current Address Info element."
}

```

这里的返回值是 `XML` 格式的, 所以还要进行相应的解析才能使用.


## 相关管理软件下载

由于单个上传文件大小的限制, 拆分为多个压缩包上传:

- [管理软件.tar.001](管理软件.tar.001)
- [管理软件.tar.002](管理软件.tar.002)
- [管理软件.tar.003](管理软件.tar.003)
- [管理软件.tar.004](管理软件.tar.004)
- [管理软件.tar.005](管理软件.tar.005)

