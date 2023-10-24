# 基于 LLaMA2 模型的端到端训练教程

本文档介绍基于 ChatLearn, Megatron-LM 框架和 LLaMA2 模型的训练流程。包含三阶段的训练：SFT, Reward 和 RLHF 训练。

# Setup: 镜像和代码准备

## 镜像
推荐参考 `https://github.com/alibaba/ChatLearn/tree/master/docker/ngc/Dockerfile.ngc22.10` 准备镜像。

## 代码

在这个示例中，我们需要下载以下相关代码。

```bash
# 下载为支持Megatron-LM
git clone https://github.com/NVIDIA/Megatron-LM.git
git checkout 954a65b04
# 下载ChatLearn代码
git clone https://github.com/alibaba/ChatLearn.git
```

**以下是这个 Tutorial 脚本中使用的通用环境变量集合：**

| ENV | 含义 |
| --- | --- |
| `CHATLEARN` | ChatLearn 代码仓库 clone 存放的位置 [https://github.com/alibaba/ChatLearn.git](https://github.com/alibaba/ChatLearn.git) |
| `MEGATRON` | Megatron-LM-ChatLearn 代码仓库 clone 存放的位置 [https://github.com/NVIDIA/Megatron-LM.git](https://github.com/NVIDIA/Megatron-LM.git) |
| `DATASET_ROOT` | 存放SFT/Reward/RLHF训练数据集合的根目录 |
| `TOKENIZER_PATH` | Tokenizer 使用的 vocab_file 所在的文件夹 |


# Step1: SFT

SFT 指的是使用有标注的对话数据来微调预训练语言模型的过程。在这个示例中，我们需要准备训练数据和下载预训练的模型，然后开始一个简单的 SFT 训练示例。

## 1.1 准备训练数据

将 SFT 数据的问题 - 回复配对的样本，整理到一个 jsonl 文件中，其中 jsonl 文件中每一行为一条 SFT 数据，形式为如下的 Python 字典格式：

```json
{'query': 问题，'response': 回复}
```

以 Anthropic 的 helpful&harmless 的数据为例，使用如下代码，会存一个 `$DATASET_ROOT/sft/train.jsonl`.

```bash
cd ${CHATLEARN}/examples/megatron/step1_sft/
DATASET_ROOT=$path_to_dataset_root
python prepare_data.py $DATASET_ROOT
```

## 1.2 下载和转化预训练模型

若使用来自于 HuggingFace transformers 的模型，首先需要下载预训练 checkpoint，比如 HuggingFace Hub 中的 LLaMA2 模型：`decapoda-research/llama-13b-hf`，或是本地保存好的 SFT 模型；
然后使用如下代码，将 HuggingFace transformers 模型转化为 Megatron-LM 模型格式；在这个例子中，我们会将模型转换成 `TP (tensor_model_parallel_size)=8，PP (pipeline_model_parallel_size)=1` 的 checkpoint, 模型会存放在`MEGATRON_LLAMA_CKPT_PATH`中。

```bash
MEGATRON=path-to-megatron
cd $MEGATRON

HF_FORMAT_DIR=path-to-hf-model
TOKENIZER_MODEL=$HF_FORMAT_DIR/tokenizer.model
MEGATRON_FORMAT_DIR=path-to-meg-model

python tools/checkpoint/util.py \
    --model-type GPT \
    --loader llama2_hf \
    --saver megatron \
    --target-tensor-parallel-size 8 \
    --load-dir ${HF_FORMAT_DIR} \
    --save-dir ${MEGATRON_FORMAT_DIR} \
    --tokenizer-model ${TOKENIZER_MODEL}
```


## 1.3 开启 SFT 训练

[阿里云 PAI DLC](https://www.aliyun.com/activity/bigdata/pai-dlc)[2]可以非常便捷高效地支持各种任务的训练。下面的脚本是一个 SFT 的训练样例。其中 `DATASET_PATH` 为 SFT 训练集路径，比如`$DATASET_ROOT/sft/train.jsonl`，在这个例子中，我们假设 tokenizer 存放的路径和模型 checkpoint 存放的路径相同。

```bash
export CHATLEARN=path-to-chatlearn
export MEGATRON=path-to-megatron-lm-extension
cd ${CHATLEARN}/examples/megatron/step1_sft/

MODEL_SIZE=7B \
LOAD_PATH=$MEGATRON_LLAMA2_CKPT_PATH \
TOKENIZER_MODEL=$LLAMA2_TOKENIZER_MODEL \
DATASET_PATH=$DATASET_ROOT/sft/ \
bash llama2_sft.sh
```

训练 log 和训练完成的模型默认会存放在`${CHATLEARN}/output/step1_sft`中，具体的定义详见`${CHATLEARN}/examples/megatron/step1_sft/llama2_sft.sh`脚本。

以下为 PAI-DLC 创建任务的页面截图，选择作业类型为 `PyTorch`, 同时将上述命令修改后粘贴到`执行命令`窗口中。13B SFT 训练需要 8 A100-80GB/A800-80GB/H800-80GB GPU 卡的资源。在这里假设集群中都是同构的 GPU 卡，同时每个节点上配置了 8 卡的资源。在申请资源的时候，需要申请节点数量为 1，GPU（每节点）卡数为 8，同时设置节点镜像为 ChatLearn 编译后的镜像地址。

![image.png](../images/sft_dlc_1.jpg)

![image.png](../images/sft_dlc_2.jpg)

如果您想在非 PAI-DLC 的其他环境提交分布式训练，在每个节点上，执行脚本需要配置以下环境变量：

```bash
export MASTER_ADDR=xxx
export MASTER_PORT=xxx
export WORLD_SIZE=xxx
export GPUS_PER_NODE=8
export RANK=xx
```

# Step2: Reward 模型训练

Reward 模型指的是在 RLHF 中作为人类评价的代理，对模型产生的问题回复进行实时评价打分的模型，Reward 模型输入问题以及模型回复，可以产生一个标量表示模型回复的质量。

## 2.1 准备训练数据

1. 首先准备问题 - 不同回复配对的样本，整理到一个 jsonl 文件中，其中 jsonl 文件中每一行为一条 Reward 模型训练数据，形式为如下的 Python 字典格式：

```json
{'query': 问题，'response': [回复 1, 回复 2, .....], 'score': [score1, score2, .....]}
```

其中 score 的值越高意味着对应回复的质量越高，越贴近人类偏好。

2. 以 Anthropic 的 helpful&harmless 的数据为例，使用如下代码，会存一个 `$DATASET_ROOT/rm/train.jsonl` 和 `$DATASET_ROOT/rm/dev.jsonl`.

```bash
cd ${CHATLEARN}/examples/megatron/step2_reward/
DATASET_ROOT=path-to-dataset-root
python prepare_data.py $DATASET_ROOT
```

## 2.2 开启 Reward 模型训练

依据 InstructGPT[1]，Reward 模型训练基于 SFT 训练产生的模型 checkpoint 初始化，训练代码如下：

```bash
export CHATLEARN=path-to-chatlearn
export MEGATRON=path-to-megatron-lm-extension
cd ${CHATLEARN}/examples/megatron/step2_reward/

LOAD_PATH=path-to-sft-ckpt \
TOKENIZER_MODEL=$LLAMA2_TOKENIZER_MODEL \
DATASET_PATH=$DATASET_ROOT/rm/ \
bash llama2_reward.sh
```

训练 log 和训练完成的模型默认会存放在`${CHATLEARN}/output/step2_reward`中，具体的定义详见`${CHATLEARN}/examples/megatron/step2_reward/llama2_reward.sh`脚本。
在 PAI-DLC 上提交运行作业可以参考 SFT 训练的提交方式。

# Step3: RLHF 训练
RLHF 指的是在一个只有指令的数据集上尝试不同的回复然后吸取 Reward 模型给不同回复的 reward 的监督信号的过程。

## 3.1 准备训练数据

1. 首先准备一个需要被探索的指令数据集，整理到一个 jsonl 文件中，其中 jsonl 文件中每一行为一条指令，格式为

```json
{"prompt": 问题}
```

2. 以 Anthropic 的 helpful&harmless 的数据为例，使用如下代码，会存一个`$DATASET_ROOT/rlhf/train.jsonl` 和`$DATASET_ROOT/rlhf/dev.jsonl`：
```bash
cd ${CHATLEARN}/examples/megatron/step3_rlhf/
DATASET_ROOT=path-to-dataset-root
python prepare_data.py $DATASET_ROOT
```
## 3.2 开启 RLHF 训练

[阿里云 PAI DLC](https://www.aliyun.com/activity/bigdata/pai-dlc)[2]可以非常便捷高效地支持 RLHF 任务的训练。以下是一个 LLaMA2-13B 的 Policy 和 13B 的 Reward 模型的训练脚本。在这个例子中，用户需要设置 `POLICY_LOAD` 为 SFT 产出的 checkpoint 路径，Policy 模型和 Reference 模型将以 SFT 的 checkpoint 初始化。`REWARD_LOAD` 为 Reward 训练产出的 checkpoint 路径，同时，用户可以指定 load checkpoint 对应的 iteration 数。Reward 模型和 Value 模型将以 Reward 模型的权重作初始化。`VOCAB_FILE` 为 `LlamaTokenizer` 所需文件所在的文件夹路径。

```bash
export CHATLEARN=path-to-chatlearn
export MEGATRON=path-to-megatron-lm-extension
export DATASET_PATH=$DATASET_ROOT/rlhf/train.jsonl

cd ${CHATLEARN}/examples/megatron/step3_rlhf

export exp_name=any_experiment_name_you_like

POLICY_LOAD=path-to-sft-ckpt \
REWARD_LOAD=path-to-trained-rm-checkpoint \
REWARD_LOAD_ITERATION=1000 \
TOKENIZER_MODEL=$LLAMA2_TOKENIZER_MODEL \
bash run_scripts/llama2/run_7b_7b.sh
```

以下为 PAI-DLC 创建任务的页面截图，选择作业类型为 PyTorch, 同时将上述命令修改后粘贴到`执行命令`窗口中。同时需要填写高级配置`customPortList=30000-30050,createSvcForAllWorkers=true`。
7B Policy + 7B Reward 的 RLHF 训练需要 8 A100-80GB/A800-80GB/H800-80GB GPU 卡的资源。在这里假设集群中都是同构的 GPU 卡，同时每个节点上配置了 8 卡的资源。在申请资源的时候，需要申请节点数量为 1，GPU（每节点）卡数为 8，同时设置节点镜像为 ChatLearn 编译后的镜像地址。
![image.png](../images/rlhf_dlc_1.jpg)
![image.png](../images/rlhf_dlc_2.jpg)


# Reference

1. Training language models to follow instructions with human feedback，[https://arxiv.org/abs/2203.02155](https://arxiv.org/abs/2203.02155)
2. 阿里云机器学习 PAI-DLC：[https://www.aliyun.com/activity/bigdata/pai-dlc](https://www.aliyun.com/activity/bigdata/pai-dlc)