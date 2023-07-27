---
title: PHP Mysql与OpenSSL
date: 2023-07-27 09:32:50
tags: PHP MySQL OpenSSL
---

前段时间开了没有SSL的FTP端口, 不到两天就中勒索病毒, 现在对于网络传输安全也开始重视起来. 这几天在折腾 `PHP` 的一些功能, 需要连接本地 `MySQL` 服务, 于是尝试使用 `SSL` 加密, 在这里记录一下踩到的坑.

## 生成自签名的证书

据说 `Mysql` 可以使用 `mysql_ssl_rsa_setup` 生成证书, 但是我已经切换到了 `mariadb`, 并没有看到这个功能, 所以只能自己用 `openssl` 生成.

网上有很多方法是先生成数字证书, 然后分别生成服务端和客户端的密钥和数字证书, 但是这个方法我用了提示证书错误. 下面这个方法生成的证书是可以使用的:

``` sh
$ openssl req -x509 -newkey rsa:1024 \            
         -keyout server-key-enc.pem -out server-cert.pem \
         -subj '/DC=com/DC=example/CN=server' -passout pass:qwerty

$ openssl rsa -in server-key-enc.pem -out server-key.pem \                                        
         -passin pass:qwerty -passout pass:

$ openssl req -x509 -newkey rsa:1024 \
         -keyout client-key-enc.pem -out client-cert.pem \
         -subj '/DC=com/DC=example/CN=client' -passout pass:qwerty

$ openssl rsa -in client-key-enc.pem -out client-key.pem \                                                         
         -passin pass:qwerty -passout pass:  

$ cat server-cert.pem client-cert.pem > ca.pem
``` 

关键在于 `ca.pem` 是最后由两个文件合成的, 原因据说是需要提供双向的验证. 注意这里我们使用的都是本地的 `IP` 进行连接, 所以需要跳过域名的验证. 然后把证书移动到一个合适的存储位置.

## 把证书的位置写入配置文件

然后编辑配置文件 `my.cnf`, 这个文件往往位于某个 `/etc` 文件夹中, 加入以下内容:

```
[mysqld]
ssl-ca=PATH/TO/ca-cert.pem
ssl-cert=PATH/TO/server-cert.pem
ssl-key=PATH/TO/server-key.pem
```

进入 `Mysql` 的命令行, 输入 `show variables like '%ssl%';` 查看当前状态

``` sh
MariaDB [(none)]> show variables like '%ssl%';
+---------------------+---------------------------------+
| Variable_name       | Value                           |
+---------------------+---------------------------------+
| have_openssl        | YES                             |
| have_ssl            | YES                             |
| ssl_ca              | /usr/local/cert/ca.pem          |
| ssl_capath          |                                 |
| ssl_cert            | /usr/local/cert/server-cert.pem |
| ssl_cipher          |                                 |
| ssl_crl             |                                 |
| ssl_crlpath         |                                 |
| ssl_key             | /usr/local/cert/server-key.pem  |
| version_ssl_library | OpenSSL 3.1.1 30 May 2023       |
+---------------------+---------------------------------+
10 rows in set (0.033 sec)
```

可以看到 `have_openssl` 和 `have_ssl` 都是 `YES`, 表示当前配置支持 `SSL`, 否则还要考虑将 `openssl` 作为扩展使用 `phpize` 加入(有空了再写).

配置好了之后重启 `Mysql` 服务, 命令可能是 `service restart mysql`, 可能是 `brew services restart mysql`, 根据自己的配置来确定. 重启之后用相同的命令应该可以看到如上述的输出, 此时 `Mysql` 仍然可以不使用 `SSL` 登录. 我们创建一个强制使用 `SSL` 的账户, 并配置访问的权限:

```sh
GRANT ALL PRIVILEGES ON *.* TO 'ssluser'@'localhost' IDENTIFIED BY 'password' require ssl;
FLUSH PRIVILEGES;  
```

使用以下命令登录:

```sh
mariadb --ssl-ca=/usr/local/cert/ca.pem --ssl-cert=/usr/local/cert/client-cert.pem --ssl-key=/usr/local/cert/client-key.pem -u ssluser -p
```

需要输入密码, 这里也可以直接在命令中填充密码, 但是 `-p` 和密码之间不能有空格. 使用 `status` 查看当前连接的状态, 确认 `SSL` 已经启用. 其实在这里不指定客户端的证书也是可以登录的, 但是只是使用 `SSL` 加密链接, 并不对身份加以验证.

使用 `Navicat` 连接时, 在 `SSL` 界面勾选启用, 并设置证书的位置, 但是不用勾选 `验证CA的服务器证书`, 不然会报错.

<img src="navicat.png" width="60%">

## PHP中使用SSL

为了源码泄露导致密码被窃取, 这里可以把密码等敏感信息放在网站的根目录之外的文件, 使用时 `include` 进来. 注意这里的 `$servername` 一定不能写 `localhost`, 不然会在连接时报如下错误:

```
Warning: mysqli_real_connect(): This stream does not support SSL/crypto in /html/sql-ssl.php on line 10

Warning: mysqli_real_connect(): Cannot connect to MySQL by using SSL in /html/sql-ssl.php on line 10
```

很多人说这是 `openssl` 没有配置好需要重新加入 `openssl` 扩展, 但是查看 `phpinfo` 可以看到明明 `openssl` 已经在编译时就加入了, 即使费劲心思把它作为 `.so` 加载进来也会报重复引入的警告. 折腾了两天, 终于找到一个回帖, 说问题的关键如下:

> 当你连接到 `localhost` 时
>> mysql_connect("localhost",...)
> 
> 如果这样的链接可用，则通信通过本地套接字转换
>> mysql_connect(): [2002]  (trying to connect via unix:/// (...)
> 
> 实际上，这样的链接“不支持SSL /加密”（加密本地通信信道没有意义） .
> 要绕过此优化并强制通过 `TCP/IP` 进行通信，请连接到 `127.0.0.1` .
> **特别鸣谢[这个链接](https://www.javaroad.cn/questions/14993), 虽然我觉得这是英文翻译过来的, 但我确实找不到原贴.**

使用以下代码建立连接:

``` php
<?php
    include("../outside/db_info.php");
    // 创建连接
    $conn=mysqli_init();
    if (!$conn)
    {
      die("mysqli_init failed");
    }
    mysqli_ssl_set($conn,"client-key.pem","client-cert.pem","ca.pem", NULL, NULL); 
    if (!mysqli_real_connect($conn,$servername, $username, $password, NULL, NULL, NULL, MYSQLI_CLIENT_SSL_DONT_VERIFY_SERVER_CERT))
    {
      die("Connect Error: " . mysqli_connect_error());
    }
    unset($servername, $username, $password);
    // 检测连接
    if ($conn->connect_error) {
        die("连接失败: " . $conn->connect_error);
    } 
    echo "连接成功";

    $conn->close();
?>
```

就可以连上啦!

## `phpize` 添加 `openssl` 扩展

`PHP` 添加扩展的只要方式有两种, 一种是 `PECL`, 一种是 `phpize`. 在能使用 `pecl` 时尽量使用, 可以减少好多奇怪的问题. 这里就来讲讲添加扩展的方式.

这里选用 `openssl` 是因为这玩意没法用 `pecl` 安装, 网上的教程也比较奇怪. 首先进入 `php/include/php/ext` 查看有没有 `openssl` 文件夹, 这里文件夹存在并不代表已经作为扩展加入, 因为这里只是些配置文件. 如果没有, 去 [`PHP` 官网](https://www.php.net/downloads.php)下载同版本的安装包, 里面有对应的文件夹拷贝过来. 把 `config0.m4` 重命名为 `config.m4`, 然后在此目录下运行以下指令:

```sh
$ /opt/homebrew/Cellar/php/8.2.8/bin/phpize
$ ./configure --with-php-config=/opt/homebrew/Cellar/php/8.2.8/bin/php-config
$ make
$ make install
```

如果没有安装 `pkg-config` 需要执行 `brew install pkg-config`. 如果提示找不到 `openssl` 的文件可以尝试指定目录或者在 `/usr/local` 进行一次标准的安装. 说实话这么重要的库我建议还是不要用 `HomeBrew` 安装, 因为它对头文件的支持实在是太差了.

安装完后会自动把 `openssl.so` 拷贝到 `pecl` 文件夹. 使用 `php --ini` 查看配置文件位置, 打开 `php.ini` 文件取消 `extension=openssl` 的注释或者加入 `extension="openssl.so"`. 执行 `php -m | grep openssl`, 有输出则已经启动扩展.


















