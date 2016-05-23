title: 在Windows中阅读Caffe代码
date: 2016-05-23 15:50:33
tags: [机器学习,Caffe]
---
#### 前言
我们都知道世界上最好的IDE是Visual Studio，如果能用VS来调试、阅读Caffe的代码，观察Caffe运行过程，相信理解的效率会大大提高，但将Caffe移植到Windows上非常耗时耗力。
最近微软官方发布了Caffe的Windows branch，使用Nuget自动下载配置Caffe各种依赖，使得Caffe在Windows下的安装运行比在原生的Linux环境下更加简单。本文勾勒一下从安装到调试运行的大致步骤，希望能对后来者有帮助。
#### Caffe的安装
* 安装Visual Studio 2013
* [下载](https://github.com/BVLC/caffe/tree/windows)Windows branch的git文件，并按照README.md中的指导进行安装(如果不使用CUDA，记得按照指导修改“CommonSettings.props”文件)。
#### 调试与运行
* 第一次进行编译时Nuget会自动下载大约1G大小的依赖，请耐心等待，如果编译过程中出现“**错误	8559	error C2220: 警告被视为错误...**”，请参考[fisherman](http://115.29.54.164/?/question/281#!answer_form)的解决方法。
* 首次编译成功后，鼠标右击classfication项目，之后单击"Set as StartUp Project"选项，设置程序的启动项目
 ![修改启动项目](http://img.blog.csdn.net/20160523200739474)
* 接着进入classfication项目的classfication.cpp文件中，拉到最下面，找到main函数，修改代码，在代码中指定模型、图片等文件的路径。然后设置断点，就可以可使用VS强大的调试功能一步步观察Caffe的运行过程了。

![修改main函数](http://img.blog.csdn.net/20160523202031729)
