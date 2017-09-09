<h2><font color=#0099FF>前言</font></h2> 

博主在上一篇中提到了两种可能的改进方法。其中方法1，扩充类似数据集，详见[Udacity Self-Driving 目标检测数据集简介与使用](http://blog.csdn.net/jesse_mx/article/details/72599220) ，由于一些原因，并未对此数据集做过多探索，一次简单训练下，mAP为64%左右，这还需要进一步探索。而方法2，说的是fine-tune已经训练好的SSD model，最近没来得及进行调参，初次实验效果有限，先把过程和原理记录下来，免得忘了，然后还会说下SSD的网络架构。

<h2><font color=#0099FF>SSD模型fine-tune</font></h2> 

<h3><font color=#0099FF>CNN fine-tune</font></h3> 

model fine-tune，就是模型微调的意思。举个例子说明一下：假如给你一个新的数据集做图片分类，这个数据集是关于汽车的，不过样本量很少，如果从零开始训练CNN（Alexnet或者VGG），容易过拟合。那怎么办呢？于是想到了迁移学习，用别人训练好的Imagenet模型来做，继承现有模型学到的特征“知识”，这样再训练的效果就会好一些。

SSD检测任务也适用fine-tune策略。之前博主介绍的SSD经典训练方法也算是一种fine-tune，为什么这么说，因为我们使用VGG_ILSVRC_16_layers_fc_reduced.caffemodel这个预训练模型，这其实是作者在Imagenet上训练的一个变种VGG，已经具备了一定的分类特征提取能力，我们继承这些“知识”，有利于模型的快速收敛。

其实，我们也可以直接去学习训练好的SSD的模型“知识”。比如，作者在COCO数据集上训练了一个SSD检测模型，这个模型对90类物体有较好的检测能力，现在我们只对其中的汽车和行人感兴趣，就只想检测这两类（COCO数据集当然也包含这两类），手头还有KITTI数据集等，现在就可以在这个模型的基础上去fine-tune，copy大部分有用的检测“知识”，并且专心学习两类物体特有的“知识”。

这里以COCO为例，介绍一下如何使用新数据集去fine-tune一个SSD COCO model，默认是SSD300x300。首先是下载[07+12+COCO](https://drive.google.com/file/d/0BzKzrI_SkD1_UFpoU01yLS1SaG8/view) ，这个是作者用VOC数据集去微调COCO模型的范例，我们以此为蓝本进行修改，主要修改点是数据集以及预训练的模型。

下载压缩包，找到`finetune_ssd_pascal.py`文件，打开后可以看到和之前的`ssd_pascal.py`区别不是很大，重要的是这么一句：

```shell
pretrain_model = "models/VGGNet/VOC0712/SSD_300x300_coco/VGG_coco_SSD_300x300.caffemodel"
```

这就是预训练的模型，可以下载的，在作者主页找到[COCO models: trainval35k](https://drive.google.com/file/d/0BzKzrI_SkD1_dUY1Ml9GRTFpUWc/view)，下载解压可得到`VGG_coco_SSD_300x300_iter_400000.caffemodel` ，再改个名字放到对应路径就行。接下来给复制一份脚本，命名为`finetune_ssd_kitti.py` ，然后修改成各种KITTI相关的名称和路径，类别数量也要改（可参考之前博文修改）。

下面运行命令开始训练看看：

```shell
cd caffe
python examples/ssd/finetune_ssd_kitti.py
```

发现有大bug，错误描述如下：

```shell
Cannot copy param 0 weights from layer 'conv4_3_norm_mbox_conf'; shape mismatch.  Source param shape is 324 512 3 3 (1492992); target param shape is 16 512 3 3 (73728). To learn this layer's parameters from scratch rather than copying from a saved net, rename the layer.
*** Check failure stack trace: ***
    @     0x7f0bddd2f5cd  google::LogMessage::Fail()
    @     0x7f0bddd31433  google::LogMessage::SendToLog()
    @     0x7f0bddd2f15b  google::LogMessage::Flush()
    @     0x7f0bddd31e1e  google::LogMessageFatal::~LogMessageFatal()
    @     0x7f0bde5c34cb  caffe::Net<>::CopyTrainedLayersFrom()
    @     0x7f0bde5ca225  caffe::Net<>::CopyTrainedLayersFromBinaryProto()
    @     0x7f0bde5ca2be  caffe::Net<>::CopyTrainedLayersFrom()
    @           0x40a849  CopyLayers()
    @           0x40bca4  train()
    @           0x4077c8  main
    @     0x7f0bdc4c6830  __libc_start_main
    @           0x408099  _start
    @              (nil)  (unknown)
Aborted (core dumped)
```

这个问题困扰了几天，后来才知道，COCO模型有90类，而本次训练的文件只有3类，conf层没有办法共享权值。解决方法就是把conf层改个名字，跳过这些层的fine-tune，意味着这部分层的参数需要从零学起。至于loc层则可以不改名，因为位置坐标和类别无关，所有类别均可共享。那么，注意到该脚本下有**两个**一模一样的CreateMultiBoxHead函数：

```python
mbox_layers = CreateMultiBoxHead(net, data_layer='data', from_layers=mbox_source_layers,
        use_batchnorm=use_batchnorm, min_sizes=min_sizes, max_sizes=max_sizes,
        aspect_ratios=aspect_ratios, steps=steps, normalizations=normalizations,
        num_classes=num_classes, share_location=share_location, flip=flip, clip=clip,
        prior_variance=prior_variance, kernel_size=3, pad=1, lr_mult=lr_mult)
```

现在，仅需要在这两个函数中添加参数`conf_postfix='_kitti'`就可以把conf层的名字了，其实就是在所有'conf'字符后面添加了'kitti'字样，新函数就变成了：

```shell
mbox_layers = CreateMultiBoxHead(net, data_layer='data', from_layers=mbox_source_layers,
        use_batchnorm=use_batchnorm, min_sizes=min_sizes, max_sizes=max_sizes,
        aspect_ratios=aspect_ratios, steps=steps, normalizations=normalizations,
        num_classes=num_classes, share_location=share_location, flip=flip, clip=clip,
        prior_variance=prior_variance, kernel_size=3, pad=1, lr_mult=lr_mult, conf_postfix='_kitti')
```

然后再运行命令，就可以正常开始训练了。博主所用脚本可以参考一下：[finetune_ssd_kitti.py](http://download.csdn.net/detail/jesse_mx/9885607)

一开始使用默认solver训练了一次，120000迭代，mAP勉强达到63%，感觉还是不够看的。后来觉着默认solver中的学习率策略可能不太对，如果使用大的学习率（比如0.001）训练太久的话，原有caffemodel的基础权重可能会被破坏的比较厉害，那效果就会打折扣，因此有大神建议博主可以每隔10000次迭代降低学习率为一半，就是暂时没空尝试，如果有效果再更新吧。

<h2><font color=#0099FF>SSD网络架构</font></h2> 

<h3><font color=#0099FF>VGGNet-SSD网络结构图</font></h3> 

如果只是训练想VGGNet的SSD，并不用太关心SSD的网络结构，毕竟有python脚本可用。可是如果要想进一步修改网络，比如把基础网络换成Mobilenet，Resnet等，或者是修改多层融合检测层，那就需要简单了解SSD的基本结构。

SSD论文给的图是这样的，大致知道了结构，细节还是看不懂。

![这里写图片描述](http://img.blog.csdn.net/20170630194322899?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当然，有人会把prototxt放进这个[Netscope](http://ethereon.github.io/netscope/quickstart.html) 中，得到可视化的结构，这是最直接的方法了，为了方便表示，训练网络train.prototxt的架构用如下简图表示（新标签页中打开可看到大图）。

![这里写图片描述](http://img.blog.csdn.net/20170630194511381?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

说明一下，为了简便，没有完整反映出6个bottom输入层；图中的“*”号表示省略写法，比如第一行的 *_perm展开后就表示conv4_3_norm_mbox_loc_perm，其他以此类推；然后mbox_priorbox层均有有第二个bottom输入，即data层，不过这里没有画出来。

下面以VGGNet-SSD为例，结合prototxt源文件说一下网络结构。

<h3><font color=#0099FF>数据层</font></h3> 

这里的数据层类型是AnnotatedData，是作者自己写的。里面有很多复杂的设置，并不能完全看懂，所以也不好随意修改，主要注意下lmdb的source，batch_size以及label_map_file即可。

```shell
# 数据层
name: "VGG_VOC0712_SSD_300x300_train"
layer {
  name: "data"
  type: "AnnotatedData"
  top: "data"
  top: "label"
  include {
    phase: TRAIN
  }
  transform_param {
    mirror: true
    mean_value: 104
    mean_value: 117
    mean_value: 123
    resize_param {
      prob: 1
      resize_mode: WARP
      height: 300
      width: 300
      interp_mode: LINEAR
      interp_mode: AREA
      interp_mode: NEAREST
      interp_mode: CUBIC
      interp_mode: LANCZOS4
    }
    emit_constraint {
      emit_type: CENTER
    }
    distort_param {
      brightness_prob: 0.5
      brightness_delta: 32
      contrast_prob: 0.5
      contrast_lower: 0.5
      contrast_upper: 1.5
      hue_prob: 0.5
      hue_delta: 18
      saturation_prob: 0.5
      saturation_lower: 0.5
      saturation_upper: 1.5
      random_order_prob: 0.0
    }
    expand_param {
      prob: 0.5
      max_expand_ratio: 4.0
    }
  }
  data_param {
    source: "examples/VOC0712/VOC0712_trainval_lmdb"
    batch_size: 32
    backend: LMDB
  }
  annotated_data_param {
    batch_sampler {
      max_sample: 1
      max_trials: 1
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        min_jaccard_overlap: 0.1
      }
      max_sample: 1
      max_trials: 50
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        min_jaccard_overlap: 0.3
      }
      max_sample: 1
      max_trials: 50
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        min_jaccard_overlap: 0.5
      }
      max_sample: 1
      max_trials: 50
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        min_jaccard_overlap: 0.7
      }
      max_sample: 1
      max_trials: 50
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        min_jaccard_overlap: 0.9
      }
      max_sample: 1
      max_trials: 50
    }
    batch_sampler {
      sampler {
        min_scale: 0.3
        max_scale: 1.0
        min_aspect_ratio: 0.5
        max_aspect_ratio: 2.0
      }
      sample_constraint {
        max_jaccard_overlap: 1.0
      }
      max_sample: 1
      max_trials: 50
    }
    label_map_file: "data/VOC0712/labelmap_voc.prototxt"
  }
}
```

### 特征提取网络

特征提取网络由两部分构成，VGGNet和新增的8个卷积层，要点有三处。

第一，这个VGGNet不同于原版，是修改过的，全名叫[VGG_ILSVRC_16_layers_fc_reduced](https://gist.github.com/weiliu89/2ed6e13bfd5b57cf81d6)是作者另一篇论文[ParseNet: Looking Wider to See Better](https://arxiv.org/abs/1506.04579)的成果，然后被用到了SSD中。

第二，为什么要新增这么8个卷积层？因为在300x300输入下，conv4_3的分辨率为38x38，而到fc7层，也还有19x19，那就必须新增一些卷积层，以产生更多低分辨率的层，方便进行**多层特征融合**。最终作者选择了6个卷积层作为bottom，分辨率如下图所示，然后依次接上若干检测层。

|   卷积层      |  分辨率  |
| :----------: | :---: |
| conv4_3_norm | 38x38 |
|      f7      | 19x19 |
|   conv6_2    | 10x10 |
|   conv7_2    |  5x5  |
|   conv8_2    |  3x3  |
|   conv9_2    |  1x1  |

第三，只有conv4_3加入了norm，这是为什么呢？原因仍然在论文中ParseNet中，大概是说，因为conv4_3的scale（尺度？）和其它层相比不同，才要加入norm使其均衡一些。

![这里写图片描述](http://img.blog.csdn.net/20170630195415585?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```shell
# 特征提取网络
layer {
  name: "conv1_1"
  type: "Convolution"
  bottom: "data"
  top: "conv1_1"
  param {
    lr_mult: 1
    decay_mult: 1
  }
  param {
    lr_mult: 2
    decay_mult: 0
  }
  convolution_param {
    num_output: 64
    pad: 1
    kernel_size: 3
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
      value: 0
    }
  }
}
#
# 省略很多层
#
layer {
  name: "fc7"
  type: "Convolution"
  bottom: "fc6"
  top: "fc7"
  param {
    lr_mult: 1
    decay_mult: 1
  }
  param {
    lr_mult: 2
    decay_mult: 0
  }
  convolution_param {
    num_output: 1024
    kernel_size: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
      value: 0
    }
  }
}
layer {
  name: "relu7"
  type: "ReLU"
  bottom: "fc7"
  top: "fc7"
}

# 到此是VGG网络层，从conv1到fc7

layer {
  name: "conv6_1"
  type: "Convolution"
  bottom: "fc7"
  top: "conv6_1"
  param {
    lr_mult: 1
    decay_mult: 1
  }
  param {
    lr_mult: 2
    decay_mult: 0
  }
  convolution_param {
    num_output: 256
    pad: 0
    kernel_size: 1
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
      value: 0
    }
  }
}
#
# 省略很多层
#
layer {
  name: "conv9_2_relu"
  type: "ReLU"
  bottom: "conv9_2"
  top: "conv9_2"
  
# 到此是新增的8层卷积层，从conv6_1到conv9_2
```

<h3><font color=#0099FF>多层融合检测网络</font></h3> 

这一部分，作者选择了6个层作为bottom，在其上连接了一系列的检测相关层，这里以f7层为例，看看它作为bottom，连接了那些层。首先是loc相关层，包括fc7_mbox_loc，fc7_mbox_loc_perm和fc7_mbox_loc_flat。

```shell
# mbox_loc层，预测box的坐标，其中24=6（default box数量）x4（四个坐标）
layer {
  name: "fc7_mbox_loc"
  type: "Convolution"
  bottom: "fc7"
  top: "fc7_mbox_loc"
  param {
    lr_mult: 1
    decay_mult: 1
  }
  param {
    lr_mult: 2
    decay_mult: 0
  }
  convolution_param {
    num_output: 24
    pad: 1
    kernel_size: 3
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
      value: 0
    }
  }
}
# mbox_loc_perm层，将上一层产生的mbox_loc重新排序  
layer {
  name: "fc7_mbox_loc_perm"
  type: "Permute"
  bottom: "fc7_mbox_loc"
  top: "fc7_mbox_loc_perm"
  permute_param {
    order: 0
    order: 2
    order: 3
    order: 1
  }
}
# mbox_loc_flat层，将perm层展平（例如将7x7展成1x49），方便拼接
layer {
  name: "fc7_mbox_loc_flat"
  type: "Flatten"
  bottom: "fc7_mbox_loc_perm"
  top: "fc7_mbox_loc_flat"
  flatten_param {
    axis: 1
  }
}

```

然后是conf相关层，包括fc7_mbox_conf，fc7_mbox_conf_perm和fc7_mbox_conf_flat。

```shell
# mbox_conf层，预测box的类别置信度，126=6（default box数量）x21（总类别数）
layer {
  name: "fc7_mbox_conf"
  type: "Convolution"
  bottom: "fc7"
  top: "fc7_mbox_conf"
  param {
    lr_mult: 1
    decay_mult: 1
  }
  param {
    lr_mult: 2
    decay_mult: 0
  }
  convolution_param {
    num_output: 126
    pad: 1
    kernel_size: 3
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
      value: 0
    }
  }
}
# box_conf_perm层，将上一层产生的mbox_conf重新排序 
layer {
  name: "fc7_mbox_conf_perm"
  type: "Permute"
  bottom: "fc7_mbox_conf"
  top: "fc7_mbox_conf_perm"
  permute_param {
    order: 0
    order: 2
    order: 3
    order: 1
  }
}
# mbox_conf_flat层，将perm层展平，方便拼接
layer {
  name: "fc7_mbox_conf_flat"
  type: "Flatten"
  bottom: "fc7_mbox_conf_perm"
  top: "fc7_mbox_conf_flat"
  flatten_param {
    axis: 1
  }
}
```

最后是priorbox层，就只有fc7_mbox_priorbox

```shell
# mbox_priorbox层，根据标注信息，经过转换，在cell中产生box真实值
layer {
  name: "fc7_mbox_priorbox"
  type: "PriorBox"
  bottom: "fc7"
  bottom: "data"
  top: "fc7_mbox_priorbox"
  prior_box_param {
    min_size: 60.0
    max_size: 111.0 # priorbox的上下界为60~111
    aspect_ratio: 2
    aspect_ratio: 3 # 共同定义default box数量为6
    flip: true
    clip: false
    variance: 0.1
    variance: 0.1
    variance: 0.2
    variance: 0.2
    step: 16 # 不同层的值不同
    offset: 0.5
  }
}
```

下面做一个表格统计一下多层融合检测层的相关重要参数：

|     bottom层     | conv4_3 |  fc7   | conv6_2 | conv7_2 | conv8_2 | conv9_2 |
| :-------------: | :-----: | :----: | :-----: | :-----: | :-----: | :-----: |
|       分辨率       |  38x38  | 19x19  |  10x10  |   5x5   |   3x3   |   1x1   |
|  default box数量  |    4    |   6    |    6    |    6    |    4    |    4    |
| loc层num_output  |   16    |   24   |   24    |   24    |   16    |   16    |
| conf层num_output |   84    |  126   |   126   |   126   |   84    |   84    |
|  priorbox层size  |  30~60  | 60~111 | 111~162 | 162~213 | 213~264 | 264~315 |
|  priorbox层step  |    8    |   16   |   32    |   64    |   100   |   300   |

<h3><font color=#0099FF>Concat层</font></h3> 

这里需要添加3个concat层，分别拼接所有的loc层，conf层以及priorbox层。

```shell
# mbox_loc层，拼接6个loc_flat层
layer {
  name: "mbox_loc"
  type: "Concat"
  bottom: "conv4_3_norm_mbox_loc_flat"
  bottom: "fc7_mbox_loc_flat"
  bottom: "conv6_2_mbox_loc_flat"
  bottom: "conv7_2_mbox_loc_flat"
  bottom: "conv8_2_mbox_loc_flat"
  bottom: "conv9_2_mbox_loc_flat"
  top: "mbox_loc"
  concat_param {
    axis: 1
  }
}
# mbox_conf层，拼接6个conf_flat层
layer {
  name: "mbox_conf"
  type: "Concat"
  bottom: "conv4_3_norm_mbox_conf_flat"
  bottom: "fc7_mbox_conf_flat"
  bottom: "conv6_2_mbox_conf_flat"
  bottom: "conv7_2_mbox_conf_flat"
  bottom: "conv8_2_mbox_conf_flat"
  bottom: "conv9_2_mbox_conf_flat"
  top: "mbox_conf"
  concat_param {
    axis: 1
  }
}
# mbox_priorbox层，拼接6个mbox_priorbox层
layer {
  name: "mbox_priorbox"
  type: "Concat"
  bottom: "conv4_3_norm_mbox_priorbox"
  bottom: "fc7_mbox_priorbox"
  bottom: "conv6_2_mbox_priorbox"
  bottom: "conv7_2_mbox_priorbox"
  bottom: "conv8_2_mbox_priorbox"
  bottom: "conv9_2_mbox_priorbox"
  top: "mbox_priorbox"
  concat_param {
    axis: 2
  }
}
```

<h3><font color=#0099FF>损失层</font></h3> 

损失层类型是MultiBoxLoss，这也是作者自己写的，在smooth_L1损失层基础上修改。损失层的bottom是三个concat层以及data层中的label，参数一般不需要改，注意其中的num_classes即可。

```shell
# 损失层
layer {
  name: "mbox_loss"
  type: "MultiBoxLoss"
  bottom: "mbox_loc"
  bottom: "mbox_conf"
  bottom: "mbox_priorbox"
  bottom: "label"
  top: "mbox_loss"
  include {
    phase: TRAIN
  }
  propagate_down: true
  propagate_down: true
  propagate_down: false
  propagate_down: false
  loss_param {
    normalization: VALID
  }
  multibox_loss_param {
    loc_loss_type: SMOOTH_L1
    conf_loss_type: SOFTMAX
    loc_weight: 1.0
    num_classes: 21
    share_location: true
    match_type: PER_PREDICTION
    overlap_threshold: 0.5
    use_prior_for_matching: true
    background_label_id: 0
    use_difficult_gt: true
    neg_pos_ratio: 3.0
    neg_overlap: 0.5
    code_type: CENTER_SIZE
    ignore_cross_boundary_bbox: false
    mining_type: MAX_NEGATIVE
  }
}
```

至此，SSD的训练网络train.prototxt基本介绍完毕，那测试网络test.prototxt和部署网络deploy.prototxt就很容易理解了，只需要保持特征检测网络和多层融合检测网络不变，copy修改其他部分就好了。

了解SSD网络架构，可以方便我们对网络进行修改，比如说训练一个最近热门的Mobilenet-SSD就是蛮好的的应用（自己构造网络尽量遵照VGGNet-SSD的范例）。


