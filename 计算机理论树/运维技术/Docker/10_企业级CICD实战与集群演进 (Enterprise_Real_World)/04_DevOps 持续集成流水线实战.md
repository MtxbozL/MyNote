
Docker 与 DevOps 理念天然契合，彻底解决了传统 DevOps 流水线中「环境不一致」的核心痛点，实现了**一次构建，全环境复用**的不可变交付模式。容器化应用的 DevOps 流水线，核心是实现从代码提交到生产部署的全流程自动化，将前序章节讲解的 Dockerfile 构建、镜像安全扫描、私有仓库推送、容器编排等能力完整串联，实现软件交付的标准化、自动化、可追溯。

### 4.1 流水线核心设计原则

1. **不可变基础设施**：应用代码与运行环境一起构建为不可变的容器镜像，一次构建后，在开发、测试、预发、生产环境完全复用，仅通过环境变量注入配置，避免环境差异导致的故障；
2. **安全左移**：将单元测试、代码扫描、镜像漏洞扫描、合规检测嵌入流水线早期环节，问题早发现早修复，降低修复成本；
3. **全流程可追溯**：镜像版本与 Git Commit 哈希、流水线构建号一一绑定，实现从代码提交到生产部署的全链路可追溯；
4. **环境隔离**：开发、测试、生产环境严格隔离，镜像需通过测试环境验证、门禁审批后，才能晋升到生产环境；
5. **声明式配置**：所有流水线配置、构建规则、部署配置均以代码形式存储在 Git 仓库中，遵循基础设施即代码（IaC）理念。

### 4.2 标准化自动化流水线全流程

严格遵循目录内定义的核心流程，扩展为企业级生产可用的 6 阶段流水线：

```plaintext
代码提交 → 单元测试与代码质量扫描 → Docker镜像构建 → 镜像安全扫描 → 推送私有镜像仓库 → 测试环境自动部署 → 生产环境审批部署
```

各阶段核心职责与门禁规则：

1. **代码触发阶段**：开发人员提交代码到 Git 仓库的指定分支，自动触发流水线执行；
2. **质量门禁阶段**：执行单元测试、代码覆盖率检测、代码静态扫描，未通过门禁直接阻断流水线；
3. **镜像构建阶段**：基于 Dockerfile 执行多阶段构建，生成标准化的容器镜像，最大化利用构建缓存，保证构建效率；
4. **安全门禁阶段**：执行镜像漏洞扫描、Dockerfile 合规检测，存在高危漏洞直接阻断流水线，禁止镜像推送；
5. **镜像推送阶段**：为镜像打上语义化版本与 Git Commit 哈希标签，推送到企业私有镜像仓库；
6. **自动部署阶段**：自动拉取最新镜像，更新测试环境的容器服务，执行自动化集成测试；
7. **生产部署阶段**：测试验证通过后，经过人工审批，执行生产环境的滚动更新部署。

### 4.3 企业级流水线实战示例

#### 4.3.1 GitLab CI 流水线（云原生主流方案）

GitLab CI 与 Git 仓库原生集成，配置简单，声明式语法，是目前中小企业最主流的 CI/CD 方案，通过`.gitlab-ci.yml`文件定义流水线全流程，完整实现上述标准化流程：

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - scan
  - push
  - deploy-test
  - deploy-prod

# 全局变量
variables:
  DOCKER_REGISTRY: "harbor.example.com"
  PROJECT_NAME: "backend-service"
  IMAGE_NAME: "${DOCKER_REGISTRY}/business/${PROJECT_NAME}"
  IMAGE_TAG: "${CI_COMMIT_SHORT_SHA}-${CI_PIPELINE_ID}"

# 1. 单元测试与代码扫描
unit-test:
  stage: test
  image: maven:3.9.6-eclipse-temurin-17
  script:
    - mvn clean test
    - mvn sonar:sonar
  only:
    - develop
    - main

# 2. 镜像构建
build-image:
  stage: build
  image: docker:26.0.0-dind
  services:
    - docker:26.0.0-dind
  script:
    - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
    - docker save ${IMAGE_NAME}:${IMAGE_TAG} -o image.tar
  artifacts:
    paths:
      - image.tar
  only:
    - develop
    - main

# 3. 镜像安全扫描
security-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - docker load -i image.tar
    - trivy image --severity HIGH,CRITICAL --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG}
  only:
    - develop
    - main

# 4. 镜像推送到私有仓库
push-image:
  stage: push
  image: docker:26.0.0-dind
  services:
    - docker:26.0.0-dind
  script:
    - docker load -i image.tar
    - docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWORD} ${DOCKER_REGISTRY}
    - docker push ${IMAGE_NAME}:${IMAGE_TAG}
    # 生产环境镜像额外打上latest-stable标签
    - if [ "${CI_COMMIT_BRANCH}" = "main" ]; then
        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest-stable;
        docker push ${IMAGE_NAME}:latest-stable;
      fi
  only:
    - develop
    - main

# 5. 测试环境自动部署
deploy-test:
  stage: deploy-test
  image: alpine:latest
  script:
    - apk add --no-cache openssh-client
    - ssh -o StrictHostKeyChecking=no test-server@test.example.com "cd /opt/${PROJECT_NAME} && docker compose pull && docker compose up -d"
  only:
    - develop
  environment:
    name: test

# 6. 生产环境审批部署
deploy-prod:
  stage: deploy-prod
  image: alpine:latest
  script:
    - apk add --no-cache openssh-client
    - ssh -o StrictHostKeyChecking=no prod-server@prod.example.com "cd /opt/${PROJECT_NAME} && docker compose pull && docker compose up -d"
  only:
    - main
  when: manual  # 手动审批触发
  environment:
    name: production
```

#### 4.3.2 Jenkins Pipeline 流水线（企业级传统方案）

Jenkins 是企业级 DevOps 的主流平台，支持高度定制化的流水线配置，通过声明式 Jenkinsfile 实现流水线定义，适配复杂的企业级权限体系、多环境部署与审批流程，核心逻辑与 GitLab CI 完全一致，仅语法格式不同。

### 4.4 生产级 CI/CD 最佳实践

1. **构建缓存优化**：合理设计 Dockerfile，将变更频率低的依赖安装步骤放在前部，最大化利用 Docker 构建缓存，大幅提升构建速度；配置 CI/CD 节点的镜像缓存，避免重复拉取基础镜像；
2. **镜像版本管理**：严禁使用`latest`标签，所有镜像必须使用`语义化版本+Git Commit哈希`作为标签，保证版本可追溯、可回滚；
3. **配置与代码分离**：应用配置通过环境变量、配置中心注入，严禁将环境配置硬编码到镜像中，实现镜像一次构建全环境复用；
4. **严格的门禁规则**：建立多层级门禁体系，单元测试不通过、代码覆盖率不达标、存在高危漏洞的镜像，禁止进入后续环节，从源头保障交付质量；
5. **渐进式部署**：生产环境采用滚动更新、蓝绿部署、金丝雀发布等渐进式部署策略，避免全量更新导致的业务中断，实现故障快速回滚；
6. **全流程可观测**：流水线执行过程、构建结果、部署状态全量可视化，异常情况自动触发告警，实现交付过程的全链路可观测。
