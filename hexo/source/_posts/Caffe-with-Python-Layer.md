title: 在Caffe中使用Python Layer
date: 2016-04-14 11:10:01
tags: [机器学习,深度学习,Caffe]
---

#### Caffe通过Boost中的Boost.Python模块来支持使用Python定义Layer：

* 使用C++增加新的Layer**繁琐**、**耗时**而且很容易**出错**
* **开发速度**与**执行速度**之间的**trade-off**


#### 编译支持Python Layer的Caffe

如果是首次编译，修改Caffe根目录下的Makefile.cinfig，uncomment
```
WITH_PYTHON_LAYER:=1
```
 如果已经编译过
```
make clean
WITH_PYTHON_LAYER=1 make&& make pycaffe
```

#### 使用Python Layer
在网络的prototxt文件中添加一个Python定义的loss层如下：
```
layer{
type: ’Python'
name: 'loss'
top: 'loss'
bottom： ‘ipx’
bottom: 'ipy'
python_param{
#module的名字，通常是定义Layer的.py文件的文件名，需要在$PYTHONPATH下
module: 'pyloss'
#layer的名字---module中的类名
layer: 'EuclideanLossLayer'
}
loss_weight: 1
}
```

#### 定义Python Layer
根据上面的要求，我们在$PYTHONPAT在创建pyloss.py，并在其中定义EuclideanLossLayer。
```Python
import caffe
import numpy as np
class EuclideadLossLayer(caffe.Layer):#EuclideadLossLayer没有权值，反向传播过程中不需要进行权值的更新。如果需要定义需要更新自身权值的层，最好还是使用C++
       def setup(self,bottom,top):
           #在网络运行之前根据相关参数参数进行layer的初始化
           if len(bottom) !=2:
              raise exception("Need two inputs to compute distance")
       def reshape(self,bottom,top):
           #在forward之前调用，根据bottom blob的尺寸调整中间变量和top blob的尺寸
           if bottom[0].count !=bottom[1].count:
              raise exception("Inputs must have the same dimension.")
           self.diff=np.zeros_like(bottom[0].date,dtype=np.float32)
           top[0].reshape(1)
       def forward(self,bottom,top):
           #网络的前向传播
            self.diff[...]=bottom[0].data-bottom[1].data
            top[0].data[...]=np.sum(self.diff**2)/bottom[0].num/2.
       def backward(self,top,propagate_down,bootm):
            #网络的前向传播
             for i in range(2):
                 if not propagate_down[i]:
                         continue
                 if i==0:
                    sign=1
                 else:
                    sign=-1
                  bottom[i].diff[...]=sign*self.diff/bottom[i].num

```
#### 原理浅析
阅读caffe源码python_layer.hpp可以知道，类PythonLayer继承自Layer，并且新增私有变量boost::python::object self_来表示我们自己定义的python layer的内存对象。

类PythonLayer类的成员函数LayerSetUP, Reshape, Forward_cpu和Backward_cpu分别是对我们自己定义的python layer中成员函数setup, reshape, forward和backward的封装调用。
```C++
class PythonLayer : public Layer<Dtype> {
 public:
  PythonLayer(PyObject* self, const LayerParameter& param)
      : Layer<Dtype>(param), self_(bp::handle<>(bp::borrowed(self))) { }

  virtual void LayerSetUp(const vector<Blob<Dtype>*>& bottom,
      const vector<Blob<Dtype>*>& top) {
    // Disallow PythonLayer in MultiGPU training stage, due to GIL issues
    // Details: https://github.com/BVLC/caffe/issues/2936
    if (this->phase_ == TRAIN && Caffe::solver_count() > 1
        && !ShareInParallel()) {
      LOG(FATAL) << "PythonLayer is not implemented in Multi-GPU training";
    }
    self_.attr("param_str") = bp::str(
        this->layer_param_.python_param().param_str());
    self_.attr("setup")(bottom, top);
  }
  virtual void Reshape(const vector<Blob<Dtype>*>& bottom,
      const vector<Blob<Dtype>*>& top) {
    self_.attr("reshape")(bottom, top);
  }

  virtual inline bool ShareInParallel() const {
    return this->layer_param_.python_param().share_in_parallel();
  }

  virtual inline const char* type() const { return "Python"; }

 protected:
  virtual void Forward_cpu(const vector<Blob<Dtype>*>& bottom,
      const vector<Blob<Dtype>*>& top) {
    self_.attr("forward")(bottom, top);
  }
  virtual void Backward_cpu(const vector<Blob<Dtype>*>& top,
      const vector<bool>& propagate_down, const vector<Blob<Dtype>*>& bottom) {
    self_.attr("backward")(top, propagate_down, bottom);
  }

 private:
  bp::object self_;
};

```

[进一步了解使用C++创建新layer](http://chrischoy.github.io/research/making-caffe-layer/)
**参考资料**

* https://github.com/BVLC/caffe/pull/1703
* https://gist.github.com//shelhamer/8d9a94cf75e6fb2df221
