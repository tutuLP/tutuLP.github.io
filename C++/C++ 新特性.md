---
title: "c++新特性"
date: 2025-03-11
categories:
  - c++
---

# c++11

c++11中有特别的经常使用的东西，所以说最少都得使用c++11

**auto  unordered_map  thread**

c++正则表达式库(RE)

##move

std::move是c++11<utility>中提供的一个函数模板，将一个左值强制转换为右值引用，从而可以调用对象的移动构造函数或移动赋值运算符，避免不必要的深拷贝，配合右值引用声明符 && 使用

### 右值引用

它为 C++ 中的移动语义和完美转发提供了基础。右值引用可以绑定到临时对象（右值），并允许对其进行修改或移动

int&& rvalueRef = 5; // 右值引用绑定到临时对象 5

右值引用声明符 &&

* 移动语义：移动构造函数和移动赋值运算符配合std::move使用

  * MyString(MyString&& other) noexcept : data(other.data), length(other.length)
  * MyString& operator=(MyString&& other) noexcept

* 完美转发：std::forward()：在函数模板中保持参数的类型和值类别（左值或右值）不变

  ```c++
  #include <iostream>
  #include <utility>
  void print(int& x) {std::cout << "Lvalue: " << x << std::endl;}
  void print(int&& x) {std::cout << "Rvalue: " << x << std::endl;}
  template<typename T> void forwardPrint(T&& arg) {print(std::forward<T>(arg));}//forward用在函数模板中，保持参数类型一致
  int main() {
      int a = 10;
      forwardPrint(a);  // 转发左值
      forwardPrint(20); // 转发右值
      return 0;}
  ```

引用折叠规则：参数为左值/左值引用，T&&将转化为int&；参数为右值/右值引用，T&&将转化为int&&

声明出来的左值引用、右值引用都是左值

## tuple

\#include<tuple>

* 创建

```c++
auto t1 = make_tuple(1, 3.14, 'a'); // 使用 make_tuple  
tuple<int, double, char> t2(1, 3.14, 'a'); // 直接初始化
```

* 访问

```c++
// 使用 std::get  
int first = get<0>(t1);  
double second = sget<1>(t1);  
char third = get<2>(t1);  
// C++17 结构化绑定  
auto [i, d, c] = t1;
```

* 大小和类型

```c++
static_assert(tuple_size<decltype(t1)>::value == 3, "Tuple size is 3");  
static_assert(is_same<tuple_element<1, decltype(t1)>::type, double>::value, "Second element is double");
```

* 修改值

```c++
tuple<int, double, char> mutableTuple(1, 3.14, 'a');  
get<0>(mutableTuple) = 2; // 修改第一个元素
```

* 比较

```c++
auto t3 = make_tuple(1, 2.0, 'b');  
if (t1 < t3) {  
    // t1 在字典序上小于 t3  
}
```

* 结构与合并 c++17

```c++
auto [a, b, c] = t1; // 解构  
auto combined = tuple_cat(t1, make_tuple(42)); // 合并
```

* 使用场景

需要从函数返回多个值时

需要存储或传递一组异质（不同类型）的数据时

模板元编程中，用于类型列表或值列表

## lambda

它是一种匿名函数的声明方式，可以在需要时内联定义函数，使得代码更加简洁和易读

`[capture list] (parameter list) -> return type { function body }`

- **capture list**：捕获列表，用于指定 lambda 表达式访问的外部变量的方式。
- * 值捕获[var]、引用捕获[&var]
  * [=],捕获所在作用域所有变量的值
  * [&]，捕获作用域内所有参数的引用
  * [=, &var] var以引用方式捕获其他以值的方式
  * [&, var]
- **parameter list**：参数列表，指定 lambda 函数的参数。与普通函数参数类似，可以为空或包含一个或多个参数。
- **return type**：返回类型，指定 lambda 函数的返回类型。可以省略，编译器会自动推导返回类型。
- **function body**：函数体，指定 lambda 函数的具体实现

可变lambda：一般捕获的变量不能修改要修改需要使用**mutable**    auto lambda = \[num]() mutable {

```c++
#include <iostream>
int main() {
    // 定义一个 lambda 表达式，接受两个参数并返回它们的和
    auto sum = [](int a, int b) -> int {
        return a + b;
    };
    // 使用 lambda 表达式计算两个整数的和
    int result = sum(3, 4);
    std::cout << "Sum: " << result << std::endl;

    return 0;
}
```

```c++
#include <iostream>
int main() {
    int x = 3;
    int y = 4;
    // 捕获 x 和 y 的值，并计算它们的和
    auto sum = [x, y]() {
        return x + y;
    };
    // 使用 lambda 表达式计算 x 和 y 的和
    int result = sum();
    std::cout << "Sum: " << result << std::endl;
    return 0;
}
```

## for each

对容器中的元素执行指定的操作,接受一个范围（通常是一个容器的迭代器对）和一个函数对象（函数指针、函数或者 Lambda 表达式），并将该函数对象应用于范围内的每个元素

```c++
#include <iostream>
#include <vector>
#include <algorithm>

// 示例函数对象
struct Print {
    void operator()(int x) const {
        std::cout << x << " ";
    }
};

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    // 使用函数对象 Print 输出每个元素
    std::for_each(vec.begin(), vec.end(), Print());
    // 使用 Lambda 表达式输出每个元素的平方
    std::cout << std::endl;
    std::for_each(vec.begin(), vec.end(), [](int x) {
        std::cout << x * x << " ";
    });

    return 0;
}
```

## 智能指针-RALL机制

自动内存管理、自动释放、防止内存泄漏等。

C++11 提供了三种主要的智能指针类型：

1. `std::unique_ptr`：
   - std::unique_ptr 独占拥有权
   - 当 std::unique_ptr 被销毁时，它所管理的资源会被自动释放。
   - 不能进行拷贝构造和拷贝赋值，但可以进行移动构造(初始化的时候move)和移动赋值(初始化之后使用)-使用std::move()移交控制权。
2. `std::shared_ptr`：
   - 多个 std::shared_pt` 可以同时指向同一块内存,共享拥有权。
   - 引用计数 管理资源的生命周期，当最后一个 std::shared_ptr被销毁时，它所管理的资源会被释放。
   - 可以使用拷贝，增加引用计数。    
3. `std::weak_ptr`：
   - std::weak_ptr 是 std::shared_ptr的弱引用，不会增加引用计数。
   - 不影响资源的生命周期，只是提供了对资源的观察。
   - 主要用于解决 std::shared_ptr 循环引用问题。(两个对象中都有一个shared_ptr指向对方导致计数至少为1，永远不会被释放)
   - 观察者模式：通过->lock() 方法来获取一个shared_ptr，如果所引用的对象还存在（即引用计数大于 0），会返回一个有效的shared_ptr，否则返回一个空的。这样就可以安全地访问可能已经被销毁的对象，同时避免了不合理地延长对象的生命周期。 缓存系统：如果存在则返回资源，不存在则创建新的

#### 使用

```cpp
//直接使用new   需要显式释放
std::unique_ptr<int> ptr(new int(42));
//使用make_unique
auto ptr = std::make_unique<int>(42);
```

```c++
unique_ptr / shared_ptr
auto ptr1 = std::make_unique<Resource>("Resource1");
std::unique_ptr<Resource> ptr2(new Resource("Resource2"));

weak_ptr
std::weak_ptr<Resource> ptr5 = ptr3;  //*从shared_ptr创建weak_ptr*
 if (auto shared_ptr = ptr5.lock()) {}  //通过lock使用
```

## 线程安全

互斥锁mutex 原子操作atomic 读写锁 shared_mutex  条件变量condition_variable 线程局部存储 无锁数据结构 单例模式的线程安全实现

### mutex

\#include <mutex>

定义：std::mutex mtx;

使用：

*lock/unlock 使用*

> mtx.lock();
>
> mtx.unlock();

*lock_guard（RAII方式）*

> std::lock_guard\<std::mutex> lock(mtx);

*unique_lock（更灵活的RAII方式）* 可以重新解锁上锁，可以配合条件变量，可以使用move转让所有权 延迟锁定defer_lock 

> std::unique_lock \<std::mutex> lock1(mtx, std::defer_lock);      
>
> std::unique_lock \<std::mutex> lock2(to.mtx, std::defer_lock);   //另一个对象to作为参数，
>
> // 避免死锁的同时加锁
>
> std::lock(lock1, lock2);

*try_lock 的使用*  非阻塞没有获取锁立即返回

>if (mtx.try_lock()) {...}
>
>​      mtx.unlock();
>
>}

### 条件变量condition_variable

\#include <condition_variable>

相对于mutex，使线程能按顺序执行

mesa语义：`while (!condition) {wait(cv);}` 防止虚假唤醒，即使被虚假唤醒也会检查条件是否满足，确保安全，而不是用if

c++11的wait带谓词(返回bool的判断条件)版本`cv.wait(lock, [this]{ return condition; });`

```c++
mutable std::mutex mtx; 
std::condition_variable cv;

//一个线程
std::unique_lock<std::mutex> lock(mtx);
...
cv.notify_one();  //通知一个正在wait的线程

//另一个线程
std::unique_lock<std::mutex> lock(mtx);
while (!condition) {
    cv.wait(lock);
}
...
```

notify_one

notify_all 

wait

wait_for 等待一段时间

wait_until 等待到一个时间点

```c++
while (queue.empty()) { 
    not_empty.wait(lock);
}

bool success = not_empty.wait_for(lock, timeout, [this]() { 
    return !queue.empty(); 
});

bool success = not_empty.wait_until(lock, deadline, [this]() {
    return !queue.empty();
});
```

防止被继承 final

# c++14

模板函数make_unique

泛型lambda表达式：允许使用auto类型作为参数

许函数在不使用 **decltype** 的情况下进行返回类型推导。

变量模板

```c++
#include <iostream>

template <typename T>
constexpr T pi = T(3.1415926535897932385);

int main() {
    std::cout << pi<double> << std::endl; 
    return 0;
}
```

二进制字面量和单引号作数字分隔符

使用 0b或 0B 前缀表示二进制数

```c++
int binary = 0b1010; 
long long largeNumber = 1'000'000'000; 
```

# c++17

* if constexpr是一个编译时条件语句，在编译时就进行条件选择不同路径
* 跨平台文件系统操作库  std::filesystem
* variant   union联合的现代更安全替代

## 结构化绑定

```c++
#include <iostream>
#include <tuple>

std::tuple<int, double> getValues() {
    return std::make_tuple(1, 2.5);
}

int main() {
    auto [a, b] = getValues();
    std::cout << a << " " << b << std::endl; 
    return 0;
}
```

## std::optional

std::optional 表示一个可能存在的值，它可以包含一个值或者为空，避免了使用指针和空指针检查

```c++
#include <iostream>
#include <optional>

std::optional<int> getValue(bool condition) {
    if (condition) {
        return 42;
    }
    return std::nullopt;
}

int main() {
    auto result = getValue(true);
    if (result) {
        std::cout << *result << std::endl; 
    }
    return 0;
}
```

