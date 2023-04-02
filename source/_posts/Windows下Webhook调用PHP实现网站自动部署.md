---
title: Windows下Webhook调用PHP实现网站自动部署
date: 2023-04-02 21:15:51
tags: webhook
---


# Windows下Webhook调用PHP实现网站自动部署

## 引言

为了远程维护的方便以及更好的兼容性和可扩展性，我把网站部署在了基于 windows 的 Nginx 上, 这样每次我要发布新的内容的时候，我就会远程登录windows界面，然后把相应的内容生成到网站文件夹里. 在之前的文章中，我已经实现了 IPv4 访问我纯 IPv6 的网站, 但是远程桌面依然不能走代理. 那现在如果可以不用远程界面就自动部署网页, 会极大减少维护的时间.

操作的想法非常简单，就是我在 Gitee 创建一个我网站文件夹的仓库在我每次要更新的时候我就在远程电脑把内容 push 上去. 在收到 push 之后，我希望我的服务器能够自动的执行 pull 操作, 这样子我的网站就被自动更新了.

前段时间在搜索相关信息的时候，我看到了 Webhook, 就是 WebHook 被触发后，自动回调一个设定的 http 地址. 网址可以动态解析发来的数据并做出相应操作, 然后返回此次调用的结果.

## Webhook

Webhook 是一种 HTTP 回调，允许应用程序通过发送 HTTP 请求来将实时数据传递到另一个应用程序。

当发生特定事件（如创建新的 GitHub 问题或付款完成）时，Webhook 会向一个特定的 URL 发送 POST 请求，将相关数据作为有效负载传递。接收方应用程序可以使用该有效负载触发进一步的操作，例如更新数据库、向用户发送通知等。

Webhook 的优点在于它能够以实时性和可靠性的方式将数据传递到其他应用程序，从而使应用程序之间的集成更加简单和高效。它还允许开发人员构建出复杂的自动化流程，减少了手动干预的需要。

现在主流的代码托管平台基本上都能够支持 Webhook. 在设置中设置回调的 http 地址及触发条件, 就可以在条件触发时收到 POST 过来的信息. 我使用的是 Gitee, 数据是 JSON 格式, 详细说明了这次触发的各种信息. 

这里的设置其实非常简单, 但是为了自动化两边的 git 应当设置好 SSH 密钥并在托管平台上登记. 最简单的方法如下 

```sh
ssh-keygen -t rsa
```

可能会提示配置全局的账号和用户名, 按照提示设置就好. 这里的账号和用户名都不需要和代码托管平台注册的信息相同, 但是会在每次 push 的时候被记录下信息.

在账号文件夹的 `.ssh` 文件夹中找到 `id_rsa.pub` 文件, 将里面所有信息复制到托管平台 SSH Key 上登记. 可以使用如下命令查看是否成功.

```sh
ssh -T git@gitee.com
```
> 不要问我为什么不用Github作说明, 这个网站国内的访问体验又不是不知道. 如果代码不是特别重要或者政治相关, 国内的托管平台也挺好用的.

最好使用这个方式拉取一次代码, 因为后期使用php了就很难看出问题到底出在哪, 之前能解决的问题一定要尽早发现解决.

## PHP文件

安装好 PHP 之后需要设置让 nginx 调用.

打开 nginx 的 `nginx.conf`, 在对应的域名下面增加处理 php 的模块. 这部分好像默认配置就有, 取消注释就好了.

```
# pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        # 项目根目录
        root           html;
        # fastcgi 监听端口，如果被占用就换一个
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
```
 
> 记得 `nginx -s reload` 重新加载配置文件. 

然后打开 php 的配置文件, 配置nginx的支持，大约在798行，删除 `cgi.fix_pathinfo = 1` 前面的分号.

把 php 的服务开起来:

```sh
php-cgi.exe -b 127.0.0.1:9000 -c php.ini
```

先写一个最简单的文件用来测试.

```php
<?php
echo "Webhooks 测试<br/>";
echo '<br/>测试：ls 输出项目路径：<br/>';
exec("dir {$path}", $output, $status);
echo '<pre>';
echo print_r($output);
echo '</pre>';
print_r($status);
?>
```

命名为 `test.php` 放到根目录下测试一下.

访问 `zhuoyuan-he.cn/test.php`, 可以得到输出, 说明之前的配置没有问题. 

接下来处理回调时发过来的信息, 总的来说分成三步: 先解析 JSON, 然后验证密码, 最后执行相关的操作然后返回信息. 

```php
<?php

header('Content-type: text/html; charset=utf-8');
ini_set("error_reporting", "E_ALL & ~E_NOTICE");

// 接收post参数
$requestBody = file_get_contents("php://input");
if (empty($requestBody)) {
    echo "data null！This site is used to update blog files using webhook.<br/>";
}
else{
	// Content type = application/json
	$content = json_decode($requestBody, true);

	// 验证 Webhooks 配置的密码
	if (empty($content['password']) || $content['password'] != '123456') {
		echo "password error";
	}

	// 项目存放物理路径，也就是站点的访问地址
	$path = "C:\\Users\\blogfiles";

	// 判断需要下拉的分支上是否有提交，我们这里的分支名称为 main
	if ($content['ref'] == 'refs/heads/master') {

	    // 执行脚本 git pull，拉取分支最新代码
	    // $res = shell_exec("cd {$path} && git pull origin main 2>&1"); // 当前为www用户
		$res = exec("cd {$path} && git pull");

	    // 记录日志 ($content 返回的是一整个对象，可以按需获取里面的内容，写入日志)
	    $res_log = '------------------------->' . PHP_EOL;
	    $res_log .= '用户 ' . $content['pusher']['name'] . ' 于 ' . date('Y-m-d H:i:s') . ' 向项目【' . $content['repository']['name'] . '】分支【' . $content['ref'] . '】PUSH ' . $content['commits'][0]['message'] . PHP_EOL;
	    $res_log .= $res . PHP_EOL;
	    // $hexolog =  exec("cd {$path} && hexo clean && hexo g");

	    // 追加方式，写入日志文件
	    file_put_contents("git_webhook_log.txt", $res_log, FILE_APPEND);
	    // file_put_contents("hexo_log.txt", $hexolog, FILE_APPEND);
	    echo 'git pull excuted!<br/>';
	}
	else
	{
		$res_log = '------------------------->' . PHP_EOL;
	    $res_log .= '用户 ' . $content['pusher']['name'] . ' 于 ' . date('Y-m-d H:i:s') . ' 向项目【' . $content['repository']['name'] . '】分支【' . $content['ref'] . '】PUSH ' . $content['commits'][0]['message'] . PHP_EOL;

	    // 追加方式，写入日志文件
	    file_put_contents("git_webhook_log.txt", $res_log, FILE_APPEND);
	    echo 'TEST SUCCESS!<br/>';
	}
	echo 'done<br/>';

}

?>

```

本来打算只同步一个资源文件夹放到服务器上自动编译, 但是 Gitee 的 Webhook 要在十秒之内返回, 不然就报错, 只能传编译好的版本了.

## `php-cgi` 开机自启与进程守护

问题的关键在这一部分. 之前的操作网上有很多教程, 也都不太会出问题, 但是一旦到了生产环节, 谈论细节的文章就很少了, 甚至很多过时的错误的文章还在一遍遍被复制.

Windows 下开机自动启动 `php-cgi` 并建立守护进程和一般的不太一样. 首先, 据说 `php-cgi` 处理500次之后就自动退出了, 我没试过但是不守护直接裸奔看起来很不靠谱. 然后为了性能考虑, 一般会建立多个进程监听同一个端口, 在 linux 上有 `php-fpm`, 但是 windows 上好像没有, 就算有也要自己编译 php. 但凡有这个水平在 windows 上编译的我相信都不会用windows来跑php(当然这个也可以讲讲有空了再说). 网上的文章有一大半说的是直接用 [winsw (Windows Service Wrapper)](https://github.com/winsw/winsw/releases/), 这个玩意会根据配置文件构建启动和停止的命令, 也会建立守护进程, 打包成系统服务之后确实挺好用, 另一个差不多的是 [nssm](https://nssm.cc/download), 这个东西有个图形界面, 但是注意选好之后一定要改服务名称, 默认会用 `./` 开头, 安装就报错. 但是它只能跑一个进程, 不太能多开. 所以另一半的文章建议不要直接用 winsw 守护 `php-cgi`, 而是搞了个 `xxfpm` 守护, 这个玩意好像可以多开. 但是这个玩意是十多年前写的好久没有维护了, 又不能处理文件路径中的空格, 甚至基本功能都实现不了. 我尝试了一个下午, 反正这玩意运行之后 `php-cgi` 没有起来, 过了几秒钟自己也退出了. 

兜兜转转, 基本上爬完了所有的文章, 最后找到了 [php-cgi-spawner](https://github.com/deemru/php-cgi-spawner). 这玩意好像是毛子用 `C` 写的, 体积小性能强运行稳定, 基本功能还丰富, 简直不要太好, 就是介绍它的文章实在太少了. 运行时按照官网提示给好参数, 就可以起设定数量个进程, 甚至还可以根据压力设定压力大时额外增加的进程. 试了一下杀掉进程, 马上就能恢复数量, 果然是强! 然后用 winsw 或者 nssm 添加这玩意到服务开机自启, 这个步骤就完成了.

> 关键部分就这么短? 花了很多时间就研究这? 

> 你上你也行! 快上吧

-------

怎么说?

死了吧......

看起来完美, 测试没错误, 服务都能起来, 生产就是不行, 推送完后无响应, 更新不了一点......还是继续看看吧.

**问题其实在于进程的用户.**

windows 服务运行进程默认的用户是内部的一个系统账户, 或者说理解成登陆账户之前更底层的设计. 这就是为什么我们不用登陆账户, 在锁屏界面就能通过网络访问到我们的电脑. git在操作的时候会使用对应账户的ssh密钥, 我们没有上传这个密钥, 当然就访问不了. 那懂的人又要说了, 我找到这个 ssh 密钥把它传上去不就好了吗? 非也! 第一次鉴权的时候会问要不要把fingerprint怎么怎么样, 得输入yes才能执行, 那你没输, 或者下次用了别的托管平台, 照样死得不明不白. [这篇文章](https://segmentfault.com/a/1190000009232433?utm_source=sf-similar-article)有探讨这个问题, 也给了一些启发,但我认为不用这么烦. 所以要在服务中修改运行的账户, 查找到我们的用户账户然后输入密码, 之后起进程就会用对应账户的信息了.

用自己账户的信息还有哪些好呢? 比如 nginx 在更新配置文件或者杀进程时不是一个账户会提示无法操作. 总之, 在自己手上的才是最好控制的.

-------

## 最后的一些碎碎念 

和机器交互总是会有各种各样意想不到的问题, 如果问题很大众化, 那么网络上很容易找到解答. 但是如果问题很小众, 或者之前流传的方法过时了, 就需要自己寻求出路.

自己找办法是一件很困难的事, 像走漆黑的夜路. 完全没有头绪的同时, 网上胡乱复制的答案还会干扰自己的判断. 很多人不愿意自己找解决问题的办法, 遇到问题就直接放弃了. 这习惯确实好, 毕竟天底下没学的知识那么多, 明明有学习成本更低的其他事, 为什么要在这里死磕呢? 这些人可以有大量的产出, 表面上吹得天花乱坠, 什么东西也都能搞一点. 但是涉及到深层的问题, 他们往往手足无措, 还会说: 这很重要吗? 这句话的意思其实是我不会所以不重要, 这是我最近才认识到的. 一个学金融的同事最近跟我说面试问了浅拷贝和深拷贝, 他完全没听说过. 但是他又说他做了三年的实习实现了几多个模型挖掘了几多个因子, 但你怎么会不知道最简单的拷贝呢? 这样别人怎么相信你的代码里面会不会全是隐含的错误呢? 最后他说他不学计算机, 这种概念很重要吗, 我无话可说.

确实现在都很看产出, 月报周报日报, 恨不得每一分钟都要有产出. 但是这怎么可能呢? 每时每刻的产出意味着没有积淀全靠以往的知识, 别人的或者是自己的. 但是现在 `ChatGPT` 知道的不比我多吗? 我前几天问它论文里一句佶屈聱牙的句子什么意思, 它甚至可以给我数值上的例子加以说明, 我问他回测的代码怎么写, 它的实现简答而精妙跟诗一样, 甚至还写好了注释(虽然仔细看隐含着骇人的错误). 但是我问他为什么一个进程起不来, 为什么会自动退出, 为什么网络上ping不通, 为什么 php 的 windows 版本没有 fpm, 它就吱哇乱叫胡扯一通, 说着很有道理却帮不上一点的废话.

我想起来我面试北大的时候组里的一位同学. 面试的题目有关多普勒效应以及量子化轨道. 想必是高考备战出身的她显然对此一无所知, 但硬是讲了很久这些东西的启示以及她自己的幼稚想法. 点评环节显然被下一位同学贬斥地体无完肤, 本以为事情到此为止的时候她竟又站起来, 近乎复述地 "总结" 了一通, 甚至还有了更深更高的感悟. 这件事情给我留下的印象极深, 每次我看到金融领域的人一知半解却十分自信地阐述最新的技术的时候(不论是区块链还是AI), 我都会想起这位同学, 她现在一定在金融行业混的很好吧, 我确实很羡慕这种胡诌的天赋, 但我也庆幸自己没有这样的能力.

我跟我的同学说我花了一天搞定了 Windows 下 Webhook 调用 PHP 实现网站自动部署, 他听完顿了几秒, 仿佛在等我继续说下去, 结果我说, 就这, 他回, 就这.

没用就没用吧, 反正这种文章也不会有人看, 如果幸运的你看到这里并真的解决了你的问题, 请一定发邮件告诉我, 给我卑微的一点点鼓励.

