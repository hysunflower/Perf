<!-- omit in toc -->
# Paddle Bert Base 性能测试

此处给出了基于 Paddle 框架实现的 Bert Base Pre-Training 任务的训练性能详细测试报告，包括执行环境、Paddle 版本、环境搭建、复现脚本、测试结果和测试日志。

相同环境下，其他深度学习框架的 Bert 训练性能数据测试流程，请参考：[OtherReports](./OtherReports)。

<!-- omit in toc -->
## 目录
- [一、测试说明](#一测试说明)
- [二、环境介绍](#二环境介绍)
  - [1.物理机环境](#1物理机环境)
  - [2.Docker 镜像](#2docker-镜像)
- [三、环境搭建](#三环境搭建)
- [四、测试步骤](#四测试步骤)
  - [1.单机（单卡、8卡）测试](#1单机单卡8卡测试)
  - [2.多机（32卡）测试](#2多机32卡测试)
- [五、测试结果](#五测试结果)
  - [1.Paddle训练性能](#1paddle训练性能)
  - [2.与业内其它框架对比](#2与业内其它框架对比)
- [六、日志数据](#六日志数据)
  - [1.单机（单卡、8卡）日志](#1单机单卡8卡日志)



## 一、测试说明

我们统一使用了 **吞吐能力** 作为衡量性能的数据指标。**吞吐能力** 是业界公认的、最主流的框架性能考核指标，它直接体现了框架训练的速度。

Bert Base 模型是自然语言处理领域极具代表性的模型，包括 Pre-Training 和 Fine-tune 两个子任务，此处我们选取 Pre-Training 阶段作为测试目标。在测试性能时，我们以 **sentences/sec** 作为训练期间的吞吐性能。在其它框架中，默认也均采用相同的计算方式。

测试中，我们选择如下3个维度，测试吞吐性能：

- **卡数**

   本次测试关注1卡、8卡、32卡情况下，模型训练的吞吐性能。选择的物理机是单机8卡配置。
   因此，1卡、8卡测试在单机下完成。32卡在4台机器下完成。

- **FP32/AMP**

   FP32 和 AMP 是业界框架均支持的两种精度训练模式，也是衡量框架性能的混合精度量化训练的重要维度。
   本次测试分别对 FP32 和 AMP 两种精度模式进行了测试。


- **BatchSize**

   经调研，大多框架的 Bert Base Pre-Training 任务在第一阶段 max_seq_len=128 的数据集训练时 ，均支持 FP32 模式下 BatchSize=32/48，AMP 模式下 BatchSize=64/96。因此我们分别测试了上述两种组合方式下的吞吐性能。

关于其它一些参数的说明：

- **XLA**

   本次测试的原则是测试 Bert Base 在 Paddle 下的最好性能表现，同时对比其与其它框架最好性能表现的优劣。

   因此，对于支持 XLA 的框架，我们默认打开 XLA 模式，以获得该框架最好的吞吐性能数据。

- **优化器**

   在 Bert Base 的 Pre-Training 任务上，各个框架使用的优化器略有不同。NGC TensorFlow、NGC PyTorch 均支持 LAMBOptimizer，PaddlePaddle 默认使用的是 AdamOptimizer。LAMBOptimizer 优化器由于支持**梯度聚合策略**，在多机参数更新时，通信开销更低，性能会比原生的 AdamOptimizer 优化器更好一些。

   此处我们以各个框架默认使用的优化器为准，并测试模型的吞吐性能。Paddle 后续也会支持性能更优的 LAMBOptimizer 优化器。

## 二、环境介绍
### 1.物理机环境

- 单机（单卡、8卡）
  - 系统：CentOS release 7.5 (Final)
  - GPU：Tesla V100-SXM2-32GB * 8
  - CPU：Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz * 80
  - Driver Version: 460.27.04
  - 内存：502 GB

- 多机（32卡）
  - 系统：CentOS release 6.3 (Final)
  - GPU：Tesla V100-SXM2-32GB * 8
  - CPU：Intel(R) Xeon(R) Gold 6271C CPU @ 2.60GHz * 48
  - Driver Version: 450.80.02
  - 内存：502 GB

### 2.Docker 镜像

- **镜像版本**: `paddlepaddle/paddle-benchmark:2.2.1-cuda11.2-cudnn8-runtime-ubuntu16.04`
- **Paddle 版本**: `2.2.0.post112`
- **模型代码**：[PaddleNLP](https://github.com/PaddlePaddle/PaddleNLP)
- **CUDA 版本**: `11.2`
- **cuDnn 版本:** `8.1.1`


## 三、环境搭建

各深度学习框架在公开的 Github 仓库中给出了详细的docker镜像和构建脚本，具体搭建流程请参考：[此处](./OtherReports)。

如下是 Paddle 测试环境的具体搭建流程:

- **拉取代码**
  ```bash
  git clone https://github.com/PaddlePaddle/PaddleNLP.git 
  cd PaddleNLP && git checkout 5af122b71cb0028882791804183e916912cd02aa
  cp requirements.txt examples/language_model/bert/static && cd examples/language_model/bert/static
  ```


- **构建镜像**

   ```bash
   # 拉取镜像
   docker pull paddlepaddle/paddle-benchmark:2.2.1-cuda11.2-cudnn8-runtime-ubuntu16.04

   # 创建并进入容器
   nvidia-docker run --name=test_bert_paddle -it \
    --net=host \
    --shm-size=1g \
    --ulimit memlock=-1 \
    --ulimit stack=67108864 \
    -e NVIDIA_VISIBLE_DEVICES=all \
    -v $PWD:/workspace/models \
    paddlepaddle/paddle-benchmark:2.2.1-cuda11.2-cudnn8-runtime-ubuntu16.04
    /bin/bash 
   ```

- **安装依赖**
   ```bash
   # 安装 PaddleNLP 中依赖库
   pip install -r requirements.txt
   pip install paddlenlp
   ```

- **准备数据**

   Bert 模型的 Pre-Training 任务是基于 [wikipedia](https://dumps.wikimedia.org/) 和 [BookCorpus](http://yknzhu.wixsite.com/mbweb) 数据集进行的训练的，原始数据集比较大。我们提供了一份小的、且已处理好的[样本数据集](https://bert-data.bj.bcebos.com/benchmark_sample%2Fbert_data.tar.gz)，大小 338M， 可以下载并解压到`models/`目录下。

   ```bash
   # 解压数据集
   tar -xzvf benchmark_sample_bert_data.tar.gz
   # 放到 models/ 目录
   mv benchmark_sample_bert_data.tar.gz models/bert_data
   ```

## 四、测试步骤

在 [PaddleNLP/examples/language_model/bert/static](https://github.com/PaddlePaddle/PaddleNLP/tree/develop/examples/language_model/bert/static) 目录下，我们提供了用于测试的 `run_pretrain.py` 脚本。

**重要参数：**
- **model_type**: 训练模型的类型，此处统一指定为 `bert`
- **model_name_or_path:** 预训练模型的名字或路径，此处统一指定为 `bert-base-uncased`
- **batch_size:** 每张 GPU 上的 batch_size 大小
- **use_amp:** 使用是否混合精度训练
- **max_steps:** 设置训练的迭代次数
- **logging_steps:** 日志打印的步长


### 1.单机（单卡、8卡）测试

为了更方便地复现我们的测试结果，我们提供了一键测试 benchmark 数据的脚本 `run_benchmark.sh` ，需放在 `PaddleNLP/examples/language_model/bert/static`目录下。

- **脚本内容如下：**
   ```bash
   #!/bin/bash   
   export PYTHONPATH=/workspace/models/
   export DATA_DIR=/workspace/models/bert_data/
   
   export CUDA_VISIBLE_DEVICES=0
   
   batch_size=${1:-32}
   num_gpus=${2:-1}
   use_amp=${3:-"True"}
   max_steps=${4:-500}
   logging_steps=${5:-20}
   
   if [ $num_gpus = 1 ]; then
      CMD="python ./run_pretrain.py"
   else
      unset CUDA_VISIBLE_DEVICES
      CMD="python -m paddle.distributed.launch --log_dir=./mylog --gpus=0,1,2,3,4,5,6,7 ./run_pretrain.py"
   fi
   
   $CMD \
               --max_predictions_per_seq 80
               --learning_rate 5e-5
               --weight_decay 0.0
               --adam_epsilon 1e-8
               --warmup_steps 0
               --output_dir ./tmp2/
               --logging_steps 10
               --save_steps 20000
               --input_dir=$DATA_DIR
               --model_type bert
               --model_name_or_path bert-base-uncased
               --batch_size ${batch_size}
               --use_amp ${use_amp}
               --gradient_merge_steps $(expr 67584 \/ $batch_size \/ 8)"
   ```

- **单卡启动脚本：**

  若测试单机单卡 batch_size=32、FP32 的训练性能，执行如下命令：

  ```bash
  bash run_benchmark.sh 32 1 False
  ```

- **8卡启动脚本：**

  若测试单机8卡 batch_size=64、FP16 的训练性能，执行如下命令：

  ```bash
  bash run_benchmark.sh 64 8 True
  ```


### 2.多机（32卡）测试
为了更方便的复现我们的测试结果，我们提供一键测试 benchmark 数据的脚本 `run_multi_node_benchmark.sh` ，需放在 `benchmark/bert`目录下(每台机器均需要执行)

- **脚本内容如下：**
   ```bash
   #!/bin/bash

   export PYTHONPATH=/workspace/models/PaddleNLP
   export DATA_DIR=/workspace/models/bert_data/
   export CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
   # 设置以下环境变量为您所用训练机器的IP地址
   export TRANER_IPS="10.10.0.1,10.10.0.2,10.10.0.3,10.10.0.4"
   export PADDLE_WITH_GLOO=0

   batch_size=${1:-32}
   use_amp=${2:-"True"}
   max_steps=${3:-500}
   logging_steps=${4:-20}

   CMD="python3.7 -m paddle.distributed.launch --gpus 0,1,2,3,4,5,6,7 --ips $TRAINER_IPS ./run_pretrain.py"

   $CMD \
      --model_type bert \
      --model_name_or_path bert-base-uncased \
      --max_predictions_per_seq 20 \
      --batch_size $batch_size   \
      --learning_rate 1e-4 \
      --weight_decay 1e-2 \
      --adam_epsilon 1e-6 \
      --warmup_steps 10000 \
      --input_dir $DATA_DIR \
      --output_dir ./tmp2/ \
      --logging_steps $logging_steps \
      --save_steps 50000 \
      --max_steps $max_steps \
      --use_amp $use_amp\
      --enable_addto True
   ````

- **启动脚本：**

  若测试 batch_size=32、FP32 的训练性能，执行如下命令：

  ```bash
  bash run_multi_node_benchmark.sh 32 False
  ```

  若测试 batch_size=64、FP16 的训练性能，执行如下命令：

  ```bash
  bash run_multi_node_benchmark.sh 64 True
  ```

## 五、测试结果

### 1.Paddle训练性能

- 训练吞吐率(sequences/sec)如下:

   |卡数 | FP32(BS=32) | FP32(BS=48) | AMP(BS=64) | AMP(BS=96) |
   |:-----:|:-----:|:-----:|:-----:|:-----:|
   |1 |147.14 | 152.19  | 584.22  | 622.30  |
   |8 | 1105.19  | 1149.4 | 3942.72  | 4188.46  |
   |32 | 4862.5 | 4948.4 | 18432.2 | 18480.0 |

### 2.与业内其它框架对比

- 说明：
  - 同等执行环境下测试
  - 单位：`sequences/sec`
  - max_seq_len: 128
  - BatchSize FP32下统一选择 32 和 48、AMP下统一选择 64、96


- FP32测试

  | 参数 | [PaddlePaddle](./Bert) | [NGC TensorFlow 1.15](./Bert/OtherReports/TensorFlow) | [NGC PyTorch](./Bert/OtherReports/PyTorch) |
  |:-----:|:-----:|:-----:|:-----:|
  | GPU=1,BS=32 | 157.328 | 145.33  | 129.85  |
  | GPU=1,BS=48 | 157.05  | 153.86  | 129.26 |
  | GPU=8,BS=32 | 1187.01  | 1105.46 | 1015.43 |
  | GPU=8,BS=48 | 1198.71   | 1190.22 | 1006.66 |
  | GPU=32,BS=32 | 4862.5 | 4379.4 | 3994.1 |
  | GPU=32,BS=48 | 4948.4 | 4723.5 | 3974.0 |



- AMP测试

  | 参数 | [PaddlePaddle](./Bert) | [NGC TensorFlow 1.15](./Bert/OtherReports/TensorFlow) | [NGC PyTorch](./Bert/OtherReports/PyTorch) |
  |:-----:|:-----:|:-----:|:-----:|
  | GPU=1,BS=64 | 590.55 | 466.13  | 517.23 |
  | GPU=1,BS=96 | 623.84 | 504.84  | 536.89 |
  | GPU=8,BS=64 | 4245.93  | 3351.06 | 4037.21 |
  | GPU=8,BS=96 | 4448.77  | 3772.37 | 4196.88 |
  | GPU=32,BS=64 | 18432.2 | 14773.4 | 15941.1 |
  | GPU=32,BS=96 | 18480.0 | 16554.3 | 16311.6 |



## 六、日志数据
### 1.单机（单卡、8卡）日志

- [单卡 bs=32、FP32](./logs/base_bs32_fp32_gpu1.log)
- [单卡 bs=48、FP32](./logs/base_bs48_fp32_gpu1.log)
- [单卡 bs=64、AMP](./logs/base_bs64_fp16_gpu1.log)
- [单卡 bs=96、AMP](./logs/base_bs96_fp16_gpu1.log)
- [8卡 bs=32、FP32](./logs/base_bs32_fp32_gpu8.log)
- [8卡 bs=48、FP32](./logs/base_bs48_fp32_gpu8.log)
- [8卡 bs=64、AMP](./logs/base_bs64_fp16_gpu8.log)
- [8卡 bs=96、AMP](./logs/base_bs96_fp16_gpu8.log)
- [32卡 bs=32、FP32 on GradAcc](./logs/base_bs32_fp32_gpu32_gradacc.log)
- [32卡 bs=48、FP32 on GradAcc](./logs/base_bs48_fp32_gpu32_gradacc.log)
- [32卡 bs=64、AMP on GradAcc](./logs/base_bs64_fp16_gpu32_gradacc.log)
- [32卡 bs=96、AMP on GradAcc](./logs/base_bs96_fp16_gpu32_gradacc.log)
- [32卡 bs=32、FP32 no GradAcc](./logs/base_bs32_fp32_gpu32.log)
- [32卡 bs=48、FP32 no GradAcc](./logs/base_bs48_fp32_gpu32.log)
- [32卡 bs=64、AMP no GradAcc](./logs/base_bs64_fp16_gpu32.log)
- [32卡 bs=96、AMP no GradAcc](./logs/base_bs96_fp16_gpu32.log)
