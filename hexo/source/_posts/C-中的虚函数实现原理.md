title: C++中的虚函数实现原理
date: 2016-03-21 20:21:47
tags: C++
---
>参考资料：[C++ Under the hood](http://www.hanese.nl/~ewout/ESE/INF2/CPP_onder_de_motorkap.pdf)


### 虚函数是C++中[多态特性](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386820044406b227b3e751cc4d5190420d17a2dc6353000)的实现基础。
### 0.首先定义一个基类B与派生类D来辅助说明问题：
```C++
   class B{
public:
B();
virtual ~ B();
virtual void fun1();
virtual void fun2();
virtual void fun3();
void fun4 const();
}

class D:public B{
D();
~D();
virtual void fun1();
void fun5();

}
```
#### 1.虚函数的原理可以概括如下：
* 对于含有虚函数的类，编译器会为其建立一个虚函数表(vtbl)，表中的元素是指向虚函数代码所在位置的指针。
![B的虚函数表](http://7xs45i.com1.z0.glb.clouddn.com/withwsfScreenshot%20from%202016-03-21%2020-41-48.png)
* 派生类的虚函数表继承自基类并添加自己的虚函数（示例代码中D新定义的虚函数D::fun5将会被添加到虚函数表）。如果基类的虚函数被派生类重载（在上面的示例代码中基类的虚函数fun1被派生类的虚函数fun1覆盖），那么由派生类虚函数代替基类虚函数的位置（B::~B和B::fun1将被D::~D和D::fun1代替）。
![D的虚函数表](http://7xs45i.com1.z0.glb.clouddn.com/withwsfSelection_001.png)
* 含有虚函数的类的每个对象中都有一个指针（称为vptr,一般放在对象所在内存的首地址），指向类的虚函数表。
![vptr与指向的vtbl](http://7xs45i.com1.z0.glb.clouddn.com/withwsfSelection_002.png)
* 由于派生对其继承自基类的虚函数表进行了改写，所以当基类的指针或引用被绑定到派生类对象时，尽管静态类型是基类，却可以调用派生类覆盖后的虚函数。

### 2.多重继承时的情况
```C++
class B1{
B1()
virtual ~B();
}
class B2{
B2()
virtual ~B2();
}

class D:public B1, public B2{
~D();
}
```
* 编译器会为继承自多个基类的派生类产生与基类个数相同的虚函数表，并在继承自第一个基类的虚函数表上添加派生类新定义的虚函数。
* 派生类的对象中也会含有与基类个数相同的vptr。
* 将派生类对象绑定到不同的基类指针（或引用）时，编译器将根据基类指针（或引用）的不同选择使用不同的虚函数表（派生类的指针(或引用)与第一个基类的指针(或引用)使用同一个虚函数表）。！
![多重继承下的虚函数表](http://7xs45i.com1.z0.glb.clouddn.com/withwsfSelection_003.png)
