## 一、核心定位

`wget` 和 `curl` 是 Linux 上**两大经典的网络下载与请求工具**，功能各有侧重：

- **wget**：简单易用的**下载工具**，适合递归下载网站、断点续传、后台下载，语法极简，专注下载。
- **curl**：被称为 **“网络请求瑞士军刀”**，支持 HTTP/HTTPS/FTP/SFTP 等多种协议，适合测试 API、发送复杂请求、上传 / 下载文件，功能极丰富。

---

## 二、wget 详解

### 1. 核心概念

- **主要用途**：下载文件、递归下载网站、断点续传、后台下载。
- **默认行为**：默认用 URL 的最后部分作为文件名，保存到当前目录。

### 2. 语法格式

```bash
wget [选项] URL
```

### 3. 高频核心命令

| 功能                 | 命令                                                             | 说明                              |
| :----------------- | :------------------------------------------------------------- | :------------------------------ |
| **简单下载**           | `wget https://example.com/file.zip`                            | 下载 file.zip 到当前目录，文件名用 URL 最后部分 |
| **指定文件名**          | `wget -O myfile.zip https://example.com/file.zip`              | 下载并保存为 myfile.zip               |
| **断点续传**           | `wget -c https://example.com/largefile.iso`                    | 继续下载之前未完成的文件                    |
| **递归下载网站**         | `wget -r https://example.com/`                                 | 递归下载整个网站（默认深度 5）                |
| **限制递归深度**         | `wget -r -l 2 https://example.com/`                            | 递归下载，深度限制为 2（避免下载太多）            |
| **后台下载**           | `wget -b https://example.com/largefile.iso`                    | 后台下载，日志写入 wget-log              |
| **从文件读取 URL 批量下载** | `wget -i urls.txt`                                             | 从 urls.txt 读取每行一个 URL 批量下载      |
| **忽略 SSL 证书错误**    | `wget --no-check-certificate https://self-signed.example.com/` | 忽略自签名 SSL 证书错误                  |

---

## 三、curl 详解

### 1. 核心概念

- **主要用途**：发送 HTTP/HTTPS 请求、测试 API、上传 / 下载文件、支持多种协议。
- **默认行为**：默认将响应输出到**标准输出**（终端），要保存文件需加 `-o` 或 `-O`。

### 2. 语法格式

```bash
curl [选项] URL
```

### 3. 高频核心选项与命令

表格

|功能|选项 / 命令|说明|
|:--|:--|:--|
|**简单下载文件**|`curl -o myfile.zip https://example.com/file.zip`|下载并保存为 myfile.zip|
|**用 URL 文件名保存**|`curl -O https://example.com/file.zip`|下载并保存为 file.zip（URL 最后部分）|
|**只看响应头**|`curl -I https://example.com/`|仅显示 HTTP 响应头，不显示响应体|
|**显示响应头 + 响应体**|`curl -i https://example.com/`|显示响应头和响应体|
|**指定请求方法**|`curl -X POST https://api.example.com/`|发送 POST 请求（默认 GET）|
|**添加请求头**|`curl -H "Content-Type: application/json" https://api.example.com/`|添加 Content-Type 请求头|
|**发送 POST 数据**|`curl -d "key1=value1&key2=value2" https://api.example.com/`|发送表单 POST 数据|
|**发送 JSON POST 数据**|`curl -H "Content-Type: application/json" -d '{"key":"value"}' https://api.example.com/`|发送 JSON POST 数据|
|**上传文件**|`curl -F "file=@/path/to/localfile" https://upload.example.com/`|上传本地文件到服务器|
|**HTTP 认证**|`curl -u username:password https://secure.example.com/`|发送用户名密码认证|
|**忽略 SSL 证书错误**|`curl -k https://self-signed.example.com/`|忽略自签名 SSL 证书错误|
|**跟随重定向**|`curl -L https://example.com/redirect`|自动跟随 HTTP 重定向|
|**显示详细请求 / 响应**|`curl -v https://example.com/`|显示详细的请求头、响应头、连接信息（调试用）|

---

## 四、避坑提示与注意事项

### wget

1. **递归下载要限制深度，避免下载整个互联网**
    
    - 默认递归深度是 5，可能下载太多文件；
    - 推荐：用 `-l N` 限制深度，如 `-l 2`。
    
2. **断点续传用 `-c`，下载大文件必备**
    
    - 下载大文件中断后，用 `-c` 继续下载，不用重新开始；
    - 典型错误：大文件中断后重新下载，浪费时间；
    - 正确示例：`wget -c https://example.com/largefile.iso`。
    
3. **后台下载用 `-b`，日志在 wget-log**
    
    - 后台下载后，日志写入当前目录的 wget-log 文件；
    - 推荐：用 `tail -f wget-log` 查看下载进度。
    

### curl

1. **默认输出到标准输出，要保存文件需加 `-o` 或 `-O`**
    
    - 新手容易忘，导致响应直接输出到终端；
    - 典型错误：`curl https://example.com/file.zip`，文件内容直接显示在终端；
    - 正确示例：`curl -O https://example.com/file.zip` 或 `curl -o myfile.zip https://example.com/file.zip`。
    
2. **POST 数据要注意编码，JSON 要加 Content-Type**
    
    - 发送 JSON 数据必须加 `-H "Content-Type: application/json"`，否则服务器可能识别为表单；
    - 典型错误：`curl -d '{"key":"value"}' https://api.example.com/`，没加 Content-Type；
    - 正确示例：`curl -H "Content-Type: application/json" -d '{"key":"value"}' https://api.example.com/`。
    
3. **跟随重定向用 `-L`，很多网站会重定向**
    
    - 很多网站会从 HTTP 重定向到 HTTPS，或从 www 重定向到非 www；
    - 推荐：日常使用加 `-L`，如 `curl -L https://example.com/`。
    

---

## 五、同类对比

| 特性        | wget           | curl                               |
| :-------- | :------------- | :--------------------------------- |
| **核心用途**  | 简单下载、递归下载、断点续传 | 网络请求、API 测试、上传 / 下载、多协议            |
| **功能丰富度** | 中等（专注下载）       | 高（网络瑞士军刀）                          |
| **协议支持**  | HTTP/HTTPS/FTP | HTTP/HTTPS/FTP/SFTP/SCP/TELNET 等多种 |
| **语法复杂度** | 极简             | 中等（选项多）                            |
| **默认安装**  | 多数发行版默认安装      | 多数发行版默认安装                          |
| **适用场景**  | 简单下载、递归下载网站    | API 测试、复杂请求、多协议需求                  |

---

## 六、课后练习

### wget 练习

1. 简单下载：`wget https://speed.hetzner.de/100MB.bin`（测试用 100MB 文件）。
2. 指定文件名：`wget -O testfile.bin https://speed.hetzner.de/100MB.bin`。
3. 断点续传：先中断下载（按 `Ctrl+C`），然后 `wget -c https://speed.hetzner.de/100MB.bin` 继续下载。
4. 递归下载（限制深度）：`wget -r -l 2 https://example.com/`。
5. 后台下载：`wget -b https://speed.hetzner.de/100MB.bin`，然后 `tail -f wget-log` 查看进度。
6. 从文件读取 URL：创建一个 urls.txt 文件，每行一个 URL，然后 `wget -i urls.txt`。

### curl 练习

1. 简单下载：`curl -O https://speed.hetzner.de/100MB.bin`。
2. 指定文件名：`curl -o testfile.bin https://speed.hetzner.de/100MB.bin`。
3. 只看响应头：`curl -I https://www.baidu.com/`。
4. 显示响应头 + 响应体：`curl -i https://www.baidu.com/`。
5. 发送 GET 请求：`curl https://api.github.com/`。
6. 发送 POST 表单数据：`curl -d "param1=value1&param2=value2" https://httpbin.org/post`（[httpbin.org](https://httpbin.org) 是测试 API 的好工具）。
7. 发送 JSON POST 数据：`curl -H "Content-Type: application/json" -d '{"name":"test"}' https://httpbin.org/post`。
8. 跟随重定向：`curl -L http://example.com/`。
9. 忽略 SSL 证书：`curl -k https://self-signed.badssl.com/`（仅测试用）。
10. 显示详细信息：`curl -v https://www.baidu.com/`。