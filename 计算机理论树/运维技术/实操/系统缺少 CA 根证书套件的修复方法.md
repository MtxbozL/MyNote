#### 方案一：临时关闭 APT 的 HTTPS 证书验证（推荐）

通过临时配置跳过证书校验，打通安装流程，用完立即恢复安全校验。

1. 创建临时配置，关闭 APT 的 HTTPS 证书验证

```bash
echo 'Acquire::https::Verify-Peer "false";
Acquire::https::Verify-Host "false";' > /etc/apt/apt.conf.d/99temp-skip-ssl
```

2. 执行 apt update，此时不会再报证书错误

```bash
apt update
```

3. 安装缺失的 CA 根证书包（核心修复步骤）

```
apt install -y ca-certificates
```

4. 删除临时配置，恢复证书安全校验（必须执行）

```
rm -f /etc/apt/apt.conf.d/99temp-skip-ssl
```

5. 重新执行 apt update，验证是否完全恢复正常

```bash
apt update
```

---

#### 方案二：临时更换 HTTP 源（备选，更稳妥）

如果方案一无效，直接把 HTTPS 源换成 HTTP 协议，绕开 SSL 证书验证的需求，安装完成后再还原。

1. 备份原有源文件

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

2. 替换为 HTTP 协议的官方源（临时使用）

```bash
sed -i 's/https:\/\/mirrors.ustc.edu.cn/http:\/\/archive.ubuntu.com/g' /etc/apt/sources.list
```

3. 更新源并安装 CA 证书包

```bash
apt update && apt install -y ca-certificates
```

4. 还原原来的中科大 HTTPS 源

```bash
cp /etc/apt/sources.list.bak /etc/apt/sources.list
```

5. 重新执行 apt update，验证正常

```bash
apt update
```

---

### 补充说明

1. 该问题常见于**精简版 Ubuntu 镜像**（比如 Docker 容器镜像、最小化安装的系统），这类镜像默认没有预装`ca-certificates`包，搭配 HTTPS 软件源就会触发该故障。
2. 临时关闭证书验证仅用于解决本次死循环，用完必须删除临时配置，否则后续 APT 的 HTTPS 请求都不会验证证书，存在安全风险。
3. 修复完成后，建议执行`apt upgrade`将系统包更新到最新版本，避免其他依赖问题。