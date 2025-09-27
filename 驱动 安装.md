### 实验环境
3080ti+ubuntu24.04

### 显卡驱动
```yaml
apt install build-essential pkg-config xserver-xorg-dev -y
lspci | grep -i vga
00:02.0 VGA compatible controller: Cirrus Logic GD 5446
00:03.0 VGA compatible controller: NVIDIA Corporation GA102 [GeForce RTX 3080 Ti] (rev a1)
# 访问链接下载 GeForce RTX 3080 Ti linux驱动
https://www.nvidia.cn/drivers/lookup/
# 根据提示安装显卡驱动
sh NVIDIA-Linux-x86_64-570.144.run
# 或者
sh NVIDIA-Linux-x86_64-570.144.run \
    --silent \
    --no-questions \
    --accept-license \
    --ui=none \
    -m=kernel-module-type=proprietary
# 查看驱动信息
nvidia-smi
```

> <font style="color:rgba(0, 0, 0, 0.9);">nvidia-uninstall 卸载</font>
>
> [下载最新官方 NVIDIA 驱动](https://www.nvidia.cn/drivers/lookup/)
>
> ![](https://cdn.nlark.com/yuque/0/2025/png/36178385/1746697914609-383175f3-87db-4007-b1a2-a4dd1b53b80d.png)
>
> NVIDIA Proprietary：适用于大多数用户，特别是需要较高性能和全面功能的用户。
>
> MIT/GPL：适用于更注重开源、自由软件的用户，但可能功能较为有限。
>

### cuda驱动
[CUDA Toolkit 12.1 Downloads](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_network)

#### 方法1(不使用,网络不好)
```yaml
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
apt update
# 安装显卡驱动
apt-get install -y nvidia-open
# 安装cuda
apt-get -y install cuda-toolkit-12-9
```

#### 方法2(runfile)
```yaml
apt install libnccl2 libnccl-dev build-essential pkg-config xserver-xorg-dev -y 
wget https://developer.download.nvidia.com/compute/cuda/12.9.0/local_installers/cuda_12.9.0_575.51.03_linux.run
root@gpu:~# sh cuda_12.9.0_575.51.03_linux.run
# sh cuda_12.9.0_575.51.03_linux.run --silent --toolkit --driver

cat >> /etc/profile << 'EOF'
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
EOF
source /etc/profile
```

```yaml
# 测试
nvcc -V
nvidia-smi
```

> 驱动：没有显卡驱动，就不能识别GPU硬件，不能调用其计算资源。
>
> [CUDA](https://zhida.zhihu.com/search?content_id=59629304&content_type=Answer&match_order=1&q=CUDA&zhida_source=entity)：是Nvidia推出的只能用于自家GPU的并行计算框架。只有安装这个框架才能够进行复杂的并行计算。
>

#### 测试
```yaml
cd /usr/local/cuda/extras/demo_suite
./deviceQuery
```

### 容器驱动
[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

```yaml
mkdir  /etc/apt/sources.list.d/ -p
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

```yaml
nvidia-ctk runtime configure --runtime=docker
# cat /etc/docker/daemon.json
systemctl restart docker
```

```yaml
nvidia-ctk runtime configure --runtime=containerd
# cat /etc/containerd/config.toml
systemctl restart containerd
```

#### 
