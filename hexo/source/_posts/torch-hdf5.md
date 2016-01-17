title: 为Torch创建hdf5训练文件
date: 2015-12-23 22:13:02
tags: [机器学习,Torch]
---
### 1. Torch与HDF5
**[Torch](http://torch.ch)** 是用**C/CUDA**作为底层实现，用**LuaJIT**作为接口的机器学习算法框架。

**[HDF5](http://docs.h5py.org/en/latest/quick.html)**是用于海量复杂数据集管理的技术,能够支持多种平台与多种语言接口（C，C++，Python等）。

Torch的tutorial只提供了处理images和random tensors的方法，并没有对其他格式提供示例。本文使用将对如何创建HDF5数据集以及如何在Torch中使用HDF5文件格式做一个梳理。

### 2. 使用Python创建HDF5文件
* **1  在Python中安装 h5py**
在Ubuntu下 `sudo pip install h5py`

* **2  创建HDF5对象**
我们需要的HDF5文件在root (group) 下应该有“data”和“label”两个dataset，并在创建时指定这两个dataset的尺寸。
``` python
import h5py, os                             
f=h5py.File('train.h5','w') #以'w'模式创建一个名为'train.h5'的HDF5对象
f.create_dataset('data',(100,3,32,32),dtype='f8') #有100个样本，每个样本有三个channel，每个channle的尺寸是32*32
f.create_dataset('label',(100,1),dtype='i') #创建存放label的dataset，尺寸是100*1
```

* **3 写入数据**
写入数据其实很简单，只需要对dataset中的每个对象赋值即可。我们使用numpy随机生成的数据为例进行赋值。
```python
import numpy as np
for i in range(100):
  temp=np.random.random((3,32,32))
  f['data'][i]=temp #写入data
  f['label'][i]=i%4 #写入label
f.close()
 
```
以上，即可生成一个用于训练的train.hdf5文件。
### 3. 在torch中使用HDF5文件
在torch中读取HDF5文件需要用到**torch-hdf5**,可以参照[官方文档来安装](https://github.com/deepmind/torch-hdf5/blob/master/doc/usage.md)。
```lua
require 'hdf5';
myFile=hdf5.open('path_to_hdf5_file','r') -- 读入HDF5文件
trainset={data=myFile:read('data'):all(),label=myFile:read('label'):all():byte()}
```
trainset是与[官方tutorial](https://github.com/soumith/cvpr2015/blob/master/Deep%20Learning%20with%20Torch.ipynb)读取'cifar10-train.t7' 后同样的对象。
``` lua
trainset --输入trainset并执行后可以看到trainset的信息如下
 ----------------执行输出的信息-------------------------------------------------
｛
data : DoubleTensor - size: 100×3×32×32
label : ByteTensor -size: 100×1
｝
```



**按照上述的方法创建trainset和testset并读取数据之后，就接着可以进行训练了，由于内容不在本文范围内，所以略过。**
