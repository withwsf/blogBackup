title: 在torch中使用cuda进行训练
date: 2015-12-29 15:50:33
tags: [机器学习,Torch]
---
### 0. 说明
本文介绍**hdf5文件的读取**、**网络的定义**与**训练**和**测试**，并强调了如何使用**cuda**对网络的训练、测试进行加速。
### 1. 过程与代码
* **1 安装hdf5** 
按照[官方tutorial](https://github.com/deepmind/torch-hdf5/blob/master/doc/usage.md)安装torch对hdf5格式文件的支持。
* **2 引入必要 package 并读入hdf5数据**
```Lua
  require 'torch';----如果是在itorch或itorch notbook中自动引入torch
  require 'hdf5';----hdf5支持
  require 'nn'; ----neural network modules 的实现
  require 'cutorch';----cuda backend支持
  require 'cunn';----neural network modules的cuda实现
```
  下面读入数据，假设train和test数据都是按照我[上篇文章](http://withwsf.github.io/2015/12/23/torch-hdf5/)的格式所存储：
```Lua
myFiletrian=hdf5.open('mytrain.h5','r')----读取hdf5文件
myFiletest=hdf5.open('mytest.h5','r')
----将读入的hdf5存到两个Table： trainset和testset中
trainset={data=myFiletrian:read('data'):all(),label=myFiletrian:read('label'):all():byte()}
testset={data=myFiletest:read('data'):all(),label=myFiletest:read('label'):all():byte()}
```
* **3 重载trainset的_index操作符并添加成员函数size()**
  Lua中也存在面向对象的概念，Lua中的类可以通过Table+function模拟出来。这里将trainset视作一个类的对象，torch要求这个类有方法trainset:size()可以返回样本个数，使用操作符trainset[i]时返回第i个样本。
```Lua
setmetatable(trainset, ----重载index操作符
    {__index = function(t, i) 
                    return {t.data[i], t.label[i]} 
                end}
);

function trainset:size()----添加成员函数 size()
    return self.data:size(1)
end
```
* **4 建立一个网络**
  我使用Torch的主要原因是因为Torch有3D conv模块，所以这里以模仿**LeNet**的3D conv网络为例：
```Lua
net=nn.Sequential()
net:add(nn.VolumetricConvolution(3,5,3,3,3))----添加3D conv层，输入3个feature cube，输出5个feature cube，filter size为3*3*3
net:add(nn.VolumetricMaxPooling(2,2,2,2,2,2))----添加3D MaxPooling层
net:add(nn.VolumetricConvolution(5,16,4,4,4))
net:add(nn.VolumetricMaxPooling(2,2,2,2,2,2))
net:add(nn.View(16*4*4*4))----连接全连接层之前要使用nn.View()进行reshpe，nn.View()里面的参数需要根据
---（接上）数据的尺寸和前面的层来计算，比如在这里我是用的训练数据是3*24*24*24的cube,分别经过3*3*3和4*4*4 filter尺寸的两次卷积和两次maxpooling之后最终的输出是16个4*4*4*的feature cube ，那么nn.View()里面的参数就应该是16*4*4*4
net:add(nn.Linear(16*4*4*4,120))
net:add(nn.Linear(120,84))----nn.Linear(a,b)，设置a个输入neurons和b个输出neurons的全全连接层
net:add(nn.Linear(84,7))
net:add(nn.LogSoftMax())-----输出每个类别概率P的log函数值log(P)

```

* **5 建立一个Solver**
```Lua
criterion=nn.ClassNLLCriterion()----使用negatice log--likehood criterion ,即计算cross-entropy损失
----注意！Torch中训练数据的Label应该从1开始，比如有四类样本，那么label应该是1、2、3、4，不能从0开始，否则报错
trainer=nn.StochasticGradient(net,criterion)----使用随机梯度下降
trainer.learningRate=0.001----设置solver参数
trainer.maxIteration=10----迭代epoch数
```
* **6 将net、criterion、trainset和testset移动到GPU中**
  如果要使用GPU加速，net、criterion、trainset和testset都需要移动到CPU内存中去
```Lua
trainset.data=trainset.data:cuda()
trainset.label=trainset.label:cuda()
testset.label=trainset.label:cuda()
testset.data=testset.data:cuda()
criterion=criterion:cuda()
net=net:cuda()
```
以trainset为例，移动到GPU之后是形式如下的对象
>{
  data : CudaTensor - size: 10500x3x24x24x24
  size : function: 0x42832ef0
  label : CudaTensor - size: 10500x1
}

* **7 训练并测试网络**
有了上面的准备，网络的训练十分简单：
```Lua
trainer:train(trainset)
```
>训练过程中输出如下：
Out[18]:StochasticGradient: training    
Out[18]:current error = 1.9459465204875 
Out[18]:current error = 1.941716547001  
Out[18]:current error = 1.921697193475  
Out[18]:current error = 1.8641934820811 
Out[18]:current error = 1.5424795499416 
Out[18]:current error = 1.3549344599815 
Out[18]:current error = 1.2374953328995 
Out[18]:current error = 1.152671923592  
Out[18]:current error = 1.094293069022  
Out[18]:current error = 1.0469601832571 
StochasticGradient: you have reached the maximum number of iterations   
training error = 1.0469601832571

由于设置的最大epoch是10，所以在对trainset迭代了10次之后训练就结束了，接下来是网络的测试：
```Lua
correct = 0
for i=1,3500 do ----我的测试集有3500个cube
    local groundtruth = testset.label[i][1]
    local prediction = net:forward(testset.data[i])
    local confidences, indices = torch.sort(prediction, true)
    if groundtruth == indices[1] then
       correct = correct + 1 
    end
end
print(correct, 100*correct/3500 .. ' % ')
```


### 2. 总结
使用Torch训练过程包括数据载入，网络定义，trainer定义，训练、测试几个部分，如果使用GPU加速，要将数据等对象移动到GPU中。
