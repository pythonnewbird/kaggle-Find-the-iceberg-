# kaggle：Find the iceberg
kaggle：Statoil/C-CORE Iceberg Classifier Challenge 赛后整理（silver 前4%） 

Iceberg
1.	概述：
在海面上漂流的冰川会对航行的船只造成威胁，许多机构和公司使用空中侦察和岸基支持来监测环境条件并评估冰山的风险。 但是，在天气特别恶劣的偏远地区，这些方法是不可行的，唯一可行的监测选择是通过卫星。比赛的训练数据来自C-CORE的卫星遥感照片，希望能设计出用来检测和减少冰川风险的准确识别冰川的算法。
   评测函数：log loss
2.	数据清洗与预处理：
卫星在地球上方约680公里处。 以特定入射角发送信号脉冲，然后将其重新编码。 基本上这些反射信号被称为反向散射。 比赛给出的数据是反向散射系数，这是反向散射系数的常规形式：
σo(dB)=βo(dB)+10log10[sin(ip)/sin(ic)]
其中：ip表示特定像素的入射角，ic是图像中心的入射角，
基本上，σo 随着从不同物体表面散射而来的不同信号而不同，对于特定的入射角度，对于不同的物体，它们之间的HH相差比较大，HV相差较小。卫星只发送在H极化的pings，而不发送在V极化的，这些H-pings在海平面发生散射，物体会改变它们的极化，并且返回H,V混合的信号，由于Sentinel只有H-发射器，返回信号仅为HH和HV的形式。我提取了两个频段，并且将他们的平均值作为第三频段来创建3频段的RGB等效。

训练数据为1604条，其中的inc_angle不是数值型变量，将其转化。
发现缺失数据：可以发现在所有数据中，inc_angle这一项数据存在缺失，缺失值为133
观察inc_angle的数据分布：iceberg的数据大致呈正态分布，not_iceberg的数据略微左偏。
 
检查是否是平衡样本：得到训练集中is_iceberg=0和is_iceberg=1的比例，大致呈1:1，样本平衡。
 

由于数据不是实际的图像，而是从雷达散射，所以形状会出现像这样的峰值和扭曲。 船的形状将会像一个点，可能就像一个细长的点。 从这里出现结构性差异，我们可以使用CNN来利用这些差异。
3.	预测模型：
 
我的方案的简单架构如上图所示，主要是采用了CNN与lightgbm这两个模型，其中CNN模型先是采用可视化的手段，通过earlystoping找出模型训练过程中的最佳epoch，然后重新用10折交叉验证训练模型，并且将epoch设置为最佳epoch 30。Lightgbm模型也是采用了10折交叉验证训练模型，并且分析了feature_importance ，发现和预想的一样，inc_angle 这个特征确实是非常重要。
 


CNN模型:
1.	权重初始化
2.	增加band_3,创建3-channel的RGB等效
3.	将图像大小都规范成75*75
4.	Zero-center：数据有过大的均值可能导致参数的梯度过大，并且在零均值化的过程中，像素的相对差异并没有被消除。
归一化：使不同维度的数据值的范围相差不要过大，不同维度相差太大的话难以优化。
5.	数据的augmentation：尝试过random crop，random mirror，random resize，flip，最后选用了flip，采用3个channel中各自进行水平与垂直翻转
6.	CNN模型的构建参考了inceptionV3
7.	特征提取层：卷积核的数目为128。采用了大小为3*3的二维卷积filter，激活函数选择ReLU（sigmoid比较容易丢失梯度，速度也比较慢）稀疏激活性，更好的挖掘相关特征，拟合训练数据，收敛速度快，步长为1*1. 池化层选用的是maxpooling，提取主要特征，filter大小为2*2或3*3。采用了dropout来一定程度上防止过拟合，消除减弱了神经元节点间的联合适应性，增强了模型的泛化能力，产生的向量具有一定的稀疏性，选择了较小的dropout，这样能使训练速度不是太慢。总体来说采用卷积层+池化层+dropout的结构，总共有4个这样的结构。
8.	Flatten层：将多维的输入一维化
9.	全连接层：采用了2个全连接层，输出维度分别为512和256，并且各自都用到了dropout来减少过拟合的风险
10.	最后是使用了sigmoid的输出层，误差和softmax差不多。
11.	模型的优化方法选用了Adam，学习速率选择了较小的0.001
12.	Batch size的大小从128开始调整，最终选择了32
13.	调用了ReduceLROnPlateau，使得当验证集上误差减小变小时，减少学习速率

LGB模型：
1.	特征提取：
（1）每个band中的：最小值，最大值，平均值，标准偏差，峰度，偏斜
（2）利用Sobel filtering（一阶差分算子，为2个3*3的滤波器，可以用来检测边缘和特征提取）：提Sober滤波后的横向与纵向的标准偏差
（3）利用Laplace filtering（二阶微分线性算子，边缘定位能力更强，锐化效果更好，能够突出图像中的细节信息）：提取Laplace滤波后的标准偏差
（4）考虑到band1和band2之间的联系，还提取了两者的Pearson相关系数矩阵，和band1与band2欧氏距离的标准偏差
（5）除背景水以外部分的体积，因为背景对于冰川的检测影响不大，利用形态分析学，利用颜色之间的差异，提取出图像中除背景意外部分的体积。
    2.  训练模型：采用10折交叉验证来训练并验证模型，并且分析各个特征的importance，发现inc_angle最重要，因此后续的工作会围绕inc_angle来展开。

        Inc_angle modified：通过观察训练数据集和测试数据集的的inc_angle，如果训练集中相应的角度在测试集中出现，则根据训练集中is_iceberg标签来预测测试集的，若训练集中is_iceberg>0.5,则在测试集中相应的结果预测为0.9，反之，预测为0.1。这个处理方式可能会有data leakage，不过对于private leaderboard 和public leaderboard的成绩都有所提高。
     3. 最后，将两个模型进行简单的ensamble。

4.	总结展望：
（1）	由于测试集的数据量远远大于训练集的数据量，可以考虑使用半监督学习来增加训练数据，比如Pseudo-labeling。
（2）	可以尝试只训练只含有2个channel的CNN模型，并将其与3个channel的模型做ensamble，提高模型的泛化能力。
（3）	迁移学习+Fine tuning 是一个比较好的用来训练模型的方法。
（4）	对于ensemble，使用的模型还是不够多，可以尝试训练一些相关性较小的模型用于ensemble
（5）	如果挖掘到了关键的特征并加以利用，树模型也能得到媲美于CNN的分类效果
（6）	对于重要特征的分析与运用非常重要。
（7）	有时候多训练几个较弱的模型比训练一个较好的模型对结果的提升更大。
