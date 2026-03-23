## 一、概述

`begin()` 是 **C++ 标准库容器（Containers）和数组** 的核心成员函数（或全局函数），作用是**返回一个指向容器「第一个元素」的迭代器（Iterator）**，是迭代器体系的 “起点标记”，广泛用于配合标准库算法（如 `sort`、`find`）、遍历容器等场景。

### 核心定位

- **迭代器起点**：标记容器元素范围的起始位置；
- **通用接口**：所有标准库容器（`vector`、`string`、`list`、`unordered_map` 等）都支持 `begin()`；
- **左闭右开区间的基础**：与 `end()` 配合，构成标准库通用的元素范围 `[begin(), end())`（左闭右开）。

---

## 二、语法与参数

### 1. 成员函数形式（最常用，适用于容器）

```cpp
container.begin();
```

- **参数**：无任何参数；
- **返回值**：指向容器第一个元素的迭代器（迭代器类型取决于容器，如 `std::string::iterator`、`std::vector<int>::iterator`）。

### 2. 全局函数形式（适用于数组和容器，C++11+）

cpp

运行

```
std::begin(container_or_array);
```

- 既可以接受容器，也可以接受 C 风格数组，通用性更强。

---

## 三、返回值详解：迭代器类型

`begin()` 的返回值是**迭代器**，可简单理解为 “指向元素的指针”（但功能更通用），分为两种版本：

表格

|版本|调用方式|返回值类型|说明|
|---|---|---|---|
|非 `const` 版本|`container.begin()`|`iterator`|可通过该迭代器**修改**指向的元素|
|`const` 版本|`container.cbegin()`（C++11+）|`const_iterator`|仅可**读取**指向的元素，不可修改（更安全，只读场景推荐）|

### 示例：两种版本的区别

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {10, 20, 30};

    // 1. 非 const 版本：可修改元素
    auto it = vec.begin();
    *it = 100; // 合法：将第一个元素改为 100
    std::cout << vec[0] << std::endl; // 输出：100

    // 2. const 版本：仅可读，不可修改
    auto cit = vec.cbegin();
    // *cit = 200; // 编译错误！const_iterator 不可修改元素
    std::cout << *cit << std::endl; // 合法：读取第一个元素，输出：100

    return 0;
}
```

---

## 四、常见容器的 `begin()` 示例

### 1. `std::string`（你之前代码中的场景）

cpp

运行

```
#include <string>
#include <iostream>

int main() {
    std::string key = "eat";
    
    // begin() 返回指向第一个字符 'e' 的迭代器
    auto it = key.begin();
    std::cout << *it << std::endl; // 输出：e

    // 通过迭代器遍历字符串
    for (auto i = key.begin(); i != key.end(); ++i) {
        std::cout << *i << " ";
    }
    // 输出：e a t
    return 0;
}
```

---

### 2. `std::vector`

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};

    // begin() 返回指向第一个元素 1 的迭代器
    auto it = vec.begin();
    std::cout << *it << std::endl; // 输出：1

    // 修改第一个元素
    *it = 100;
    std::cout << vec[0] << std::endl; // 输出：100

    return 0;
}
```

---

### 3. C 风格数组（全局 `std::begin`）

cpp

运行

```
#include <iostream>
#include <iterator> // std::begin 头文件

int main() {
    int arr[] = {10, 20, 30};

    // 全局 std::begin 用于数组
    auto it = std::begin(arr);
    std::cout << *it << std::endl; // 输出：10

    return 0;
}
```

---

## 五、与 `end()` 的配合：左闭右开区间

`begin()` 必须和 `end()` 成对使用，共同构成标准库通用的元素范围：

- `begin()`：指向**第一个有效元素**；
- `end()`：指向**最后一个有效元素的「下一个位置」**（尾后迭代器，不指向任何有效元素）；
- 范围特性：**左闭右开** `[begin(), end())`，即包含 `begin()` 指向的元素，不包含 `end()` 指向的位置。

### 为什么是左闭右开？

1. **空容器判断简单**：如果 `begin() == end()`，说明容器为空；
2. **遍历终止条件清晰**：循环到 `it != end()` 即可，无需额外判断；
3. **标准库算法通用**：所有标准库算法（`sort`、`find`、`copy` 等）都遵循这个范围约定。

---

## 六、在 `sort(key.begin(), key.end())` 中的具体作用

回到你之前的字母异位词分组代码：

cpp

运行

```
string key = s;
sort(key.begin(), key.end());
```

这段代码中 `begin()` 和 `end()` 的作用是：

1. **给 `sort` 算法传递排序范围**：`sort` 是标准库排序算法，需要两个迭代器参数 —— 第一个是排序起点，第二个是排序终点；
2. **`key.begin()`**：指向字符串 `key` 的**第一个字符**（比如 `key = "eat"` 时，指向 `'e'`）；
3. **`key.end()`**：指向字符串 `key` 的**尾后位置**（不指向任何有效字符）；
4. **`sort` 的执行逻辑**：遍历 `[begin(), end())` 区间内的所有字符，按字典序排序。

### 完整示例（结合你的代码场景）

cpp

运行

```
#include <iostream>
#include <string>
#include <algorithm> // sort 头文件

int main() {
    std::string key = "eat";
    std::cout << "排序前：" << key << std::endl; // 输出：eat

    // sort 接收 [begin(), end()) 区间，对整个字符串排序
    std::sort(key.begin(), key.end());

    std::cout << "排序后：" << key << std::endl; // 输出：aet
    return 0;
}
```

---

## 七、注意事项与避坑指南

### 1. 空容器的 `begin()`

如果容器为空，`begin()` 返回的迭代器等于 `end()`，此时**不可解引用**（访问 `*begin()` 会导致未定义行为，程序崩溃）。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> emptyVec;
    auto it = emptyVec.begin();
    
    // 错误！空容器的 begin() 等于 end()，解引用会崩溃
    // std::cout << *it << std::endl; 

    // 正确做法：先判断是否为空
    if (it != emptyVec.end()) {
        std::cout << *it << std::endl;
    }
    return 0;
}
```

---

### 2. 迭代器失效问题

部分容器（如 `vector`、`deque`）在**修改容量**（如 `push_back` 触发扩容、`insert`/`erase` 元素）时，会导致 `begin()` 返回的旧迭代器**失效**（指向已释放的内存），继续使用会崩溃。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3};
    auto oldBegin = vec.begin(); // 保存旧的 begin()

    // 插入元素，触发扩容
    vec.push_back(4);
    vec.push_back(5);
    vec.push_back(6);

    // 错误！oldBegin 已失效，访问会崩溃
    // std::cout << *oldBegin << std::endl; 

    // 正确做法：重新获取 begin()
    auto newBegin = vec.begin();
    std::cout << *newBegin << std::endl; // 合法，输出：1
    return 0;
}
```

---

### 3. 只读场景优先用 `cbegin()`

如果不需要修改元素，推荐用 `cbegin()`（C++11+）替代 `begin()`，更安全（避免误修改），代码意图也更清晰。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3};

    // 只读遍历：用 cbegin() 和 cend()
    for (auto it = vec.cbegin(); it != vec.cend(); ++it) {
        std::cout << *it << " "; // 仅读取，不可修改
    }
    return 0;
}
```

---

## 八、总结

1. **核心功能**：`begin()` 返回指向容器第一个元素的迭代器，是容器遍历和标准库算法的基础；
2. **版本选择**：非 `const` 版本用于修改元素，`cbegin()` 用于只读场景（更安全）；
3. **范围约定**：必须与 `end()` 配合，构成左闭右开区间 `[begin(), end())`；
4. **典型应用**：配合 `sort`、`find` 等算法，或用于手动遍历容器；
5. **避坑要点**：空容器不可解引用 `begin()`，容器扩容后需重新获取迭代器。