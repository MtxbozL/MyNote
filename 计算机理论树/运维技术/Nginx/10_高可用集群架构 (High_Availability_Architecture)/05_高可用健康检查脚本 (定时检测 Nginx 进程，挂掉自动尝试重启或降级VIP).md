
Keepalived 原生的 VRRP 机制仅能监控自身进程与服务器网络状态，**无法感知 Nginx 服务的运行状态**—— 若 Nginx 进程崩溃、服务异常无法响应，但 Keepalived 进程仍正常运行，主节点会继续持有 VIP，导致客户端流量进入后无法处理，业务中断。

自定义健康检查脚本是解决该问题的核心方案，也是企业级生产环境必须配置的核心能力，可实时监控 Nginx 服务状态，异常时自动调整节点优先级，触发 VIP 漂移，实现故障自愈。

### 5.1 健康检查的核心设计目标

1. **全维度监控**：不仅监控 Nginx 进程是否存活，还要监控 Nginx 服务是否可正常响应请求，避免进程存活但服务僵死的情况；
2. **故障自愈**：Nginx 异常时，先尝试自动重启服务，无需人工干预，减少故障切换次数；
3. **自动切换**：重启失败后，自动降低当前节点的 VRRP 优先级，触发 VIP 漂移至备节点，保证业务不中断；
4. **状态恢复**：Nginx 服务恢复正常后，自动恢复节点优先级，保证集群角色符合预期；
5. **日志记录**：完整记录检查过程、异常事件、操作结果，便于故障溯源与审计。

### 5.2 标准 Nginx 健康检查脚本

脚本路径：`/etc/keepalived/check_nginx.sh`，主备节点需完全一致，添加执行权限`chmod +x /etc/keepalived/check_nginx.sh`。

```bash
#!/bin/bash
# Nginx高可用健康检查脚本
# 日志文件路径
LOG_FILE="/var/log/keepalived_check_nginx.log"
# Nginx PID文件路径，需与nginx.conf中的pid指令一致
NGINX_PID="/usr/local/nginx/logs/nginx.pid"
# 健康检查地址，使用本地回环地址，避免网络波动影响
CHECK_URL="http://127.0.0.1/nginx_status"
# 最大重启尝试次数
MAX_RETRY=2
# 当前重试次数
RETRY_COUNT=0

# 日志输出函数
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# 步骤1：检查Nginx进程是否存活
check_process() {
    if [ -f $NGINX_PID ]; then
        NGINX_PID_VALUE=$(cat $NGINX_PID)
        if kill -0 $NGINX_PID_VALUE > /dev/null 2>&1; then
            log "Nginx进程正常，PID: $NGINX_PID_VALUE"
            return 0
        else
            log "Nginx PID文件存在，但进程已不存在"
            return 1
        fi
    else
        log "Nginx PID文件不存在，进程未启动"
        return 1
    fi
}

# 步骤2：检查Nginx服务是否可正常响应
check_service() {
    HTTP_STATUS=$(curl -o /dev/null -s -w "%{http_code}" --connect-timeout 2 --max-time 3 $CHECK_URL)
    if [ $HTTP_STATUS -eq 200 ]; then
        log "Nginx服务响应正常，HTTP状态码: $HTTP_STATUS"
        return 0
    else
        log "Nginx服务响应异常，HTTP状态码: $HTTP_STATUS"
        return 1
    fi
}

# 步骤3：重启Nginx服务
restart_nginx() {
    log "尝试重启Nginx服务，第$((RETRY_COUNT+1))次尝试"
    # 先尝试平滑重载，失败再强制重启
    /usr/local/nginx/sbin/nginx -t > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        /usr/local/nginx/sbin/nginx -s reload
    else
        /usr/local/nginx/sbin/nginx -s stop > /dev/null 2>&1
        sleep 1
        /usr/local/nginx/sbin/nginx
    fi
    sleep 2
    # 检查重启结果
    if check_process && check_service; then
        log "Nginx服务重启成功"
        return 0
    else
        log "Nginx服务重启失败"
        return 1
    fi
}

# 主执行逻辑
log "==================== 开始健康检查 ===================="
# 先检查进程，再检查服务响应
if check_process && check_service; then
    log "Nginx服务完全正常，健康检查通过"
    exit 0
else
    log "Nginx服务异常，进入故障处理流程"
    # 循环尝试重启
    while [ $RETRY_COUNT -lt $MAX_RETRY ]; do
        RETRY_COUNT=$((RETRY_COUNT+1))
        if restart_nginx; then
            exit 0
        fi
    done
    # 重启失败，退出码1，触发Keepalived优先级降低
    log "Nginx重启失败，已达到最大重试次数，触发VIP漂移"
    exit 1
fi
```

### 5.3 脚本核心配置说明

1. **NGINX_PID**：必须与 Nginx 配置文件中`pid`指令指定的路径完全一致，否则进程检查会失效；
2. **CHECK_URL**：建议使用 Nginx 的`stub_status`状态页面，该页面轻量无业务依赖，可真实反映 Nginx 服务的响应能力，需提前在 Nginx 中配置本地可访问的状态页面；
3. **MAX_RETRY**：最大重启尝试次数，建议设置为 2-3 次，避免频繁重启导致服务异常；
4. **超时设置**：curl 命令设置了 2 秒连接超时、3 秒总超时，避免检查脚本阻塞，影响 Keepalived 的 VRRP 报文收发；
5. **退出码规则**：脚本正常退出码为 0，Keepalived 不调整优先级；退出码为 1 时，Keepalived 按照`weight`配置降低节点优先级，触发故障切换。

### 5.4 进阶健康检查能力

生产环境可根据业务需求，扩展健康检查脚本的能力，实现更全面的监控：

1. **后端服务健康检查**：检查 Nginx 反向代理的后端服务可用性，若所有后端节点均宕机，触发 VIP 漂移，避免流量进入后全部返回 502 错误；
2. **系统资源检查**：监控服务器 CPU、内存、磁盘使用率，若资源耗尽导致服务无法正常运行，触发切换；
3. **端口存活检查**：检查 Nginx 监听的 80/443 端口是否正常监听，避免端口未绑定导致的服务异常；
4. **脑裂检测**：脚本中添加脑裂检测逻辑，若发现两个节点同时持有 VIP，自动停止低优先级节点的 Keepalived 服务，避免 IP 冲突。

### 5.5 故障通知脚本

节点角色发生切换时，可通过自定义通知脚本发送告警至运维人员，及时感知故障事件，脚本路径`/etc/keepalived/notify.sh`，添加执行权限：

```bash
#!/bin/bash
# Keepalived角色切换通知脚本
LOG_FILE="/var/log/keepalived_notify.log"
NOTIFY_TYPE=$1
SERVER_IP=$(hostname -I | awk '{print $1}')
DATETIME=$(date +'%Y-%m-%d %H:%M:%S')

# 日志记录
echo "[$DATETIME] 节点$SERVER_IP角色切换为: $NOTIFY_TYPE" >> $LOG_FILE

# 钉钉/企业微信/邮件告警
# 此处替换为实际的告警接口地址
WEBHOOK_URL="https://oapi.dingtalk.com/robot/send?access_token=xxx"
curl -H "Content-Type: application/json" -X POST $WEBHOOK_URL -d '{
    "msgtype": "text",
    "text": {
        "content": "【Nginx高可用告警】\n时间：'$DATETIME'\n节点IP：'$SERVER_IP'\n事件：节点角色切换为'$NOTIFY_TYPE'\n请及时排查故障！"
    }
}'
```

在 Keepalived 配置文件的`vrrp_instance`块中引用该脚本：

```bash
notify_master "/etc/keepalived/notify.sh master"
notify_backup "/etc/keepalived/notify.sh backup"
notify_fault "/etc/keepalived/notify.sh fault"
```

---

## 核心风险点：脑裂问题与解决方案

### 脑裂问题定义

脑裂（Split-Brain）是高可用集群最严重的故障，指主备节点之间的心跳链路中断，双方都判定对方故障，同时切换为主节点，持有同一个 VIP，导致局域网内出现 IP 地址冲突，客户端流量被随机分发到两个节点，出现请求错乱、业务中断、数据不一致等严重问题。

### 脑裂问题的核心诱因

1. 防火墙拦截了 VRRP 组播报文，备节点无法收到主节点的 VRRP 广播；
2. 主节点服务器负载过高、CPU 使用率 100%，无法正常发送 VRRP 广播报文；
3. 主备节点之间的物理链路故障，二层网络不通；
4. VRRP 实例的 VRID、认证密码配置不一致，导致报文无法正常解析；
5. 网卡故障、驱动异常，导致 VRRP 报文无法正常收发。

### 脑裂问题的解决方案

1. **严格配置防火墙规则**：明确放行 VRRP 协议流量，避免规则拦截导致的心跳中断；
2. **双链路心跳检测**：除了 VRRP 组播心跳，额外添加 TCP 直连心跳检测，双重校验节点存活状态；
3. **脑裂检测与自动处理**：在健康检查脚本中添加脑裂检测逻辑，通过 arping 命令检测 VIP 是否在其他节点存在，若发现脑裂，自动停止自身 Keepalived 服务，释放 VIP，保证集群角色唯一；
4. **硬件与网络冗余**：主备节点部署在同一机柜的不同交换机下，避免单交换机故障导致的链路中断；
5. **实时告警监控**：配置 VIP 绑定状态监控、节点角色监控，发现脑裂事件立即触发最高级别告警，运维人员第一时间介入处理。