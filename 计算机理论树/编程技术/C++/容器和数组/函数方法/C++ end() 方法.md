## 一、概述

`end()` 是 **C++ 标准库容器（Containers）和数组** 的核心成员函数（或全局函数），作用是**返回一个指向容器「最后一个有效元素的下一个位置」的迭代器**（称为 “尾后迭代器”，Past-the-end Iterator）。

它与 `begin()` 成对使用，共同构成标准库通用的**左闭右开区间 `[begin(), end())`**，是迭代器体系的 “终点标记”，广泛用于配合标准库算法（如 `sort`、`find`）、遍历容器等场景。

### 核心特点

- **不指向有效元素**：`end()` 返回的迭代器仅用于标记 “范围结束”，**绝对不能解引用**（访问 `*end()` 会导致未定义行为）；
- **通用接口**：所有标准库容器（`vector`、`string`、`list`、`unordered_map` 等）和 C 风格数组都支持 `end()`；
- **左闭右开的关键**：与 `begin()` 配合，确保范围判断简单、遍历逻辑清晰。

---

## 二、语法与参数

### 1. 成员函数形式（最常用，适用于容器）

cpp

运行

```
container.end();
```

- **参数**：无任何参数；
- **返回值**：指向容器 “尾后位置” 的迭代器（迭代器类型取决于容器，如 `std::string::iterator`、`std::vector<int>::iterator`）。

### 2. 全局函数形式（适用于数组和容器，C++11+）

cpp

运行

```
std::end(container_or_array);
```

- 既可以接受容器，也可以接受 C 风格数组，通用性更强。

---

## 三、返回值详解：尾后迭代器

`end()` 的返回值是**尾后迭代器**，这是理解 `end()` 的核心：

- **位置含义**：指向容器中 “最后一个有效元素的下一个内存位置”，不存储任何有效数据；
- **作用**：仅作为 “范围结束的哨兵”，用于判断迭代是否到达终点；
- **绝对禁止**：**永远不要解引用 `end()` 返回的迭代器**（即不要写 `*container.end()`），否则会导致未定义行为（程序崩溃、数据损坏等）。

### 迭代器类型（与 `begin()` 对应）

表格

|版本|调用方式|返回值类型|说明|
|---|---|---|---|
|非 `const` 版本|`container.end()`|`iterator`|仅用于范围判断，不可解引用|
|`const` 版本|`container.cend()`（C++11+）|`const_iterator`|只读场景的尾后迭代器，更安全（推荐用于只读遍历）|

---

## 四、常见容器的 `end()` 示例

### 1. `std::string`（结合你之前的 `sort` 场景）

cpp

运行

```
#include <string>
#include <iostream>

int main() {
    std::string key = "eat";
    
    // end() 返回指向 't' 下一个位置的尾后迭代器
    auto it_end = key.end();
    
    // 错误！绝对不能解引用 end()
    // std::cout << *it_end << std::endl; 

    // 正确用法：与 begin() 配合遍历
    for (auto it = key.begin(); it != key.end(); ++it) {
        std::cout << *it << " ";
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

    // end() 返回指向 5 下一个位置的尾后迭代器
    auto it_end = vec.end();

    // 正确用法：判断迭代是否结束
    for (auto it = vec.begin(); it != it_end; ++it) {
        std::cout << *it << " ";
    }
    // 输出：1 2 3 4 5
    return 0;
}
```

---

### 3. C 风格数组（全局 `std::end`）

cpp

运行

```
#include <iostream>
#include <iterator> // std::end 头文件

int main() {
    int arr[] = {10, 20, 30};

    // 全局 std::end 用于数组，返回指向 30 下一个位置的指针
    int* p_end = std::end(arr);

    // 正确用法：遍历数组
    for (int* p = std::begin(arr); p != p_end; ++p) {
        std::cout << *p << " ";
    }
    // 输出：10 20 30
    return 0;
}
```

---

## 五、与 `begin()` 配合：左闭右开区间 `[begin(), end())`

`end()` 的核心价值是与 `begin()` 配合，构成标准库通用的**左闭右开区间**，这是 C++ 迭代器体系的基础设计。

### 左闭右开的含义

- **左闭**：包含 `begin()` 指向的第一个有效元素；
- **右开**：不包含 `end()` 指向的尾后位置；
- **数学表示**：`[begin(), end())`。

### 为什么设计成左闭右开？（核心优势）

1. **空容器判断简单**：如果 `begin() == end()`，说明容器中没有任何元素；
    
    cpp
    
    运行
    
    ```
    std::vector<int> emptyVec;
    if (emptyVec.begin() == emptyVec.end()) {
        std::cout << "容器为空" << std::endl;
    }
    ```
    
2. **遍历终止条件清晰**：循环只需判断 `it != end()`，无需额外计算元素数量；
    
    cpp
    
    运行
    
    ```
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        // 自动遍历所有元素，到达尾后位置时停止
    }
    ```
    
3. **标准库算法通用**：所有标准库算法（`sort`、`find`、`copy`、`reverse` 等）都严格遵循 `[begin(), end())` 的范围约定。

---

## 六、在 `sort(key.begin(), key.end())` 中的具体作用

回到你之前的字母异位词分组代码：

cpp

运行

```
string key = s;
sort(key.begin(), key.end());
```

这段代码中 `end()` 的具体作用是：

1. **给 `sort` 算法传递排序的「终点」**：`sort` 是标准库排序算法，需要两个迭代器参数 ——`begin()` 是排序起点，`end()` 是排序终点；
2. **明确排序范围**：`sort` 会遍历 `[begin(), end())` 区间内的所有字符（即从第一个字符到最后一个有效字符，不包含尾后位置），按字典序排序；
3. **确保算法安全**：`end()` 作为尾后标记，让 `sort` 知道 “在哪里停止”，避免越界访问。

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

    // sort 接收 [begin(), end()) 区间：
    // - begin() 指向 'e'（第一个有效字符）
    // - end() 指向 't' 的下一个位置（尾后标记）
    // sort 会处理 'e'、'a'、't' 三个字符，不包含 end() 位置
    std::sort(key.begin(), key.end());

    std::cout << "排序后：" << key << std::endl; // 输出：aet
    return 0;
}
```

---

## 七、注意事项与避坑指南

### 1. 绝对禁止解引用 `end()`（最常见错误）

`end()` 返回的是尾后迭代器，不指向任何有效元素，解引用会导致**未定义行为**（程序崩溃、数据损坏、随机输出等）。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3};
    auto it_end = vec.end();

    // 错误！绝对不要解引用 end()
    // std::cout << *it_end << std::endl; 

    return 0;
}
```

---

### 2. 空容器时 `begin() == end()`

如果容器为空，`begin()` 和 `end()` 返回的迭代器相等，此时既不能解引用 `begin()`，也不能解引用 `end()`。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> emptyVec;

    // 正确判断空容器
    if (emptyVec.begin() == emptyVec.end()) {
        std::cout << "容器为空，无需遍历" << std::endl;
    }

    return 0;
}
```

---

### 3. 迭代器失效问题

部分容器（如 `vector`、`deque`）在**修改容量**（如 `push_back` 触发扩容、`insert`/`erase` 元素）时，会导致 `end()` 返回的旧迭代器**失效**（指向已释放的内存），需重新获取。

cpp

运行

```
#include <vector>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3};
    auto oldEnd = vec.end(); // 保存旧的 end()

    // 插入元素，触发扩容
    vec.push_back(4);
    vec.push_back(5);
    vec.push_back(6);

    // 错误！oldEnd 已失效，不能用于判断
    // for (auto it = vec.begin(); it != oldEnd; ++it) { ... }

    // 正确做法：重新获取 end()
    auto newEnd = vec.end();
    for (auto it = vec.begin(); it != newEnd; ++it) {
        std::cout << *it << " ";
    }
    // 输出：1 2 3 4 5 6
    return 0;
}
```

---

### 4. 只读场景优先用 `cend()`

如果不需要修改元素，推荐用 `cend()`（C++11+）替代 `end()`，配合 `cbegin()` 使用，更安全（避免误修改），代码意图也更清晰。

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

1. **核心功能**：`end()` 返回指向容器 “尾后位置” 的迭代器，仅用于标记范围结束，**绝对不能解引用**；
2. **左闭右开区间**：与 `begin()` 配合构成 `[begin(), end())`，是标准库算法和遍历的基础；
3. **版本选择**：非 `const` 版本用于一般场景，`cend()` 用于只读场景（更安全）；
4. **典型应用**：配合 `sort`、`find` 等算法，或用于手动遍历容器的终止判断；
5. **避坑要点**：禁止解引用 `end()`、空容器时 `begin() == end()`、容器扩容后需重新获取 `end()`。