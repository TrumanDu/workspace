# AI路之ubuntu系统cuda升级
最近想使用一下llama3，发现pytorch版本低不支持，升级pytorch版本就需要升级cuda版本，升级cuda版本需要提前升级nvidia驱动版本。
## 升级nvidia驱动版本
卸载驱动
```
sudo apt-get purge nvidia-*
sudo apt-get autoremove
```
[https://www.nvidia.com/Download/driverResults.aspx/224022/en-us/](https://www.nvidia.com/Download/driverResults.aspx/224022/en-us/)
下载驱动并安装
```
sudo sh NVIDIA-Linux-x86_64-550.76.run
```


## 升级cuda版本
在[网站](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=20.04&target_type=runfile_local)上根据不同版本选择不同安装方式，我选择离线安装。

在安装之前需要先卸载之前安装的cuda版本，否则会安装失败。
```
sudo /usr/local/cuda-X.Y/bin/cuda-uninstaller
```
或者
```
sudo /usr/bin/nvidia-uninstall
```

卸载完成后要重启机器
```
reboot
```

接下来就是安装了
```
wget https://developer.download.nvidia.com/compute/cuda/12.4.1/local_installers/cuda_12.4.1_550.54.15_linux.run
sudo sh cuda_12.4.1_550.54.15_linux.run
```

保持默认选择安装就好，没有报错证明安装没问题。

最后验证
```
$ nvidia-smi

Sat Apr 27 00:56:09 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.15              Driver Version: 550.54.15      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        Off |   00000000:01:00.0 Off |                  N/A |
| 65%   52C    P0             N/A /  390W |       0MiB /  24576MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+

```


## 参考
1. [NVIDIA CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html)

