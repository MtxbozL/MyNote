
基础设施与中间件监控仅能反映系统的底层运行状态，**自定义业务埋点**是实现业务白盒监控的核心手段，通过 Prometheus 官方提供的客户端库，在业务代码中嵌入自定义指标，实现业务维度的全链路监控。

### 5.5.1 业务埋点核心设计规范

1. **指标类型选型严格匹配语义**：严格遵循第三章的指标类型规范，事件总数用 Counter，瞬时状态用 Gauge，耗时分布用 Histogram，禁止类型误用；
2. **指标命名标准化**：遵循`[业务域]_[模块]_[测量对象]_[单位]`的命名规范，保证语义清晰，禁止使用无意义的缩写；
3. **标签设计低基数原则**：标签仅用于维度分组，禁止在标签中加入用户 ID、订单 ID、请求 ID 等高基数唯一标识，避免时序基数爆炸；
4. **标签枚举值可控**：标签的枚举值数量必须可控，例如`method`标签仅包含`GET/POST/PUT/DELETE`，`status`标签仅包含`success/failed`；
5. **性能损耗最小化**：埋点操作仅为内存级计数，无 IO 操作，无阻塞逻辑，避免埋点影响业务代码的性能。

### 5.5.2 Golang 埋点实战

Golang 是 Prometheus 生态的原生开发语言，官方客户端库为`prometheus/client_golang`，是 Go 服务埋点的标准方案。

#### 1. 核心步骤与代码示例

##### 步骤 1：引入依赖

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp
```

##### 步骤 2：初始化指标

```go
package main

import (
  "net/http"
  "time"

  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

// 1. 定义Counter指标：统计HTTP请求总数
var httpRequestsTotal = prometheus.NewCounterVec(
  prometheus.CounterOpts{
    Name: "api_http_requests_total",
    Help: "Total number of HTTP requests processed by the API",
  },
  []string{"method", "path", "status"}, // 标签维度
)

// 2. 定义Histogram指标：统计HTTP请求耗时分布
var httpRequestDuration = prometheus.NewHistogramVec(
  prometheus.HistogramOpts{
    Name:    "api_http_request_duration_seconds",
    Help:    "Duration of HTTP requests processed by the API",
    Buckets: prometheus.DefBuckets, // 预定义桶，可自定义
  },
  []string{"method", "path"},
)

// 3. 定义Gauge指标：统计当前活跃请求数
var httpActiveRequests = prometheus.NewGauge(
  prometheus.GaugeOpts{
    Name: "api_http_active_requests",
    Help: "Current number of active HTTP requests",
  },
)

// 初始化：注册指标到默认Registry
func init() {
  prometheus.MustRegister(httpRequestsTotal)
  prometheus.MustRegister(httpRequestDuration)
  prometheus.MustRegister(httpActiveRequests)
}
```

##### 步骤 3：业务代码中埋点

```go
// 业务HTTP处理函数
func helloHandler(w http.ResponseWriter, r *http.Request) {
  // 活跃请求数+1，请求结束后-1
  httpActiveRequests.Inc()
  defer httpActiveRequests.Dec()

  // 记录请求开始时间，用于计算耗时
  start := time.Now()
  defer func() {
    // 请求结束后，记录耗时到Histogram
    httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(time.Since(start).Seconds())
  }()

  // 业务逻辑处理
  w.WriteHeader(http.StatusOK)
  w.Write([]byte("Hello Prometheus!"))

  // 请求完成，计数+1
  httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
}

func main() {
  // 注册业务路由
  http.HandleFunc("/hello", helloHandler)
  // 暴露/metrics接口
  http.Handle("/metrics", promhttp.Handler())
  // 启动HTTP服务
  http.ListenAndServe(":8080", nil)
}
```

##### 步骤 4：Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "go_api_service"
    scrape_interval: 15s
    metrics_path: "/metrics"
    static_configs:
      - targets: ["127.0.0.1:8080"]
        labels:
          env: prod
          service: "user-api"
```

#### 2. 核心最佳实践

- 使用`MustRegister`注册指标，启动时即可发现指标重复注册等错误；
- Histogram 的桶需根据业务实际耗时分布自定义，避免使用默认桶导致精度不足；
- 标签维度需提前规划，禁止动态生成标签键与标签值；
- 使用`promauto`包可简化指标注册流程，无需手动调用`MustRegister`。

### 5.5.3 Java/Spring Boot 埋点实战

Java/Spring Boot 生态的埋点标准方案是**Micrometer**，它是 Spring Boot 官方默认的应用监控门面，原生支持 Prometheus，完全兼容 Prometheus 指标模型，无需直接使用底层客户端库。

#### 1. 核心步骤与代码示例

##### 步骤 1：引入 Maven 依赖

```xml
<!-- Spring Boot Actuator：提供应用监控端点 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Micrometer Prometheus 注册器：适配Prometheus格式 -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

##### 步骤 2：application.yml 配置

```yaml
server:
  port: 8080
spring:
  application:
    name: user-api
# Actuator 端点配置
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health # 暴露prometheus与health端点
  endpoint:
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name} # 全局标签，标识应用名称
```

##### 步骤 3：业务代码中埋点

```java
package com.example.demo.controller;

import io.micrometer.core.annotation.Timed;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.PostConstruct;
import java.util.concurrent.atomic.AtomicInteger;

@RestController
public class HelloController {

    // 注入MeterRegistry，Micrometer的核心注册器
    private final MeterRegistry meterRegistry;
    // 定义Counter：请求总数
    private Counter requestTotalCounter;
    // 定义Gauge：活跃请求数
    private final AtomicInteger activeRequests = new AtomicInteger(0);

    // 构造器注入
    public HelloController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    // 初始化指标
    @PostConstruct
    public void initMetrics() {
        // 初始化Counter，指定名称、帮助文本、标签
        requestTotalCounter = Counter.builder("api_http_requests_total")
                .description("Total number of HTTP requests processed by the API")
                .tags("method", "GET", "path", "/hello")
                .register(meterRegistry);

        // 初始化Gauge，绑定AtomicInteger
        Gauge.builder("api_http_active_requests", activeRequests, AtomicInteger::get)
                .description("Current number of active HTTP requests")
                .register(meterRegistry);
    }

    @GetMapping("/hello")
    @Timed(value = "api_http_request_duration_seconds", description = "Duration of HTTP requests")
    public String hello() {
        // 活跃请求数+1
        activeRequests.incrementAndGet();
        try {
            // 业务逻辑处理
            requestTotalCounter.increment();
            return "Hello Prometheus!";
        } finally {
            // 活跃请求数-1
            activeRequests.decrementAndGet();
        }
    }
}
```

- 核心说明：`@Timed`注解会自动生成 Histogram 类型的耗时指标，无需手动编写耗时统计逻辑；
- 指标访问地址：`http://127.0.0.1:8080/actuator/prometheus`。

##### 步骤 4：Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "spring_boot_service"
    scrape_interval: 15s
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["127.0.0.1:8080"]
        labels:
          env: prod
          service: "user-api"
```

#### 2. 核心最佳实践

- 优先使用`@Timed`、`@Counted`等注解实现通用埋点，减少代码侵入；
- 全局统一添加`application`、`env`等标签，便于多服务、多环境的指标聚合；
- 自定义 Histogram 的桶分布，通过`@Timed`注解的`histogramBuckets`属性配置；
- 禁用不必要的内置指标，减少时序基数，通过`management.metrics.enable.*`配置关闭。

---
