---
title: "面向对象"
date: 2025-03-11
categories:
  - c++
---

# 类

局部类：函数内部定义类

### 访问控制

public 任意都可访问 

private 被类的成员函数访问 （只能自己）

protected 当前类或者子类  (血脉连接，自己和子孙)

### 友元

==friend== 可以访问类的所有成员

friend void getName(A& a); 友元类
friend void B::getName(A& a); 友元类函数
friend class B; 友元类   //**必须在此之前声明class B;**

###struct与class的区别

1. 默认的访问权限
   struct的成员默认为public，class的成员默认为private

2. 继承时的默认访问权限

- struct的继承默认是public继承，而class的继承默认是private继承。

3. 语义上的区别
   struct：关注数据集合而不是行为。class：OOP

### 静态成员static

静态函数谁都可以调用，静态成员变量谁都可以修改

静态成员变量

* 类里声明，类外初始化  static int count;  int A::count= 0;
* 类内初始化 static constexpr或const int constVar = 42; 
* 生命周期全局
* 类的静态成员的类型可以是这个类，而非静态成员只能是该类的指针或引用，因为分配内存的时候会导致无限嵌套

静态成员函数

* 只能访问静态数据成员和其他静态成员函数
* 没有this指针，只能显式传入对象的指针或引用才能访问成员变量。
* 类内声明和定义 或者类外不加static定义

### const

1. struct和class前+const 不会报错，但是没有意义
2. 常量对象

~~~c++
class base{}
const base a;
base const a;
//常量类对象，只能调用const函数，只能修改mutable声明的变量
const int b;
int const b;
~~~

> **mutable**声明的变量，表示可以被const成员对象或函数修改。

3. const类成员函数
   void getname() const {}
   只能调用const函数，只能修改mutable成员变量
   可以const重载，给不同的对象调用
   this指针默认为(const Base*)this，所以如果返回值为this，返回类型必须是const Base\*，如果返回值为\*this，返回值类型可以为Base，因为返回的是副本不影响。

   > 有一种情况可以通过const函数修改成员变量的值：那就是const函数内部调用静态成员函数，然后静态成员函数的参数是非指向const成员的指针或者非const对象的引用，这时可以通过这个参数修改。参数是类对象不能修改，因为修改的是副本。

4. const成员变量，**必须提供构造函数的初始化列表**

5. const指针
   指向cosnt对象的指针：const Base* ptr = &obj1;  不能通过该指针修改对象的值，指针可以指向其他对象。
   const指针：Base* const ptr = &obj;  指针不能指向其他对象。
   指向const对象的const指针：const Base* const ptr = &obj;既不能通过该指针修改对象的值，指针也不能再指向其他对象。

6. const引用
   const Base& y=a; 不能修改a的内容。
   Base& const y=a; ❌没有这种表达

### this指针

this指针指向调用该函数的对象
a.com(b); a隐式绑定到this上
*this，相当于this指针的解引用代表这个对象

~~~c++
MyClass& operator=(const MyClass& other) {  
    // 赋值操作...  
    return *this; // 返回当前对象的引用  
}
~~~

### 类的作用域

类允许包含指向自身的引用和指针

* 声明中的名字，参数名字，返回类型都必须在使用前声明

* 函数体会在整个类全部可见后再处理，所以会看到后续public中的成员变量，可以做函数体的返回值。

* 类外已经定义了一个全局变量名为name，则类里不能再定义一个同名的成员name

###聚合类

* 所有成员都是public 没有private，protected

* 没有定义构造函数
* 没有基类 不从其他地方继承而来
* 没有虚函数 不支持多态

~~~c++
struct Point {  
    int x;  
    int y;  
};  
// 聚合类初始化 ，使用花括号初始化列表
Point p = {1, 2};
~~~

### 字面值常量类

在编译时创建对象，并且其对象能用于常量表达式的类

* 数据成员都是**字面值类型**

> 字面值类型：整形，浮点型，枚举类型，指针(指向字面值类型或空指针)，引用类型，字面值常量类

* 有一个**constexpr构造函数** constexpr函数的参数值和返回值必须是字面值类型，且不能抛出异常，用noexcept
* 如果定义了析构函数，必须是**constexpr**的

constexpr构造函数可以声明成=default的形式、或者删除函数的形式。否则，constexpr构造函数必须符合构造函数的要求又符合constexpr函数的要求（拥有的唯一可执行语句就是返回语句）
constexpr构造函数必须初始化所有的数据成员

~~~c++
class Debug {
private:
    bool hw;
    bool io;
    bool other;
public:
    constexpr Debug(bool b = true) :hw(b), io(b), other(b) {}
    constexpr Debug(bool h, bool i, bool o) : hw(h), io(i), other(o) {}
    constexpr bool any() { return hw || io || other; }
    void set_io(bool b) { io = b; }
    void set_hw(bool b) { hw = b; }
    void set_other(bool b) { other = b; }
};
 
int main()
{
    constexpr Debug io_sub(false, true, false);  //调试IO
    constexpr Debug prod(false);  //无调试
    if (io_sub.any())  //等价于if(true)
        cerr << "print appropriate error messages;" << endl;
    if(prod.any()) //等价于if(false)
        cerr << "print an error message;" << endl;
    return 0;
}
~~~

# 类函数

## 自由函数

不属于任何结构体和类，在全局任何地方都可以调用 不接受this指针
std::ostream& print(std::ostream& os, const sals& item)

~~~c++
#include <iostream>
#include <fstream>
#include <string>
// 定义 sals 类
class sals {
private:
    std::string bookNumber;
public:
    // 构造函数
    sals(const std::string& number) : bookNumber(number) {}
    // 返回书籍编号
    std::string isbn() const {
        return bookNumber;
    }
};
// 输出 sals 对象的内容到任何输出流
std::ostream& print(std::ostream& os, const sals& item) {
    os << "Book Number: " << item.isbn();
    return os;
}
int main() {
    // 创建一个 sals 对象
    sals myBook("1234567890");
    // 输出到标准输出流
    print(std::cout, myBook) << std::endl;
    // 输出到文件流
    std::ofstream outFile("book_info.txt");
    if (outFile.is_open()) {
        print(outFile, myBook) << std::endl;
        outFile.close();
        std::cout << "Data written to file successfully." << std::endl;
    } else {
        std::cerr << "Unable to open file." << std::endl;
    }
    return 0;
}
~~~

## 构造函数

只要定义了构造函数就不会自动生成默认构造函数，如果需要应该显示定义
成员是**const**或者**引用**，某种**未提供默认构造函数的类类型**，必须通过构造函数初始值列表提供初始值，而不是在函数体内赋值

### 默认构造函数

* 没有参数
* 如果类中没有显式定义任何构造函数，编译器会自动生成一个默认构造函数（仅当类中有需要初始化的成员时）。
* 显示定义构造函数

1. MyClass() = **default**;
2. MyClass(){}; 使用这种方式，编译器就不会再生成其他隐式的特殊成员函数（如拷贝构造函数、拷贝赋值运算符等），除非你显式地定义它们。

> 有一个成员变量int c;无论如何不会自动初始化为0(内置参数类型),使用默认构造函数则c会初始化为一个类似32691的值

### 拷贝构造函数

使用原有对象的值初始化一个新的对象。

* 第一个参数是自身引用类型(推荐为const const MyClass& other)，任何额外参数都有默认值
  如果不为const自身引用类型：
  1. 不能接受const和volatile
  2. 不能使用 MyClass obj2 = MyClass (obj1);因为obj1是一个临时变量，**临时变量是一个右值，不能被修改**

- 可以存在多个拷贝构造函数
- 一般隐式使用 MyClass B=A;     显式使用：MyClass B = MyClass(A); 

什么时候会使用拷贝构造函数？

对象作为函数参数，以值传递的方式传入函数体，会使用拷贝构造函数生成副本。(建议传指针或者引用)
对象作为函数返回值，以值传递的方式从函数返回
使用原有对象的值初始化一个新的对象。

==浅拷贝==
默认拷贝构造函数 ClassName(const ClassName& other);
定义了拷贝构造函数就不会自动生成默认的
不能处理需要动态分配内存的的情况(含指针等)，使多个对象指向同一个内存
静态成员变量count用于计数，计数在构造函数中实现，调用默认拷贝不会实现计数

==深拷贝==

~~~c++
MyClass(const MyClass& other) {
    data = new int(*(other.data));
}
~~~

自定义实现，解决动态分配问题。

### 移动构造函数

高效转移资源，避免不必要的深拷贝

什么时候使用？

函数返回临时对象：将临时对象的资源转移到接收返回值的对象中。
容器插入临时对象：在向容器（如 std::vector）中插入临时对象时，会调用移动构造函数来避免不必要的拷贝。
MyClass obj2(std::move(obj1));
MyClass obj = MyClass(5);  //区别MyClass obj = MyClass(obj1);  **MyClass(5)是一个右值**

- 参数是该类对象的一个右值引用（T&&），用于通过移动一个已存在的对象（通常是临时对象）来初始化新创建的对象。
- 定义ClassName(ClassName&& other); 
- 转移之后需要置空原资源，防止原对象被析构时释放了转移的资源
-  MyClass(MyClass&&) = delete;

~~~c++
MyClass(MyClass&& other) noexcept : data(other.data), size(other.size) {
    other.data = nullptr;   //原对象资源置空
    other.size = 0;
    std::cout << "Move constructor called" << std::endl;
}
//noexcept 不会抛出异常
~~~

### 转换构造函数

* 转换构造函数允许将一个其他类型的数据转换成一个**临时类对象**。

* 只有一个参数，并且该参数不是本类的const引用时，该构造函数被视为转换构造函数。

* 转换构造函数可以隐式地转换其他类型的对象到本类的对象，使用`explicit`关键字可以禁用这种隐式转换。
* 可以用类对象直接与int相加

使用：
Complex c2 = 12；隐式，非explict
Complex c2 = Complex(12);  显式
c1 = Complex(9);  转换构造函数返回的是一个临时类对象
c1 + 1; 隐式，非explict配合+运算符重载

~~~c++
class Complex{
public:
    double real , imag;
    explicit Complex(int i){
        std::cout << "IntConstructor called" << std::endl;
        real = i;
        imag = 0;
    }
};
void operator+(const Complex& a) {
    this->real += a.real;
    this->imag += a.imag;
}
~~~

#### explicit

使用explicit关键字可以防止构造函数进行隐式类型转换。
一个类的对象与另一个类型的值进行赋值或比较时，如果类的构造函数可以接受该类型的参数，编译器可能会尝试使用构造函数来隐式地将一个类型的值转换为类的对象。

用于转换构造函数或者单参数模板构造函数 ，explicit TemplateClass(T value) : data(value)

### 委托构造函数

- 委托构造函数允许一个构造函数调用同一个类中的另一个构造函数来执行初始化工作。
- 委托构造函数通过初始化列表中的'类名后跟参数列表'来调用另一个构造函数。

~~~c++
 Person(const std::string& name, int age) : name(name), age(age) {}
//委托构造函数
 Person(const std::string& name) : Person(name, 0) {}
~~~

### 总结

~~~c++
#include <iostream>
#include <string>

class MyClass {
private:
    int value;
    std::string name;

public:
    // 默认构造函数
    MyClass() : value(0), name("default") {
        std::cout << "Default constructor called" << std::endl;
    }
    // 拷贝构造函数
    MyClass(const MyClass& other) : value(other.value), name(other.name) {
        std::cout << "Copy constructor called" << std::endl;
    }
    // 移动构造函数
    MyClass(MyClass&& other) noexcept : value(other.value), name(std::move(other.name)) {//更高效
        other.value = 0;
        std::cout << "Move constructor called" << std::endl;
    }
    // 转换构造函数
    MyClass(int val) : value(val), name("converted") {
        std::cout << "Conversion constructor called" << std::endl;
    }
    // 委托构造函数
    MyClass(int val, const std::string& str) : MyClass(val) {//委托转换构造函数
        name = str;
        std::cout << "Delegating constructor called" << std::endl;
    }
    // 打印对象信息的成员函数
    void printInfo() const {
        std::cout << "Value: " << value << ", Name: " << name << std::endl;
    }
};

int main() {
    // 使用默认构造函数
    std::cout << "Using default constructor:" << std::endl;
    MyClass obj1;
    obj1.printInfo();
    // 使用转换构造函数
    std::cout << "\nUsing conversion constructor:" << std::endl;
    MyClass obj2(42);
    obj2.printInfo();
    // 使用委托构造函数
    std::cout << "\nUsing delegating constructor:" << std::endl;
    MyClass obj3(100, "custom");
    obj3.printInfo();
    // 使用拷贝构造函数
    std::cout << "\nUsing copy constructor:" << std::endl;
    MyClass obj4(obj3);
    obj4.printInfo();
    // 使用移动构造函数
    std::cout << "\nUsing move constructor:" << std::endl;
    MyClass obj5(std::move(obj2));
    obj5.printInfo();
    return 0;
}
~~~

##内联函数

是在编译的时候将函数嵌到调用了它的地方
隐式内联：类声明和定义放在一起，只是建议编译器内联
显示内联：使用inline

## 析构函数

~MyClass() {}

隐式调用：当离开当前作用域会自动调用析构函数，如果有动态内存分配的成员在函数实现中添加释放
显示调用：当用new管理对象时

~~~c++
char* buffer = new char[sizeof(MyClass)];
MyClass* obj = new (buffer) MyClass();
obj->~MyClass();
delete[] buffer;
~~~

* 不可重载
* 没有参数和返回值
* private多用于设计模式中，单例等
* 一个类有虚函数或作为基类，析构函数应该声明为虚函数 **virtual** ~Base() {...}
* * 首先调用该类的析构函数，然后按继承层次依次向上调用基类的析构函数。
* 纯虚析构函数 
  **抽象类：**如果一个类的析构函数被声明为纯虚函数，该类是抽象类，不能实例化。
  纯虚析构函数仍然需要定义，因为对象销毁时必须调用析构函数。

~~~c++
class AbstractBase {
public:
    virtual ~AbstractBase() = 0;  // 纯虚析构函数
};
AbstractBase::~AbstractBase() {
    // 纯虚析构函数的定义
}
~~~

* 禁止析构 以禁止对象被销毁。这在某些设计模式中可能会用到。~NonDestructible() = **delete**;

## 拷贝赋值运算符

已存在的对象给一已存在的对象赋值
ClassName& operator=(const ClassName& other);
ClassName a;ClassName b; b=a;

* 默认拷贝赋值运算符-参考浅拷贝
* 深拷贝需要自己定义
* 禁止对象被拷贝 删除拷贝赋值运算符 MyClass& operator=(const MyClass&) = delete;
* * 检查自赋值-避免资源意外释放
  * 释放已有资源 释放当前对象已经持有的资源（例如内存）
  * 深拷贝新资源
  * 返回当前对象的引用

~~~c++
   MyClass& operator=(const MyClass& other) {
        if (this == &other) return *this; // 防止自我赋值
        delete data; // 释放原来的内存
        data = new int(*(other.data)); // 深拷贝
        return *this;
    }
~~~

## 移动赋值运算符

实现步骤

* 检查自赋值
* 释放当前资源
* 转移资源
* 返回 *this

~~~c++
MyClass& operator=(MyClass&& other) noexcept {
        if (this != &other) { // 检查自赋值
            delete[] data; // 释放当前资源
            data = other.data; // 转移资源
            other.data = nullptr; // 防止资源释放
        }
        return *this;
    }
obj5 = std::move(obj2);
~~~

## 三五法则 

定义其中一个需要定义其他的

* **三法则** 

析构函数 拷贝构造函数 拷贝赋值运算符
比如：当成员中有动态分配的数组时，需要显示定义析构函数来释放内存。同时避免浅拷贝的问题(多个对象共享一块内存，导致多次释放)，所以需要实现深拷贝。

* **五法则**

移动构造函数  移动赋值运算符
移动构造函数和移动赋值运算符通过将临时对象的资源所有权转移到新对象，避免了不必要的深拷贝，提高了性能

~~~c++
#include <iostream>

class MyClass {
private:
    int* data;
public:
    // 默认构造函数
    MyClass(int value) : data(new int(value)) {
        std::cout << "默认构造函数被调用" << std::endl;
    }

    // 拷贝构造函数
    MyClass(const MyClass& other) : data(new int(*(other.data))) {
        std::cout << "拷贝构造函数被调用" << std::endl;
    }

    // 拷贝赋值运算符
    MyClass& operator=(const MyClass& other) {
        if (this == &other) return *this; // 防止自我赋值
        *data = *(other.data); // 拷贝数据
        std::cout << "拷贝赋值运算符被调用" << std::endl;
        return *this;
    }

    // 移动构造函数
    MyClass(MyClass&& other) noexcept : data(other.data) {
        other.data = nullptr; // 将源对象的数据指针设为nullptr
        std::cout << "移动构造函数被调用" << std::endl;
    }

    // 移动赋值运算符
    MyClass& operator=(MyClass&& other) noexcept {
        if (this == &other) return *this; // 防止自我赋值
        delete data; // 释放当前对象的资源
        data = other.data; // 接管源对象的数据指针
        other.data = nullptr; // 将源对象的数据指针设为nullptr
        std::cout << "移动赋值运算符被调用" << std::endl;
        return *this;
    }

    // 析构函数
    ~MyClass() {
        delete data; // 释放资源
        std::cout << "析构函数被调用" << std::endl;
    }

    // 获取值
    int getValue() const { return *data; }
};

int main() {
    MyClass obj1(42);           // 默认构造函数
    MyClass obj2 = obj1;        // 拷贝构造函数
    MyClass obj3(0);            // 默认构造函数
    obj3 = obj1;                // 拷贝赋值运算符

    MyClass obj4 = std::move(obj1);  // 移动构造函数
    MyClass obj5(0);                 // 默认构造函数
    obj5 = std::move(obj2);          // 移动赋值运算符

    std::cout << "obj1的值: " << (obj1.getValue() ? std::to_string(obj1.getValue()) : "nullptr") << std::endl;
    std::cout << "obj2的值: " << (obj2.getValue() ? std::to_string(obj2.getValue()) : "nullptr") << std::endl;
    std::cout << "obj3的值: " << obj3.getValue() << std::endl;
    std::cout << "obj4的值: " << obj4.getValue() << std::endl;
    std::cout << "obj5的值: " << obj5.getValue() << std::endl;

    return 0;
}
~~~

## 操作重载

<img src="./images/C++ 面向对象.assets/image-20240608132237830.png" alt="image-20240608132237830" style="zoom:33%;" />

de1+de2 或 operator+(de1+de2)

de1+=de2 或 de1.operator+=(de2) this绑定到de1 de2作为实参

#面向对象程序设计

OOP： 封装 继承 多态

## 继承

class Dog : public Animal
继承会继承私有成员，占用内存，但是不能访问

==继承类型== 默认为private

- 公有继承（public）：不能访问基类的private，其他类型不变
- 保护继承（protected）：public，protected -> protected
- 私有继承（private）：public，protected -> private
- 继承来的成员变量 可以使用using改变可访问性 public: using A::num;

一个派生类**继承**了所有的基类方法，但下列情况除外：

- 基类的构造函数。c++ 11中使用using Base::Base;可以继承构造函数
- 析构函数。 
- 赋值运算符重载函数
- 友元关系。

### 多继承

一个子类有多个父类

class C: public A, public B

==**菱形继承**==
A B有一个共同的祖宗O，这时A B都有O的元素，C不知道继承A和B哪一个的元素

~~~c++
class Base {public:int data;};
// 中间基类 A
class DerivedA : public Base {};
// 中间基类 B
class DerivedB : public Base {};
// 最终派生类
class FinalDerived : public DerivedA, public DerivedB {};

// obj.data = 10; 二义性报错
obj.DerivedA::data = 10; //需要指定路径
//虚继承解决
class A : virtual public Base {};
class B : virtual public Base {};
obj.data = 10;
~~~

### 赋值

子类的构造函数用**初始化列表**的方式时不能直接给继承来的变量赋值，必须要在列表中调用父类的构造函数，否则就在函数体内赋值。

~~~c++
Fu(int value) : baseValue(value) {}
Zi() : Fu(2), derivedValue(2) {}
//或者
Zi() : derivedValue(2) {baseValue=2;}
~~~

子类对象给父类对象赋值

~~~c++
Zi a;
Fu b;
b=a; //会将继承来的变量的值赋值给父类对象
//a=b;不可以
~~~

父类引用子类对象或者父类指针指向子类对象，只能访问子类能访问的父类对象

~~~c++
Zi a;
Fu& b = a;
Fu* p = &a;
~~~

==**同名**==

如果子类有一个和父类同名的成员，子类对象默认访问子类成员，访问父类成员要加fu::

如果子类有一个和父类同名的函数，访问也是先访问子类的函数，如果子类的函数参数与父类不一样，调用函数也不会自动去匹配参数

==**构造函数**==

会先调用父类的构造函数，然后调用子类的构造函数
子类隐式调用父类的默认构造函数，或显示调用构造函数

~~~c++
class human {
protected: std::string _name;
public:
    human(std::string name = "小明") : _name(name) {std::cout << "调用 human 构造函数，姓名: " << _name << std::endl;}
};
class student : public human {
private: int _age;
public:
    student(std::string name, int age) : _age(age){std::cout << "调用 student 构造函数，姓名: " << _name << ", 年龄: " << _age << std::endl;}
};
int main() {
    student st("小红", 18);
    return 0;
}
//human的构造函数带有默认参数，它可以当作默认构造函数来使用，隐式调用 小明 18  这时如果还有一个默认构造函数human(){_name="小黄";}，那么编译会有歧义，因为不知道隐式调用哪一个函数
student(std::string name, int age) : _age(age),human(name)
//显示调用父类构造函数 小红 18
~~~

==**析构函数**==

先调用子类的析构再父类的析构，禁止子类中调用父类的析构-造成空间释放两次。



子类拷贝构造函数调用父类的拷贝构造函数时，直接传入子类的对象，父对象会拿到他有的哪部分，剩下的在子类拷贝构造函数中完成

~~~c++
student(student& s)
		:human(s)//直接将子类传过来通过切片拿到父类中的值
		,_age(s._age)//拿除了父类之外的值
	{
		cout << s._age << endl<<s._name<<endl;
	}
~~~

* 显示调用父类的拷贝赋值运算符

~~~c++
student& operator=(const student& s){
		if (this != &s){
			cout << "调用了子类" << endl;
			human::operator=(s);//必须调用父类运算符
			_age = s._age;
			_name = s._name;
		}
		return *this;
	}
~~~

## 多态

编译时多态：静态绑定，编译时就确定了要调用的函数或方法，比如重载(函数重载，运算符重载)，模板。
运行时多态：动态绑定，在运行时确定调用哪个函数 ，实现：通过对象的虚函数表指针找到并调用正确的函数

虚函数：在基类中使用关键字virtual声明的函数。
虚函数表：每个有虚函数的类都有一个虚函数表。虚函数表是一个指针数组，其中每个指针指向类的虚函数。存放在全局数据空间。就算是继承父类的虚函数也有自己的虚函数表，如果没有重写，那么虚函数表中的指针还是指向父类的虚函数。
虚函数表指针：指向虚函数表(开头位置)。大小：32位系统(4字节)，64位系统(8字节)
调用顺序：判断函数是否为虚函数，如果是则通过虚函数表指针找到虚函数表，再在虚函数表中找到该虚函数调用。

> 如果派生类还有一个int类型的变量，64位系统中，虚函数表指针为8 节，int为4字节，考虑内存对齐后，对象大小通常为16字节（按8字节对齐），32位系统中为8字节(四字节对齐)。
>
> 多继承：如果一个派生类继承了两个类，两个类都有虚函数，这个派生类就有两个虚函数表指针，如果这个派生类再次声明自己新的虚函数，那么会放到第一个虚函数表的末尾。

### vitual override

vitual 声明虚函数，当基类的指针或者引用指向派生类，调用虚函数时，子类如果对虚函数重写，调用的是子类重写的虚函数，没有重写调用的就是父类的虚函数，如果没有声明为虚函数，则调用的都是父类的函数。
override 显示声明重写了虚函数，void reload() override{}; 有override的函数编译时会检查重写是否正确。

重写虚函数传入参数和返回类型不能改变-如果类型不同会造成重定义
例外：

1. 协变 ，父类的虚函数返回类型是自身的指针或引用，子类的虚函数返回类型可以改为子类的指针或引用
2. 重写析构函数

~~~c++
Base* ptr = new Derived();
delete ptr;
//如果Base的析构函数没有声明为虚函数将不会调用Derived的析构函数，造成无法释放的内存泄露。如果声明为虚函数，则先调用子类的析构函数，再调用基类的虚构函数。
~~~

> 注意区别重载，重载是同一个作用域内函数名相同，所以子类中可以有与继承的虚函数相同函数名的函数，但是一个是重写，一个重载，当没有重写虚函数时，子类对象调用该函数名函数不会做动态匹配，如果要使用父类的同名函数需要加作用域::

**建议析构函数都声明为虚函数，避免内存泄漏**
构造函数不能声明为虚函数

### final

c++11，用于类名后或者虚函数后防止被继承。
class student final{};
virtual void abc() override final{}; 一般是某个子类对继承的虚函数进行重写之后，它的子类不会用到这个虚函数了就,子类也不能出现和这个虚函数相同名字，参数，返回值的函数。

### 抽象基类

含纯虚函数 virtual void draw()= 0;  子类必须提供实现
**不能创建抽象基类对象**

~~~c++
// 这里常考一道笔试题：sizeof(Base)是多少？
class Base{
public:
    virtual void Func1(){
        cout << "Func1()" << endl;
    }
private:
    int _b = 1;
};
//虚函数表指针(32位系统4字节，64位系统8字节)+int成员4字节,考虑内存对齐，8/32
//类的成员函数不占空间，存放在程序的代码段
~~~

# union联合

union成员共享一个内存空间，给其中一个成员赋值，另一个已经赋值的成员就没有值了。
只给unino分配最大的成员所需的空间

~~~c++
union MyUnion {  
    int i;  
    float f;  
    char str[20];  
};
MyUnion u;
u.i=10;

//匿名联合
union {
    int intValue;
    char charValue;
};
// 直接访问成员
intValue = 10;

//构造与析构
union MyUnion {
    int intValue;
    std::string stringValue;
    // 手动定义构造函数
    MyUnion() {
        new (&intValue) int(0);
    }
    // 手动定义析构函数
    ~MyUnion() {
        // 由于 string 有非平凡析构函数，需要手动调用
        stringValue.~basic_string();
    }
};
~~~

* 标准库中的 std::variant：C++17 引入了 std::variant，它提供了一种类型安全的方式来存储一个值，这个值可以是几种类型中的任何一种。std::variant 是 union 的一个更现代、更安全的替代品。

# variant

c++ 17中对union更现代，安全的替代品

```cpp
#include <variant>
// 定义一个 variant 类型，它可以存储 int、double 或 std::string 类型的值
std::variant<int, double, std::string> myVariant;
// 存储一个 int 值
myVariant = 42;
// 使用 std::get_if 检查当前存储的是否为 int 类型, 是则返回其值的指针
if (const int* intPtr = std::get_if<int>(&myVariant)) {
     td::cout << "The stored value is an int: " << *intPtr << std::endl;
}
```

类型匹配

~~~c++
#include <iostream>
#include <variant>
#include <string>
#include <vector>
// 定义一个访问器结构体，使用重载的 () 运算符处理不同类型
struct VariantVisitor {
    void operator()(int value) const {
        std::cout << "Received an int: " << value << std::endl;
    }
    void operator()(double value) const {
        std::cout << "Received a double: " << value << std::endl;
    }
    void operator()(const std::string& value) const {
        std::cout << "Received a string: " << value << std::endl;
    }
};
int main() {
    std::vector<std::variant<int, double, std::string>> variants = {42, 3.14, "Hello"};
    for (const auto& var : variants) {
        // 使用 std::visit 调用访问器来处理 variant 中的值
        std::visit(VariantVisitor{}, var);
    }
    return 0;
}
~~~

异常处理

~~~c++
std::variant<int, std::string> myVariant = 42;
try {
    // 尝试获取一个 std::string 类型的值，由于当前存储的是 int，会抛出异常
    std::string str = std::get<std::string>(myVariant);
} catch (const std::bad_variant_access& e) {
    std::cout << "Exception caught: " << e.what() << std::endl;
}
~~~
