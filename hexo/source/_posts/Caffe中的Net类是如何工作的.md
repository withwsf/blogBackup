title: Caffe中的Net类是如何工作的
date: 2016-05-24 20:21:47
tags: [C++,Caffe]
---
Net类是Caffe中Blobs，Layers，Nets三个抽象层次中最高层的抽象。Nets类负责按照网络定义文件将需要的layers和中间blobs进行实例化，并将所有的Layers组合成一个有向无环图。Nets还提供了在整个网络上进行前向传播与后向传播的接口。下面从观察Net运行的角度来解析一下Net类如何工作。
### Net类数据成员概述
**下面对Net类中比较重要的数据成员进行说明：**

* **`vector<shared_ptr<Layer<Dtype> > > layers_;`**
layers_中存放着网络的所有layers，也就是Net类的实例保存着网络定义文件中所有layer的实例

* **`vector<shared_ptr<Blob<Dtype> > > blobs_;`**
blobs_中保存着网络所有的中间结果，即所有layer的输入数据（bottom blob）和输出数据（top blob）

* **`vector<vector<Blob<Dtype>*> > bottom_vecs_;`**
* **`vector<vector<Blob<Dtype>*> > top_vecs_;`**  
`bottom_vecs_`保存的是各个layer的bottom blob的指针，这些指针指向`blobs_`中的blob。`bottom_ves.size()`与网络layer的数量相等，由于layer可能有多个bottom blob，所以使用`vector<Blob<Dtype>*>`来存放layer-wise的bottom blob。同理可以知道`top_vecs`的作用。

* **`vector<shared_ptr<Blob<Dtype> > > params_;`**
* **`vector<Blob<Dtype>*> learnable_params_;`**
上述两个数据成员存放的是指向网络参数的指针，注意，直接拥有参数的是layer，`params_`保存的只是网络中各个layer的参数的指针；而`learnable_params_`也如其名字所指，保存的是各个layer中可以被学习的参数。


### Net类的实例化(一个网络的建立)
#### 构造函数
Net类有两个构造函数，分别是`Net(const NetParameter& param, const Net* root_net)`和`Net(const string& param_file, Phase phase, const Net* root_net)`,前者接受`NetParameter`的const引用作为参数(后面参数root_net与多GPU并行训练有关，忽略掉并不影响理解)，后者接受定义网络prototxt文件路径和`phase`作为输入。
前者直接调用`Init()`函数，后者将prototxt文件解析为`NetPrameter`后调用`Init()`函数。
#### Init()函数
`Init()`函数承担初始化一个网络的任务，摘取主干代码描述如下（忽略细节，大致描述过程）：

```C++
for (int layer_id = 0; layer_id < param.layer_size(); ++layer_id) {//param是网络参数，layer_size()返回网络拥有的层数
    const LayerParameter& layer_param = param.layer(layer_id);//获取当前layer的参数
    layers_.push_back(LayerRegistry<Dtype>::CreateLayer(layer_param));//根据参数实例化layer


//下面的两个for循环将此layer的bottom blob的指针和top blob的指针放入bottom_vecs_和top_vecs_,bottom blob和top blob的实例全都存放在blobs_中。相邻的两层，前一层的top blob是后一层的bottom blob，所以blobs_的同一个blob既可能是bottom blob，也可能使top blob。
    for (int bottom_id = 0; bottom_id < layer_param.bottom_size();++bottom_id) {
       const int blob_id=AppendBottom(param,layer_id,bottom_id,&available_blobs,&blob_name_to_idx);
    }

    for (int top_id = 0; top_id < num_top; ++top_id) {
       AppendTop(param, layer_id, top_id, &available_blobs, &blob_name_to_idx);
    }

//接下来的工作是将每层的parameter的指针塞进params_，尤其是learnable_params_。
   const int num_param_blobs = layers_[layer_id]->blobs().size();
   for (int param_id = 0; param_id < num_param_blobs; ++param_id) {
       AppendParam(param, layer_id, param_id);
       //AppendParam负责具体的dirtywork
    }

    }
```
#### 初始化之后
经过上述过程的网络，参数都是随机产生或者指定的，如果进行预测或这fine-tuning，就需要将载入预训练的权值，Net类提供的函数`CopyTrainedLayersFrom(const string& trained_file)`可以实现这个过程。
### 网络的运行(前向传播, 反向传播和权值更新)
Net类可以提供网络级的前向前向传播、反向传播和权值更新(即在网络的所有层上有序执行前述动作)。
#### 前向传播
与前向传播相关的函数有`Forward(const vector<Blob<Dtype>*> & bottom, Dtype* loss)`,`Forward(Dtype* loss)`,`ForwardTo(int end)`，`ForwardFrom(int start)`和`ForwardFromTo(int start, int end)`，前面的四个函数都是对第五个函数封装，第五个函数定义如下：
    ```
    template <typename Dtype>
Dtype Net<Dtype>::ForwardFromTo(int start, int end) {
  CHECK_GE(start, 0);
  CHECK_LT(end, layers_.size());
  Dtype loss = 0;
  for (int i = start; i <= end; ++i) {
    // LOG(ERROR) << "Forwarding " << layer_names_[i];
    Dtype layer_loss = layers_[i]->Forward(bottom_vecs_[i], top_vecs_[i]);
    loss += layer_loss;
    if (debug_info_) { ForwardDebugInfo(i); }
  }
  return loss;
}
    ```
    重点语句是`layers_[i]->Forward(bottom_vecs_[i], top_vecs_[i]);`，使用layer对应bottom blob和top blob进行前向传播。

#### 反向传播
与前向传播一样，反向传播也有很多相关函数，但都是对`BackwardFromTo(int start, int end)`的封装。
```void Net<Dtype>::BackwardFromTo(int start, int end) {
  CHECK_GE(end, 0);
  CHECK_LT(start, layers_.size());
  for (int i = start; i >= end; --i) {
    if (layer_need_backward_[i]) {
      layers_[i]->Backward(top_vecs_[i], bottom_need_backward_[i], bottom_vecs_[i]);
      if (debug_info_) { BackwardDebugInfo(i); }
    }
  }
}
```
与前向传播相反，反向传播是从尾到头进行的。
#### 权值更新

```
template <typename Dtype>
void Net<Dtype>::Update() {
  for (int i = 0; i < learnable_params_.size(); ++i) {
    learnable_params_[i]->Update();
  }
}
```
在训练的过程中layer的权值要根据反向传播并累积的梯度进行更新，更新的过程由`Update()`完成。这个函数的功能十分明确，对每个存储learnable_parms的blob调用blob的`Update()`函数，来更新权值。
