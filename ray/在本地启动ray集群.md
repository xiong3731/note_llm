[Launching an On-Premise Cluster — Ray 2.46.0](https://docs.ray.io/en/latest/cluster/vms/user-guides/launching-clusters/on-premises.html#on-prem)

<font style="color:rgb(34, 39, 43);">在 AWS、GCP、Azure 及更多平台上部署您的应用程序到 Ray 集群，通常只需对现有代码进行少量更改。</font>

<font style="color:rgb(34, 39, 43);">Ray 能够从笔记本电脑到大型集群无缝扩展工作负载。虽然 Ray 只需调用 </font>`<font style="color:rgb(145, 37, 131);background-color:rgb(243, 244, 245);">ray.init</font>`<font style="color:rgb(34, 39, 43);"> 就能在单机上立即使用，但要运行多节点的 Ray 应用程序，你必须首先部署一个 Ray 集群。</font>

### <font style="color:rgb(34, 39, 43);">部署conda</font>
```shell
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh


source ~/miniconda3/bin/activate

cat > ~/.condarc <<EOF
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
EOF

pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

conda init --all
conda install conda-forge::conda-bash-completion -y
echo 'source /root/miniconda3/etc/profile.d/bash_completion.sh' >> ~/.bashrc
source ~/.bashrc
conda create -n myenv python=3.10
conda activate myenv
```

### 手动设置ray集群
```yaml
编辑 hosts文件,创建集群中ip和主机名的对应
```

```shell

pip install -U "ray[default]"
echo -e "export GLOO_SOCKET_IFNAME=eth0" >> /etc/profile && source /etc/profile
```

#### head
```shell
ray start --head --port=6379 --include-dashboard=True --dashboard-host=0.0.0.0
```

> 为什么用6379端口?
>
> 因为 **Ray 的核心底层通信机制** 就是通过 **Redis 实例** 来协调各个节点的状态和任务调度的。
>

#### worker
```shell
# ray start --address=<head-node-address:port>
ray start --address=10.60.202.139:6379
```

`ray list nodes`

#### 测试
每台都要 `pip install vllm`

```yaml
vllm serve ~/.cache/huggingface/hub/DeepSeek-R1-Distill-Qwen-1.5B \
    --tensor-parallel-size 2 \
    --pipeline-parallel-size 3 \
    --max-model-len 16384 \
    --port 8000

```

> 问题1: RuntimeError: NCCL error: unhandled system error  
答案: cuda和nvidia驱动版本不匹配,重新安装,使用cuda.run安装nvidia驱动
>
> 问题2: gloo无法连接
>
> 答案: 编辑 hosts文件,创建集群中ip和主机名的对应 ;
>
> 并设置环境变量echo -e "export GLOO_SOCKET_IFNAME=eth0" >> /etc/profile && source /etc/profile
>
> 问题3: FileNotFoundError: /root/.cache/huggingface/DeepSeek-R1-Distill-Qwen-1.5B
>
> 答案: 检查目录下是否有文件,并且ray集群下的每一台都要有模型
>

```yaml
curl http://127.0.0.1:8000/v1/chat/completions   -H "Content-Type: application/json"   -d '{
      "model": "/root/.cache/huggingface/hub/DeepSeek-R1-Distill-Qwen-1.5B",
      "messages": [
        {"role": "system", "content": "你是DeepSeek-R1-Distill-Qwen-14B"},
        {"role": "user", "content": "1+1=?"}
      ],
      "max_tokens": 1024,
      "temperature": 0.7
  }'
```

