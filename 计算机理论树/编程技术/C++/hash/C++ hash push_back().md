## 一、概述

`push_back()` 是 **C++ 顺序容器（Sequence Containers）** 的核心成员函数，作用是**在容器的「尾部」直接添加一个新元素**，是最常用的容器插入操作之一。

### 核心特点

- **插入位置固定**：仅在容器末尾插入，无需指定位置；
- **均摊时间复杂度低**：对于 `vector`、`deque` 等容器，平均时间复杂度为 **O(1)**（均摊分析）；
- **使用场景广泛**：适用于 “动态收集数据”“按顺序追加元素” 的场景（如读取输入、构建数组等）。

---

## 二、支持 `push_back()` 的容器

`push_back()` 是**顺序容器专属方法**，关联容器（如 `unordered_map`、`set`）和无序容器不支持。具体支持列表：

表格

| 容器类型                  | 是否支持 `push_back()` | 说明                            |
| --------------------- | ------------------ | ----------------------------- |
| `std::vector`         | ✅ 支持               | 最常用，尾部插入效率高（扩容时除外）            |
| `std::string`         | ✅ 支持               | 专门用于在字符串末尾**添加单个字符**          |
| `std::deque`          | ✅ 支持               | 双端队列，尾部插入效率稳定（无连续内存扩容开销）      |
| `std::list`           | ✅ 支持               | 双向链表，尾部插入效率极高（O (1)，无扩容）      |
| `std::forward_list`   | ❌ 不支持              | 单向链表，仅支持 `push_front()`（头部插入） |
| `unordered_map`/`map` | ❌ 不支持              | 关联容器，通过键值对存储，无 “尾部” 概念        |

---

## 三、语法与参数

### 1. 基本语法

```cpp
container.push_back(value);
```

### 2. 参数说明

- `container`：要操作的顺序容器对象（如 `vector<int> vec`）；
- `value`：要插入的元素，类型必须与容器的 `value_type` 完全一致（或可隐式转换）。

### 3. C++11 扩展：支持移动语义

C++11 起，`push_back()` 新增**右值引用重载**，支持通过 `std::move()` 转移元素所有权，避免拷贝开销：

```cpp
container.push_back(std::move(value));
```

---

## 四、底层原理（以 `std::vector` 为例，核心重点）

`vector` 的内存是**连续存储**的，内部维护两个关键指标：

- `size`：当前容器中实际存储的元素个数；
- `capacity`：容器在不重新分配内存的情况下，最多能存储的元素个数。

`push_back()` 的执行逻辑分两种情况：

### 情况 1：容量足够（`size < capacity`）

直接在容器**尾部的空闲内存中构造新元素**，然后将 `size` 加 1。

- 时间复杂度：O (1)（极快）。

### 情况 2：容量不足（`size == capacity`）

触发**扩容（Reallocation）**，步骤如下：

1. **分配新内存**：申请一块更大的连续内存（通常是原容量的 **1.5 倍或 2 倍**，取决于编译器实现，如 GCC 用 2 倍，MSVC 用 1.5 倍）；
2. **迁移元素**：将旧内存中的元素 ** 拷贝（或移动，C++11+）** 到新内存；
3. **释放旧内存**：销毁旧内存中的元素并释放原内存空间；
4. **插入新元素**：在新内存的尾部构造新元素，更新 `size` 和 `capacity`。

- 时间复杂度：O (n)（扩容时较慢，但均摊到所有 `push_back()` 操作，平均仍为 O (1)）。

---

## 五、使用示例

### 示例 1：`vector` 基本使用（最常用）

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec;

    // 1. 插入普通值
    vec.push_back(10);
    vec.push_back(20);
    vec.push_back(30);

    // 输出：10 20 30
    for (int num : vec) {
        std::cout << num << " ";
    }
    return 0;
}
```

---

### 示例 2：`string` 插入单个字符

```cpp
#include <string>
#include <iostream>

int main() {
    std::string str = "hello";
    
    // push_back 仅用于添加单个字符
    str.push_back(' ');
    str.push_back('w');
    str.push_back('o');
    str.push_back('r');
    str.push_back('l');
    str.push_back('d');

    std::cout << str << std::endl; // 输出：hello world
    return 0;
}
```

---

### 示例 3：插入自定义类型

```cpp
#include <vector>
#include <string>
#include <iostream>

// 自定义结构体
struct Person {
    std::string name;
    int age;

    // 构造函数
    Person(std::string n, int a) : name(std::move(n)), age(a) {}
};

int main() {
    std::vector<Person> people;

    // 插入临时对象（自动调用构造函数）
    people.push_back(Person("Alice", 25));
    people.push_back(Person("Bob", 30));

    // 输出
    for (const auto& p : people) {
        std::cout << p.name << ": " << p.age << std::endl;
    }
    return 0;
}
```

---

### 示例 4：C++11 移动语义（避免拷贝，优化性能）

```cpp
#include <vector>
#include <string>
#include <iostream>

int main() {
    std::vector<std::string> vec;
    std::string longStr = "这是一个很长的字符串，拷贝开销很大";

    // 1. 传统方式：拷贝 longStr 到容器中（有拷贝开销）
    vec.push_back(longStr);
    std::cout << "拷贝后 longStr 的内容：" << longStr << std::endl; // longStr 仍有内容

    // 2. 移动语义：用 std::move() 转移所有权（无拷贝，极快）
    vec.push_back(std::move(longStr));
    std::cout << "移动后 longStr 的内容：" << longStr << std::endl; // longStr 内容已被移走（为空或未定义）

    return 0;
}
```

---

## 六、注意事项与避坑指南

### 1. 迭代器 / 引用 / 指针失效（`vector`/`deque` 重点）

当 `vector` 或 `deque` 触发**扩容**时，容器内的**所有迭代器、引用、指针都会失效**（指向旧内存，已被释放），继续使用会导致程序崩溃（未定义行为）。

#### 错误示例

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3};
    int& ref = vec[0]; // 引用第一个元素
    auto it = vec.begin(); // 迭代器指向第一个元素

    // 插入多个元素，触发扩容
    vec.push_back(4);
    vec.push_back(5);
    vec.push_back(6);

    // 错误！ref 和 it 已失效，访问会崩溃
    // std::cout << *it << std::endl; 
    // std::cout << ref << std::endl;

    return 0;
}
```

#### 正确做法

扩容后**重新获取迭代器 / 引用**，或提前用 `reserve()` 预留空间（避免扩容）。

---

### 2. 性能优化：提前用 `reserve()` 预留空间

如果已知元素的大致数量，提前调用 `reserve(n)` 预留至少 `n` 个元素的容量，**避免频繁扩容**（扩容的拷贝开销很大）。

#### 优化示例

```cpp
#include <vector>

int main() {
    std::vector<int> vec;

    // 优化前：可能触发多次扩容（比如从 0→1→2→4→8→16...）
    // for (int i = 0; i < 1000; i++) {
    //     vec.push_back(i);
    // }

    // 优化后：提前预留 1000 个元素的空间，仅需 0 次扩容
    vec.reserve(1000);
    for (int i = 0; i < 1000; i++) {
        vec.push_back(i);
    }

    return 0;
}
```

---

### 3. `push_back()` vs `emplace_back()`（C++11+ 重点对比）

C++11 引入了 `emplace_back()`，功能与 `push_back()` 类似，但**更高效**，推荐优先使用。

| 特性        | `push_back()`        | `emplace_back()`       |
| --------- | -------------------- | ---------------------- |
| 插入方式      | 先构造临时对象，再拷贝 / 移动到容器中 | **直接在容器内存中构造对象**，无临时对象 |
| 性能（自定义类型） | 有拷贝 / 移动开销           | 无额外开销，最优               |
| 语法灵活性     | 仅支持传入已构造的对象          | 支持传入构造函数参数，直接在容器内构造    |

#### 对比示例

```cpp
#include <vector>
#include <string>

struct Person {
    std::string name;
    int age;
    Person(std::string n, int a) : name(std::move(n)), age(a) {}
};

int main() {
    std::vector<Person> people;

    // 1. push_back：需先构造临时对象 Person("Alice", 25)，再拷贝/移动到容器
    people.push_back(Person("Alice", 25));

    // 2. emplace_back：直接在容器内存中构造 Person，无需临时对象（更高效）
    people.emplace_back("Bob", 30); // 直接传构造函数参数即可

    return 0;
}
```

---

## 七、总结

1. **核心功能**：`push_back()` 是顺序容器的尾部插入方法，适用于 `vector`、`string`、`deque`、`list`；
2. **性能特点**：平均 O (1)，但 `vector` 扩容时会退化为 O (n)，需用 `reserve()` 优化；
3. **避坑要点**：扩容会导致迭代器失效，避免扩容后使用旧迭代器；
4. **现代 C++ 推荐**：优先用 `emplace_back()` 替代 `push_back()`，性能更优。