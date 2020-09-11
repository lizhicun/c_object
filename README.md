# 深度探索c++对象模型

## 第一章关于对象
### 前言
c语言中，“数据”和“处理数据的操作(函数)” 是分开声明的，将这种程序方法称为程序性(procedural),比如声明一个struct Point3d
```
struct Point3d {
    float x;
    float y;
    folat z;
};
```
而操作该数据数据的函数例如打印函数，只能另外定义成
```
void Point3d_print(const Point3d *pd) {
    print("(%f, %f, %f)", pd->x, pd->y, pd->z);
}
```
或者定义成一个宏
```
#define Point3d_print(pd) print("(%f, %f, %f)", pd->x, pd->y, pd->z);
```
而在C+中，可以采用独立的“抽象数据类型”（ADT,abstract data type）来实现
```
class Point3d {
private:
    float _x;
    float _y;
    float _z;
public:
    Point3d(float x = 0.0, float y = 0.0, float z = 0.0) :
        _x(x), _y(y), _z(z) { }
    float x() { return _x; }
    float y() { return _y; }
    float z() { return _z; }
};
inline osteream&
operator<< (ostream& os, const Point3d &pt) {
    os << "(" << pt.x() << ", " << pt.y() << ", " << pt.z() << ")";
}
```
从软件工程的角度来看，C++中的一个ADT(尤其是使用template)比C中的程序性使用全局数据要更好，威力更大。  
封装之后的成本：
* 1.数据和普通的成员函数并不会带来额外的成本，数据成员被直接内含在每一个类对象中，而成员函数虽然含在类的声明之中，却不出现在对象之中，每一个non-inline成员函数只会诞生一个函数实例，inline function则会在其每一个使用者身上产生一个函数实例
* 主要额外负担由virtual引起，virtual function机制：用以支持一个有效率的“执行期绑定”；virtual base class：用以实现“多次出现在继承体系中的base class，有一个单一而被共享的实例”

### c++对象模型
用到的例子
```
class Point {
public:
    Point(float xval);
    virtual ~Point();  //virtual member function(虚析构函数)
    float x() const;  //non-static member function
    static int PointCount();  //static member function
private:
    float _x;    //static  data member
    static int _point_count;  //non-static  data member
}
```
#### 简单对象模型
members按照其声明顺序，各被指定一个slot；members本身并不放在object之中，只有“指向member的数据成员”才放在object中
 ![Image text](https://github.com/lizhicun/c_object/blob/master/src/1.png)

很容易知道一个class object的大小为指针大小乘上class中声明的member的个数
#### 表格驱动对象模型
class object内含指向两个表格的指针，一个指针指向data member table（数据成员表，存放数据），另一个指针指向Member function table（成员函数表，存放成员函数）
![Image text](https://github.com/lizhicun/c_object/blob/master/src/2.png)

#### C++对象模型
C++对象模型将静态数据成员，静态成员函数和一般非静态成员函数均存放在个别的class object之外(单独存取，和对象无关)，而非静态数据成员则被放在每一个class object内，虚函数则以下面两个步骤支持：
* 每一个class产生一堆指向虚函数的指针，并且按照顺序置于虚函数表(virtual table，vbtl)。
* 每一个class object安插一个指针(vptr)指向第一步的virtual table。vptr的设定以及重置都由每一个class的construtor, destrutor和copy assignment自动完成。(由于构造函数来设定vptr,故构造函数无法称成为虚函数)。每一个class 所关联的type_info object(用于支持RTTI）也通常放在虚函数表的第一个slot。
![Image text](https://github.com/lizhicun/c_object/blob/master/src/3.png)

#### 加上继承后的对象模型
在虚继承，virtual base class不管在继承链上被派生多少次，派生类中永远只会存在一个virtual base class的一个实例(有且仅有一个subobject),例如在以下iostream继承体系中，派生类iostream只有virtual ios base 的一个实例.
![Image text](https://github.com/lizhicun/c_object/blob/master/src/4.png)

那么一个派生类是如何模塑其基类的实例呢？
base table模型：每一个class object内含一个bptr(pointer to base ctable),例如
![Image text](https://github.com/lizhicun/c_object/blob/master/src/5.png)

iostream类中的pbtr指向它的两个基类的table,此外，这种模型下存取的时间和继承的深度是密切相关的。例如：iostream object 要存取到ios subobject必然要经过两次。更加详细的讨论可以见第三章第四节
#### 对象模型是如何影响程序
举个栗子
```
X foobar() {
    X xx;
    X *px = new X;
    xx.foo();
    px->foo();
    delete px;
    return xx;
}
```
编译器内部的转化可能是
```
void foobar(X& _result) {   //NRV优化
    //构造_result来代替返回的local xx
    _result.X::X(); 

    //扩展X *px = new X;
    X *px = new(sizeof(X));
    if(px != 0) px->X::X();
    foo(&_result);

    //扩展px->foo();
    (*px->vtbl[2])(px);

    //delete px;
    if(px != 0) {
            (*px->vtbl[1])(px);//destructor
        _delete(px);
    }
    return;
}
```
不同的对象模型会导致现有的程序代码在内部转换结果不同
类X的对象模型如下
![Image text](https://github.com/lizhicun/c_object/blob/master/src/6.png)

### 关键字多带来的差异
struct 和 class 关键字的意义：
* 它们之间在语言层面并无本质的区别，更多的是概念和编程思想上的区别。
* struct 用来表现那些只有数据的集合体 POD（Plain OI' Data）、而 class 则希望表达的是ADT（abstract data type）的思想；
* 由于这 2 个关键字在本质是无区别，所以 class 并没有必须要引入，但是引入它的确非常令人满意，因为这个语言所引入的不止是这个关键字，还有它所支持的封装和继承的哲学；
* 可以这样想象：struct 只剩下方便 C 程序员迁徙到 C++的用途了。

### 对象的差异
C++支持三种程序设计范式(programming paradigms):程序模型(procedural model),抽象数据类型模型(abstract data type model, ADT)以及面向对象模型(object-orient model)。
只有通过pointer或者reference的间接处理，才能支持多态(用基类对象的指针或者引用处理派生类接口)，否则很可能产生切割(将派生类对象直接赋给基类对象)。  
举个栗子  
class Z public继承自 class X,X中有个虚函数rotate()，考虑下面的代码
```
void rotate(X datum, const X *pointer, const X& reference) {
    //下面这个操作总是调用X::rotate();
    datum.rotate();
    //下面这两个操作必须在运行期才能知道调用的是那个rotate()实例
    (*pointer).rotate();
    reference.rotate();
}
main() {
    Z z;
    rotate(z, &z, z);
    return 0;
}
```
在main()函数中，rotate(z, &z, z);中的第一个参数z不经过virtual机制，因此总是调用X::rotate()，而另外两个参数由于一个是指针，一个是引用，因此都调用的是Z::rotate();

指针的类型有什么用：
指向不同的类型的指针之间的差异在于：其所寻地址出来的object的类型不同，也就是说，指针类型会教导编译器如何解释某个特定的地址中的内存内容与大小；另外转换cast其实是一种编译器指令，只影响地址中的内存内容与大小的解释方式，并不会改变指针所指向的地址。
举个栗子
```
class ZooAnimal {
public:
    ZooAnimal();
    virtual ~ZooAnimal();
    virtual rotate();
protected:
    int loc;
    string name; 
};
ZooAnimal za("Zoey");
ZooAnimal *pza = &za;
```
内存布局
![Image text](https://github.com/lizhicun/c_object/blob/master/src/7.png)

int在32位机器上一般是4bits，内存中的地址涵盖1000 ~ 1003，string通常是8bits(包括4bits的字符指针以及4bits的字符长度的整数)，地址涵盖1004 ~ 1011，最后是4bits的vptr，地址涵盖1012 ~ 1015.

加上多态后
```
class Bear : public ZooAnimal {
public:
    Bear();
    ~Bear();
    void rotate();
    virtual void dance();
protected:
    enum Dances{...}
    Dances dances_known;
    int cell_block;
};
Bear b("yogi");
Bear *pb = &b;
Bear &rb = *pb;
ZooAnimal *pz = &b;
```
b,pb,rb的内存需求是怎样的呢？  
b是一个Bear Object,所需要的内存位24bytes，  
包括ZooAnimal的16bytes以及Bear本身的8bytes，而指针pb以及引用rb只需要4bytes(在32位机器上)。具体见下图：  
![Image text](https://github.com/lizhicun/c_object/blob/master/src/8.png)

假设Bear object b放在地址1000处，那么Bear指针pb 和ZooAnima指针pz有什么区别呢？它们都是指向Bear Object的首地址(第一个byte即1000)，差别在于pb所涵盖地址包括整个Bear Object即1000 ~ 1023，而pz所涵盖地址仅仅包括ZooAnimal Subobject即1000~1015.一个指针或者引用之所以支持多态，是因为它们不引发内存中与类型相关的改变，而只改变它们所指向内存的“大小和内容解释方式”。
cast是一种编译器指令，大部分情况下它并不改变一个指针所含的真正地址，它只影响“被指出之内存的大小和其内容”的解释方式
