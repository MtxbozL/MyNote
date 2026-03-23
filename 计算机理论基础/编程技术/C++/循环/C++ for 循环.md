## 一、概述

循环是程序控制流的核心，用于重复执行一段代码。C++ 提供了三种主要循环结构：`for`、`while`、`do-while`，其中 **`for` 循环是最灵活、最常用的一种**，特别适合 “已知循环次数” 或 “需要遍历容器 / 数组” 的场景。

---

## 二、传统 `for` 循环（C++98 及以前）

### 1. 语法结构

```cpp
for (初始化语句; 循环条件; 迭代语句) {
    // 循环体（要重复执行的代码）
}
```

### 2. 执行流程（核心！必须记牢）

执行顺序严格遵循以下 4 步，**每次循环都重复步骤 2-4**：

1. **执行初始化语句**：仅在循环开始前执行 **1 次**，用于声明 / 初始化循环变量（如 `int i = 0`）；
2. **判断循环条件**：如果条件为 `true`，进入循环体；如果为 `false`，直接结束循环；
3. **执行循环体**：运行大括号内的代码；
4. **执行迭代语句**：更新循环变量（如 `i++`、`i--`、`i += 2` 等），然后回到 **步骤 2** 重复。

### 3. 经典示例

#### 示例 1：遍历数组（索引循环）

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    vector<int> nums = {10, 20, 30, 40, 50};
    
    // 传统索引循环：i 从 0 到 nums.size()-1
    for (int i = 0; i < nums.size(); i++) {
        cout << "nums[" << i << "] = " << nums[i] << endl;
    }
    return 0;
}
```

#### 示例 2：计算 1 到 100 的和

```cpp
int sum = 0;
for (int i = 1; i <= 100; i++) {
    sum += i;
}
cout << "1-100的和：" << sum << endl; // 输出 5050
```

### 4. 传统 `for` 循环的灵活写法

三个部分都可以省略（但分号不能少），甚至可以写多个语句：

- **省略初始化**：如果循环变量已在外部声明

```cpp
    int i = 0;
    for (; i < 10; i++) { /* ... */ }
```

- **省略循环条件**：相当于 `true`，会变成**无限循环**（需配合 `break` 退出）

```cpp
    for (int i = 0; ; i++) { 
        if (i >= 10) break; // 手动退出
    }
```

- **省略迭代语句**：在循环体内部更新变量

```cpp
    for (int i = 0; i < 10;) { 
        /* ... */
        i++; // 在循环体里迭代
    }
```
    
- **多变量循环**：用逗号分隔多个初始化 / 迭代语句

```cpp
    for (int i = 0, j = 10; i < j; i++, j--) {
        cout << i << " " << j << endl;
    }
```
    

### 5. 新手避坑：传统 `for` 循环的常见错误

1. **循环变量作用域陷阱**：在 C++11 及以后，初始化语句中声明的变量（如 `int i`）仅在 `for` 循环内部有效，外部无法访问：

```cpp
    for (int i = 0; i < 10; i++) {}
    // cout << i; // 错误！i 已超出作用域
```
    
2. **浮点数比较的精度问题**：不要用浮点数作为循环变量（精度丢失会导致条件判断错误）：

```cpp
    // 错误示例！
    for (double x = 0.0; x != 1.0; x += 0.1) { 
        // x 可能永远不会等于 1.0（浮点数精度问题）
    }
```
    
3. **循环条件写反**：比如把 `i < 10` 写成 `i > 10`，导致循环一次都不执行。

---

## 三、范围 `for` 循环（C++11 及以后，推荐优先使用）

### 1. 核心作用

专门用于**简化容器 / 数组的遍历**，无需手动管理索引或迭代器，代码更简洁、更安全。

### 2. 语法结构

```cpp
for (元素类型 循环变量 : 要遍历的容器/数组) {
    // 循环体（直接使用循环变量）
}
```

### 3. 经典示例

#### 示例 1：遍历 `vector`（只读场景）

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    vector<string> strs = {"eat", "tea", "tan", "ate"};
    
    // 范围 for 循环：依次把 strs 里的每个字符串赋值给 s
    for (const string& s : strs) { // 加 const & 避免拷贝，更高效
        cout << s << endl;
    }
    return 0;
}
```

#### 示例 2：修改容器元素（必须加引用 `&`）

```cpp
vector<int> nums = {1, 2, 3, 4};
// 不加 &：循环变量是拷贝，修改不影响原数组
for (int num : nums) {
    num *= 2; 
}
// nums 还是 {1, 2, 3, 4}

// 加 &：循环变量是原元素的引用，修改直接生效
for (int& num : nums) {
    num *= 2;
}
// nums 变成 {2, 4, 6, 8}
```

### 4. 范围 `for` 循环的底层原理

它的本质是**迭代器的语法糖**，编译器会自动将其转换为类似以下的代码：

```cpp
// 范围 for 循环
for (int x : vec) { cout << x; }

// 编译器自动转换为（伪代码）：
for (auto it = vec.begin(); it != vec.end(); ++it) {
    int x = *it;
    cout << x;
}
```

### 5. 新手避坑：范围 `for` 循环的注意事项

1. **遍历过程中禁止修改容器大小**：比如在循环里 `push_back`、`erase`、`clear` 元素，会导致迭代器失效，程序崩溃：

```cpp
    vector<int> vec = {1, 2, 3};
    for (int x : vec) {
        vec.push_back(4); // 错误！会导致迭代器失效
    }
```
    
2. **只读场景推荐加 `const &`**：如果不需要修改元素，用 `const 类型&` 既可以避免拷贝（提高效率），又能保证只读安全（防止误修改）。
3. **支持的遍历对象**：只要是有 `begin()` 和 `end()` 成员函数的对象都能遍历，包括：
    
    - 标准容器：`vector`、`string`、`unordered_map`、`set`、`list` 等；
    - C 风格数组（普通数组）；
    - 初始化列表：`for (int x : {1, 2, 3}) { ... }`。
    

---

## 四、循环控制语句：`break` 和 `continue`

### 1. `break`：立即终止整个循环

用于**提前退出循环**，执行 `break` 后，循环体剩余代码不再执行，直接跳出循环。

#### 示例：找到第一个偶数就退出

```cpp
vector<int> nums = {1, 3, 4, 5, 6};
for (int num : nums) {
    if (num % 2 == 0) {
        cout << "找到第一个偶数：" << num << endl;
        break; // 立即终止循环，后面的 5、6 不会再遍历
    }
    cout << "当前数：" << num << endl;
}
```

### 2. `continue`：跳过当前循环，进入下一次迭代

用于**跳过本次循环的剩余代码**，直接执行下一次循环的 “迭代语句” 或 “条件判断”。

#### 示例：只打印奇数，跳过偶数

```cpp
for (int i = 1; i <= 10; i++) {
    if (i % 2 == 0) {
        continue; // 跳过本次循环，后面的 cout 不会执行
    }
    cout << i << " 是奇数" << endl;
}
```

---

## 五、嵌套 `for` 循环

### 1. 概念

在一个 `for` 循环内部再写另一个 `for` 循环，称为**嵌套循环**。外层循环执行一次，内层循环会完整执行一轮。

### 2. 经典示例：打印 9x9 乘法表

```cpp
#include <iostream>
using namespace std;

int main() {
    for (int i = 1; i <= 9; i++) { // 外层循环：控制行数
        for (int j = 1; j <= i; j++) { // 内层循环：控制每行的列数
            cout << j << "x" << i << "=" << i*j << "\t";
        }
        cout << endl; // 每行结束换行
    }
    return 0;
}
```

### 3. 新手避坑：嵌套循环的性能问题

嵌套循环的时间复杂度是 **O (外层次数 × 内层次数)**，如果层数过多或次数过大，会导致性能急剧下降（比如 3 层循环各 1000 次，就是 10 亿次操作）。

---

## 六、最佳实践总结

1. **优先使用范围 `for` 循环**：遍历容器 / 数组时，优先用 C++11 的范围 `for`，代码更简洁、更不易出错；
2. **传统 `for` 循环的适用场景**：需要精确控制索引（比如反向遍历、间隔遍历）、或遍历非标准容器时，用传统 `for`；
3. **循环变量的命名**：推荐用有意义的名字（如 `i`、`j` 用于索引，`num`、`s` 用于元素），避免混淆；
4. **避免无限循环**：除非明确需要（如服务器主循环），否则不要写省略循环条件的 `for(;;)`，容易导致程序卡死；
5. **减少循环内的计算**：如果循环体内有固定不变的计算，尽量提到循环外部，提高效率：

```cpp
    // 不推荐
    for (int i = 0; i < 100; i++) {
        int x = 10 * 10; // 固定计算，每次循环都执行
        /* ... */
    }
    
    // 推荐
    int x = 10 * 10; // 提到循环外，只执行一次
    for (int i = 0; i < 100; i++) {
        /* ... */
    }
```
    

---

## 七、综合练习（结合之前的算法题）

### 练习 1：用传统 `for` 循环实现两数之和（暴力解法）

```cpp
#include <vector>
using namespace std;

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        int n = nums.size();
        // 嵌套传统 for 循环：遍历所有两两组合
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                if (nums[i] + nums[j] == target) {
                    return {i, j};
                }
            }
        }
        return {};
    }
};
```

### 练习 2：用范围 `for` 循环实现字母异位词分组

```cpp
#include <vector>
#include <string>
#include <unordered_map>
#include <algorithm>
using namespace std;

class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string, vector<string>> hashMap;
        
        // 范围 for 循环遍历所有字符串
        for (const string& s : strs) {
            string key = s;
            sort(key.begin(), key.end());
            hashMap[key].push_back(s);
        }
        
        // 范围 for 循环遍历哈希表，收集结果
        vector<vector<string>> result;
        for (auto& pair : hashMap) {
            result.push_back(pair.second);
        }
        return result;
    }
};
```