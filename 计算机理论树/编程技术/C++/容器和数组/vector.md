`vector` 是 C++ STL（标准模板库）中最常用的**动态数组容器**，自动管理内存，支持随机访问，是面试和日常开发的核心工具。

---

## 一、基础前置

### 1. 引入头文件

使用 `vector` 必须包含对应的头文件：

```cpp
#include <vector>
// 为了简化代码，通常配合 using namespace std; 使用
using namespace std;
```

### 2. 核心特点

- **动态大小**：无需预先指定固定长度，可自动扩容 / 缩容；
- **连续内存**：元素在内存中连续存储，支持**O (1) 时间的随机访问**；
- **自动内存管理**：无需手动 `new/delete`，容器生命周期结束后自动释放内存；
- **仅支持尾部高效插入 / 删除**：中间插入 / 删除需要移动元素，时间复杂度 O (n)。

---

## 二、常用构造函数（创建 vector 的方式）

| 构造方式           | 说明                             | 代码示例                           |
| -------------- | ------------------------------ | ------------------------------ |
| 默认构造           | 创建空的 vector                    | `vector<int> v;`               |
| 填充构造           | 创建长度为 n，每个元素都初始化为 val 的 vector | `vector<int> v(26, 0);`        |
| 拷贝构造           | 用另一个 vector 初始化新 vector        | `vector<int> v1(v2);`          |
| 初始化列表构造（C++11） | 用花括号直接指定初始元素                   | `vector<int> v = {1,2,3,4,5};` |

---

## 三、元素访问（获取 / 修改 vector 中的元素）

|方法|说明|注意事项|代码示例|
|---|---|---|---|
|`operator[]`|用下标访问元素|**不检查越界**，越界会导致程序崩溃（未定义行为）|`v[0] = 10;`|
|`at()`|用下标访问元素|**会检查越界**，越界抛出 `std::out_of_range` 异常，更安全|`v.at(0) = 10;`|
|`front()`|获取第一个元素|空 vector 调用会崩溃|`int first = v.front();`|
|`back()`|获取最后一个元素|空 vector 调用会崩溃|`int last = v.back();`|

---

## 四、迭代器（遍历 vector 的通用方式）

迭代器是 STL 容器的 “通用指针”，用于遍历和操作容器元素。

|迭代器方法|说明|
|---|---|
|`begin()`|返回指向第一个元素的迭代器|
|`end()`|返回指向 “最后一个元素的下一位” 的迭代器（标记结束，不代表有效元素）|
|`rbegin()`|返回反向迭代器，指向最后一个元素|
|`rend()`|返回反向迭代器，指向第一个元素的前一位|

### 遍历示例

```cpp
vector<int> v = {1,2,3,4,5};

// 1. 范围for循环（C++11，最简洁，推荐日常使用）
for (int num : v) {
    cout << num << " ";
}

// 2. 正向迭代器遍历
for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
    cout << *it << " "; // 用*解引用迭代器获取元素
}

// 3. 反向迭代器遍历（从后往前）
for (vector<int>::reverse_iterator it = v.rbegin(); it != v.rend(); ++it) {
    cout << *it << " ";
}
```

---

## 五、容量操作（管理 vector 的大小和空间）

### 核心概念区分

- **size**：vector 中**实际存储的元素个数**；
- **capacity**：vector**预分配的内存空间大小**（不重新分配内存的前提下最多能存的元素数），`capacity >= size`。

|方法|说明|代码示例|
|---|---|---|
|`size()`|返回当前元素个数|`int n = v.size();`|
|`empty()`|判断是否为空（size==0），返回 bool|`if (v.empty()) { ... }`|
|`capacity()`|返回预分配的空间大小|`int cap = v.capacity();`|
|`reserve(n)`|预分配至少 n 个元素的空间（只改 capacity，不改变 size，不初始化元素）|`v.reserve(100);`（提前扩容，避免多次重新分配，优化性能）|
|`resize(n, val)`|改变 size 为 n：<br><br>1. n > size：在尾部补充 val（val 默认 0）<br><br>2. n < size：删除尾部多余元素|`v.resize(10, 0);`|
|`shrink_to_fit()`|释放多余的预分配空间，让 capacity 等于 size|`v.shrink_to_fit();`（节省内存）|

---

## 六、修改操作（添加、删除、替换元素）

|方法|说明|代码示例|
|---|---|---|
|`push_back(val)`|在尾部添加一个元素（最常用，平均 O (1)）|`v.push_back(10);`|
|`emplace_back(args...)`|在尾部**原地构造**一个元素（C++11，比 push_back 更高效，避免临时对象拷贝）|`v.emplace_back(10);`|
|`pop_back()`|删除尾部的一个元素|`v.pop_back();`|
|`insert(pos, val)`|在迭代器 pos 指向的位置插入 val，返回新元素的迭代器|`v.insert(v.begin(), 5);`（在开头插入 5）|
|`erase(pos)`|删除迭代器 pos 指向的元素，返回下一个元素的迭代器|`v.erase(v.begin());`（删除第一个元素）|
|`erase(first, last)`|删除区间 [first, last) 内的所有元素|`v.erase(v.begin(), v.begin()+3);`（删除前 3 个元素）|
|`clear()`|清空所有元素（size 变为 0，capacity 不变）|`v.clear();`|
|`swap(v2)`|交换两个 vector 的内容（O (1)，仅交换内部指针）|`v.swap(v2);`|

---

## 七、高级常用操作（配合 STL 算法）

`vector` 可以配合 STL 算法库（`<algorithm>`）实现排序、查找等功能。

### 1. 排序

```cpp
#include <algorithm>
vector<int> v = {3,1,4,1,5};
sort(v.begin(), v.end()); // 默认升序排序，结果：[1,1,3,4,5]
sort(v.rbegin(), v.rend()); // 降序排序，结果：[5,4,3,1,1]
```

### 2. 查找

```cpp
#include <algorithm>
vector<int> v = {1,2,3,4,5};
// 查找元素3，返回指向3的迭代器，没找到返回v.end()
auto it = find(v.begin(), v.end(), 3);
if (it != v.end()) {
    cout << "找到了，下标是：" << it - v.begin() << endl;
}
```

---

## 八、核心注意事项（面试高频考点）

### 1. 迭代器失效

以下操作会导致迭代器失效，失效的迭代器不能再使用，否则程序崩溃：

- **`push_back/emplace_back`**：如果触发了重新分配（size 超过 capacity），所有迭代器失效；
- **`insert`**：插入位置之后的所有迭代器失效；
- **`erase`**：删除位置之后的所有迭代器失效。

### 2. 性能优化

- 提前用 `reserve(n)` 预分配空间，避免多次重新分配内存（重新分配需要拷贝所有元素，开销大）；
- 尽量用 `emplace_back` 代替 `push_back`，减少临时对象的拷贝。

### 3. 内存管理

- `vector` 自动管理内存，不要手动 `delete` 容器内的元素（除非存的是指针，需要手动释放指针指向的内存）；
- `clear()` 只清空元素，不释放预分配的空间，想释放空间用 `shrink_to_fit()`。