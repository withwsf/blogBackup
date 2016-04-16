title: C++中的虚
tags: C++
date: 2016-04-16 20:32:56
---
C++中主要有虚函数，纯虚函数和虚继承三种“虚”。之前的[一篇文章](http://withwsf.github.io/2016/03/21/C-%E4%B8%AD%E7%9A%84%E8%99%9A%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86/)中我们讲过虚函数的实现原理。下面继续讲一讲纯虚函数和虚继承。
#### 纯虚函数
而对于纯虚函数与虚函数的区别，可以概括为————虚函数可以让派生类选择单纯继承接口还是同时继承接口与实现，而纯虚函数强制规定派生类只继承接口并自己实现函数。

* 定义了纯虚函数的类称为抽象类，抽象类不能够被实例化
```C++
class abstractClass{
public:
virtual void pure_virtual()=0; //通用的纯虚函数声明方式
};
```
* 纯虚函数一般只给出声明而不给出定义，如果要给出定义的话，形式如下：
```C++
void abstractClass::pure_virtual(){
std::cout<<"This is abstractClass::pure_virtual !"
}
```
* 只有给出纯虚函数实现的派生类才能够被实例化，否则派生类仍然是抽象类
```C++
class child: public abstractClass{
  virtual void pure_virtual();
}
void child::pure_virtual(){
  abstractClass::pure_virtual();//这里只是距离如果纯虚函数给出了实现的话如何使用
}

```

#### 虚继承
* 在普通继承中，派生类对象来将从基类对象继承来的非静态成员变量与自身的非静态成员变量放在同一块连续的内存中
```C++
class Parent{//打印出基类对象中非静态成员变量与派生类对象中非静态成员变量在内存中的offset
  //可以帮助我们进行理解
public:
    int a;
    int b;

    virtual void foo(){
        cout << "parent" << endl;
    }
};

class Child : public Parent{
public:
    int c;
    int d;

    virtual void foo(){
        cout << "child" << endl;
    }
};

int main(){

    Parent p;
    Child c;

    p.foo();
    c.foo();

    cout << "Parent Offset a = " << (size_t)&p.a - (size_t)&p << endl;
    cout << "Parent Offset b = " << (size_t)&p.b - (size_t)&p << endl;

    cout << "Child Offset a = " << (size_t)&c.a - (size_t)&c << endl;
    cout << "Child Offset b = " << (size_t)&c.b - (size_t)&c << endl;
    cout << "Child Offset c = " << (size_t)&c.c - (size_t)&c << endl;
    cout << "Child Offset d = " << (size_t)&c.d - (size_t)&c << endl;

    system("pause");
}
```
最终输出结果是：
```
parent
child
Parent Offset a = 8
Parent Offset b = 12
Child Offset a = 8
Child Offset b = 12
Child Offset c = 16
Child Offset d = 20
```
* 在多重继承中，如果继续使用单继承中的内存布局方法，可以会引起多种问题，需要另外进行设计。

考虑如下的继承关系:
```C++
class B{
public:
void foo(){
  cout<<"B"<<endl;
}

};

class C:public B{
};

class D:public B{
};

class E:public C,public D{

};

int main(){
  E e;
  e.foo();//将会报错，比如在qt creator中编译器将抛出
          //error: request for member 'foo' is ambiguous e.foo()
}
```
上面的代码中，派生类E有两个基类C和D，而C和D都有一个共同的基类B。这样e中就有两个B的对象，那么自然
在调用e.foo()的时候会产生歧义。另外，由于重复保存了B的对象，还会造成内存浪费。

为了防止上面的现象，引入了虚继承————继承基类时使用virtual关键字：
```C++
class B{
public:
void foo(){
  cout<<"B"<<endl;
}

};

class C:virtual public B{
};

class D:virtual public B{
};

class E:public C,public D{

};
```
“虚”可以认为意味着“在运行时决定”，使用虚继承的派生类在运行时才决定绑定哪个基类。C++并没有具体规定虚继承的
对象的内存布局，而是由编译器决定如何实现。一般情况下，会在虚继承的派生类对象中增添一个虚基类指针，指向基类对象。比如上面代码中E继承C和D时，会继承C和D的虚基类指针，而C和D的虚基类指针会指向同一个基类B。这样，就实现了E对象中只有一个B对象。


#### 参考资料：
[wikipedia-虚函数](https://wikipedia.kfd.me/zh-cn/%E8%99%9A%E5%87%BD%E6%95%B0_(%E7%A8%8B%E5%BA%8F%E8%AF%AD%E8%A8%80)
[memory layout of inherited class](http://stackoverflow.com/questions/8672218/memory-layout-of-inherited-class)
