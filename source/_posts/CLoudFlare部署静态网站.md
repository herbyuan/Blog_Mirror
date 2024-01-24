---
title: CLoudFlare部署静态网站
date: 2023-04-18 16:53:42
tags: CLoudFlare
---

# CLoudFlare部署静态网站

在之前的文章中, 已经实现了纯 ipv6 网页的 ipv4 代理, 极大增加了可访问性. 但是这样就需要家中的服务器始终在线, 网络环境的稳定性也十分重要. 现在想要更进一步, 把这些静态内容直接托管, 岂不是美滋滋?
 
现在许多代码托管平台都提供了静态网页的部署, 但是能够申请到的仅仅是三级域名. 而且主流的 Github 在大陆的访问非常不稳定. 今天查看 Cloudflare 控制台时发现有 pages 选项卡, 可以部署一些网页, 且可以直接从代码托管平台接入, 于是马上尝试.

------

## Github 接入 Cloudflare 并设置 Gitee 自动推送

转了一圈还得是 Github 啊, 下面原先的 Gitlab 还是比较麻烦.

事情的起因是这样的, 虽然 Gitlab 可以访问但是速度还是不尽人意. 且下面我看到明明可以设置 `mirror` 实际上并没有做到, 实现起来各种问题. 然后我看到 Gitee 支持与 Github 的双向同步了 (其实是两个单向拼起来的实际上也建议只开一个方向不然可能会有代码丢失的风险), 那我何不直接使用 Github?

进入 Gitee 对应仓库的管理, 选择 `仓库镜像管理`, 右上方添加镜像, 授权 Github 之后选择同步的方向, 我们要从 Gitee 推送仓库到 Github, 所以选择 Push 方向, 然后选好我们要镜像的目标仓库.

<img src="mirror.png" width="80%">

最下方需要添加 Github 的令牌, 进入 Github, 在[个人设置](https://github.com/settings/profile) 界面, 左边栏最下方 `Develop settings - Personal access tokens - Tokens(classic)`, 生成一个新的 Token, 权限根据 Gitee 官方文档:

> 「Select scopes」字段请根据你的需求进行勾选；
> repo 字段为必选字段，请您直接勾选；
> admin:repo_hook 字段为可选字段，用于自动生成 webhook；当您需要 Gitee 自动从 GitHub 同步仓库时，建议您勾选。

生成的 Token 只会显示一次, 复制到 Gitee.

确认之后就会自动同步, 注意 Github 对应仓库的内容会被清空.

然后在 Cloudflare 设置 Pages, 参考下一段基本上没有区别.

每次推送部署成功或者失败都会收到邮件通知, 还是挺好的.

最后, 注意 Gitee 的单个文件大小限制是 `140+ MB`, Cloudflare Pages 的限制是 `26 MB`, 如果单个文件超出限制了记得提前处理一下.

-------

## Gitlab 接入 Cloudflare

现在可用的代码托管平台有两个, 分别是 Github 和 Gitlab. 前者的访问由于 DNS 污染极度不稳定, Gitlab 的状态要好不少. 我估计 Gitlab 现在是由 Cloudflare 提供的 CDN 服务, 因为两者有太多的相似之处了. 注册 Gitlab 时注意后缀有 `.com` 和 `.cn`, 只有 `.com` 的账号才能接入 Cloudflare. 

<img src="pages.png" width="80%">

注册完后可以导入我们之前的仓库. 直接从已有的 repo 导入, 并且勾选 mirror 选项, 这样在原先仓库内容变动的时候就会自动同步变化到新的仓库. 这里如果不是列出的几家代码托管平台, 则直接以 `.git` 的链接导入.

<img src="import.png" width="80%">

在 Cloudflare 的 Pages 中选择 `Connect to git`, 然后在 Gitlab 中授权, 就可以读取到托管平台上的仓库了. 

<img src="connect.png" width="80%">

选择我们使用的仓库后可以指定一些构建的方法, 这里我们是把编译好的文件夹整个上传的, 所以这里直接指定目标文件夹, 不用指定编译的命令.

<img src="build.png" width="80%">

点击 `save and deploy`, 就会自动从 Gitlab 上拉取代码并完成部署. 最顶上会显示 Cloudflare 给我们分配的 URL, 点击就进入了我们的网页. 第一次打开可能会比较慢, 之后会自动完成 CDN 的分发和缓存, 速度会有所提升.

## 用我们自己的域名访问网页

前面的部署许多平台都可以做到, 但是可以绑定到自己的域名的就很少了. 进入我们的网页控制页面之后, 在 `Custom domains` 可以自定义域名. CloudFlare 会自动为我们添加 `CNAME` 解析. 过一两分钟之后刷新就可以用这个域名来进行访问了.

-------

> 补充: 下面这段是有问题的, `CNAME` 解析只能是自己家的域名, 比如在 Github 设置了自己的主页, 也是不能挂自己的二级域名的. 这么看来目前此方案仍然是最优解.

到最后才注意到这个花里胡哨的玩意到了最后只是个 `CNAME` 解析而已, 早知道这样就不必吹捧了. 但是应当注意到从一个没有备案的域名设置 `CNAME` 到别的网站在很多平台其实是不被允许的, 比如想把邮箱网址定向到阿里企业邮箱, 会显示不能从这个域名访问. 这里可能 Cloudflare 的服务器都在国外就没有这些限制了, 也算是挺方便的.



