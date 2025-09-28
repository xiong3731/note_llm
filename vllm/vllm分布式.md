### docker
#### 环境
| 10.60.215.44 ubuntu | 3090*2gpu |
| --- | --- |
| 10.60.246.98 ubuntu | 3090*2gpu |
| 10.60.87.236 ubuntu | 3080ti*2gpu |


docker

```yaml
apt install docker.io -y
systemctl enable docker --now
```

安装三个驱动[驱动 安装](https://www.yuque.com/xiong3731/tz203d/nohdgymaq9wvt492)

```yaml
apt install build-essential pkg-config xserver-xorg-dev -y
lspci | grep -i vga
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/570.144/NVIDIA-Linux-x86_64-570.144.run
# 3090和3080ti用的是同一类型的驱动
sh NVIDIA-Linux-x86_64-570.144.run
nvidia-smi
# 图形化界面选择 ,日志 /var/log/nvidia-installer.log
 NVIDIA Proprietary -> ok -> ok -> ok -> ok -> No -> ok
 # sudo ./NVIDIA-Linux-x86_64-570.144.run --advanced-options 可以查看命令行安装的参数
 wget https://developer.download.nvidia.com/compute/cuda/12.9.0/local_installers/cuda_12.9.0_575.51.03_linux.run
  sh cuda_12.9.0_575.51.03_linux.run --silent  --toolkit
  cat >> /etc/profile << 'EOF'
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
EOF
source /etc/profile

mkdir  /etc/apt/sources.list.d/ -p
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

nvidia-ctk runtime configure --runtime=docker
# cat /etc/docker/daemon.json
systemctl restart docker
```

#### 分布式部署
[Distributed Inference and Serving — vLLM](https://docs.vllm.ai/en/latest/serving/distributed_serving.html)

```yaml
git clone git@github.com:vllm-project/vllm.git
# 找到这个文件
vllm/examples/online_serving/run_cluster.sh
```

<details class="lake-collapse"><summary id="u1d9c5839"><span class="ne-text">run_cluster.sh</span></summary><pre data-language="yaml" id="Ys80u" class="ne-codeblock language-yaml"><code>#!/bin/bash

# Check for minimum number of required arguments
if [ $# -lt 4 ]; then
    echo &quot;Usage: $0 docker_image head_node_address --head|--worker path_to_hf_home [additional_args...]&quot;
    exit 1
fi

# Assign the first three arguments and shift them away
DOCKER_IMAGE=&quot;$1&quot;
HEAD_NODE_ADDRESS=&quot;$2&quot;
NODE_TYPE=&quot;$3&quot;  # Should be --head or --worker
PATH_TO_HF_HOME=&quot;$4&quot;
shift 4

# Additional arguments are passed directly to the Docker command
ADDITIONAL_ARGS=(&quot;$@&quot;)

# Validate node type
if [ &quot;${NODE_TYPE}&quot; != &quot;--head&quot; ] &amp;&amp; [ &quot;${NODE_TYPE}&quot; != &quot;--worker&quot; ]; then
    echo &quot;Error: Node type must be --head or --worker&quot;
    exit 1
fi

# Define a function to cleanup on EXIT signal
cleanup() {
    docker stop node
    docker rm node
}
trap cleanup EXIT

# Command setup for head or worker node
RAY_START_CMD=&quot;ray start --block&quot;
if [ &quot;${NODE_TYPE}&quot; == &quot;--head&quot; ]; then
    RAY_START_CMD+=&quot; --head --port=6379&quot;
else
    RAY_START_CMD+=&quot; --address=${HEAD_NODE_ADDRESS}:6379&quot;
fi

# Run the docker command with the user specified parameters and additional arguments
docker run \
    --entrypoint /bin/bash \
    --network host \
    --name node \
    --shm-size 10.24g \
    --gpus all \
    -v &quot;${PATH_TO_HF_HOME}:/root/.cache/huggingface&quot; \
    &quot;${ADDITIONAL_ARGS[@]}&quot; \
    &quot;${DOCKER_IMAGE}&quot; -c &quot;${RAY_START_CMD}&quot;</code></pre></details>
在头节点上

> 第四个参数,是因为我把model放在了/root/下面,所以是/root      
>
> /root/DeepSeek-R1-Distill-Qwen-14B
>

```yaml
# 对于头节点上的这个脚本可以修改33行,启动dashboard,否则后续ray list nodes 无法使用(可选)
# 33 RAY_START_CMD="pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple && pip install -U ray[default]==2.45.0 && ray start --block --include-dashboard=True --dashboard-host=0.0.0.0"

bash run_cluster.sh \
                uhub.service.ucloud.cn/xiong/vllm/vllm-openai:20250511 \
                10.60.215.44 \
                --head \
                /root \
                -e VLLM_HOST_IP=10.60.215.44 \
                -e GLOO_SOCKET_IFNAME=eth0
```

其他节点

```yaml
RAY_START_CMD="pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple && pip install -U ray[default]==2.45.0 && ray start --block"
bash run_cluster.sh \
                uhub.service.ucloud.cn/xiong/vllm/vllm-openai:20250511 \
                10.60.215.44 \
                --worker \
                /root \
                -e VLLM_HOST_IP=10.60.246.98 \
                -e GLOO_SOCKET_IFNAME=eth0
```

```yaml
bash run_cluster.sh \
                uhub.service.ucloud.cn/xiong/vllm/vllm-openai:20250511 \
                10.60.215.44 \
                --worker \
                /root \
                -e VLLM_HOST_IP=10.60.87.236 \
                -e GLOO_SOCKET_IFNAME=eth0
```

[http://10.60.215.44:8265/#/cluster](http://10.60.215.44:8265/#/cluster)

#### 测试
```yaml
docker exec -it node /bin/bash
ray status
ray list nodes

vllm serve /root/.cache/huggingface/DeepSeek-R1-Distill-Qwen-14B \
     --tensor-parallel-size 2 \
     --pipeline-parallel-size 3 \
  --max-model-len 16384 \
  --port 8000 
```

> q: 为什么--tensor-parallel-size不是显卡的数量6
>
> a: 
>
> + **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">模型结构</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">DeepSeek-R1-Distill-Qwen-14B</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);"> </font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">的 </font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">注意力头数（attention heads）是 40</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">。</font>
> + **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">Tensor Parallelism（TP）</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：你设置了 </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">--tensor-parallel-size 6</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">（6 路张量并行），但 </font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">40 不能被 6 整除</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">，导致 vLLM 无法正确分配计算任务。</font>
> + <font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">推荐的参数     --tensor-parallel-size 2  --pipeline-parallel-size 3  2卡+3节点并行</font>
>

```yaml
curl http://127.0.0.1:8000/v1/chat/completions   -H "Content-Type: application/json"   -d '{
      "model": "/root/.cache/huggingface/DeepSeek-R1-Distill-Qwen-14B",
      "messages": [
        {"role": "system", "content": "你是DeepSeek-R1-Distill-Qwen-14B"},
        {"role": "user", "content": "你是谁,你是多大参数的模型"}
      ],
      "max_tokens": 1024,
      "temperature": 0.7
  }
```



