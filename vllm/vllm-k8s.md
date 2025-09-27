[Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

### k3s 环境
master <font style="color:rgb(10, 22, 51);background-color:rgb(246, 246, 251);">10.60.202.139</font>

```shell
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

node <font style="color:rgb(10, 22, 51);background-color:rgb(234, 238, 253);">10.60.135.71 & </font><font style="color:rgb(10, 22, 51);background-color:rgb(246, 246, 251);">10.60.24.6</font>

```shell
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://10.60.202.139:6443 K3S_TOKEN=K10c2d3cfd2469fbde9e7f9941286bba873f650c95109980622b5b4a75035af06bf::server:75731e405f9d08cbb095969a18a24cd3 sh -
```

[k3s-containerd 代理](https://www.yuque.com/xiong3731/yunwei/exeppmogrcaduzbg)

### 驱动安装
[https://www.yuque.com/xiong3731/tz203d/nohdgymaq9wvt492](https://www.yuque.com/xiong3731/tz203d/nohdgymaq9wvt492)

安装显卡驱动和containerd驱动

```shell
# 显卡驱动
sudo sh NVIDIA-Linux-x86_64-570.144.run \
    --silent \
    --no-questions \
    --accept-license \
    --ui=none
nvidia-smi

# 容器驱动
mkdir  /etc/apt/sources.list.d/ -p
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

systemctl restart k3s-agent # node
systemctl restart k3s # master
# k3s 会自动查找nvidia容器运行时
grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
```

### k8s驱动
[GitHub - NVIDIA/k8s-device-plugin: NVIDIA device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin?tab=readme-ov-file#quick-start)

+ <font style="color:rgb(31, 35, 40);">NVIDIA 驱动 ~= 384.81</font>
+ <font style="color:rgb(31, 35, 40);">nvidia-docker >= 2.0 || nvidia-container-toolkit >= 1.7.0 (>= 1.11.0 以在基于 Tegra 的系统中使用集成 GPU)</font>
+ <font style="color:rgb(31, 35, 40);">nvidia-container-runtime 设置为默认的低级运行时</font>
+ <font style="color:rgb(31, 35, 40);">Kubernetes 版本 >= 1.10</font>

```shell
wget https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
# spec.template.spec 新增一条    runtimeClassName: nvidia
kubectl apply -f nvidia-device-plugin.yml
# 使用以下命令检查有没有nvidia gpu
kubectl get nodes -o json | jq '.items[].status.allocatable'

```

> 测试
>

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

root@10-60-202-139:~# kubectl logs gpu-pod
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

### vllm
[https://docs.vllm.ai/en/latest/deployment/k8s.html](https://docs.vllm.ai/en/latest/deployment/k8s.html)

```shell
# 创建仓库和秘钥
cat <<EOF |kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-models
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: hf-token-secret
type: Opaque
data:
  token: dGhpc19pc190b2tlbg==
EOF
```

使用DeepSeek-R1-Distill-Qwen-1.5B

```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deepseek-7b
  namespace: default
  labels:
    app: deepseek-7b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deepseek-7b
  template:
    metadata:
      labels:
        app: deepseek-7b
    spec:
      runtimeClassName: nvidia
      volumes:
      - name: cache-volume
        persistentVolumeClaim:
          claimName: vllm-models
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: "5Gi"
      containers:
      - name: deepseek-7b
        image: uhub.service.ucloud.cn/xiong/vllm/vllm-openai:20250511
        command: ["/bin/sh", "-c"]
        args: [
          "vllm serve /root/.cache/huggingface/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B --tensor-parallel-size 2 --enable-chunked-prefill --max_num_batched_tokens 1024"
        ]
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token-secret
              key: token
        ports:
        - containerPort: 8000
        resources:
          limits:
            cpu: "10"
            memory: 20G
            nvidia.com/gpu: "2"
          requests:
            cpu: "2"
            memory: 6G
            nvidia.com/gpu: "2"
        volumeMounts:
        - mountPath: /root/.cache/huggingface
          name: cache-volume
        - name: shm
          mountPath: /dev/shm
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 300
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 300
          periodSeconds: 5
```

> scp -r  DeepSeek-R1-Distill-Qwen-1.5B g3:/var/lib/rancher/k3s/storage/pvc-db592924-ac30-4288-9209-3cc2b939e13d_default_vllm-models/deepseek-ai/  
传输一下文件
>

```shell
curl http://127.0.0.1:8000/v1/chat/completions   -H "Content-Type: application/json"   -d '{
      "model": "/root/.cache/huggingface/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B",
      "messages": [
        {"role": "system", "content": "你是DeepSeek-R1-Distill-Qwen-14B"},
        {"role": "user", "content": "你是谁,你是多大参数的模型"}
      ],
      "max_tokens": 1024,
      "temperature": 0.7
  }'
```

> success
>

