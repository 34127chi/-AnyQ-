# 对百度开源的simnet模块梳理
**新增**
1) 代码的注释，包括预处理以及tensorflow部分;
2) 梳理[模型网络结构](#6);

应用介绍: 
1) 提供语义相似度模型的训练和预测流程;
2) 支持paddle和tensorflow两种框架进行模型训练; 
3) 提供pointwise和pairwise等方法的神经网络配置;

使用说明: 
1) 试用: 用户指定运行平台, 如下：
   paddle框架训练: sh train.sh paddle
   paddle框架预测: sh predict.sh paddle
   tensorflow框架训练: sh train.sh tensorflow
   tensorflow框架预测: sh predict.sh tensorflow

2) 用户自定义训练: 
   paddle训练目录: anyq/tools/simnet/train/paddle
   run_train.sh内为训练流程:
       指定训练参数in_conf_file_path, 训练参数参见anyq/tools/simnet/train/paddle/examples/cnn_pointwise.json
   run_infer.sh内为预测流程

   tensorflow训练目录: anyq/tools/simnet/train/tf
   run_train.sh内为训练流程:
       指定训练参数in_task_conf, 训练参数参见anyq/tools/simnet/train/tf/examples/cnn-pointwise.json
   run_infer.sh内为预测流程
   
注意事项: 
用户在本地安装paddle和tensorflow包后即可使用;

问题咨询: 
欢迎用户提交任何问题和Bug Report                                                                                                                                                                                                                               

# TensorFlow版SimNet
TensorFlow版本Simnet提供了BOW，CNN，LSTM等多种网络实现，您可以通过配置文件的形式灵活选择您需要的网络，损失函数，训练方式。

## 内容
- [代码结构](#1)
- [安装](#2)
- [数据准备](#3)
- [模型训练](#4)
- [模型推断](#5)
- [模型网络结构](#6)

## <span id="5">代码结构</span>
```
simnet
    |-tf
        |- date //示例数据
        |- examples //示例配置文件
        |- layers //网络中使用操作层的实现
        |- losses //损失函数实现
        |- nets //网络结构实现
        |- tools //数据转化及评价工具
        |- util //工具类
```

## <span id="1">安装</span>

TensorFlow版本Simnet需要TensorFlow v1.2.0，详细安装方式见[TensorFlow官网](https://www.tensorflow.org/)。

## <span id="2">数据准备</span>

SimNet可以使用Pointwise与Pairwise两种类型的数据进行训练

**Pointwise训练及测试数据格式** 

* *训练数据格式*：训练数据包含三列，依次为Query1的ID序列（ID间使用空格分割），Query2的ID序列（ID间使用空格分割），Label，每列间使用TAB分割，例如；
```
1 1 1 1 1   2 2 2 2 2   0
1 1 1 1 1   1 1 1 1 1   1
...
```
* *测试数据格式*：Pointwise测试数据格式与训练数据格式相同。

**Pairwise训练及测试数据格式** 

* *训练数据格式*：训练数据包含三列，依次为Query1的ID序列（ID间使用空格分割），Positive Query2的ID序列（ID间使用空格分割），Negative Query3的ID序列（ID间使用空格分割），每列间使用TAB分割，例如；
```
1 1 1 1 1   1 1 1 1 1   2 2 2 2 2   
1 1 1 1 1   1 1 1 1 1   3 3 3 3 3
...
```
* *测试数据格式*：测试数据格式包含三列，依次为Query1的ID序列（ID间使用空格分割），Query2的ID序列（ID间使用空格分割），Label，每列间使用TAB分割，例如；
```
1 1 1 1 1   1 1 1 1 1   1
1 1 1 1 1   2 2 2 2 2   0
3 3 3 3 3   3 3 3 3 3   1
...
```

**数据转换**
通过如下方式，可以完成数据转化
```
python tools/tf_record_writer.py
        pointwise //数据类型
        data/train_pointwise_data //输入路径
        data/convert_train_pointwise_data //输出路径
        0 //pad的ID，一般为0
        32 //序列最大长度
```
* *数据类型说明*：当数据样式为Query1的ID序列，Position Query2的ID序列，Negative Query3的ID序列时数据类型参数选择pointwise，当数据样式为Query1的ID序列，Query2的ID序列，Label时数据类型参数为pairwise

## <span id="3">模型训练</span>

数据准备完毕后，可以通过执行脚本run_train.sh启动示例训练，也可以通过如下方式启动自定义训练：
```
python tf_simnet.py
        --task train
        --task_conf examples/cnn_pointwise.json
```
**参数说明：**
* **task**: 任务类型 ，可选择train或predict。
* **task_conf**: 使用配置文件地址

**配置文件说明：** 通过配置文件您可以灵活的选择网络类型，数据类型，损失函数以及其他超参数，例如：
```
{
    "train_data":{
        "train_file": "tmp/quora/train-data-record", //训练文件路径
        "data_size": 356430, //训练集大小
        "left_slots" : [["left",32]], //left slot的名字及最大长度
        "right_slots" : [["right",32]] //right slot的名字及最大长度
    },

    "model":{
        "net_py": "./nets/mlpcnn", //网络对应模块路径
        "net_class": "MLPCnn", //网络对应类名
        "vocabulary_size": 100001, //词典大小
        "embedding_dim": 128, // Embedding维度
        "num_filters": 256, //卷机核数量
        "hidden_size": 128, //隐藏层大小
        "window_size": 3, //卷机核大小
        "loss_py": "./losses/simnet_loss", //损失对应模块路径
        "loss_class": "SoftmaxWithLoss" //损失对应类名
    },

    "global":{
        "training_mode": "pointwise", //训练模式
        "n_class": 2, //类别数目
        "max_len_left": 32, //Left slot的最大长度
        "max_len_right": 32 //Right slot的最大长度
    },

    "setting":{
        "batch_size": 64, // Batch Size
        "num_epochs": 1, // Number of Epochs
        "thread_num": 6, //线程数
        "print_iter": 100, //显示间隔
        "model_path": "tmp/model_cnn_pointwise", //模型保存路径
        "model_prefix": "cnn", //模型保存名
        "learning_rate": 0.001, //学习率
        "shuffle": 1 //是否打乱数据
    },

    "test_data":{
        "test_file": "tmp/quora/test-data-record", //测试数据路径
        "test_model_file":  "tmp/model_cnn_pointwise/cnn.epoch1", //测试使用模型
        "test_result": "tmp/result_cnn_pointwise" //测试结果文件
    },

    "graph":{
        "graph_path": "graph",
        "graph_name": "tmp/model_cnn_pointwise.protxt"
    }
}
```

## <span id="4">模型推断</span>
模型推断可以得到模型预测结果或得分，您可以通过执行脚本run_infer.sh启动示例训练，也可以通过如下方式启动自定义训练：
```
python tf_simnet.py
        --task predict
        --task_conf examples/cnn_pointwise.json
```
**参数说明：**
同模型训练部分

**配置文件说明：**
同模型训练部分


## <span id="6">模型网络结构</span>
**Bow网络结构:**
input1 embedding + input2 embedding => input1 embedding的平均 + input2 embedding的平均 => input1  + input2 的concat=> 全连接层

**KNRM网络结构:**
input1 embedding + input2 embedding => input1 与 input2 相似度矩阵(归一化后的input) => kenerl池化层(多种尺度的池化层 池化层的泛化)  => 全连接层

对应的论文:End-to-End Neural Ad-hoc Ranking with Kernel Pooling

**CNN-KNRM网络结构:**
input1 embedding + input2 embedding => cnn卷积(得到多个尺度的ngram) => input1 与 input2 相似度矩阵(归一化后的input) => kenerl池化层(多种尺度的池化层 池化层的泛化)  => 全连接层

对应的论文:Convolutional Neural Networks for Soft-Matching N-Grams in Ad-hoc Search

**LSTM网络结构:**
input1 embedding + input2 embedding => input1 encode(rnn) + input2 encode(rnn) => input1 与 input2 拼接 => 全连接层

**MatchPyramid:**
input1 embedding + input2 embedding => input1 与 input2 相似度矩阵 => 动态卷积 + 池化层 => 全连接层

**MMDNN网络结构:**
input1 embedding + input2 embedding => input1 encode(birnn) + input2 encode(birnn) => input1 与 input2 相似度矩阵 => 动态卷积+池化层  => 全连接层

**MLPCnn网络结构:**
input1 embedding + input2 embedding => input1 encode(cnn) + input2 encode(cnn) => 全连接层

**MVLSTM网络结构:**
input1 embedding + input2 embedding => input1 encode(birnn) + input2 encode(birnn) => input1 与 input2 相似度矩阵 => k-max-pooling => 全连接层
