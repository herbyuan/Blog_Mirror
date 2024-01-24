---
title: WSL开荒与机器学习环境
date: 2023-03-09 23:51:20
tags: WSL
updated:
type:
comments:
description:
keywords:
top_img:
mathjax:
katex:
aside:
aplayer:
highlight_shrink:
---



# WSL2开荒与机器学习环境（纯新手入门向）

每次换主机或者系统重装都要进行一大堆开荒的操作, 虽然操作起来流程都是一样的, 但总有地方需要查. 现在的网络环境下懂的不懂的人都在乱说, 每次寻找信息也要花费不少时间, 于是决定自己写一个文档供以后的自己参考. 

## WSL和WSL2

新装的系统默认的WSL应该是WSL1, 如果想要更好更完整的体验可以升级到WSL2. 大致来说需要的操作是在系统的 `启动或关闭windows功能` 里面打开 `Hyper-V` 和 `适用于Linux的Windows子系统`.  大致来说WSL1和2的区别就是WSL2更像是一个虚拟机, 它的底层和 Docker 有着千丝万缕的联系；而WSL1更像是Linux和Win的无缝衔接, 这也导致了WSL1的功能和性能都在很大程度上被限制了. 

这里再说一句网络上二者的区别. WSL1直接使用宿主机的网络, 如果你在WSL1里面监听一个端口, 直接使用 `<宿主ip>:<监听port>` 就可以建立连接. 这样的好处就是直接可以暴露WSL里面的端口, 简单方便. 但是如果宿主和WSL的端口号冲突了, 可想而知就会有一定的问题. 而使用WSL2, 默认的方式是利用 `hyperv` 建立了内部的虚拟交换机, 让WSL可以访问外部的网络. 但是这样外部就不能直接访问到WSL2了. 解决的方式有如下两种：

第一种就是使用windows自带的 `netsh` [工具](https://learn.microsoft.com/zh-tw/windows-server/networking/technologies/netsh/netsh-interface-portproxy)做一个端口转发.  一般来说外网访问都只有ipv6的地址, 2019年开始运营商网络都会支持ipv6了, 如果没有ipv6地址的话排查一下老旧路由器和光猫设置应该很好解决. 所以这里做一个把ipv6端口信息转发到WSL内部端口的代理：

```sh
add v6tov6 listenport= {Integer | ServiceName} [[connectaddress=] {IPv6Address | HostName} [[connectport=] {Integer | ServiceName}] [[listenaddress=] {IPv6Address | HostName} [[protocol=]tcp]
```

监听地址 `listenaddress` 如果想设置成全部地址, 可以简写成 `::`, 如果是ipv4就写 `0.0.0.0`, 总之是全零的地址. `listenport`是在外部监听的地址, 记得在防火墙里开放这个端口. `connectaddress` 是连接的地址, 考虑到地址总是变来变去, 这里直接用 `::1` 指定成本机就好了, `connectport` 就是希望转发到的端口. 所以最后使用管理员权限运行

```sh
 netsh interface portproxy add v6tov6 listenaddress=:: listenport=22 connectport=22 connectaddress=::1
```

当然如果有很多业务放在虚拟机上, 给每个端口设置转发也比较麻烦. 第二个办法是直接建立外部的虚拟交换机把虚拟机加入本地网络, 也获得自己的ip地址. 首先搜索 `hyper-v管理器`, 打开虚拟交换机管理器, 新建一个外部的虚拟交换机, 对应到上网的网卡, 假设名字为 `WSLBRIDGE`. 在windows的对应自己用户文件夹建立名为 `.wslconfig` 的文件, 这是对所有WSL的全局设置. 输入一下内容配置网络
```
[wsl2]
networkingMode=bridged # 桥接模式
vmSwitch=WSLBRIDGE # 虚拟交换机名字
ipv6=true # 启用 IPv6
```

管理员权限运行 `wsl --shutdown && wsl` 重启 WSL2, 不出意外的话网络就是桥接模式了.  具体ip可以使用 `ifconfig` 命令查看. 注意和windows不同, windows的命令是 `ipconfig`. 可以使用 `netstat -tln` 查看使用中的端口. 


## 基本工具安装

闲话少说, 打开虚拟化后就可以在windows商店找到 `Ubuntu`, 商店提供了不同年份的LTS版本和最新的版本, 在这里就使用 `Ubuntu 20.04.5 LTS`. 首先拉取一下最新的文件列表：

```sh
sudo apt update
sudo apt-get update
```

`sudo` 的含义是提升权限, 大家都知道, 在权限不够的时候加上就行, 只要不是 `rm -rf /*`. 本来还想写一下怎么更换国内的源的, 但是最近不管是PIP, Anaconda还是 Ubuntu 的网站访问速度都还行, 可能是全民数据科学的缘故, 这块内容就有空再补充好了. 

上来先无脑安装一些编译任何文件都会用到的包小试牛刀一下：

```sh
sudo apt install build-essential
```

这个包会安装gcc的一些东西, 如果后期要自己编译一些文件可能会用到. 这里先不装python了, 一方面ubuntu列表里的python就像摸奖, 不指定版本装出来不知道是什么玩意, 另一方面反正后期用conda管理环境的话在里面装更方便. 

接下来安装`Anaconda`. 先从网上找个安装脚本然后执行：

```sh
wget https://mirrors.bfsu.edu.cn/anaconda/archive/Anaconda3-2022.10-Linux-x86_64.sh --no-check-certificate
bash Anaconda3-2021.11-Linux-x86_64.sh
```

按照提示读完许可输入同意傻瓜式安装到底. 最后会询问要不要初始化`conda`环境, 这里建议初始化, 不然可能直接敲`conda`都找不到命令. 

安装完成之后直接输入`conda`, 提示找不到命令

>conda: command not found

不要紧张, 关掉窗口重新打开就好了. 或者使用命令

```sh
source ~/.bashrc
```

就可以立刻加载修改后的设置, 使之生效. 

进来之后就会发现前面多了一个 `(base)`, 说明已经处在conda的环境中了. conda对比直接安装各个软件的好处就是可以创建多个虚拟环境, 只要切换不同的环境就能实现不同配置的切换. 

使用如下命令新建一个python3.9的环境

```sh
conda create -n python39 python=3.9
```

通过 `conda activate python39` 切换到这个环境中来. 如果想退出环境, 输入 `conda deactivate` 即可. 使用 `conda env list` 可以看到系统中存在的所有环境. 

这里安装 `Tensorflow` 和 `Pytorch` 两个主流的机器学习包, 并配置好 `CUDA` 和 `CUDNN` 的加速. Ubuntu本身就会带Nvidia显卡的驱动, 可以使用命令 `nvidia-smi` 查看显卡的信息. 首先安装 `CUDA` 和 `CUDNN`. 使用 `conda search cudatoolkit` 查看可安装的版本, 可以看到并不全. 我的显卡是RTX3070, 所以考虑使用`cuda11.3`, 这也是兼容大多数包的版本. 

```sh
conda install cudatoolkit=11.3 cudnn
``` 

安装完成后可以继续安装 `Tensorflow` 和 `Pytorch` . 不知道为什么如果先装 `Pytorch`, 安装 `Tensorflow` 时就会消耗大量的时间做冲突检查. 所以在这里先安装 `Tensorflow`. 注意默认情况下安装的是CPU的版本, 所以要先把版本列出来选择对应的GPU版本. 

可以看到大致信息如下


|  Name   |  Version  | Build | Channel
|  ----  | ----  |----  |----  |
| tensorflow |  2.10.0 | eigen_py39he2aad1f_0  | pkgs/main
| tensorflow |  2.10.0 | gpu_py310hb074053_0  | pkgs/main
| tensorflow |  2.10.0 | gpu_py37h84cb581_0  | pkgs/main
| tensorflow |  2.10.0 | gpu_py38h11b98a5_0  | pkgs/main
| tensorflow |  2.10.0 | gpu_py39h039f4ff_0  | pkgs/main
| tensorflow |  2.10.0 | mkl_py310h24f4fea_0  | pkgs/main


所以不仅要指定版本号, 还要指定对应的 `Build`

```sh
conda install tensorflow=2.10.0=gpu_py39h039f4ff_0
```

安装完成后可以安装 `Pytorch`, 在[官网](https://pytorch.org/get-started/previous-versions/)可以看到历史版本的安装指令

```sh
conda install pytorch==1.11.0 torchvision==0.12.0 torchaudio==0.11.0 cudatoolkit=11.3 -c pytorch
```

## GPU的识别与问题调试

这样两个包就安装好了, 接下来看一下能否识别到GPU

```sh
python
>>> import torch
>>> torch.cuda.is_available()
True
>>> torch.cuda.get_device_name(0)
'NVIDIA GeForce RTX 3070'

>>> import tensorflow as tf 
>>> tf.test.is_gpu_available()

```

`Tensorflow` 的输出会带有很多告警信息, 大致有如下几种

```sh
2023-03-08 18:41:43.367643: I tensorflow/core/platform/cpu_feature_guard.cc:193] This TensorFlow binary is optimized with oneAPI Deep Neural Network Library (oneDNN) to use the following CPU instructions in performance-critical operations:  SSE4.1 SSE4.2 AVX AVX2 AVX_VNNI FMA
To enable them in other operations, rebuild TensorFlow with the appropriate compiler flags.
2023-03-08 18:41:43.450813: I tensorflow/core/util/util.cc:169] oneDNN custom operations are on. You may see slightly different numerical results due to floating-point round-off errors from different computation orders. To turn them off, set the environment variable `TF_ENABLE_ONEDNN_OPTS=0`.
2023-03-08 18:41:43.509932: E tensorflow/stream_executor/cuda/cuda_blas.cc:2981] Unable to register cuBLAS factory: Attempting to register factory for plugin cuBLAS when one has already been registered
2023-03-08 18:42:00.096412: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:966] could not open file to read NUMA node: /sys/bus/pci/devices/0000:01:00.0/numa_node
Your kernel may have been built without NUMA support.
```

第一个是使用CPU的指令集, 第二条是说近似计算会有些许误差, 第三条直接报了 `ERROR` 说无法注册 `cuBLAS`, 这应该是和内存访问有关的, 官方表示是WSL系统导致的, 不会对计算造成影响. 最后一条 `NUMA` 也是WSL的问题, 这两种信息可以不用理会, 不会造成问题. 

接下来搞一个真的要算的东西给它做做看, 就用最简单的 `MNIST` 分类

```python
print('hello world')

import tensorflow as tf

print('step1')

mnist = tf.keras.datasets.mnist

(x_train, y_train), (x_test, y_test) = mnist.load_data()
x_train, x_test = x_train / 255.0, x_test / 255.0
# 搭建模型
model = tf.keras.models.Sequential([tf.keras.layers.Flatten(input_shape=(28, 28)),tf.keras.layers.Dense(128, activation='relu'),tf.keras.layers.Dropout(0.2),tf.keras.layers.Dense(10, activation='softmax')])

model.compile(optimizer='adam',loss='sparse_categorical_crossentropy',metrics=['accuracy'])
# 训练并验证模型
model.fit(x_train, y_train, epochs=5)
```


训练一代需要大约4秒钟, 这和直接跑在实体机器上的速度是相当的, 说明现在WSL的性能以及N卡的虚拟化做的已经足够好了. 

再测试一个需要用到卷积神经网络的模型

```python
import tensorflow as tf

from tensorflow.keras import datasets, layers, models

(train_images, train_labels), (test_images, test_labels) = datasets.cifar10.load_data()

# Normalize pixel values to be between 0 and 1
train_images, test_images = train_images / 255.0, test_images / 255.0

class_names = ['airplane', 'automobile', 'bird', 'cat', 'deer',
               'dog', 'frog', 'horse', 'ship', 'truck']

model = models.Sequential()
model.add(layers.Conv2D(32, (3, 3), activation='relu', input_shape=(32, 32, 3)))
model.add(layers.MaxPooling2D((2, 2)))
model.add(layers.Conv2D(64, (3, 3), activation='relu'))
model.add(layers.MaxPooling2D((2, 2)))
model.add(layers.Conv2D(64, (3, 3), activation='relu'))

model.add(layers.Flatten())
model.add(layers.Dense(64, activation='relu'))
model.add(layers.Dense(10))

model.summary()

model.compile(optimizer='adam',
              loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
              metrics=['accuracy'])

history = model.fit(train_images, train_labels, epochs=10, 
                    validation_data=(test_images, test_labels))
```

在训练的一开始报出了如下信息

```sh
Epoch 1/10
2023-03-08 18:50:35.307252: I tensorflow/stream_executor/cuda/cuda_dnn.cc:384] Loaded cuDNN version 8201
2023-03-08 18:50:37.047238: I tensorflow/core/platform/default/subprocess.cc:304] Start cannot spawn child process: No such file or directory
2023-03-08 18:50:37.138750: I tensorflow/core/platform/default/subprocess.cc:304] Start cannot spawn child process: No such file or directory
2023-03-08 18:50:37.138811: W tensorflow/stream_executor/gpu/asm_compiler.cc:80] Couldn't get ptxas version string: INTERNAL: Couldn't invoke ptxas --version
2023-03-08 18:50:37.229208: I tensorflow/core/platform/default/subprocess.cc:304] Start cannot spawn child process: No such file or directory
2023-03-08 18:50:37.229326: W tensorflow/stream_executor/gpu/redzone_allocator.cc:314] INTERNAL: Failed to launch ptxas
Relying on driver to perform ptx compilation.
Modify $PATH to customize ptxas location.
This message will be only logged once.
```

这次挺幸运, 模型还可以继续跑, 不像上次直接就报找不到动态链接库文件了. 虽然我不知道这个 `ptxas` 是个什么玩意, 但是国外有大神说出现问题的原因是 `nvcc` 没有被安装进来, 不信你看

```sh
~$ nvcc -V
Command 'nvcc' not found, but can be installed with:
sudo apt install nvidia-cuda-toolkit
```

使用如下命令安装nvcc
```sh
conda install -c nvidia cuda-nvcc
```

安装完后再测试就不会有问题了. 但是这里有一点很奇怪, 就是如果我直接在系统里面安装官方的 CUDA 和 CUDNN, `nvcc` 识别到的就是正确的CUDA版本, 但是这样安装显示的就是显卡支持的最高版本了. 

虽然如此, 无伤大雅, 现在基本上机器学习最基本的环境就构建好了. 

----------------

好了吗, 不一定, 前面说的根本不能跑的问题还没说呢. 

在 `Ubuntu 22.04` 上执行的时候, 会在训练开始前跳出如下信息

```sh
Could not load library libcudnn_cnn_infer.so.8. Error: libcuda.so: cannot open shared object file: No such file or directory
Please make sure libcudnn_cnn_infer.so.8 is in your library path!
Aborted
```

这里显示链接库中找不到文件 `libcudnn_cnn_infer.so.8`. 很自然的我们要看看这个文件到底有没有. Anaconda的所有文件存放在`~/anaconda3/envs/{myenv}` 中, 只要查看里面的 `/lib` 文件夹有没有东西就好了. 但是很奇怪, 这里明明白白放着一个相同命名的可执行文件. 所有的网上资料都很少提及可行的解决方案, 大致意思就是说既然是链接那么动态链接库的目录就应该有这个文件夹, 把这个文件夹加入到 `$PATH` 和 `$LD_LIBRARY_PATH` 就可以了. 那不妨我们来尝试一下

```sh
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/anaconda3/envs/python39/lib
export PATH=$PATH:~/anaconda3/envs/python39/lib
```

export 的意思是立即修改环境变量, 但是重启之后就变回去了. 但是没关系, 因为. . . 

这根本就没有用！！！

其实道理很好理解, Anaconda 把所有的可执行文件和链接库都放在以环境命名的文件夹里面了. 作为一个开发软件的人不会蠢到不把这玩意加入到环境变量中, 不然我还要什么虚拟环境, 我自己去修改环境变量不就好了. 问题的根源在于 `libcuda.so`, 这玩意的链接出了点问题. 如果你也有这个问题, 不妨跟我一起操作下：

```sh
$ sudo ldconfig
/usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link

$ sudo rm -r libcuda.so.1
$ sudo rm -r libcuda.so

$ sudo ln -s libcuda.so.1.1 libcuda.so.1
$ sudo ln -s libcuda.so.1.1 libcuda.so
$ sudo sudo ldconfig
```

`ldconfig` 是一个动态链接库管理命令, 命令的用途主要是在默认搜寻目录(`/lib` 和 `/usr/lib`)以及动态库配置文件 `/etc/ld.so.conf` 内所列的目录下搜索出可共享的动态链接库(格式如前介绍, `lib*.so*`),进而创建出动态装入程序(`ld.so`)所需的连接和缓存文件.缓存文件默认为 `/etc/ld.so.cache`, 此文件保存已排好序的动态链接库名字列表. 

然后再运行代码, 发现错误已经消除了. 网上的评论说这个问题只在使用 `Ubuntu22.04` 才出现, 所以链接没处理好应该也是系统的问题. 
