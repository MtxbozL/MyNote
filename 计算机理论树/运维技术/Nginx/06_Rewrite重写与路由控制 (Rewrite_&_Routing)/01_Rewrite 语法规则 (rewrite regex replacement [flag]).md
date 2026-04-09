### 1. 模块概述

`rewrite` 指令隶属于 Nginx 原生内置的 **ngx_http_rewrite_module** 标准 HTTP 模块，无需额外编译配置即可启用。其核心能力是通过 PCRE（Perl Compatible Regular Expressions）正则表达式匹配客户端请求的 URI，完成 URI 内部重写、客户端重定向、路由转发操作，是 Nginx 实现 URL 语义化、域名迁移适配、流量调度、伪静态配置的核心基础指令。

### 2. 核心语法定义

**官方标准语法**：`rewrite regex replacement [flag];`

**默认配置**：无

**合法作用域**：`server`、`location`、`if`

> 注：该指令仅可在上述三个作用域内配置，禁止在`main`、`events`、`http`全局块中使用。

### 3. 语法元素详解

#### 3.1 regex 正则匹配字段

`regex` 为遵循 PCRE 语法规范的正则表达式，用于匹配目标请求字符串，核心规则如下：

1. **匹配对象边界**：匹配目标为请求的**URI 路径部分**，不包含协议、域名、端口，也不包含请求查询字符串（query string，对应`$args`内置变量）。例如请求`https://example.com/user/123?from=index`，匹配对象仅为`/user/123`。
2. **捕获组规则**：通过`()`包裹的正则片段可定义捕获组，捕获内容可在`replacement`字段中通过`$1`、`$2`、…、`$n`按顺序引用，`$0`代表正则匹配到的完整字符串。
3. **匹配锚定符**：`^`用于锚定 URI 的起始位置，`$`用于锚定 URI 的结束位置，可实现精准匹配，避免泛匹配导致的规则冲突。
4. **匹配失败处理**：若`regex`与当前 URI 不匹配，该条`rewrite`指令终止执行，同作用域内后续`rewrite`指令按书写顺序继续执行。

#### 3.2 replacement 替换字符串字段

`replacement` 为正则匹配成功后，用于替换原始 URI 的字符串，其执行行为、参数处理遵循严格的规则约束：

1. **两大执行分支**
    
    - **协议开头的重定向分支**：当`replacement`以`http://`、`https://`、`$scheme`等应用层协议标识符开头时，Nginx 直接终止当前`rewrite`处理流程，向客户端返回 HTTP 重定向响应。无 flag 指定时，默认返回 302（临时重定向）状态码；重定向地址可拼接正则捕获组与 Nginx 内置变量，实现动态跳转。
    - **非协议开头的内部重写分支**：当`replacement`不以协议标识符开头时，Nginx 仅在服务内部执行 URI 重写，不向客户端返回重定向响应，重写后的 URI 继续参与后续 Nginx 配置的 location 匹配与指令执行。
    
2. **查询字符串处理规则**
    
    - 默认行为：若`replacement`未显式添加`?`结尾符，原始请求的 query string 会自动追加到替换后的 URI 末尾；
    - 丢弃原始参数：若需禁止原始 query string 追加，需在`replacement`字符串末尾添加`?`结尾符；
    - 自定义参数：可在`replacement`中直接拼接自定义查询参数，也可通过`$args`、`$query_string`内置变量引用原始参数。
    
3. **变量引用规则**：`replacement`支持引用 regex 捕获组变量（`$1-$n`）与所有 Nginx HTTP 核心内置变量，实现动态重写逻辑。

#### 3.3 [flag] 可选标志位字段

`flag`为可选参数，用于控制`rewrite`匹配成功后的后续处理行为，是 rewrite 规则流程控制的核心。合法 flag 取值共 4 类，本节仅做语法层面定义，各 flag 的执行逻辑差异与适用场景将在 6.02 节详细解析。

|flag 取值|语法层面核心定义|
|---|---|
|`last`|终止当前作用域的 rewrite 指令集执行，以重写后的 URI 重新发起 location 匹配|
|`break`|终止当前作用域的所有 rewrite 指令集执行，不发起新的 location 匹配，继续执行当前 location 内的后续非 rewrite 指令|
|`redirect`|强制返回 302 临时重定向状态码，适用于 replacement 非协议开头的场景|
|`permanent`|强制返回 301 永久重定向状态码，适用于 replacement 非协议开头的场景|

### 4. 指令执行逻辑与约束

1. **执行顺序规则**
    
    - 同一作用域内（server/location/if），`rewrite`指令按配置文件的书写顺序自上而下串行执行；
    - server 块内的`rewrite`指令，优先于 location 块内的所有指令执行；
    - if 块内的`rewrite`指令，仅在 if 条件判断为真时执行。
    
2. **循环执行约束**
    
    当 URI 被 rewrite 重写后，若未被`break`/`redirect`/`permanent`等 flag 终止，Nginx 会重新进入配置匹配流程。该循环执行的最大次数为 10 次，超过 10 次后 Nginx 将向客户端返回`500 Internal Server Error`错误，避免配置错误导致的死循环。
3. **执行优先级**
    
    `rewrite`指令的执行优先级，高于`proxy_pass`、`root`、`alias`等内容处理类指令，早于 location 的二次匹配流程。

### 5. 语法合规性校验与典型示例

#### 5.1 语法校验规范

Nginx 配置修改后，必须通过`nginx -t`命令执行语法预校验。该命令会检测 rewrite 正则表达式的合法性、作用域合规性等核心问题，校验通过后方可执行`nginx -s reload`重载配置，避免服务中断。

#### 5.2 典型语法示例

示例 1：基础内部 URI 重写（伪静态场景）

```nginx
rewrite ^/article/(\d+)\.html$ /article.php?id=$1 last;
```

- 语法解析：regex 锚定以`/article/`开头、`.html`结尾的 URI，捕获文章 ID 为`$1`；
- 行为说明：内部重写为`/article.php?id=$1`，`last`标志位触发新的 location 匹配，原始请求参数自动追加。

示例 2：域名永久重定向（域名迁移场景）

```nginx
rewrite ^/(.*)$ https://www.newdomain.com/$1 permanent;
```

- 语法解析：regex 匹配根路径下所有 URI，捕获完整路径为`$1`；
- 行为说明：replacement 以 https 协议开头，`permanent`标志位强制返回 301 永久重定向，全量流量跳转至新域名。

示例 3：丢弃原始请求参数的重写

```nginx
rewrite ^/product/(\d+)$ /detail.php?pid=$1? last;
```

- 语法解析：replacement 末尾添加`?`结尾符；
- 行为说明：仅保留自定义的`pid`参数，原始请求的 query string 完全丢弃，避免参数污染。

示例 4：多捕获组动态参数传递

```nginx
rewrite ^/category/(\d+)/item/(\d+)$ /goods.php?cid=$1&gid=$2 last;
```

- 语法解析：regex 定义两个捕获组，分别匹配分类 ID 与商品 ID；
- 行为说明：重写后通过`$1`、`$2`分别引用捕获组，实现多参数动态传递。

### 6. 核心边界条件与注意事项

1. 大小写匹配控制：默认 regex 为大小写敏感匹配，若需实现大小写不敏感匹配，需在正则开头添加`(?i)`修饰符，示例：`rewrite (?i)^/img/(.*)$ /images/$1 last;`。
2. 编码 URI 匹配规则：rewrite 匹配的是**解码后的 URI**，而非客户端原始请求的 URL 编码字符串。例如客户端请求`/user/a%20b`，匹配对象为`/user/a b`。
3. 空 URI 匹配：当请求 URI 为`/`时，需严格约束正则的泛匹配规则，避免出现误匹配。
4. 规则冲突规避：rewrite 重写后的 URI 会重新匹配 location，需提前规划规则逻辑，避免出现循环匹配与规则冲突。