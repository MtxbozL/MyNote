
Prometheus 的配置文件、规则文件修改后，默认不会自动生效，需重启进程或触发**热加载**。热加载是生产环境的核心运维能力，可在不中断服务、不丢失时序数据、不停止抓取与规则评估的前提下，完成配置的动态更新。

### 2.5.1 热加载生效前提

1. 待加载的新配置文件、规则文件必须通过`promtool`的合法性校验，否则热加载直接失败，原有配置继续生效；
2. 若通过 HTTP 接口触发热加载，Prometheus 启动时必须添加`--web.enable-lifecycle`启动参数，开启生命周期管理接口，否则`/-/reload`接口将返回 404。

### 2.5.2 标准热加载方式

#### 方式 1：SIGHUP 信号触发（生产环境首选）

向 Prometheus 主进程发送`SIGHUP`信号，触发配置重加载，无需开启 web 生命周期接口，安全性最高，适配离线、内网等无 web 访问权限的场景。

```bash
# 1. 获取Prometheus主进程PID
pidof prometheus
# 2. 发送SIGHUP信号触发热加载
kill -HUP <prometheus-pid>
```

#### 方式 2：HTTP 接口触发（自动化场景首选）

通过 POST 请求访问 Prometheus 的`/-/reload`接口，触发热加载，适配自动化脚本、CI/CD 流水线、远程运维场景。

```bash
# 本地触发
curl -X POST http://localhost:9090/-/reload
# 远程触发
curl -X POST http://<prometheus-ip>:9090/-/reload
```

### 2.5.3 热加载生效范围与限制

- **可热加载更新的配置**：全局配置、抓取配置、规则文件路径、Alertmanager 配置、远程读写配置、服务发现配置等绝大多数运行时配置；
- **不可热加载更新的配置**：进程启动参数（如监听端口、存储路径、web 生命周期开关、TSDB 配置等），此类参数修改必须重启 Prometheus 进程方可生效。

---
