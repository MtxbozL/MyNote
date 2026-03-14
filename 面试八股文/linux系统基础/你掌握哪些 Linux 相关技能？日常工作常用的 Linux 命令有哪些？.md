#### 核心标准答案

我熟练掌握 Linux 系统的**日常运维、故障排查、性能优化、权限管理、自动化脚本编写**等能力，可独立完成服务器的交付、监控、维护与故障修复。日常高频使用的命令按场景分类如下：

1. **系统信息与状态查看**：`uname`、`hostname`、`uptime`、`top`、`htop`、`free`、`df`、`du`
2. **文件与目录操作**：`ls`、`cd`、`pwd`、`mkdir`、`rm`、`cp`、`mv`、`chmod`、`chown`、`find`、`locate`
3. **进程与服务管理**：`ps`、`top`、`htop`、`kill`、`killall`、`systemctl`、`service`
4. **网络排查与配置**：`ping`、`telnet`、`netstat`、`ss`、`ip`、`ifconfig`、`curl`、`wget`、`tcpdump`
5. **性能监控与故障排查**：`vmstat`、`iostat`、`dstat`、`sar`、`dmesg`、`strace`
6. **文本处理与日志分析**：`cat`、`more`、`less`、`tail`、`head`、`grep`、`awk`、`sed`、`wc`、`sort`
7. **压缩与包管理**：`tar`、`zip`、`unzip`、`yum`、`dnf`、`apt`、`rpm`、`dpkg`

#### 专家级拓展

- 生产环境高频场景的进阶用法：
    
    1. 日志排查：`tail -f xxx.log | grep "ERROR"` 实时过滤错误日志；`awk '{print $1}' access.log | sort | uniq -c | sort -nr` 统计 Nginx 日志的 IP 访问量 Top10
    2. 进程排查：`ps aux --sort=-%cpu | head -10` 快速定位 CPU 占用 Top10 的进程；`lsof -p pid` 查看进程打开的文件句柄
    3. 磁盘排查：`du -sh * | sort -hr` 快速定位当前目录下占用空间最大的文件 / 目录；`df -i` 查看 inode 使用情况，排查小文件过多导致的 inode 耗尽问题
    4. 网络排查：`ss -tulnp` 快速查看系统监听的端口与对应进程，比 netstat 性能更高；`tcpdump -i eth0 port 80` 抓包分析网络请求
    
- 核心能力体现：我不仅会使用基础命令，还能通过命令组合完成复杂的故障排查、数据统计、自动化脚本编写，比如用 shell 脚本实现日志自动切割、服务健康检查、批量服务器巡检等。

#### 面试避坑指南

- 严禁只零散罗列命令名称，不做场景分类，会被面试官判定为「只会背命令，无实际应用能力」；
- 避免说一些不常用的冷门命令，面试官大概率会追问用法，容易翻车；优先说高频、有落地场景的命令，体现实际工作经验。