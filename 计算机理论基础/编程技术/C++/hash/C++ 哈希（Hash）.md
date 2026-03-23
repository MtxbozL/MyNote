## 一、概述

哈希表（Hash Table）是一种通过 “键（Key）” 直接访问 “值（Value）” 的数据结构，核心优势是**平均 O (1) 的查找、插入、删除速度**，远快于线性表和树结构。

C++11 及以后标准引入了**无序关联容器**（Unordered Associative Containers），底层基于哈希表实现，主要包括：

- `unordered_map`：键值对映射（Key-Value），键唯一
- `unordered_set`：单值集合，值唯一
- `unordered_multimap`：键值对映射，键可重复
- `unordered_multiset`：单值集合，值可重复

### 与有序容器（`map`/`set`）的对比

|特性|无序容器（`unordered_*`）|有序容器（`map`/`set`）|
|---|---|---|
|底层实现|哈希表|红黑树|
|查找速度|快（平均 O (1)）|较慢（O (logn)）|
|元素顺序|无序|按键升序排列|
|内存占用|稍多|稍少|
|适用场景|追求极致性能，无需有序|需要有序遍历，对稳定性要求高|

---

## 二、核心组件

哈希表的高效运行依赖三个核心组件：

1. **哈希函数（Hash Function）**：将键（Key）映射为一个整数（哈希值），用于确定元素在 “桶（Bucket）” 中的位置
2. **相等性谓词（Equality Predicate）**：判断两个键是否相等，用于解决哈希冲突
3. **桶（Bucket）**：哈希表的存储单元，每个桶存储一个或多个元素（冲突时用链表 / 红黑树存储）

---

## 三、标准库无序容器详解

### 1. `unordered_map`（键值对映射，最常用）

#### 语法定义

```cpp
#include <unordered_map>
#include <string>

// 基本定义：Key 为 string，Value 为 int
std::unordered_map<std::string, int> umap;
```

#### 常用操作

| 操作                          | 说明                                   |
| --------------------------- | ------------------------------------ |
| `umap[key] = value`         | 插入 / 更新键值对（若 key 不存在则插入，存在则覆盖 value） |
| `umap.insert({key, value})` | 插入键值对（若 key 已存在则不覆盖，返回插入是否成功）        |
| `umap.find(key)`            | 查找 key，返回迭代器；若未找到返回 `umap.end()`     |
| `umap.count(key)`           | 返回 key 出现的次数（0 或 1，因为键唯一）            |
| `umap.erase(key)`           | 删除 key 对应的键值对                        |
| `umap.size()`               | 返回键值对数量                              |
| `umap.empty()`              | 判断是否为空                               |
| `umap.clear()`              | 清空所有元素                               |

#### 遍历方式

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

int main() {
    std::unordered_map<std::string, int> umap = {
        {"apple", 5},
        {"banana", 3},
        {"orange", 7}
    };

    // 方式1：范围 for 循环（推荐，C++11+）
    for (const auto& pair : umap) {
        std::cout << pair.first << ": " << pair.second << std::endl;
    }

    // 方式2：迭代器遍历
    for (auto it = umap.begin(); it != umap.end(); ++it) {
        std::cout << it->first << ": " << it->second << std::endl;
    }

    return 0;
}
```

---

### 2. `unordered_set`（单值集合）

#### 语法定义

```cpp
#include <unordered_set>
#include <string>

// 基本定义：存储 string 类型的唯一值
std::unordered_set<std::string> uset;
```

#### 常用操作

| 操作                   | 说明                                 |
| -------------------- | ---------------------------------- |
| `uset.insert(value)` | 插入值（若 value 已存在则不插入）               |
| `uset.find(value)`   | 查找 value，返回迭代器；若未找到返回 `uset.end()` |
| `uset.count(value)`  | 返回 value 出现的次数（0 或 1，因为值唯一）        |
| `uset.erase(value)`  | 删除值                                |
| `uset.size()`        | 返回值的数量                             |
| `uset.empty()`       | 判断是否为空                             |
| `uset.clear()`       | 清空所有元素                             |

#### 典型场景：去重

```cpp
#include <iostream>
#include <unordered_set>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 2, 1, 4, 5};
    std::unordered_set<int> uset(nums.begin(), nums.end());

    // 输出去重后的结果：1 2 3 4 5（顺序无序）
    for (int num : uset) {
        std::cout << num << " ";
    }
    return 0;
}
```

---

### 3. `unordered_multimap`/`unordered_multiset`（允许重复）

- `unordered_multimap`：键可重复的键值对映射
- `unordered_multiset`：值可重复的单值集合

#### 示例：`unordered_multimap` 存储多对一关系

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

int main() {
    std::unordered_multimap<std::string, int> ummap;
    ummap.insert({"apple", 5});
    ummap.insert({"apple", 10}); // 键 "apple" 可重复
    ummap.insert({"banana", 3});

    // 查找键 "apple" 对应的所有值
    auto range = ummap.equal_range("apple");
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << it->first << ": " << it->second << std::endl;
    }
    return 0;
}
```

---

## 四、哈希函数与相等性谓词

### 1. 内置哈希函数：`std::hash`

C++ 标准库为**内置类型**（如 `int`、`double`、`std::string`、指针等）提供了默认的哈希函数 `std::hash`，定义在 `<functional>` 头文件中。

#### 示例：使用 `std::hash` 计算哈希值

```cpp
#include <iostream>
#include <functional> // std::hash 头文件
#include <string>

int main() {
    std::hash<int> hash_int;
    std::hash<std::string> hash_str;

    std::cout << "Hash of 42: " << hash_int(42) << std::endl;
    std::cout << "Hash of 'hello': " << hash_str("hello") << std::endl;
    return 0;
}
```

---

### 2. 自定义类型的哈希支持

标准库**没有为自定义类型（如自定义类 / 结构体）提供默认哈希函数**，需要手动实现，有两种常用方法：

#### 方法 1：特化 `std::hash`（推荐，与标准库风格一致）

假设我们有一个自定义结构体 `Person`，包含 `name` 和 `age` 两个字段：

```cpp
#include <unordered_map>
#include <string>
#include <functional>

// 自定义结构体
struct Person {
    std::string name;
    int age;

    // 必须重载 == 运算符（相等性谓词默认使用 std::equal_to）
    bool operator==(const Person& other) const {
        return name == other.name && age == other.age;
    }
};

// 特化 std::hash<Person>
namespace std {
    template<>
    struct hash<Person> {
        size_t operator()(const Person& p) const {
            // 组合多个字段的哈希值（常用方法：异或 + 位移）
            size_t h1 = hash<std::string>()(p.name);
            size_t h2 = hash<int>()(p.age);
            return h1 ^ (h2 << 1);
        }
    };
}

// 使用示例
int main() {
    std::unordered_map<Person, std::string> umap;
    Person p1 = {"Alice", 25};
    umap[p1] = "Engineer";
    return 0;
}
```

#### 方法 2：自定义哈希函数对象

如果不想修改 `std` 命名空间，可以定义独立的哈希函数对象：

```cpp
#include <unordered_map>
#include <string>

struct Person {
    std::string name;
    int age;
    bool operator==(const Person& other) const {
        return name == other.name && age == other.age;
    }
};

// 自定义哈希函数对象
struct PersonHash {
    size_t operator()(const Person& p) const {
        size_t h1 = std::hash<std::string>()(p.name);
        size_t h2 = std::hash<int>()(p.age);
        return h1 ^ (h2 << 1);
    }
};

// 使用时需在模板参数中指定哈希函数和相等性谓词
int main() {
    std::unordered_map<Person, std::string, PersonHash> umap;
    Person p1 = {"Bob", 30};
    umap[p1] = "Designer";
    return 0;
}
```

---

### 3. 相等性谓词

默认的相等性谓词是 `std::equal_to`，它会调用键的 `==` 运算符，因此自定义类型**必须重载 `==`**（除非自定义相等性谓词）。

自定义相等性谓词的示例（与自定义哈希函数配合使用）：

```cpp
struct PersonEqual {
    bool operator()(const Person& a, const Person& b) const {
        return a.name == b.name && a.age == b.age;
    }
};

// 使用时需同时指定哈希函数和相等性谓词
std::unordered_map<Person, std::string, PersonHash, PersonEqual> umap;
```

---

## 五、性能优化与高级操作

### 1. 负载因子（Load Factor）与 Rehash

- **负载因子**：`load_factor = 元素数量 / 桶数量`，衡量哈希表的拥挤程度
- **最大负载因子**：`max_load_factor()`，默认值为 1.0，超过该值会自动触发 `rehash`（重新分配桶数量，减少冲突）

#### 手动调整桶数量

```cpp
#include <unordered_map>
#include <string>

int main() {
    std::unordered_map<std::string, int> umap;
    
    // 查看当前桶数量和负载因子
    std::cout << "Bucket count: " << umap.bucket_count() << std::endl;
    std::cout << "Load factor: " << umap.load_factor() << std::endl;

    // 手动设置最大负载因子（降低可减少冲突，但增加内存）
    umap.max_load_factor(0.7);

    // 手动 rehash，预留至少 100 个桶（提前分配空间，避免频繁 rehash）
    umap.rehash(100);

    // 更推荐的方式：reserve 预留元素空间（自动计算桶数量）
    umap.reserve(1000); // 预计存储 1000 个元素，提前分配

    return 0;
}
```

---

### 2. 性能优化建议

1. **提前预留空间**：如果已知元素数量，用 `reserve()` 提前分配桶，避免频繁 `rehash`（性能开销大）
2. **选择合适的哈希函数**：自定义类型的哈希函数要尽量 “分散”，减少哈希冲突
3. **避免过深的嵌套**：哈希冲突过多时，桶内的链表 / 红黑树会变长，性能退化为 O (n)
4. **优先使用 `find()` 而非 `count()`**：`find()` 找到后可直接获取值，`count()` 仅判断存在性，若后续需要值还需再次查找

---

## 六、常见陷阱与避坑指南

### 1. 自定义类型未提供哈希函数或相等谓词

**错误示例**：

```cpp
struct Person { std::string name; int age; };
std::unordered_map<Person, std::string> umap; // 编译错误！
```

**解决**：按 “四、2” 节的方法实现哈希函数和 `==` 运算符。

---

### 2. 哈希冲突导致性能急剧下降

**现象**：插入 / 查找速度变慢，从 O (1) 退化为 O (n)

**原因**：哈希函数设计不合理，大量键映射到同一个桶

**解决**：优化哈希函数，组合多个字段的哈希值（如异或、位移、乘法）

---

### 3. 遍历过程中修改容器大小

**错误示例**：

```cpp
std::unordered_map<std::string, int> umap = {{"a", 1}, {"b", 2}};
for (auto& pair : umap) {
    umap.erase(pair.first); // 错误！遍历中删除会导致迭代器失效
}
```

**解决**：使用迭代器的返回值安全删除：

```cpp
auto it = umap.begin();
while (it != umap.end()) {
    if (it->second == 1) {
        it = umap.erase(it); // erase 返回下一个有效迭代器
    } else {
        ++it;
    }
}
```

---

### 4. 线程安全问题

C++ 标准库的无序容器**不是线程安全的**：

- 多线程同时读取是安全的
- 多线程同时写入或读写混合会导致数据竞争
    
    **解决**：使用互斥锁（`std::mutex`）保护容器访问。

---

## 七、综合练习

### 练习 1：用 `unordered_map` 实现两数之和

```cpp
#include <vector>
#include <unordered_map>

class Solution {
public:
    std::vector<int> twoSum(std::vector<int>& nums, int target) {
        std::unordered_map<int, int> hashMap;
        for (int i = 0; i < nums.size(); ++i) {
            int complement = target - nums[i];
            if (hashMap.find(complement) != hashMap.end()) {
                return {hashMap[complement], i};
            }
            hashMap[nums[i]] = i;
        }
        return {};
    }
};
```

---

### 练习 2：用 `unordered_set` 实现数组去重

```cpp
#include <vector>
#include <unordered_set>

std::vector<int> removeDuplicates(std::vector<int>& nums) {
    std::unordered_set<int> uset(nums.begin(), nums.end());
    return std::vector<int>(uset.begin(), uset.end());
}
```

---

### 练习 3：自定义类型的 `unordered_map` 使用

```cpp
#include <unordered_map>
#include <string>
#include <functional>
#include <iostream>

struct Student {
    std::string id;
    std::string name;

    bool operator==(const Student& other) const {
        return id == other.id;
    }
};

namespace std {
    template<>
    struct hash<Student> {
        size_t operator()(const Student& s) const {
            return hash<std::string>()(s.id);
        }
    };
}

int main() {
    std::unordered_map<Student, int> scoreMap;
    Student s1 = {"2024001", "Alice"};
    scoreMap[s1] = 95;

    std::cout << "Alice's score: " << scoreMap[s1] << std::endl;
    return 0;
}
```