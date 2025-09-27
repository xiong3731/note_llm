[欢迎来到 vLLM！ — vLLM - 高效开源AI工具平台](https://vllm-zh.llamafactory.cn/)

## 前言
+ **vLLM** 是一个 **推理引擎**，用于高效运行大语言模型，优化显存使用和推理速度，适合企业级的多用户、低延迟需求。
+ vLLM 是基于 **PyTorch + CUDA** 的高性能推理引擎。三者关系是：**PyTorch 负责构建和管理模型，CUDA 负责加速计算，vLLM 构建在它们之上优化推理性能。**
+ **Dataflow** 是 vLLM 内部的 **任务调度机制**，通过高效调度任务，确保多任务并发推理不会发生资源冲突，提供低延迟。
+ **Transformer** 是大语言模型的基础架构，提供了计算语言的能力，是 GPT、LLaMA 等模型的核心。
+ **CUDA** 提供了 GPU 加速的底层支持，确保 vLLM 能在 GPU 上高效运行，减少计算时间。

实际应用场景：

+ **vLLM + CUDA**：大模型推理（如 GPT）通过 CUDA 实现 GPU 加速，vLLM 管理多个推理任务，确保低延迟和高吞吐。
+ **Transformer**：构成了 GPT、LLaMA 等模型的核心，支持多种应用，如文本生成、对话系统等。

## 安装(本地)
### 从源代码构建(pip3 install编译失败,放弃)
```yaml
sudo apt install -y libopenblas-dev python3-dev ninja-build cmake libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
sudo yum install -y openssl-devel zlib-devel bzip2-devel readline-devel sqlite-devel  xz-devel libffi-devel tk-devel

# 操作系统：Linux
# Python: 3.8 – 3.12
# GPU: 计算能力 7.0 或更高（例如，V100、T4、RTX20xx、A100、L4、H100 等）
wget -c https://www.python.org/ftp/python/3.12.9/Python-3.12.9.tgz
tar -zxvf Python-3.12.9.tgz  && cd Python-3.12.9/
./configure --prefix=/usr/local/python3 --with-ssl 
make clean && make && make install  

ln -sf /usr/local/python3/bin/python3 /usr/bin/python3
ln -sf /usr/local/python3/bin/pip3 /usr/bin/pip3
ln -sf /usr/bin/pip3 /usr/bin/pip
pip3 config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
```

```yaml
pip3 install numpy wheel setuptools ninja cmake
git clone https://github.com/vllm-project/vllm.git
cd vllm/
pip3 install -e .
```

### 使用pip安装
[conda](https://www.yuque.com/xiong3731/yunwei/uyqwzsarn04eilzi) # conda安装

```yaml
(base) root@gpu:~# conda create -n vllm python=3.12.9 -y
(base) root@gpu:~# conda activate vllm
(vllm) root@gpu:~# which pip
/root/miniconda3/envs/vllm/bin/pip

pip install vllm

(vllm) root@gpu:~# which vllm
/root/miniconda3/envs/vllm/bin/vllm
```

### 测试
#### 下载
`cd /root/coderepo`

```yaml
pip install modelscope
python -c "
from modelscope import snapshot_download
model_dir = snapshot_download('deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B', local_dir='/root/share/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B', revision='master')
print(model_dir)
"
```

#### 运行
```yaml
pip install bitsandbytes
vllm serve /root/coderepo/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --port 8102 \
  --quantization bitsandbytes \
  --enforce-eager
# 使用单卡加载模型，支持16K长文本推理，启用8bit量化,并禁用内核融合以便调试。
```

#### 命令行访问
```yaml
curl http://localhost:8102/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
      "model": "/root/coderepo/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B",
      "messages": [
        {"role": "system", "content": "你是一个数学助手，擅长计算和解释数字。"},
        {"role": "user", "content": "请计算：144500 ÷ 1156 等于多少？"}
      ],
      "max_tokens": 1024,
      "temperature": 0.7
  }'
```

  


## 安装(docker)
```yaml
apt install docker.io -y
```

#### 安装驱动
[容器驱动](https://www.yuque.com/xiong3731/tz203d/nohdgymaq9wvt492#MyUXv)

#### 测试
```yaml
# 以下是官方示例
docker run --runtime nvidia --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=<secret>" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model mistralai/Mistral-7B-v0.1
# /root下存放了model,所以此处挂载了/root
docker run --runtime nvidia --gpus all \
    -v /root/DeepSeek-R1-Distill-Qwen-1.5B:/root/.cache/huggingface/hub/models--DeepSeek-R1-Distill-Qwen-1.5B \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    -p 8000:8000 \
    --ipc=host \
    uhub.service.ucloud.cn/xiong/vllm/vllm-openai:20250511 \
    --model /root/.cache/huggingface/hub/models--DeepSeek-R1-Distill-Qwen-1.5B
```

```yaml
curl http://127.0.0.1:8000/v1/chat/completions   -H "Content-Type: application/json"   -d '{
      "model": "/root/.cache/huggingface/hub/models--DeepSeek-R1-Distill-Qwen-1.5B",
      "messages": [
        {"role": "system", "content": "你是一个数学助手，擅长计算和解释数字。"},
        {"role": "user", "content": "请计算：144500 ÷ 1156 等于多少？"}
      ],
      "max_tokens": 1024,
      "temperature": 0.7
  }'
  
# 成功返回
```

