# NPU训练最佳实践

## 目录
- [环境准备](#环境准备)
- [微调](#微调)
- [推理](#推理)


## 环境准备

实验环境：8 * 昇腾910B3 64G

```shell
# 创建新的conda虚拟环境(可选)
conda create -n npu python=3.10.12 -y
conda activate npu
# 设置pip全局镜像 (可选,加速下载)
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装ms-swift(当前推荐从源码安装, 待发版后可直接pip安装)
git clone https://github.com/modelscope/swift.git
cd swift
pip install -e '.[llm]'
# 安装torch-npu
pip install torch-npu
# 如果你想要使用deepspeed(控制显存占用,训练速度会有一定下降)
pip install deepspeed -U
# datasets==2.19.0不向下兼容,需指定安装2.18.0版本
pip install datasets==2.18.0
# 安装依赖缺失的包
pip install decorator

# 环境对齐 (可选,通常不需要运行. 如果你运行错误, 可以跑下面的代码, 仓库使用最新环境测试)
pip install -r requirements/framework.txt  -U
pip install -r requirements/llm.txt  -U

```

测试环境是否安装正确,NPU能否被正常加载：
```python
from transformers.utils import is_torch_npu_available
import torch
import torch_npu

torch.randn((10,), device='npu:0')
torch.npu.set_device(0)

print(is_torch_npu_available())  # True
print(torch.npu.device_count())  # 8
```
查看NPU的P2P连接,这里看到每个NPU都通过7条HCCS与其他NPU互联
```shell
(valle) root@valle:~/src# npu-smi info -t topo
	   NPU0       NPU1       NPU2       NPU3       NPU4       NPU5       NPU6       NPU7       CPU Affinity
NPU0       X          HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       144-167
NPU1       HCCS       X          HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       144-167
NPU2       HCCS       HCCS       X          HCCS       HCCS       HCCS       HCCS       HCCS       96-119
NPU3       HCCS       HCCS       HCCS       X          HCCS       HCCS       HCCS       HCCS       96-119
NPU4       HCCS       HCCS       HCCS       HCCS       X          HCCS       HCCS       HCCS       0-23
NPU5       HCCS       HCCS       HCCS       HCCS       HCCS       X          HCCS       HCCS       0-23
NPU6       HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       X          HCCS       48-71
NPU7       HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       HCCS       X          48-71

Legend:

  X    = Self
  SYS  = Path traversing PCIe and NUMA nodes. Nodes are connected through SMP, such as QPI, UPI.
  PHB  = Path traversing PCIe and the PCIe host bridge of a CPU.
  PIX  = Path traversing a single PCIe switch
  PXB  = Path traversing multipul PCIe switches
  HCCS = Connection traversing HCCS.
  NA   = Unknown relationship.

```
查看NPU状态,
[npu-smi命令详解](https://support.huawei.com/enterprise/zh/doc/EDOC1100079287/10dcd668)
```shell
(valle) root@valle:~/src# npu-smi info
+------------------------------------------------------------------------------------------------+
| npu-smi 24.1.rc1.b030            Version: 24.1.rc1.b030                                        |
+---------------------------+---------------+----------------------------------------------------+
| NPU   Name                | Health        | Power(W)    Temp(C)           Hugepages-Usage(page)|
| Chip                      | Bus-Id        | AICore(%)   Memory-Usage(MB)  HBM-Usage(MB)        |
+===========================+===============+====================================================+
| 0     910B3               | OK            | 101.8       43                0    / 0             |
| 0                         | 0000:C1:00.0  | 0           0    / 0          3318 / 65536         |
+===========================+===============+====================================================+
| 1     910B3               | OK            | 92.0        39                0    / 0             |
| 0                         | 0000:C2:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 2     910B3               | OK            | 102.0       40                0    / 0             |
| 0                         | 0000:81:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 3     910B3               | OK            | 99.8        40                0    / 0             |
| 0                         | 0000:82:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 4     910B3               | OK            | 98.6        45                0    / 0             |
| 0                         | 0000:01:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 5     910B3               | OK            | 99.7        44                0    / 0             |
| 0                         | 0000:02:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 6     910B3               | OK            | 103.8       45                0    / 0             |
| 0                         | 0000:41:00.0  | 0           0    / 0          3314 / 65536         |
+===========================+===============+====================================================+
| 7     910B3               | OK            | 98.2        44                0    / 0             |
| 0                         | 0000:42:00.0  | 0           0    / 0          3315 / 65536         |
+===========================+===============+====================================================+

```
## 微调
以下介绍LoRA的微调, 全参数微调设置参数`--sft_type full`即可.

| 模型大小 | NPU数量 | deepspeed类型 | 最大显存占用量   |
|------|-------|-------------|-----------|
| 7B   | 1     | None        | 1 * 28 GB |
| 7B   | 4     | None        | 4 * 22 GB |
| 7B   | 4     | zero2       | 4 * 28 GB |
| 7B   | 4     | zero3       | 4 * 22 GB |
| 7B   | 8     | None        | 8 * 22 GB |
| 14B  | 1     | None        | 1 * 45 GB |
| 14B  | 8     | None        | 8 * 51 GB |
| 14B  | 8     | zero2       | 8 * 49 GB |
| 14B  | 8     | zero3       | 8 * 31 GB |
### 单卡训练

通过如下命令启动单卡微调:

```shell
# 实验环境: 昇腾910B3
# 显存需求: 28 GB
# 运行时长: 8小时
ASCEND_RT_VISIBLE_DEVICES=0 \
swift sft \
    --model_type qwen1half-7b-chat \
    --dataset blossom-math-zh \
    --num_train_epochs 5 \
    --sft_type lora \
    --output_dir output \
```


### 数据并行训练,4卡ddp, qwen1.5-7B-Chat

```shell
# 实验环境: 4 * 昇腾910B3
# 显存需求: 4 * 22 GB
# 运行时长: 2小时
NPROC_PER_NODE=4 \
ASCEND_RT_VISIBLE_DEVICES=0,1,2,3 \
swift sft \
    --model_type qwen1half-7b-chat \
    --dataset blossom-math-zh \
    --num_train_epochs 5 \
    --sft_type lora \
    --output_dir output \
```


### Deepspeed训练

ZeRO2:
```shell
# 实验环境: 4 * 昇腾910B3
# 显存需求: 4 * 28GB
# 运行时长: 3.5小时
NPROC_PER_NODE=4 \
ASCEND_RT_VISIBLE_DEVICES=0,1,2,3 \
swift sft \
    --model_type qwen1half-7b-chat \
    --dataset blossom-math-zh \
    --num_train_epochs 5 \
    --sft_type lora \
    --output_dir output \
    --deepspeed default-zero2 \
```

ZeRO3:
```shell
# 实验环境: 4 * 昇腾910B3
# 显存需求: 4 * 22 GB
# 运行时长: 8.5小时
NPROC_PER_NODE=4 \
ASCEND_RT_VISIBLE_DEVICES=0,1,2,3 \
swift sft \
    --model_type qwen1half-7b-chat \
    --dataset blossom-math-zh \
    --num_train_epochs 5 \
    --sft_type lora \
    --output_dir output \
    --deepspeed default-zero3 \
```


## 推理

原始模型:
```shell
ASCEND_RT_VISIBLE_DEVICES=0 swift infer --model_type qwen1half-7b-chat
```

LoRA微调后:
```shell
ASCEND_RT_VISIBLE_DEVICES=0 swift infer --ckpt_dir xxx/checkpoint-xxx --load_dataset_config true
```
