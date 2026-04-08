
## 3.2 首页与目录浏览

Nginx 默认提供**目录请求处理**与**目录文件列表浏览**的能力，通过`index`指令配置默认首页，通过`autoindex`指令开启目录浏览功能，适配静态站点的基础访问需求。

### 3.2.1 首页配置（index 指令）

#### 核心作用

当客户端请求**目录路径**（如`http://example.com/`、`http://example.com/static/`）时，Nginx 会按`index`指令配置的文件名顺序，依次在对应目录中查找文件，找到第一个存在的文件即返回，若未找到则根据`autoindex`配置决定是否开启目录浏览。

#### 语法与作用域

- 语法：`index 文件名1 文件名2 ...;`
- 作用域：`http`、`server`、`location`块，支持多层覆盖。

#### 示例与最佳实践

nginx

```
# server块中配置全局首页
index index.html index.htm index.php default.html;
# 特定location块覆盖，适配前端单页应用
location /spa/ {
    root /var/www;
    index index.html; # 仅以index.html为首页
}
```

#### 关键特性

- 文件名顺序决定查找优先级，建议将最常用的首页文件（如`index.html`）放在最前；
- 支持配置带路径的文件名，如`index /html/default.html`，但不推荐，易导致路径混乱。

### 3.2.2 目录浏览配置（autoindex 指令）

#### 核心作用

当客户端请求目录路径且`index`指令配置的首页文件均不存在时，开启`autoindex`后 Nginx 会生成**可视化的目录文件列表页面**，展示该目录下的所有文件与子目录，支持文件名、大小、修改时间、类型等信息查看，适合静态资源文件服务器场景。

#### 核心指令集

表格

|指令|语法|作用|默认值|
|---|---|---|---|
|autoindex|autoindex on \| off;|开启 / 关闭目录浏览功能|off|
|autoindex_exact_size|autoindex_exact_size on \| off;|开启：文件大小以字节为单位（精确）；关闭：以 KB/MB/GB 为单位（人性化）|on|
|autoindex_localtime|autoindex_localtime on \| off;|开启：文件修改时间为服务器本地时间；关闭：为 UTC 时间|off|
|autoindex_format|autoindex_format html \| xml \| json \| jsonp;|指定目录列表的返回格式，适配程序调用场景|html|

#### 完整配置示例

nginx

```
location /files/ {
    root /var/www;
    autoindex on; # 开启目录浏览
    autoindex_exact_size off; # 人性化显示文件大小
    autoindex_localtime on; # 显示服务器本地时间
    autoindex_format html; # 以HTML格式返回
}
```

#### 配置注意事项

1. 生产环境中，**非必要场景禁止开启目录浏览**，避免服务器目录结构泄露，带来安全风险；
2. 若开启目录浏览，建议配合`allow/deny`指令做 IP 访问控制，仅允许内网或指定 IP 访问。
