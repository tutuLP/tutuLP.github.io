---
title: "c++面试"
date: 2024-06-09
categories:
  - 就业
tags:
  - c++面试
---

RPC：是一种允许程序调用另一个地址空间（通常在另一台共享网络的计算机上）的程序的技术，就像调用本地子程序一样

数据序列化：数据序列化是将数据结构或对象状态转换成可以存储或传输的格式的过程。通过序列化，复杂的数据结构可以被转换为字节流，从而便于存储到文件、数据库，或者通过网络传输。json xml

反序列化则是将字节流恢复成原始数据结构或对象状态的过程。



网络传输用的大端



c++20 协程 比线程更小的



## 引用和指针

~~~c++
int x=10;
int *p=&x;
int &y=x;

int z=*p;使用需要解引用
int z=y; y是x的别名
~~~

目前计划->看stl库->实现数据结构部分->

### 顺序容器

1. **`std::vector`**： 动态数组，可以随机访问元素，支持高效的尾部插入和删除操作。

   ```
   cpp复制代码#include <vector>
   std::vector<int> vec;
   ```

2. **`std::deque`**： 双端队列，支持高效的头部和尾部插入和删除操作。

   ```
   cpp复制代码#include <deque>
   std::deque<int> deq;
   ```

3. **`std::list`**： 双向链表，支持高效的插入和删除操作，但不支持随机访问。

   ```
   cpp复制代码#include <list>
   std::list<int> lst;
   ```

4. **`std::forward_list`**： 单向链表，仅支持单向遍历，节省空间。

   ```
   cpp复制代码#include <forward_list>
   std::forward_list<int> fwd_lst;
   ```

### 关联容器

1. **`std::map`**： 有序关联容器，存储键值对，键是唯一的，按照键的顺序排序。

   ```
   cpp复制代码#include <map>
   std::map<int, std::string> mp;
   ```

2. **`std::multimap`**： 有序关联容器，允许键重复，按照键的顺序排序。

   ```
   cpp复制代码#include <map>
   std::multimap<int, std::string> mmp;
   ```

3. **`std::set`**： 有序集合，存储唯一的元素，按照元素的顺序排序。

   ```
   cpp复制代码#include <set>
   std::set<int> st;
   ```

4. **`std::multiset`**： 有序集合，允许元素重复，按照元素的顺序排序。

   ```
   cpp复制代码#include <set>
   std::multiset<int> mst;
   ```

### 无序容器

1. **`std::unordered_map`**： 无序关联容器，存储键值对，键是唯一的，基于哈希表实现。

   ```
   cpp复制代码#include <unordered_map>
   std::unordered_map<int, std::string> ump;
   ```

2. **`std::unordered_multimap`**： 无序关联容器，允许键重复，基于哈希表实现。

   ```
   cpp复制代码#include <unordered_map>
   std::unordered_multimap<int, std::string> ummp;
   ```

3. **`std::unordered_set`**： 无序集合，存储唯一的元素，基于哈希表实现。

   ```
   cpp复制代码#include <unordered_set>
   std::unordered_set<int> ust;
   ```

4. **`std::unordered_multiset`**： 无序集合，允许元素重复，基于哈希表实现。

   ```
   cpp复制代码#include <unordered_set>
   std::unordered_multiset<int> umst;
   ```

### 容器适配器

1. **`std::stack`**： 栈适配器，通常基于 `std::deque` 实现，也可以基于 `std::vector` 或 `std::list` 实现。

   ```
   cpp复制代码#include <stack>
   std::stack<int> stk;
   ```

2. **`std::queue`**： 队列适配器，通常基于 `std::deque` 实现。

   ```
   cpp复制代码#include <queue>
   std::queue<int> que;
   ```

3. **`std::priority_queue`**： 优先队列适配器，通常基于 `std::vector` 实现，用于维护一个堆结构。

   ```
   cpp复制代码#include <queue>
   std::priority_queue<int> pq;
   ```

### 特殊容器

1. **`std::array`**： 定长数组，大小在编译时确定。

   ```
   cpp复制代码#include <array>
   std::array<int, 10> arr;
   ```

2. **`std::bitset`**： 定长二进制数组，用于高效存储和操作二进制位。

   ```
   cpp复制代码#include <bitset>
   std::bitset<8> bs;
   ```

### 选择合适的容器

选择合适的容器取决于具体需求：

- 需要随机访问：`std::vector` 或 `std::deque`
- 需要高效的头尾部插入/删除：`std::deque`
- 需要高效的任意位置插入/删除：`std::list` 或 `std::forward_list`
- 需要键值对存储且按键排序：`std::map`
- 需要键值对存储且按哈希表存储：`std::unordered_map`
- 需要存储唯一元素且按顺序排序：`std::set`
- 需要存储唯一元素且按哈希表存储：`std::unordered_set`
- 需要LIFO结构：`std::stack`
- 需要FIFO结构：`std::queue`
- 需要优先级队列：`std::priority_queue`