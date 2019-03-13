# Data

ImageNet [http://image-net.org/](http://image-net.org/)

# GAN

## GAN入门

知乎上通俗易懂的入门文章 [https://zhuanlan.zhihu.com/p/27440393](https://zhuanlan.zhihu.com/p/27440393)

![](https://pic2.zhimg.com/80/v2-7319eab235d83fe971fb769f62cbb15d_hd.png)

GAN模型没有损失函数，优化过程是一个“二元极小极大博弈(minimax two-player game)”问题，下面是模型的价值函数：


## StyleGAN

可以生成高清人脸图像的生成网络，论文：[StyleGAN Thesis](https://arxiv.org/pdf/1812.04948.pdf)，Source Code: [StyleGAN @ Github](https://github.com/NVlabs/stylegan)

在生成人脸的基础上，还需要考虑让生成的人脸图像都是正视摄像头的，这就涉及到Head Pose Estimation，这里，我们使用[Deepgaze](https://github.com/mpatacchiola/deepgaze)预测头部扭动的角度，将角度限制在0-20度，超过这个限制的，需要做惩罚。

虽然，StyleGAN生成的人脸图像已经比较高清，但是，也还是会有一些生成的图像清晰度不够高，这样，就需要增加另外一个约束条件，强制模型生成高清的人脸图像。这里，对图像的清晰度判断又涉及到Image Quality Assessment(IQA)的技术。

## 有趣的GAN

让目标人物生成跳舞动作：[Everybody Dance Now!](https://twitter.com/hardmaru/status/1032762806796312576)

## Github上优秀的GAN项目

使用Keras实现的各种GAN：[Link](https://github.com/eriklindernoren/Keras-GAN)

# Deep Learning

Deep Learning(花书)：[英文版](http://www.deeplearningbook.org/), [中文翻译版](https://github.com/exacity/deeplearningbook-chinese)

Python版本的深度学习：[中文版](https://cnbeining.github.io/deep-learning-with-python-cn/)

莫烦介绍TensorFlow：[TensorFlow](https://morvanzhou.github.io/tutorials/machine-learning/tensorflow/)，[进化算法](https://morvanzhou.github.io/tutorials/machine-learning/evolutionary-algorithm/)

## 各种layer的作用

### 池化层(Pooling Layer)

- pooling layer的存在价值

本质上，是在精简feature map数据量的同时，最大化保留空间信息和特征信息，的处理技巧；目的是，通过feature map进行压缩浓缩，给到后面hidden layer的input就小了，计算效率能提高；CNN的invariance的能力，本质是由convolution创造的；

- pooling layer的工作原理

feature map作为input，用不含参数的filter（小方框自由设计尺寸）对其做扫描，平移，跳格；filter没有参数，但要对方框内的数值做最大值（max pooling）或均值（average pooling）保留，抹去方框内其他值，实现压缩浓缩；把留下的值形成的matrix，也就是浓缩后的feature map 传给下一层hidden layer

- convolution 也能对input做压缩，为什么非要用pooling layer?

因为pooling layer的工作原理，在压缩上比convolution更专注和易用

![](https://pic3.zhimg.com/80/v2-18dd32366cd7fc7a097c108832d7dad6_hd.jpg)

### 激活函数(activation function)
目的是解决线性不可分问题

## Github上优秀的Deep Learning项目

Deep Reinforcement Learning for Keras: [Link](https://github.com/keras-rl/keras-rl)

# Adversarial Machine Learning

## 鉴别黄色图像/生成模型无法检测的黄色图像

[黄色图片数据库](https://github.com/alexkimxyz/nsfw_data_scrapper)
