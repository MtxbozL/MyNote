
本节提供标准化的 Elasticsearch+Kibana 学习环境部署方案，采用 Docker 容器化技术实现环境的快速交付与一致性保障，通过 Docker-Compose 实现服务编排与一键启停，适配学习与测试场景的使用需求。

### 1.4.1 部署方案选型依据

Docker 容器化部署具备环境一致性、资源隔离、快速交付、运维便捷等核心优势，Docker-Compose 可实现多容器服务的协同编排，解决 Elasticsearch 与 Kibana 的服务依赖、网络通信、配置管理等问题，是学习环境的最优部署方案。

本方案采用 Elastic Stack 8.x 稳定版本（当前企业级主流生产版本），实现单节点 Elasticsearch 集群与配套 Kibana 服务的部署，同时兼容生产环境的基础配置规范。

### 1.4.2 前置环境要求

1. 软件版本：Docker 20.10+，Docker-Compose v2+；
2. 硬件资源：宿主机可用内存≥4GB，Elasticsearch 默认堆内存配置 2GB，保障服务稳定运行；
3. 系统配置：Linux 系统需调整内核参数`vm.max_map_count=262144`，满足 Elasticsearch 内存映射的最低要求，否则服务将启动失败。
    
    - 临时生效命令：`sysctl -w vm.max_map_count=262144`
    - 永久生效配置：将`vm.max_map_count=262144`写入`/etc/sysctl.conf`文件，执行`sysctl -p`生效。
    

### 1.4.3 Docker-Compose 标准化配置

创建`docker-compose.yml`配置文件，写入以下标准化配置，核心配置项均标注设计说明，保障配置的可解释性与可扩展性：

```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    container_name: es-learning-node
    environment:
      # 单节点发现模式，关闭集群bootstrap检查，适配学习环境
      - discovery.type=single-node
      # 堆内存初始值与最大值保持一致，避免动态调整带来的性能波动
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
      # 学习环境关闭xpack安全认证，简化操作，生产环境严禁开启此配置
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      # 数据卷挂载，实现数据持久化，避免容器重启数据丢失
      - es-data:/usr/share/elasticsearch/data
    networks:
      - elastic-stack-net
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.15.0
    container_name: kibana-learning
    environment:
      # 关联Elasticsearch服务，基于Docker内部网络通信
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - elastic-stack-net
    restart: unless-stopped

volumes:
  es-data:
    driver: local

networks:
  elastic-stack-net:
    driver: bridge
```

### 1.4.4 部署与服务验证流程

1. 配置文件准备：在指定目录创建`docker-compose.yml`文件，写入上述配置；
2. 服务启动：在配置文件所在目录执行`docker-compose up -d`，实现服务的一键后台启动；
3. 服务状态检查：执行`docker-compose ps`，查看 elasticsearch 与 kibana 服务的状态均为`Up`，确认服务正常启动；
4. Elasticsearch 服务验证：执行`curl http://localhost:9200`，若返回包含集群名称、节点名称、版本号的 JSON 数据，说明 ES 服务正常运行；
5. Kibana 服务验证：通过浏览器访问`http://<宿主机IP>:5601`，成功进入 Kibana 控制台，即可完成环境可用性验证。可通过左侧菜单进入`Dev Tools`控制台，执行 ES 的 RESTful API，开启后续的学习与实践。

### 1.4.5 部署核心注意事项

1. 版本一致性：Elasticsearch 与 Kibana 的主版本、次版本必须严格一致，否则将出现 API 不兼容、服务无法关联等问题；
2. 内存配置规范：ES 堆内存`-Xms`与`-Xmx`必须设置为相同值，最大值不得超过 32GB，否则将触发 JVM 压缩指针失效，导致内存利用率大幅下降（详见后续调优章节）；
3. 数据持久化：必须通过数据卷挂载 ES 数据目录，避免容器销毁或重启导致的学习数据丢失；
4. 生产环境适配：本方案的单节点模式、关闭安全认证等配置仅适用于学习环境，严禁直接用于生产环境。生产环境需配置多节点集群、分片副本、TLS 加密、身份认证、权限管控等企业级安全与高可用能力。

## 本章小结

本章完成了 Elastic Stack 的入门认知与基础环境搭建，核心知识点如下：

1. 明确了 Elastic Stack 从传统 ELK 到现代架构的演进逻辑，核心是通过 Beats 组件解决了边缘数据采集的资源瓶颈，拓展了技术栈的场景适配能力；
2. 掌握了四大核心组件的定位与协同关系，建立了 “采集 - 清洗 - 存储 - 分析 - 可视化” 的全链路架构认知框架；
3. 深入理解了 Lucene 的底层核心原理，包括倒排索引的三级存储结构、检索流程，以及 Doc Values 列式正排索引的设计逻辑与场景适配；
4. 完成了标准化 Docker 学习环境的部署，验证了 Elasticsearch 与 Kibana 服务的可用性，为后续的 ES 基础操作、映射配置、查询语法等实践环节奠定了环境基础。