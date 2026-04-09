
Nginx 本身不提供自动日志切割能力，随着服务持续运行，日志文件会持续膨胀，导致磁盘空间耗尽、日志查询与归档效率极低，甚至引发服务 IO 性能瓶颈。Linux 系统原生的**logrotate**工具是行业标准的日志切割方案，无需额外安装，可实现自动化、平滑的日志切割、压缩、归档、清理全流程，不中断 Nginx 服务。

### 3.1 logrotate 核心工作原理

logrotate 是 Linux 系统内置的日志轮转工具，通过系统定时任务`crontab`每日自动执行，核心执行流程如下：

1. 读取`/etc/logrotate.d/`目录下的 Nginx 专属配置文件，匹配目标日志文件；
2. 按照配置的规则，对达到切割条件的日志文件进行重命名归档；
3. 向 Nginx 主进程发送`USR1`信号，触发 Nginx 重新打开新的日志文件，实现平滑切换，无请求丢失、无服务中断；
4. 按照配置规则，对旧日志进行压缩、保留指定份数、过期自动删除。

> 核心关键：`USR1`信号是 Nginx 官方指定的「重新打开日志文件」信号，属于轻量级操作，仅重新加载日志文件句柄，不重载配置、不重启进程，完全不影响业务请求处理，禁止使用`HUP`信号（重载配置）替代。

### 3.2 logrotate 标准配置详解

Nginx 通过 yum/apt 包管理器安装时，会自动在`/etc/logrotate.d/`目录下生成`nginx`配置文件；源码编译安装的 Nginx 需手动创建该配置文件。

#### 3.2.1 配置文件核心参数详解

|参数|功能说明|生产环境推荐值|
|---|---|---|
|`daily/weekly/monthly`|切割周期，按日 / 周 / 月切割|高流量场景`daily`，低流量场景`weekly`|
|`rotate N`|保留归档日志的份数，超出份数的旧日志自动删除|30（保留 30 天），合规场景可调整至 90/180|
|`compress`|开启归档日志的 gzip 压缩，压缩比可达 10:1，大幅节省磁盘空间|开启`compress`|
|`delaycompress`|延迟压缩，切割后的第一个归档文件不压缩，下一次切割时再压缩|开启，避免切割时压缩大文件导致 IO 峰值|
|`missingok`|日志文件不存在时不报错，继续执行，避免定时任务异常|开启|
|`notifempty`|日志文件为空时不执行切割，避免生成无效空归档文件|开启|
|`create mode user group`|新建日志文件的权限、所有者、所属组|与 Nginx 运行用户匹配，如`create 640 nginx nginx`|
|`postrotate/endscript`|切割完成后执行的 Shell 脚本，核心是向 Nginx 发送 USR1 信号|必须配置，实现日志平滑切换|
|`sharedscripts`|多个日志文件匹配时，仅执行一次 postrotate 脚本，而非每个文件执行一次|开启，避免重复发送信号|
|`dateext`|归档文件添加日期后缀，替代默认的数字序号，便于日志溯源|开启|
|`dateformat`|日期后缀格式，如`-%Y%m%d`|推荐`-%Y%m%d`，按天切割|

#### 3.2.2 生产环境标准配置示例

##### 1. yum/apt 安装的 Nginx（默认路径）

```bash
# 配置文件路径：/etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 nginx nginx
    sharedscripts
    dateext
    dateformat -%Y%m%d
    postrotate
        # 检查Nginx主进程是否存在，存在则发送USR1信号
        if [ -f /run/nginx.pid ]; then
            kill -USR1 `cat /run/nginx.pid`
        fi
    endscript
}
```

##### 2. 源码编译安装的 Nginx（自定义路径）

```bash
# 配置文件路径：/etc/logrotate.d/nginx
/usr/local/nginx/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 640 www www
    sharedscripts
    dateext
    dateformat -%Y%m%d
    postrotate
        # 源码安装的Nginx PID文件路径
        if [ -f /usr/local/nginx/logs/nginx.pid ]; then
            kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
        fi
    endscript
}
```

##### 3. 高流量场景小时级切割配置

对于日均日志量超过 10GB 的高流量场景，需配置小时级切割，避免单日志文件过大，影响查询与归档效率：

```bash
# 配置文件路径：/etc/logrotate.d/nginx_hourly
/var/log/nginx/*.log {
    hourly
    rotate 720
    compress
    delaycompress
    missingok
    notifempty
    create 640 nginx nginx
    sharedscripts
    dateext
    dateformat -%Y%m%d%H
    postrotate
        if [ -f /run/nginx.pid ]; then
            kill -USR1 `cat /run/nginx.pid`
        fi
    endscript
}
```

> 小时级切割需额外配置定时任务：将 logrotate 执行脚本从`/etc/cron.daily/`复制到`/etc/cron.hourly/`，确保每小时执行一次。

### 3.3 配置验证与手动执行

1. **调试模式验证配置**：配置完成后，通过调试模式检查配置是否合法，无实际执行操作：
    
    ```bash
    logrotate -d /etc/logrotate.d/nginx
    ```
    
2. **强制手动触发切割**：可手动强制执行切割，验证流程是否正常，用于紧急场景或配置上线验证：
    
    ```bash
    logrotate -f /etc/logrotate.d/nginx
    ```
    
3. **执行结果验证**：执行后检查日志目录是否生成归档文件，Nginx 是否生成新的日志文件，新请求是否正常写入新日志，无报错信息。

### 3.4 常见陷阱与最佳实践

1. **PID 文件路径必须准确**：postrotate 脚本中的 PID 文件路径必须与 Nginx 配置文件中`pid`指令指定的路径完全一致，否则信号无法发送到 Nginx 主进程，导致切割后 Nginx 仍写入旧日志文件，是最常见的配置错误。
2. **权限约束**：logrotate 以 root 用户执行，归档目录必须对 root 用户有读写权限，新建日志文件的权限必须与 Nginx 运行用户匹配，否则会导致 Nginx 无法写入新日志。
3. **禁止使用 reload 替代 USR1**：`kill -HUP`是重载 Nginx 配置，会重新加载所有配置、重建 Worker 进程，属于重量级操作，可能导致业务短暂波动；`USR1`是专门用于日志重新打开的轻量级信号，是官方唯一推荐的日志切割信号。
4. **归档目录规划**：生产环境建议将日志归档到独立的磁盘分区，避免日志文件占满系统盘，导致操作系统异常；同时配置磁盘空间监控，当使用率超过 85% 时触发告警。
5. **合规性适配**：等保 2.0、金融行业合规标准要求日志至少保留 6 个月，需调整`rotate`参数至 180 以上，同时将归档日志备份到离线存储，满足审计要求。
6. **压缩性能优化**：高流量场景可配置`compresscmd /usr/bin/pigz`，使用多线程 gzip 压缩工具替代单线程 gzip，大幅降低压缩耗时与 CPU 开销。

---
