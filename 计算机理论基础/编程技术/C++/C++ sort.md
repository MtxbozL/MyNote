### 1. 基础概述

| 项目    | 核心说明                                                            |
| :---- | :-------------------------------------------------------------- |
| 必备头文件 | `#include <algorithm>`（新手高频遗漏点）                                 |
| 支持标准  | C++98 及以上所有标准                                                   |
| 核心特性  | 原地排序、默认升序、**不稳定排序**、高性能                                         |
| 时间复杂度 | C++11 及以后：最坏 / 平均均为 `O(n log n)`；C++98：平均`O(n log n)`，最坏`O(n²)` |
| 底层实现  | 主流编译器均采用**内省排序（Introsort）**，融合快速排序、堆排序、插入排序，兼顾效率与最坏时间复杂度        |

---

### 2. 完整函数原型

`std::sort` 有两个核心重载，均无返回值：

#### 重载 1：默认升序排序（使用 `<` 运算符比较）

```cpp
template<class RandomIt>
void sort(RandomIt first, RandomIt last);
```

#### 重载 2：自定义排序规则

```cpp
template<class RandomIt, class Compare>
void sort(RandomIt first, RandomIt last, Compare comp);
```

#### 参数详解

1. **`first` / `last`**：随机访问迭代器，定义排序的**左闭右开区间 `[first, last)`**
    
    - 左闭右开：`last` 指向最后一个元素的下一个位置，`sort(vec.begin(), vec.end())` 才是对整个容器排序
    - 迭代器要求：必须是**随机访问迭代器**，仅支持 `vector`/`array`/`deque`/ 原生数组；**不支持 `list`/`forward_list`**（需使用自带的 `sort` 成员函数），`set`/`map` 本身有序无需排序
    
2. **`comp`**：二元谓词（可调用对象），返回 `bool` 值，自定义排序规则
    
    - 规则：`comp(a, b)` 返回 `true` 时，排序后 `a` 会排在 `b` 前面
    - 必须满足**严格弱序**，否则会触发未定义行为（程序崩溃、排序异常）
    

---

### 3. 基础用法示例

#### 示例 1：原生数组排序

```cpp
#include <algorithm>
#include <iostream>
using namespace std;

int main() {
    int arr[] = {3, 1, 4, 2, 5};
    int n = sizeof(arr) / sizeof(arr[0]);
    sort(arr, arr + n); // 数组名是首地址，arr+n对应尾后位置
    for (int x : arr) cout << x << " "; // 输出：1 2 3 4 5
    return 0;
}
```

#### 示例 2：vector 升序 / 降序排序

```cpp
#include <algorithm>
#include <vector>
#include <iostream>
#include <functional> // 用于greater<T>
using namespace std;

int main() {
    vector<int> nums = {3, 1, 4, 2, 5};
    
    // 1. 默认升序排序
    sort(nums.begin(), nums.end());
    // 输出：1 2 3 4 5
    
    // 2. 降序排序（使用预定义的greater仿函数）
    sort(nums.begin(), nums.end(), greater<int>());
    // 输出：5 4 3 2 1
    
    // 3. 区间部分排序：只排序前3个元素
    sort(nums.begin(), nums.begin() + 3);
    return 0;
}
```

---

### 4. 自定义排序规则（核心高频用法）

支持 4 种常用的自定义方式，其中 Lambda 表达式（C++11+）最简洁通用。

#### 示例 1：自定义结构体排序（多条件排序）

```cpp
#include <algorithm>
#include <vector>
#include <string>
using namespace std;

// 自定义结构体
struct Student {
    string name;
    int score;
    int age;
};

int main() {
    vector<Student> stus = {
        {"张三", 90, 20},
        {"李四", 85, 19},
        {"王五", 90, 18}
    };

    // 方式1：Lambda表达式（推荐，C++11+）
    // 规则：先按分数降序，分数相同按年龄升序
    sort(stus.begin(), stus.end(), [](const Student& a, const Student& b) {
        if (a.score != b.score) {
            return a.score > b.score; // 分数高的在前
        }
        return a.age < b.age; // 分数相同，年龄小的在前
    });

    // 方式2：自定义全局函数
    bool cmp(const Student& a, const Student& b) {
        return a.score > b.score;
    }
    sort(stus.begin(), stus.end(), cmp);

    // 方式3：仿函数（函数对象）
    struct Cmp {
        bool operator()(const Student& a, const Student& b) {
            return a.age < b.age;
        }
    };
    sort(stus.begin(), stus.end(), Cmp());

    return 0;
}
```

---

### 5. 核心约束：严格弱序（最容易踩的坑）

自定义的 `comp` 函数必须满足**严格弱序**，否则会触发未定义行为（排序乱序、程序崩溃、死循环），核心规则：

1. **反自反性**：`comp(a, a)` 必须返回 `false`
2. **反对称性**：`comp(a, b)` 为 `true`，则 `comp(b, a)` 必须为 `false`
3. **传递性**：`comp(a,b)=true` 且 `comp(b,c)=true`，则 `comp(a,c)` 必须为 `true`
4. **等价传递性**：`a` 和 `b` 等价（`comp(a,b)` 和 `comp(b,a)` 都为 `false`），`b` 和 `c` 等价，则 `a` 和 `c` 必须等价

#### 反面典型（绝对禁止）

```cpp
// 错误！使用 <= 违反严格弱序，会导致未定义行为
sort(vec.begin(), vec.end(), [](int a, int b) {
    return a <= b; 
});

// 正确写法
sort(vec.begin(), vec.end(), [](int a, int b) {
    return a < b; 
});
```

---

### 6. 高频错误汇总

表格

|错误场景|错误原因|修复方案|
|:--|:--|:--|
|`vector<int> vec = sort(...)`|`sort` 返回 `void`，无返回值|先拷贝容器，再对副本执行 `sort`|
|编译报错：`sort` 未定义|忘记包含 `<algorithm>` 头文件|顶部添加 `#include <algorithm>`|
|排序后程序崩溃 / 乱序|比较函数违反严格弱序|严格遵循严格弱序规则，禁用 `<=`/`>=`|
|对 `list` 调用 `std::sort` 编译报错|`list` 迭代器不是随机访问迭代器|使用 `list` 自带的成员函数 `list.sort()`|
|排序区间越界|迭代器超出容器范围，比如对空容器执行 `vec.begin()+5`|先判断容器长度，保证区间 `[first, last)` 合法|
|对 const 容器排序|const 容器的迭代器是 `const_iterator`，无法修改元素|移除 const 修饰，或拷贝到非 const 容器再排序|

---

### 7. 进阶相关排序函数

表格

|函数|功能|适用场景|
|:--|:--|:--|
|`stable_sort`|稳定排序，保持相等元素的相对顺序|需要保留相等元素原有顺序的场景（如多字段排序）|
|`partial_sort`|部分排序，只保证前 k 个元素有序|只需要获取前 N 大 / 小的元素，无需全排序|
|`nth_element`|找到第 n 位的元素，左侧都小于它，右侧都大于它|找 TopK、中位数，时间复杂度 O (n)，效率远高于全排序|

---

### 8. 最佳实践与性能优化

1. **传参优化**：自定义比较函数中，参数使用 `const &` 常量引用，避免大对象拷贝开销
2. **场景选型**：无需稳定排序优先用 `sort`，比 `stable_sort` 性能更高；只需要 TopK 优先用 `partial_sort`/`nth_element`
3. **迭代器安全**：排序前保证容器不会扩容 / 缩容，避免迭代器失效
4. **空容器兼容**：`sort` 对空区间、单元素区间是安全的，无需额外判空，但要保证迭代器合法
5. **C++20 优化**：可使用 `std::ranges::sort`，支持直接传入容器，写法更简洁，安全性更高
    
    cpp
    
    运行
    
    ```
    // C++20 ranges::sort 写法
    vector<int> nums = {3,1,4,2,5};
    ranges::sort(nums); // 直接传入容器，无需写begin/end
    ranges::sort(nums, ranges::greater()); // 降序
    ```