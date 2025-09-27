## 安装
前提条件： 安装好cuda & conda ，<font style="color:#DF2A3F;">并进入conda虚拟环境，此处不再赘述</font>

<font style="color:#DF2A3F;">环境： ubuntu + cuda 12.8 + Nvidia 2080Ti + pytorch 2.7.0</font>

### 编译安装（没有magma128版本，放弃）
[https://github.com/pytorch/pytorch/tree/v2.7.0?tab=readme-ov-file#from-source](https://github.com/pytorch/pytorch/tree/v2.7.0?tab=readme-ov-file#from-source)

```yaml
export HTTP_PROXY=http://10.60.100.1:7890/
export HTTPS_PROXY=http://10.60.100.1:7890/
export NO_PROXY=localhost,127.0.0.1,.example.com,.xiong03.cn,192.168.43.0/24
```

```yaml
sudo apt install -y build-essential cmake ninja-build git python3-dev python3-venv libpython3-dev libopenblas-dev liblapack-dev libjpeg-turbo8-dev libpng-dev libtiff5-dev libavformat-dev libswscale-dev libfreetype6-dev libxext-dev libxrender-dev libxcb1-dev libxkbcommon-dev libxcb-xkb-dev libfontconfig1-dev libglu1-mesa-dev libffi-dev liblz4-dev zlib1g-dev libncurses5-dev libgomp1 libatlas3-base libhdf5-dev libzmq3-dev libprotobuf-dev protobuf-compiler libsnappy-dev libgoogle-glog-dev libgflags-dev liblmdb-dev libbz2-dev libdbus-1-dev libnuma-dev
```

cudnn安装

> [cuDNN 9.10.1 Downloads](https://developer.nvidia.com/cudnn-downloads?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_local)
>

```yaml
wget https://developer.download.nvidia.com/compute/cudnn/9.10.1/local_installers/cudnn-local-repo-ubuntu2404-9.10.1_1.0-1_amd64.deb
sudo dpkg -i cudnn-local-repo-ubuntu2404-9.10.1_1.0-1_amd64.deb
sudo cp /var/cudnn-local-repo-ubuntu2404-9.10.1/cudnn-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cudnn
```

获取源代码

```yaml
git clone https://github.com/pytorch/pytorch
cd pytorch

git checkout v2.7.0 # 切换到稳定分支
# if you are updating an existing checkout
# 同步子模块配置（若URL有变更）
git submodule sync 
# 初始化/更新子模块
git submodule update --init --recursive 
```

安装依赖项

```yaml
conda install cmake ninja
pip install -r requirements.txt
pip install mkl-static mkl-include
conda install -c pytorch magma-cuda128
```

安装pytorch

```yaml
export CMAKE_PREFIX_PATH="${CONDA_PREFIX:-'$(dirname $(which conda))/../'}:${CMAKE_PREFIX_PATH}"
python setup.py develop
```

### pip安装
```yaml
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

```yaml
# 验证
python -c "import torch;x = torch.rand(5, 3);print(x)"
```

```yaml
# 验证cuda是否可用
python -c "import torch; print(torch.cuda.is_available())"
```

