### 2019年12月4日
#### 主体内容
1. 使用滑动窗口生成数据，**窗口的大小为1000*1000，滑动步长为500**。 数据标注判断： 1、**相交面积比大于50%（在癌细胞在滑动窗口内也是属于这个集合）** 2、**相交面积大于500000**（滑动窗口面积的一半，这个是考虑到癌细胞标注尺寸太大，超过1000，又不能简单的用相交面积比来衡量，500000这个数值有待改善）。  
2. Caffe1和Caffe2(Pytorch中)相互编译难度太大，例如：Caffe1中的set_random_seed(), set_mode_cpu()和set_mode_gpu()等与caffe2中的调用方式很不一样，**建议更换Detectron2**，Detectron是Facebook AI Reserch开源的目标检测代码库，方便调用多种不同的目标检测算法（如：Faster R-CNN, RetinaNet等），它有两个版本：  
    (1) [Detectron1](https://github.com/facebookresearch/Detectron/): 使用Python2和Caffe2。  
    (2) [Detectron2](https://github.com/facebookresearch/detectron2): 使用Python3和Pytorch。  
3. Train的文件放置新生成的patch文件。
4. 关于bashrc文件备份且替换旧文件的解决方案：  
```
# 该方法不可行
cp -f ./bashrc ~/.bashrc
source ~/.bashrc
# 得手动输入：
export PYTHONPATH=/home/admin/jupyter/kfbreader-linux:$PYTHONPATH
export LD_LIBRARY=/opt/conda/lib:/home/admin/jupyter/kfbreader-linux:$LD_LIBRARY_PATH
```
5. 关于需要做的数据统计内容：  
    (1) 训练数据1690张kfb文件，1440张有标注（即POS），250张没标注（即NEG，未见上皮内细胞病变）。  
    (2) ROI总数情况-异常细胞POS数量:阴性细胞NEG数量： 46441:28881（1.61：1.00）  
    (3) ROI的平均宽和高：3324，3233    
    (4) 关于六类异常细胞POS各自标注框特别大的情况（宽或高大于1000的数量情况）：  
    
|细胞类别|数量|
|:---:|:---:|
|ASC-H(非典型鳞状细胞倾向上皮细胞内高度)|2|  
|ASC-US(非典型鳞状细胞不能明确意义)|0|  
|HSIL(上皮内高度病变)|1|  
|LSIL(上皮内低度病变)|0|  
|Candida(某类阴性异常细胞)|134|    
|Trichomonas(某类阴性异常细胞)|0|  
        
   (5) 关于六类异常细胞POS的统计情况：  

|细胞类别|数量|平均宽/高|最小面积宽/高|最大面积宽/高|最小面积|最大面积|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|ASC-H|6194|100,101|12,14|856，959|168|820904|  
|ASC-US|5945|104,104|19,20|875,694|380|607250|  
|HSIL|2818|136,134|18,21|941,740|378|696340|  
|LSIL|3382|130,130|25,25|883,770|625|679910|  
|Candida|1651|366,348|19,22|5500,4176|418|22968000|  
|Trichomonas|11510|24,24|9,9|219,282|81|61758|  

6. 复赛测试集提供350张fkb数文件，给出ROI区域内6类异常细胞的位置、类别和概率。评测指标为**mAP(IOU=0.5, VOC2010)**。复赛评测没有其他超参数，不会过滤选手提交的预测结果。（没有概率和目标个数阈值）。  
7. 生成数据有**75322张patch图和json标注文件**，其中json文件占295M，jpg图文件占20G。  

#### 后续需要留意：
1. 相交面积相对百分比和绝对面积的阀值考虑，和滑动时可以考虑在滑动窗口中心点上加入随机波动。对于初赛有些大的异常细胞，有些医生会标注为大框，有些医生标注多个小框，复赛医生审核环节统一为大框。所以复赛数据可能出现标注框特别大的情况，比如上千个像素宽或高。而就目前数据生成方式来说，当遇到比patch面积更大的标签框时，滑动的得到的patch会产生一整个都是POS（即1000*1000都是异常细胞），那样模型无法得到适量的负样本进行训练，有正负样本不平均问题，这可得看标注框特别大的情况是怎样的。(补充：据数据统计情况来看，这个问题不大，但为了保险起见**把size>900x900的pos筛选掉**了)  
2. 复赛允许使用初赛预训练模型。我们暂时先不用初赛数据去训练得到一个预训练模型，还是用公开数据集如ImageNet预训练好的模型作为复赛预训练模型，后续我们再尝试对比下初赛数据得到的预训练模型会不会对复赛有帮助。

### 2019年12月7日
#### 主体内容
下面记录运行Detectron2遇到的一些Bugs及解决方案：
1. 文件/FSF/detectron2/detectron2/utils/events.py中from torch.utils.tensorboard  import SummaryWriter导入模块错误，原因是现有pytorch版本中torch.utils没有SummaryWriter这个模块，因此改用from tensorboardX import SummaryWriter。  
2. AssertionError: Box regression deltas become infinite or NaN! 因为学习率设置不合理，导致无法回归无法收敛。这个得看论文[Large Minibatch SGD论文](https://arxiv.org/pdf/1706.02677.pdf)。GPU数量，学习率和SOLVER.IMS_PER_BATCH设置如下：  

|GPU|学习率|SOLVER.IMS_PER_BATCH|
|:---:|:---:|:---:|  
|1|0.0025|2|  
|2|0.0050|4|  

### 2019年12月9日
#### 主体内容
1. Faster-RCNN初次实验在LeaderBoard的结果如下：

|Hyperparameter|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|基础设置|0.1697|0.1095|0.0961|0.1496|0.2922|0.1651|0.2059|9.1hrs|1.6hrs  

注：基础设置如下：  
(1) 模型：ResNet50  
(2) 训练数据集：train+val， IOU>900*900的pos筛选掉了  
(3) SOLVER.STEPS: (12000, 16000)  # LR衰减的步数，当到SOLVER.STEPS时，衰减率减0.1×LR。  
(4) MAX_ITER: 40000  
(5) BASE_LR: 0.0050  
(6) IMS_PER_BATCH: 4  
(7) FINAL_NMS_THRESH: 0.1  
(8) 数据增强： 无  
(9) stride_proportion: 0.5  

2. 未来优化手段（优先级：高到底）：  

|手段|负责人|完成情况|  
|:---:|:---:|:---:|  
|模型结构换ResNet101+FPN|思凡|已完成于2019/12/10|
|SOLVER.STEPS|思凡|已完成2019/12/11|
|stride_proportion|宏峰|已完成2019/12/12|  
|MAX_ITER, SOLVER.STEPS|宏峰|已完成于2019/12/13|  
|数据增强-水平/垂直翻转|宏峰|已完成于2019/12/15|   

### 2019年12月10日
#### 主体内容
1. Faster-RCNN实验2主要是研究**模型结构**，在LeaderBoard的结果如下：

|模型结构|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|ResNet50|0.1697|0.1095|0.0961|0.1496|0.2922|0.1651|0.2059|9.1hrs|1.6hrs|   
|ResNet101+FPN|0.2050|0.1307|0.1407|0.1557|0.3339|0.2236|0.2453|5.54hrs|1.1hrs|    

注：ResNet50的实验是基础配置，ResNet101+FPN使用的其他变量除了模型结果，其他与基础实验设置一样。  

总结：**提高模型结构复杂度利于性能提升**。而且ResNet101+FPN耗时更短，原因有待深入了解(原因补充至主体内容第3点)。  
2. [Large Minibatch SGD论文](https://arxiv.org/pdf/1706.02677.pdf)论文阅读小结：该论文指导了关于在多个GPU下使用Large Minibatch SGD时对学习率该如何调整。一般情况为了省时，进行对损失函数的梯度下降会采用Minibatch SGD，例如Faster-RCNN每个minibatch大小是2张图像，但因为可以使用多个GPU，那么GPU并行计算可实现Large Minibatch，假设GPU数量k=2，Large Minibatch size = k x minibatch size = 4。根据Linear Scaling Rule (即**若minibatch size乘以k，学习率LR也要乘以k**)，2 GPU下的学习率便是原始LR x 2的结果。此外，作者提到，如果直接在Large Minibatch上使用LR x K的学习率或者使用Constant Warmup(即前几个epochs先固定用小的LR，之后再固定用LR x k学习率)，都会对Loss的收敛造成一定程度的不利，因此使用Gradual Warmup(即LR是逐渐以常数递增至LR x k)解决这个不利因素。  
3. [FPN](https://zpascal.net/cvpr2017/Lin_Feature_Pyramid_Networks_CVPR_2017_paper.pdf)论文阅读小结：该论文讲述了使用Feature Pyramid Networks对目标检测的帮助，目标检测模型有两个部分：一是backbone用于提取特征图，二是head用于在特征图上的实现预测。因为FPN是操作在不同ConvNet之间的不同尺寸特征上，使得它相比于在图像层面上的图像金字塔(image pyramid)更节省时间和空间，而且FPN有利于检测小目标。而**ResNet101+FPN比ResNet50省时是因为前者把conv5放在backbone中，head只是两个FC全连接层，而后者把conv5放在head里，backbone的resnet只到conv4为止，这样ResNet50在head上的权重更新负担比ResNet101+FPN重，训练时间自然久些。注意backbone并不是不进行权重更新，只是权重更新的负担比head的小。**  

### 2019年12月11日
#### 主体内容
1. Faster-RCNN实验3主要是研究**学习率衰减起始点SOLVER.STEPS**，在LeaderBoard的结果如下：

|SOLVER.STEPS|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|(12000,16000)|0.2050|0.1307|0.1407|0.1557|0.3339|0.2236|0.2453|5.54hrs|1.1hrs|    
|30000|0.2125|0.1317|0.1391|0.1772|0.3369|0.2430|0.2474|5.55hrs|1.0hrs|    

注：对比实验使用统一的模型结果ResNet101+FPN，但SOLVER.STEPS不同。  

总结：学习率衰减起始点延迟到30000，这过程中梯度下降幅度比原来大，帮助模型收敛更快些(即拟合速度加快)，因此模型性能有所提升。提升不大的原因可能是因为最大循环数MAX_ITER只设置到40000，按照每iteration输入4张图片（2 image per batch x 2 GPU），一个epoch = 45707张图/4 = 11426.75 iterations，则epoch size = MAX_ITER/11426.75 = 3.5 epochs，很大可能训练epochs数不够，导致模型还没有彻底收敛到最优，这也是为什么学习率衰减起始点延迟导致性能提升。建议先不使用数据增强，因为数据增强的目的一是数据本身太少模型不能很好拟合，二是避免模型训练出现过拟合需要增加数据提高通用性。我们现在的情况很可能是MAX_ITER设置不当导致模型欠拟合，因为本身数据量很大而epoch明显太小，所以当前应提高训练次数实现很好的拟合状态再考虑数据增强和扩充数据。  

### 2019年12月12日
#### 主体内容
1. Faster-RCNN实验4主要是研究**预测时滑动窗口步伐比例stride_proportion**，在LeaderBoard的结果如下：

|stride_proportion|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|0.5|0.2125|0.1317|0.1391|0.1772|0.3369|0.2430|0.2474|5.55hrs|1.0hrs|    
|0.25|0.2207|0.1241|0.1352|0.1769|0.3294|0.2928|0.2656|5.55hrs|3.8hrs|  

注：对比实验使用统一的MAX_ITER=40000, SOLVER.STEPS=30000，但stride_proportion不同。  
总结：stride_proportion=0.25带来了0.01的提升，时间多了近3个小时，还是可以speed/accuracy tradeoff是可以容忍的，因此**后续实验保持stride_proportion=0.25的设置**。

### 2019年12月13日
#### 主体内容
1. Faster-RCNN实验5主要是研究**MAX_ITER**，在LeaderBoard的结果如下：

|MAX_ITER|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|40000|0.2207|0.1241|0.1352|0.1769|0.3294|0.2928|0.2656|5.55hrs|3.8hrs|  
|80000|0.1770|0.1009|0.1698|0.1238|0.2842|0.1656|0.2178|11.83hrs|3.44hrs|  
|200000|0.1758|0.1347|0.2099|0.0874|0.2935|0.1746|0.1546|28.36hrs|3.8hrs|    

注：前者使用MAX_ITER=40000(3.5 epochs), SOLVER.STEPS=30000，后者使用MAX_ITER=200000(17.5 epochs), SOLVER.STEPS=150000。  
总结：**17.5 epochs使得模型过拟合了**，训练时也发现最后收敛的总Loss在0.01左右，而且cls_loss是0。而**之前MAX_ITER下的模型收敛的总Loss在0.18-0.25的范围内**，由于阿里云暂时不支持tensorboard看训练的损失曲线，我们只能主观先作出一个猜测，即在epoch size=[3.5, 17.5]之间存在一个最优收敛情况，为了验证这个猜测，我们需要进一步对MAX_ITER做出调整，这次选取了MAX_ITER=80000(7 epochs)，直接取实验5的断点保存模型运行预测即可，根据实验5的训练日志来看，epoch size = 7时总损失大概在0.13-0.18范围内。**后续实验如果做数据增强或者数据扩增需要调整到更高的MAX_ITER，调整方法建议：先跑个大的MAX_ITER，根据日志的Loss选择拟合较好的断点保存模型结果**。  

### 2019年12月14日
#### 主体内容
1. MAX_ITER=80000的实验结果已补充至2019年12月13日主体内容第1点的表格内，这次**第6次实验（即loss在0.13-0.18）还是反映模型过拟合**了，所以现在还是回到MAX_ITER=40000(3.5 epochs), SOLVER.STEPS=30000的设置下进行下一步的实验，先加入水平/垂直翻转数据增强，观察其对模型性能的影响。如果模型性能提升，说明MAX_ITER=40000(3.5 epochs), SOLVER.STEPS=30000的设置下存在拟合强的表现。反之，说明无数据增强的模型已经够好了，不需要额外在数据量上做功夫，不然会导致MAX_ITER又要做相应增加的操作调整。
2. 结合这几天关于MAX_ITER和SOLVER.STEPS的研究，有以下三点建议：  
(1) **epoch数和学习率衰减起始点不宜同时更改。**  
(2) **除非收敛很慢或很快，否则学习率最好不要乱动。**  
(3) **某学习率下模型表现不错的话，想要验证是否过拟合或欠拟合，改epoch数比改学习率更好。**  
因此，**建议以后保持SOLVER.STEPS=30000不变，若怀疑3.5epochs-7.5epochs之间存在最优模型表现，只调节MAX_ITER即可。**  

### 2019年12月15日
#### 主体内容
1. Faster-RCNN实验7主要是研究**数据增强-水平/垂直翻转**，在LeaderBoard的结果如下：

|数据增强|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|无|0.2207|0.1241|0.1352|0.1769|0.3294|0.2928|0.2656|5.55hrs|3.8hrs|  
|水平/垂直翻转|0.2292|0.1435|0.1488|0.1572|0.3510|0.3108|0.2640|5.57hrs|2-3hrs|  

注：由于优化了预测代码使得其能在多GPU下多进程运算，预测时间有所缩短。  
总结：**水平/垂直翻转数据增强能提高现有模型表现**，因此无数据增强的原模型存在拟合强，后续可以考虑其他数据增强和数据扩充手段。  
2. 近期观看论文及回顾历史实验，近期思考的新优化可能，它们对性能的帮助可能不错：  

|序号|手段|负责人|完成情况|  
|:---:|:---:|:---:|:---:|  
|1|全正样本数据集|思凡|已完成于2019/12/18|  
|2|解决数据类别不平衡-Weight Balanced Cross Entropy|宏峰|已完成待实验|  
|3|将FPN中head里2-fc改为传统的conv|宏峰|已完成待实验|  
|4|解决数据类别不平衡-Focal Loss|宏峰|已完成待实验|  
|5|重置滑窗规则+数据增强|-|-|  
|6|增加额外随机数据集|思凡|已完成于2019/12/19|  
|7|用初赛数据预训练模型|宏峰|已完成于2019/12/20|  
|8|模型集成|-|-|  

注：  
(1) 序号4的关于类别损失函数Focal Loss是来自RetinaNet论文，据说比序号2手段中我采用的Median balanced loss这种基于类别频次分配权重更好。  
(2) 序号5手段思路来自阿里最新目标检测模型[ATLDETv2](https://mp.weixin.qq.com/s/jmMw3KVOgNp-KFjOTO_5qw)，虽然没有公布论文和源码，但其中的实例平衡增强（Instance-Balanced Augmentation）技术我们可以借鉴。其中说道：“研究者会对图像按照特定的尺度（如 1.5 倍和 2 倍大小）进行缩放操作，即定义了一批大小不同的「滑窗」。同时，他们也会定义滑窗的步长。定义后，使用滑动窗口在样本图像中滑动，产生滑动区域。在这些滑窗中，**选择包含少量目标的最优数据加入到训练集**中。在选择滑窗的过程有一定的规则。例如，**滑窗在某个步长上和已有目标有界限重叠的滑窗目标不会被取用**，同时滑窗目标的选择也会**参考数据集已有的样本类别分布情况**。当选择了一定的滑窗目标后，研究者会**根据分辨率和尺度等进行一定的变化，加入一些随机扰动**，使得选出的样本能够增强原有的数据集样本。”，它的这种方式好处在于虽然刚开始滑动窗口得到最优数据不多，但是加入了尺度变化后重新滑动窗口，数据会得到增强。但因为技术细节未公布，如何根据样本类别分布情况选择滑窗目标是我们该研究的。另外，这感觉并无真正解决类别不平衡问题，理想状态应该是各类别目标数量一致或者相差很小。我们也能借助它部分思想，比如我们只取不含任何假阳性的最优数据，然后观察数据集分布情况，根据数据扩充的需求，改变stride，如果之前相交目标，这次在滑窗内，我们把它取出来加入数据集，直到实现足够的数据量。如果还是不够，加入额外的数据增强，例如旋转。  
(3) 序号6的实验不着急做，看模型拟合状况再说。  
(4) 序号7的实验也是参考[ATLDETv2](https://mp.weixin.qq.com/s/jmMw3KVOgNp-KFjOTO_5qw)的思路，如果我们用初赛数据预训练模型，再拿次预训练模型去在复赛数据上训练，可能会增加模型的通用性。  
(5) 序号8的数据集成方式请参考：[Ensemble](https://www.zhihu.com/question/41631631/answer/94816420)，但提到的下面方法得询问阿里人员是否可用：  
- 同样的参数,不同的初始化方式。  
- **不同的参数,通过cross-validation,选取最好的几组**。  
- **同样的参数,模型训练的不同阶段，即不同迭代次数的模型**。  
- 不同的模型,进行线性融合. 例如RNN和传统模型。  

### 2019年12月16日
#### 主体内容
1. Faster-RCNN实验8主要是研究的是**模型backbone**，在LeaderBoard的结果如下：

|backbone|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|R101|0.2292|0.1435|0.1488|0.1572|0.3510|0.3108|0.2640|5.57hrs|2-3hrs|  
|X101|0.2226|0.1295|0.1530|0.1571|0.3268|0.3014|0.2679|12.37hrs|4.8hrs|  

注：X101是ResNeXt-101-32x8d模型。  
总结：根据Validation表现，在相同epoch size下，X101比R101拟合多了，但这并不反映X101不好，减少epoch训练数或者数据扩充都可能提高X101的表现。该实验说明了**复杂模型比简单模型更易拟合**。[ResNeXt](https://arxiv.org/abs/1611.05431)在Resdual模块基础上引入了Inception思想并加以改进，在resdual block并列32 group（也称cardinality基数）个bottleneck（即1x1,3x3,1x1 conv），其中bottleneck的宽度width推荐使用4，因此该设置模型叫ResNeXt101-32x4d。我们这bottleneck宽度是8，速度会慢点，但可承受。据论文和detectron2在Github所示，X101会比R101在AP上高0.01左右，所以最后如果模型提升不明显也不用感到意外。  


### 2019年12月18日
#### 主体内容
1. Faster-RCNN实验9主要是研究的是**加入含大目标框的数据样本**，在LeaderBoard的结果如下：

|数据|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|去大目标框数据|0.2226|0.1295|0.1530|0.1571|0.3268|0.3014|0.2679|12.37hrs|4.8hrs|  
|加大目标框数据|0.2361|0.1423|0.1463|0.1604|0.3338|0.3472|0.2867|12.36hrs|4.7hrs|  

注：之前实验因为担心大目标框数据(即滑动窗与目标框IOU>900*900)会给实验带来正负样本不平均（负样本<正样本），因此筛选掉了它们。 实验9的加大目标框数据指把之前筛选掉的补充回来。  
总结：**加入含大目标框的数据样本对模型有帮助。**  
1. 下面是两个数据集中各类别目标框数统计:  

|细胞类别|去大目标框数据|加大目标框数据|变化量|  
|:---:|:---:|:---:|:---:|  
|AH|20195|20198|+3|  
|AS|19383|19383|0|  
|HL|9187|9189|+2|  
|LL|10952|10954|+2|  
|CA|5026|5765|+739|  
|TS|36716|36716|0|  
|total|101459|102205|+746|  

2. 加入含大目标框数据后，Candida明显增加多张图，而且尽管它总数量少，但在历史实验中该类AP@50也比其他类别高，反映**大目标的Candida易检测**。该实验证实2019年12月4日在后续留意内容第一点是多余的，因为模型每个patch同时输入2张图，即使某张图是全正样本，另外一张图也能提取出足够的负样本出来，出现2张全正样本图同时输入到模型的概率十分小。另外，虽然该实验只有Candida数量有明显增加，但是其他类别的表现也有所提升，结合实验8的表现，可以猜测数据量的增加在一定程度下缓解了X101的强拟合，增加了模型的通用性能。  

### 2019年12月19日
#### 主体内容  
1. Faster-RCNN实验10主要是研究的是**增加小目标的数据样本**，在LeaderBoard的结果如下：  

|数据|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|未增加小目标框数据|0.2361|0.1423|0.1463|0.1604|0.3338|0.3472|0.2867|12.36hrs|4.7hrs|  
|增加小目标框数据|0.2211|0.1242|0.1612|0.1879|0.3103|0.3170|0.2261|12.39hrs|4.7hrs|  

注：此次增加的小样本数据是是目标的宽和高均小于100就进行一次数据增强。  

总结：该实验单纯判定目标的宽和高均小于100作为小目标样本，而实际上Faster-rcnn对32×32以下的目标难检测（因最小的anchor box尺寸是32），所以真正的32x32以下的小目标，模型很难对小目标提出较多的anchor bbox，因此模型在训练过程中得不到足够的小目标样本训练，那么在预测时则会忽略掉小目标。回过头看，增强目标的宽和高均小于100的样本则引入了中型目标的增强，这样小目标样本占总样本比例还是不够明显，而且带来多的冗余中型目标样本，使模型欠拟合。另外TS类很多小目标样本，其他类就只是最多上到几百，最低到个位数。因此这次小目标样本的问题在该实验中并没有很好的解决，而且TS类小目标多了，把类别比重拉的更大，使得当模型预测遇到可疑小目标时，更多会选择TS类而不是其他异常细胞类别。  
解决方案： 1、减少小样本数据增强的数量，并且不要增强TS类的小样本数据（因为TS类的数据大部分都是小目标样本来着）。2、使用Augmentation，增加小样本在图片所占面积的比重。  

### 2019年12月20日
#### 主体内容
1. 针对小中大型目标数量对与三种不同数据集的统计情况如下：  

**(1) 原始数据集:**

|细胞类别|小目标|中目标|大目标|细胞总数|  
|:---:|:---:|:---:|:---:|:---:|  
|ASC-H|401|3549|2244|6224|  
|ASC-US|55|3496|2394|5945|  
|HSIL|183|1111|1524|2818|  
|LSIL|26|1419|1937|3382|  
|Candida|9|286|1357|1652|  
|Trichomonas|10436|1095|23|11554|  
|目标总数|11409|10956|9479|21109|  

**(2) 在ROI内常规窗口滑动所得数据集:（normal patchs:46420, large patchs:746）**

|细胞类别|小目标|中目标|大目标|细胞总数|  
|:---:|:---:|:---:|:---:|:---:|  
|ASC-H|1261|11649|7288|20198|  
|ASC-US|196|11605|7582|19383|  
|HSIL|585|3586|5018|9189|  
|LSIL|102|4688|6164|10954|  
|Candida|33|913|4813|5729|  
|Trichomonas|33320|3463|63|36846|  
|目标总数|35497|35904|30928|102329|  

**(3) 根据bbox中心点窗口滑动所得数据集（normal patchs:31404, large patchs:255）:**

|细胞类别|小目标|中目标|大目标|细胞总数|  
|:---:|:---:|:---:|:---:|:---:|  
|ASC-H|2609|12978|4726|20313|  
|ASC-US|181|8250|4108|12539|  
|HSIL|1178|4960|3587|9725|  
|LSIL|136|4735|3898|8769|  
|Candida|17|489|1972|2478|  
|Trichomonas|114096|7563|112|121771|  
|目标总数|118217|38975|18401|175593|  

### 2019年12月21日
#### 主体内容  
1. Faster-RCNN实验11主要是研究的是**使用初赛数据训练的预训练模型**，在LeaderBoard的结果如下：  

|预训练数据|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|COCO|0.2361|0.1423|0.1463|0.1604|0.3338|0.3472|0.2867|12.36hrs|4.7hrs|  
|COCO+初赛数据|0.2406|0.1339|0.1654|0.1752|0.3493|0.3366|0.2833|12.21hrs|4.7hrs|  

注：此次实验用初赛数据训练了5000iterations，预训练的1个小时并不计入上述训练时间内。初赛数据预训练模型X101-FPN-pretrained-wsi.pkl已放置configs/IMageNetPretrained/MSRA下了。  

总结：**初赛数据训练的预训练模型能帮助模型提升一点性能**。由于担心预训练模型过拟合与初赛数据集，所以并没有设置很大的iteration数。预训练模型思路如下：  
（1）修改配置文件（yaml及pascal_voc数据格式转换的py文件）并更改训练数据文件夹为初赛训练集;  
（2）将第一步的操作回归为原本实验配置;  
（3）用FSF/detectron2/下的convert_pth_to_pkl.py文件将第一步得到的模型权重pth文件转为模型结构+模型权重的pkl文件;  
（4）修改checkpoint/detection_checkpoint.py中的内容使得保存的pkl能以正确的格式被载入。    
2. 在可视化预测结果和测试集数据研究发现，存在下面三种情况并已改进了预测代码：  
（1）**预测细胞图上可疑异常细胞会有多类别的重叠出现**，原因是NMS放在不同类下进行，现已将NMS放置类外统一进行;  
（2）**测试集中给定json文件中的ROI存在重叠可能**，因此NMS再做调整，放置单图所有ROI结束预测后再进行NMS;  
（3）测试集所有图片的所有ROI总数为857个，而有**10个medium ROI是roi_h<1000或者roi_w>1000，另外有2个small ROI的宽和高都小于1000**，之前实验没考虑这些，使得这12个ROI没有任何检测结果出现，现在已对代码进行改进能实现对medium ROI和small ROI的检测了。  

### 2019年12月22日
#### 主体内容  
1. Faster-RCNN实验12主要是研究的是**NMS的摆放位置**，在LeaderBoard的结果如下： 

|NMS|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|ROI下的类内|0.2406|0.1339|0.1654|0.1752|0.3493|0.3366|0.2833|12.21hrs|4.7hrs|  
|ROI外及所有类外|0.2264|0.1308|0.1640|0.1465|0.2819|0.3442|0.2909|12.21hrs|4.6hrs|  

注：关于NMS摆放位置的调整详情请见2019年12月21日主体内容第2点。’ROI下的类内’指NMS是在每个ROI下的类内进行，因此可视化会出现重叠框（分别来自重叠ROI和不同类间的重叠）。‘ROI外及所有类外’指NMS放在所有ROI集及类外进行的，因此可视化不会出现重叠框，除了IOU<0.1的相交框。  
总结：虽然NMS放到最后在全局检测结果上做会让可视化呈现上变得很好，但是基于MAP的计算方式，还是偏向于NMS在ROI下的类内检测结果。  

### 2019年12月23日
#### 主体内容  
1. Faster-RCNN实验13主要是**修改之前预测代码的遗漏**，在LeaderBoard的结果如下： 

|预测ROI|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|单large ROI|0.2406|0.1339|0.1654|0.1752|0.3493|0.3366|0.2833|12.21hrs|4.7hrs|  
|所有ROI|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  

注：该实验将NMS放在ROI下的类内，同时修复了如2019年12月21日主体内容第2点的(1)的遗漏。  

### 2019年12月24日
#### 主体内容
1. Faster-RCNN实验13主要是研究的是**NMS阀值**，在LeaderBoard的结果如下： 

|NMS阀值|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|0.1|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|0.4|0.2514|0.1535|0.1854|0.1856|0.3743|0.3131|0.2965|12.21hrs|4.8hrs|  

注：NMS阀值越高，留下的检测结果越多。  
总结：NMS阀值的提高对于AH，AS和TS的AP有所提高，其他类均有所下降。因此后期可以**考虑精细化NMS阀值到不同类别**。  

### 2019年12月25日
#### 主体内容
1. Faster-RCNN实验14主要是研究的是**训练集获取方法**，在LeaderBoard的结果如下： 

|方法|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|ROI滑动|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|pos滑动|0.2278|0.1491|0.1767|0.1498|0.3450|0.3170|0.2289|12.21hrs|4.7hrs|  

注：‘ROI滑动’是指定大小的patch是从ROI左上角依次根据stride滑动获取得到的。‘pos滑动’指patch根据pos中心点依次滑动获取。  
总结：虽然pos滑动的数据集效果不好，但是AH的分数比较高，之后实验试着保持ROI滑动数据不变，然后加入pos滑动所得的AH和AS类别数据进入看看能不能提高关于这两类的检测表现。

### 2019年12月27日
#### 主体内容
1. Faster-RCNN实验15主要是研究的是**增加额外的AH样本量**，在LeaderBoard的结果如下： 

|数据|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|原数据|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|加额外AH样本|0.2535|0.1474|0.1706|0.1819|0.3700|0.3611|0.2901|12.21hrs|4.6hrs|  

注：该实验是沿袭2019年12月25日的总结思路，在原ROI滑动获取数据基础上，额外加入了由pos滑动所得的AH类别数据（4000张）。  
总结：AH的分数并没有增加，因此接下来**不建议后续实验再去对数据进行扩充操作了，现在数据量够了**。 因为本质上ROI滑动获取数据跟POS滑动获得数据其实差不多，只是POS更多考虑每个标注目标都能在至少一张patch图内有完整性，而ROI滑动会有更多的样本相交可能，个体完整性随stride数少而更好，但样本量也可能会增加，目前这种状况，还是维持现有数据量现状。  
  
### 2019年12月28日
#### 主体内容
1. Faster-RCNN实验16主要是研究的是**数据增强-旋转**，在LeaderBoard的结果如下： 

|数据|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|原数据|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|旋转数据|0.2458|0.1452|0.1674|0.1881|0.3569|0.3756|0.2415|12.21hrs|4.6hrs|  

注：该实验的旋转数据是指在原数据基础上以0.5随机概率进行旋转操作，不改变原有数据量大小。此外该实验还精细化了NMS阀值，具体细节在2019年12月29日实验内容进行详细描述。    
总结：虽然该实验分数不高，但是可能是因为模型欠拟合导致的，因此**最后如果有时间，可以使用resume=True继续训练至MAX_ITER=50000看看**。

### 2019年12月29日
#### 主体内容
1. Faster-RCNN实验17主要是研究的是**NMS和CONF_THRESHOLD的精细化**，在LeaderBoard的结果如下： 

|NMS与CONF|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|    
|普通|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|精细化|0.2590|0.1551|0.1848|0.2021|0.3888|0.3322|0.2911|12.21hrs|4.6hrs|

注：（以下某些内容无特别说明都是按照普通配置来配的）  
（1）‘普通’：NMS都放在ROI下的类内，置信阀值是0.05，NMS阀值为0.1。  
（2）‘精细化’：CA和TS都是全局NMS（即无类间重叠和ROI间重叠），CA置信阀值是0.5，AS、AH和TS的三个NMS阀值提到0.5。  

总结：  
1. 通过对比其他选手分数发现，我们在CA、AS和AH三类上的检测率比较低，而之前的实验关于NMS和其它研究中，我们发现有些类别的变动不太一致。其中对于CA、AS和AH三类有以下发现：  
（1）**CA易检测**。菌丝特征便能很好区分其他异常细胞，而且在训练集上的结果可视化发现，CA检测置信度一般都高达0.8以上，而且它类一般是0.2-0.4这样的。另外CA实例数量相比其他类别是最少的，但是别的分数确排名第二。通过2019年12月13日发现CA在高MAX_ITER下表现很差，容易过拟合，说明特征好学习。 对此提出三种解决方式：**提高CA的置信度阀值（提高到0.5反而不好，估计是在训练和测试集存在一定的不匹配性，医学细胞切片的共性，因此该方法不合适，建议保持所有置信阀值都为0.05）**; **取MAX_ITER<40000以下模型关于CA的预测结果（将在‘不同收敛阶段的模型集成’实验中做）**;**将CA的NMS放在最外面（即全局NMS）**。   
（2）由2019年12月22日和24日关于‘NMS的摆放位置’和‘NMS阀值’实验看出，**TS用全局NMS会好点，且TS的NMS阀值提高也会有所帮助**。   
（3）同样地，**AH和AS的NMS阀值提高会有帮助，但不建议使用全局NMS**。   
2. **NMS阀值的调整取决于该类异常细胞是否不确定性高。AS和AH易混淆**，它们不确定性高，所以提出多点的检测框会好点。另外**关于是否做全局NMS，是根据该类细胞是否易检测**，CA如果与其他类别细胞检测框重叠，那么会选择置信高的框，而易检测的CA的置信度一般都比较高，所以这样能够有效的减少其他类的误判检测结果。  
3. 精细化NMS与CONF_THRESHOLD要谨慎，因为调整的参数设计比较多变，同时可能不同收敛阶段精细化调整又不一样。  
4. 在[宫颈细胞图像分割和识别方法研究](http://www.wanfangdata.com.cn/details/detail.do?_type=degree&id=Y1777462)第27页提到**ASC是一个意义尚不明确的类别-非典型鳞状细胞。其中ASC-US指那些LSIL的改变，大部分ASC-US的判定结果是LSIL。ASC-H指HSIL的病例，很多时候在判断识别时，取消ASC这一类，将ASC-US划入LSIL中，将ASC-H划入HSIL中，这样可以降低将病变细胞识别为正常细胞的识别率**。因此ASC-US和LSIL这一组与ASC-H和HSIL这一组在该比赛中如何把他们的混淆很好区分开是很重要的一环，暂时还没有思路。  

### 2019年12月30日
#### 主体内容
1. Faster-RCNN实验19主要是研究的是**收敛阶段**，在LeaderBoard的结果如下： 

|收敛阶段|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|40000|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs| 
|30000|0.2467|0.1637|0.1498|0.1817|0.3765|0.3361|0.2723|<12.21hrs|4.7hrs| 

总结：可以试着提高收敛阶段看看效果。  

### 2019年12月31日
#### 主体内容
1. Faster-RCNN实验20主要是研究的是**混搭配置**，在LeaderBoard的结果如下： 

|配置|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|正常|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs| 
|混搭|0.2232|0.1359|0.1556|0.1926|0.3643|0.2711|0.2198|12.21hrs|4.8hrs| 

注：‘正常配置’是：未旋转数据集+MSCOCO预训练和初赛数据预训练5000iters+头部2fc。 ‘混搭配置’是：旋转数据集+MSCOCO预训练+头部2fc和1conv。该实验不是针对某变量进行对比实验，无太大的参考价值。  
总结：设置该实验是因为有点想碰运气，病急乱头医。实验到瓶颈也不该这样做。  

2. 未来几天的实验计划如下：  
（2020.1.1）跑旋转复赛数据集+MSCOCO+初赛数据预训练模型+45000iters下的实验，该实验是接着2019年12月28日来的，因为本质上数据带有旋转会更好，表现不好可能是拟合不当原因。  
（2020.1.2）如果上面实验表现好，则使用旋转数据，如果不好则使用回原来的无旋转数据。之后在roi_heads上加入额外2个conv。  
（2020.1.3）更具上面实验的好坏决定conv的去留。之后将stride_proportion换成0.05。  
（2020.1.4）精细化nms。    
注：上述实验可根据实验结果进行微调。  

### 2020年1月1日
#### 主体内容
1. Faster-RCNN实验21主要是研究的是**旋转数据**，在LeaderBoard的结果如下： 

|配置|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|未旋转原数据|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|旋转数据-40000|0.2458|0.1452|0.1674|0.1881|0.3569|0.3756|0.2415|12.21hrs|4.6hrs|  
|旋转数据-45000|0.2407|0.1599|0.1626|0.2001|0.3605|0.3256|0.2352|12.21hrs|4.7hrs|  

注：‘旋转数据-40000’是来自2019年12月28日的结果。  
总结：效果不是想象中的好，暂时搁置，未来可能可以将旋转数据-40000下CA结果和旋转数据-45000下的AH结果集成到未旋转原数据里。 这个不好的原因是很难把握不同旋转图像类别的收敛拟合情况，如果旋转到过拟合类别，可能会提高表现，如果旋转到不是过拟合，可能会减少表现。而类别的过拟合需要一个固定模型观察不同类别的拟合情况去判断的，例如2019年12月30日的实验。  

2. 更改未来几天实验计划如下：  
（2020.1.2）使用回原来的无旋转数据+切片染色色调均一化。  
（2020.1.3）之后在roi_heads上加入额外2个conv。  
（2020.1.4）更具上面实验的好坏决定conv的去留。之后将stride_proportion换成0.05。  
（2020.1.5）精细化nms。  

### 2020年1月2日
#### 主体内容
1. Faster-RCNN实验22主要是研究的是**切片染色色调均一化**，在LeaderBoard的结果如下： 

|染色|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|原数据|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|染色均一化数据|0.2250|0.1269|0.1546|0.1698|0.3692|0.2794|0.2499|12.21hrs|4.6hrs|  

注：染色均一化指将BGR图像转化为HSV图像，然后统一将色调Hue调成100，之后再转化为BGR图像保存作为图像数据集。  
总结：切片让色色调均一化在这里貌似表现不太好，原因暂无猜测。  

2. 更改未来几天实验计划如下：  
（2020.1.3）将stride_proportion换成0.05。  
（2020.1.4）在roi_heads上加入额外2个conv。  
（2020.1.5）在具上面实验的好坏决定conv的去留，精细化nms。  

### 2020年1月4日
#### 主体内容
1. Faster-RCNN实验23主要是补充调整**预测时滑动窗口步伐比例stride_proportion**，在LeaderBoard的结果如下： 

|stride_proportion|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|0.25|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|0.15|0.2495|0.1389|0.1734|0.1938|0.3711|0.3225|0.2975|12.21hrs|11.6hrs|  

总结：在模型准确度不是那么高的情况下，stride_proportion调的很低不一定是件好事，因为它有可能会增多误判检测量，导致测试分数下降。  
2. 关于CA的预测结果，我们由于输入尺寸的限制，所以CA预测框最大都不会大于1000×1000，而我们根据起初的数据统计发现：CA存在一部分大于1000×1000的目标框，因为CA本身属于大目标。为了解决这个大目标检测问题，个人提出以下三种方法：  
（1）方法一：在预测部分，使用3000x3000的大窗口滑动测试图，取patch后resize成1000x1000的尺寸图输入到预测器当中，最后将得到的检测框乘以3回复到原用尺寸。（该方法已尝试，最后得到1000x1000~2000x2000的检测框有15个，大于2000x2000的检测框有28个，可是可视化后发现大部分都是不合理的，猜测原因是图片resize后，一些细胞相交边缘重叠处变细导致模型误认为是菌丝，所以放大后的结果再细看发现是误判）  
（2）方法二：在预测部分，类内NMS时采用相交检测框大于某阀值就进行目标框相结合的操作。（这个方法能不能取得好的表现在于模型对CA的检测准不准确，以及标注手法情况是怎样的，很难有个直接判断）  
（3）方法三：在训练部分，生成宽高均大于1000尺寸的patch输入模型。（这个得看显卡内存，而且生成数据量和收敛情况得重新调整）  
3. 除了上述第2点可能会造成CA分数低，从2019年12月18日能猜测还有个原因可能是CA数量太少。因此这里再多生成一组全正样本（大部分都是CA）并且都进行旋转加入到现有数据集看看。  

### 2020年1月6日
#### 主体内容
1. Faster-RCNN实验24主要是研究的是**再增加额外的正样本数**，在LeaderBoard的结果如下： 

|正样本|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|原数据|0.2589|0.1487|0.1787|0.2021|0.3888|0.3441|0.2909|12.21hrs|4.7hrs|  
|增加一组额外|0.2643|0.1745|0.1948|0.1962|0.3556|0.3736|0.2908|12.54hrs|4.7hrs|  
 
 注：‘原数据’是包括正常滑动分割得到的非全正样本及全正样本，而‘增加一组额外’是在原数据基础上增加多一组原数据集的全正样本（并且全都进行旋转操作）。本质上这一组额外全正样本是主要增加了CA的数据量。  
 
|roi数|AH|AS|HL|LL|CA|TS|总|  
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|增加前|20198|19383|9189|10954|5759|36846|102329|   
|增加后|20201|19383|9191|10956|6498|36846|103095|   

注：roi数是指该异常细胞在数据集里有多少bbox框。  
总结：该实验证实了2020年1月4日第3点的猜测。  

### 2020年1月7日
#### 主体内容
1. Faster-RCNN实验25主要是研究的是**Soft NMS**，在LeaderBoard的结果如下： 

|NMS|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|普通NMS|0.2643|0.1745|0.1948|0.1962|0.3556|0.3736|0.2908|12.54hrs|4.7hrs|  
|Soft NMS|0.0878|0.0677|0.1618|0.0283|0.2038|0.0331|0.0318|12.21hrs|>5hrs|  

注：Soft NMS使用的是高斯赋值权重法。  
总结：不建议使用Soft NMS。  

2. Faster-RCNN实验26主要是修正了**6198.json和9379.json**，在LeaderBoard的结果如下： 

|修正|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|无|0.2643|0.1745|0.1948|0.1962|0.3556|0.3736|0.2908|12.54hrs|4.7hrs|  
|有|0.3102|0.1939|0.3192|0.2008|0.4770|0.3748|0.2956|12.54hrs|4.7hrs|  

### 2020年1月8日
#### 主体内容
1. Faster-RCNN实验27主要是研究的是**不同收敛阶段**，在LeaderBoard的结果如下： 

|收敛数|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|35000|0.3088|**0.2106**|0.3190|0.1924|0.4759|0.3685|0.2865|～11hrs|4.7hrs|  
|40000|0.3102|0.1939|0.3192|**0.2008**|**0.4770**|**0.3748**|**0.2956**|12.54hrs|4.7hrs|  
|45000|0.3088|0.2105|**0.3240**|0.1874|0.4715|0.3728|0.2869|～13hrs|4.7hrs|  

集成后结果如下：  

|方法|score|AH|AS|HL|LL|CA|TS|训练时间|预测时间|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|集成|0.3138|0.2106|0.3240|0.2008|0.4770|0.3748|0.2956|～11hrs|4.7hrs|  
 
