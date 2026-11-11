# 14 Kubernetes 核心与实践

> 本章目标：判断**何时需要 K8s**，建立"**声明期望状态 + 控制循环**"的心智模型，读懂并审查 AI 生成的 K8s YAML（Deployment、Service、Gateway API、ConfigMap / Secret、HPA、PDB），把**资源限额、探针、优雅停机、滚动更新**配对，并能按状态快速排障。
>
> 版本基线：Kubernetes 1.37（2026 年 8 月发布，社区同时维护 1.35~1.37 三个小版本）；Gateway API v1.7（2026 年 10 月）；社区版 Ingress-NGINX 控制器已于 2026 年 3 月退役；Helm 4（2025 年 11 月发布）。托管集群的可用版本以云厂商为准。
>
> 边界：部署形态选型见 [15 章](./15-部署架构与云.md#一部署形态选型)；探针语义与应用侧优雅停机见 [08 章](../L2-核心能力/08-并发与可靠性.md#七优雅停机)；镜像构建见 [13 章](./13-Docker与Compose.md)；流水线、GitOps 与发布策略见 [16 章](./16-CICD与发布策略.md)；模型推理集群见 [企业级 03 章](../../企业级AI应用开发实战/03-模型层-选型私有化推理与微调.md#369-从单机到集群)。

---

## 一、何时需要 K8s

### 1.1 需要与不需要

| 信号 | 倾向 |
|---|---|
| 服务数量少（API + Worker），单机或双机就能承载 | **不需要**：Compose 或托管容器服务（Cloud Run、SAE、ECS Fargate 等） |
| 团队没有人能负责集群升级、网络、存储问题 | **不需要**：再多的功能也抵不过凌晨三点没人会修 |
| 多个服务、多个团队，需要统一的部署、配置、伸缩、权限模型 | 需要 |
| 需要按队列积压、GPU 资源等精细调度和伸缩 | 需要 |
| 公司已有平台团队维护的集群 | 直接用，不要另起炉灶 |

> 🎯 **落地要点**：K8s 的价值在于**标准化**——一套 YAML 描述所有服务的部署、网络、配置与伸缩。代价是概念多、故障面广。即使确定要用，也优先**托管 K8s**（ACK、TKE、CCE、EKS、GKE、AKS），控制平面的高可用与升级交给云厂商。

### 1.2 本地实验环境

| 工具 | 原理 | 特点 |
|---|---|---|
| **kind** | 用 Docker 容器充当节点 | 启动快，适合 CI 与多节点实验 |
| **k3d** | 在 Docker 中运行轻量发行版 k3s | 资源占用低，自带简单的负载均衡 |
| **minikube** | 虚拟机或容器中的单节点集群 | 插件丰富（metrics-server、ingress 等） |
| **OrbStack** | macOS 上的容器与轻量 K8s | 本地构建的镜像可直接被集群使用，Mac 用户上手成本最低 |
| Docker Desktop 内置 K8s | 一键开启 | 与 Docker Desktop 绑定 |

本地集群用来练习 YAML、探针、滚动更新和排障命令；不要用来评估性能。

---

## 二、架构与控制循环

### 2.1 组件直觉

```mermaid
graph TB
    U["kubectl / CI / GitOps"] -->|提交期望状态| API
    subgraph CP["控制平面（托管 K8s 中由云厂商运维）"]
        API[kube-apiserver<br/>唯一入口，鉴权与校验] <--> ETCD[(etcd<br/>集群状态存储)]
        SCH[scheduler<br/>为 Pod 选择节点] <--> API
        CM[controller-manager<br/>各类控制器] <--> API
    end
    subgraph N["工作节点（多台）"]
        KL[kubelet<br/>按分配启动容器、执行探针] <--> API
        KL --> CR[容器运行时<br/>containerd]
        KP[kube-proxy / CNI<br/>Service 转发与 Pod 网络]
    end
```

所有组件都只和 apiserver 通信：提交 YAML 只是把"期望状态"写进 etcd，真正干活的是各组件在各自的循环里**观察差异、采取行动**。

### 2.2 声明式与控制循环

```mermaid
graph LR
    A["期望状态<br/>replicas: 3"] --> C{控制器比较}
    B["实际状态<br/>当前 2 个 Pod 运行"] --> C
    C -->|"有差异"| D[采取行动<br/>创建 1 个 Pod]
    D --> B
    C -->|"一致"| E[等待下一次变化]
```

前端类比：React 中你声明 `UI = f(state)`，由框架负责把 DOM 调和成期望的样子；K8s 中你声明"要 3 个副本、用这个镜像"，由控制器负责把集群调和成期望的样子。两个推论：

- **不要手动修改由控制器管理的对象**。直接删一个 Pod，Deployment 会立刻补一个；`kubectl edit` 改了线上配置而不改 Git，下次发布就被覆盖
- **"已提交"不等于"已生效"**。`kubectl apply` 成功只代表 apiserver 接受了期望状态，要用 `kubectl rollout status` 等命令确认实际状态收敛

---

## 三、核心对象

### 3.1 对象全景

| 对象 | 作用 | 类比 |
|---|---|---|
| **Pod** | 最小调度单位，一个或多个共享网络与卷的容器 | 一个"逻辑主机"上的进程组 |
| **ReplicaSet** | 维持 N 个相同 Pod | 通常不直接使用 |
| **Deployment** | 管理 ReplicaSet，提供滚动更新与回滚 | 无状态服务的标准载体 |
| **Service** | 给一组 Pod 提供稳定的虚拟 IP 与 DNS 名 | 内部负载均衡器 |
| **Gateway / HTTPRoute** | 集群入口的 L7 路由（新标准） | 反向代理配置 |
| **Ingress** | 旧的入口 API（功能冻结） | 同上 |
| **ConfigMap / Secret** | 配置 / 敏感配置 | 环境变量文件 |
| **Namespace** | 资源分组、权限与配额边界 | 项目 / 环境隔离 |
| **StatefulSet** | 有稳定名称与独立存储的有状态副本 | 自建数据库、消息队列 |
| **DaemonSet** | 每个节点运行一个 Pod | 日志采集、监控 agent |
| **Job / CronJob** | 运行到完成的任务 / 定时任务 | 迁移、批处理、定时对账 |
| **PV / PVC / StorageClass** | 持久卷 / 卷申请 / 卷的供给方式 | 云盘 |

### 3.2 Pod、Deployment 与 Service 的关系

```mermaid
graph LR
    GW["Gateway + HTTPRoute<br/>api.example.com"] --> SVC["Service chat-api<br/>ClusterIP，按标签选择 Pod"]
    DEP[Deployment chat-api] --> RS1["ReplicaSet v1.4.2"]
    RS1 --> P1[Pod] & P2[Pod] & P3[Pod]
    SVC -. "selector: app=chat-api" .-> P1 & P2 & P3
```

- Pod 是**一次性**的：重建后 IP 会变，所以访问一律通过 Service
- Service 与 Pod 之间**只靠标签关联**。标签写错（`app: chat-api` vs `app: chatapi`）时一切看起来都正常，只是没有流量，这是 AI 生成 YAML 的高发错误
- Service 背后的实际端点记录在 **EndpointSlice** 中；旧的 Endpoints API 自 1.33 起已标记弃用

| Service 类型 | 访问范围 | 用途 |
|---|---|---|
| **ClusterIP**（默认） | 仅集群内部 | 服务间调用；对外流量经 Gateway 进入 |
| NodePort | 每个节点的固定端口（默认 30000~32767） | 实验环境或自建负载均衡对接 |
| LoadBalancer | 云厂商为其创建外部负载均衡 | 非 HTTP 协议对外暴露；HTTP 一般统一走 Gateway |

集群内 DNS：`<service>.<namespace>.svc.cluster.local`，同一 Namespace 内直接写 `chat-api` 即可。

> ⚠️ **注意**：Pod 内 `/etc/resolv.conf` 默认 `ndots:5`，访问 `api.openai.com` 这类外部域名时，会先依次拼接集群搜索域查询多次失败后才查原始域名，增加解析延迟和 DNS 压力。频繁调用外部 API 的服务可在域名末尾加点写成完全限定域名（`api.openai.com.`），或在 Pod 的 `dnsConfig` 中调低 `ndots`。

### 3.3 集群入口：Ingress 与 Gateway API

**现状**：Ingress API 仍受支持但已**功能冻结**；使用最广的社区版 **Ingress-NGINX 控制器已于 2026 年 3 月退役**，仓库归档，不再发布安全补丁（F5 维护的另一个 NGINX Ingress Controller 是不同项目，不受影响）。新集群应使用 **Gateway API**，存量迁移可借助官方的 ingress2gateway 工具。

> ⚠️ **注意**：AI 生成的 K8s 配置大量带有 `ingressClassName: nginx` 和 `nginx.ingress.kubernetes.io/*` 注解（如 `proxy-read-timeout`、`proxy-buffering`），这是基于已退役控制器的写法。审查时确认集群实际使用的入口控制器；新写的路由用 Gateway API。

Gateway API 把入口配置按**角色拆分**，解决了 Ingress 时代"所有能力都塞进注解"的问题：

| 资源 | 负责人 | 内容 |
|---|---|---|
| **GatewayClass** | 基础设施提供方 | 使用哪个实现（Envoy Gateway、Istio、Cilium、NGINX Gateway Fabric、Traefik、云厂商控制器等） |
| **Gateway** | 平台 / 运维团队 | 监听端口、TLS 证书、允许哪些 Namespace 挂载路由 |
| **HTTPRoute** | 应用团队 | 域名、路径匹配、后端 Service、超时、重试、流量拆分 |

超时、重试、按权重拆分流量（金丝雀，[16 章](./16-CICD与发布策略.md#六发布策略)）都是 HTTPRoute 的标准字段，不再依赖某个实现的私有注解。各实现对扩展功能的支持度不同，以其一致性测试报告为准。

### 3.4 有状态与批处理工作负载

- **StatefulSet**：Pod 名称固定（`pg-0`、`pg-1`）、每个副本有独立 PVC、按序启停。能在 K8s 里跑数据库，但备份、升级、故障切换都要自己负责；中小团队**数据库优先用托管服务**（[15 章](./15-部署架构与云.md#六数据层托管还是自建)）
- **Job**：数据库迁移、一次性数据修复。迁移随发布执行的方式见 [16 章 5.4](./16-CICD与发布策略.md#54-迁移在哪里执行)
- **CronJob**：设置 `concurrencyPolicy: Forbid` 防止上一次没跑完下一次又开始；任务本身要幂等，因为调度器可能漏跑或重跑
- **PV / PVC**：Deployment 的多个副本挂同一块只支持 `ReadWriteOnce` 的云盘会失败。用户上传的文件放对象存储，而不是 PVC

---

## 四、完整示例：部署一个 AI 对话服务

以下是一个 FastAPI 流式对话服务的完整清单，逐段说明设计意图。

### 4.1 Namespace、ConfigMap 与 Secret

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-chat
  labels:
    gateway-access: public            # 被 Gateway 允许挂载路由的标记（见 4.4）
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: chat-api-config
  namespace: ai-chat
data:
  LOG_LEVEL: info
  LLM_TIMEOUT_SECONDS: "120"
  OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector.observability:4318
---
# 仅演示结构：真实环境由 External Secrets 等从密钥管理服务同步生成，不提交到 Git（第八节）
apiVersion: v1
kind: Secret
metadata:
  name: chat-api-secret
  namespace: ai-chat
type: Opaque
stringData:
  DATABASE_URL: postgresql://app:CHANGE_ME@pg.internal:5432/app
  LLM_API_KEY: CHANGE_ME
```

### 4.2 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-api
  namespace: ai-chat
spec:
  replicas: 3                          # 启用 HPA 后由 HPA 管理；GitOps 中通常省略此字段避免互相覆盖
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: chat-api }     # 必须与下方 template 的标签一致，创建后不可修改
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                      # 先多起 1 个新 Pod
      maxUnavailable: 0                # 新 Pod 就绪前不减少旧 Pod，容量不下降
  template:
    metadata:
      labels: { app: chat-api }
    spec:
      terminationGracePeriodSeconds: 90   # ≥ preStop + 应用停机超时（流式对话需更长，见 6.3）
      securityContext:
        runAsNonRoot: true                 # 要求镜像 USER 为数字 UID
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:           # 副本尽量分散到不同可用区
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: chat-api }
      containers:
        - name: app
          image: ghcr.io/acme/chat-api:1.4.2   # 不可变版本号，禁止 latest
          ports:
            - { name: http, containerPort: 8000 }
          envFrom:
            - configMapRef: { name: chat-api-config }
            - secretRef: { name: chat-api-secret }
          env:
            - { name: WEB_CONCURRENCY, value: "2" }   # 显式指定 worker 数（13 章 3.3）
          resources:
            requests: { cpu: 500m, memory: 768Mi }    # 调度依据，也是 HPA 利用率的分母
            limits: { memory: 768Mi }                 # 内存 limit = request；CPU 不设 limit 的取舍见 5.2
          startupProbe:                     # 启动完成前暂停 liveness，最多给 24 × 5 = 120 秒
            httpGet: { path: /healthz, port: http }
            periodSeconds: 5
            failureThreshold: 24
          livenessProbe:                    # 只检查进程自身是否卡死，不检查任何外部依赖
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:                   # 是否能接流量：初始化完成、关键依赖可达、未处于停机中
            httpGet: { path: /readyz, port: http }
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
          lifecycle:
            preStop:
              sleep: { seconds: 5 }         # 原生 sleep 动作，等待端点摘除传播，不依赖镜像中的 sleep
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: tmp, mountPath: /tmp }   # 根文件系统只读时，给需要写临时文件的路径挂 emptyDir
      volumes:
        - name: tmp
          emptyDir: {}
```

### 4.3 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: chat-api
  namespace: ai-chat
spec:
  selector: { app: chat-api }          # 与 Pod 标签一致
  ports:
    - name: http
      port: 80                         # Service 端口
      targetPort: http                 # 引用容器端口名，改容器端口时不必同步修改这里
```

### 4.4 Gateway 与 HTTPRoute

```yaml
# 平台团队维护：一个共享的对外 Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: gateway-system
spec:
  gatewayClassName: envoy-gateway      # 取决于集群安装的实现
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-example-com # 证书 Secret，通常由 cert-manager 签发续期
      allowedRoutes:
        namespaces:
          from: Selector               # 只允许打了标记的 Namespace 挂载路由
          selector:
            matchLabels: { gateway-access: public }
---
# 应用团队维护：本服务的路由
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: chat-api
  namespace: ai-chat
spec:
  parentRefs:
    - name: public-gateway
      namespace: gateway-system
  hostnames: ["api.example.com"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /v1/chat }
      timeouts:
        request: 300s                  # 流式对话需要更长的超时
      backendRefs:
        - { name: chat-api, port: 80 }
    - matches:
        - path: { type: PathPrefix, value: / }
      timeouts:
        request: 30s
      backendRefs:
        - { name: chat-api, port: 80 }
```

> ⚠️ **注意**：`timeouts.request` 对**流式响应**的计时语义、空闲超时、响应缓冲行为，各 Gateway 实现存在差异；网关前面的云负载均衡还有自己的空闲超时。上线前必须在**生产链路**上实测一次持续数分钟的流式回答（SSE 的反代要点见 [01 章](../L1-认知切换/01-网络与HTTP-服务端视角.md)）。

### 4.5 HPA 与 PDB

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chat-api
  namespace: ai-chat
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: chat-api }
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 60 }   # 相对 requests.cpu 的百分比
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # 缩容前观察 5 分钟，避免抖动
      policies:
        - { type: Percent, value: 20, periodSeconds: 60 }   # 每分钟最多缩 20%，给长连接留排空时间
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chat-api
  namespace: ai-chat
spec:
  minAvailable: 2                        # 节点维护驱逐时至少保留 2 个可用 Pod
  selector:
    matchLabels: { app: chat-api }
```

---

## 五、资源 requests / limits

### 5.1 两个数字的含义

| | requests | limits |
|---|---|---|
| 用途 | **调度依据**：节点剩余可分配资源 ≥ requests 才会被调度上去 | **运行时上限**：由 cgroup 强制执行 |
| CPU 超出 | — | 被**限流（throttling）**，进程变慢但不会被杀 |
| 内存超出 | — | 被内核 **OOM kill**，状态显示 `OOMKilled`，退出码 137 |
| 不设置的后果 | 调度器认为它不占资源，节点被超卖 | 一个泄漏的 Pod 可以吃光节点内存，波及同节点其他 Pod |

### 5.2 QoS 等级与配置建议

| QoS | 条件 | 节点内存紧张时 |
|---|---|---|
| **Guaranteed** | 每个容器 CPU 与内存的 requests = limits | 最后被驱逐 |
| **Burstable** | 至少设置了部分 requests / limits | 按超出 requests 的程度排序驱逐 |
| **BestEffort** | 完全未设置 | **最先被驱逐** |

配置建议：

- **内存：limit 等于 request**。内存不可压缩，超卖内存等于随机 OOM
- **CPU：必须设 request；limit 取决于团队策略**。CPU limit 会在突发时触发 CFS 限流，表现为 P99 延迟毛刺而 CPU 利用率看起来不高；不设 limit 则允许使用节点空闲 CPU，但需要配合 Namespace 的资源配额防止滥用。对延迟敏感的在线服务，不少团队选择不设 CPU limit
- **按实测设置**：先按压测或线上观察的 P95 用量设置 request，再用 `kubectl top` 与监控持续校准
- 1.35 起**原地调整 Pod 资源**（in-place resize）已 GA，可在不重建 Pod 的情况下修改 requests / limits；但 JVM 堆等启动时确定的参数不会随之变化

### 5.3 三栈运行时在容器限额下的行为

| 运行时 | 内存 | CPU |
|---|---|---|
| **Java** | 识别 cgroup 内存限额，但默认堆只占 25%；用 `-XX:MaxRAMPercentage=70~75` | 可用核数按 **CPU limit** 推算；不设 limit 时看到整个节点的核数，GC 线程、`ForkJoinPool` 偏大，可用 `-XX:ActiveProcessorCount` 显式指定 |
| **Node** | V8 堆上限按可用内存推算，具体行为随版本变化；用 `--max-old-space-size` 显式设为容器内存的 60%~75%，并用 `v8.getHeapStatistics().heap_size_limit` 验证 | 单线程事件循环，CPU 限流直接表现为延迟升高；libuv 线程池默认 4 个线程（`UV_THREADPOOL_SIZE`） |
| **Python** | 无堆上限概念，按 worker 数 × 单 worker 内存估算 | `os.cpu_count()` 返回**节点核数**，不感知配额；worker 数显式配置 |
| **Go** | 可用 `GOMEMLIMIT` 设置软上限 | 1.25 起 `GOMAXPROCS` 默认遵循 CPU limit（不看 requests） |

> ⚠️ **注意**：AI 生成的 YAML 常见两种极端：完全不写 `resources`（BestEffort，最先被驱逐，也无法做 HPA），或者照抄 `limits: {cpu: 100m, memory: 128Mi}` 这类示例值（JVM 或 Python AI 服务启动即 OOM，CPU 严重限流导致探针超时、反复重启）。

---

## 六、探针与优雅停机

### 6.1 三种探针的配置要点

三种探针的语义与"liveness 不查外部依赖"的原则见 [08 章第八节](../L2-核心能力/08-并发与可靠性.md#八健康检查)。这里只讲配置：

| 参数 | 默认值 | 建议 |
|---|---|---|
| `periodSeconds` | 10 | readiness 可以更短（5s），让摘除和恢复更及时 |
| `timeoutSeconds` | **1** | 偏短；Python / Java 在 GC 或负载高时偶尔超过 1 秒，建议 2~3 秒 |
| `failureThreshold` | 3 | liveness 宁可宽松，误杀的代价比晚发现卡死更大 |
| `initialDelaySeconds` | 0 | 启动慢的服务**用 startupProbe**，不要用很大的 initialDelay 硬等 |

### 6.2 常见错误配置

```yaml
# 反例：liveness 与 readiness 共用一个会检查数据库和模型 API 的 /health
livenessProbe:
  httpGet: { path: /health, port: 8000 }   # /health 内部 ping 了 Postgres、Redis、LLM API
  initialDelaySeconds: 60
  timeoutSeconds: 1
```

后果链：模型 API 短暂超时 → 所有 Pod 的 `/health` 同时失败 → kubelet **同时重启全部 Pod** → 重启期间无可用实例，全站 503 → 新 Pod 同时启动、同时建连，冲击数据库 → 一次外部抖动被放大为全面故障。

其他高发错误：

| 错误 | 后果 |
|---|---|
| readiness 检查非关键依赖（如模型 API） | 下游抖动时所有 Pod 同时摘除，Service 没有任何端点 |
| 健康检查接口走了鉴权、限流或数据库会话中间件 | 探针被 401 / 429，或在连接池耗尽时失败 |
| 健康检查在 CPU 密集请求的同一个事件循环 / 线程池中处理 | 高负载时探针超时，正在干活的 Pod 被重启，负载转移到其他 Pod 形成雪崩 |
| 没有 startupProbe，启动要 90 秒却只给了 30 秒的 initialDelay | 永远启动不起来，进入 CrashLoopBackOff |

### 6.3 停机时序与长连接 / SSE

Pod 删除时，"从端点中摘除"与"执行 preStop → 发送 SIGTERM"**并行发生**，完整时序见 [08 章 7.2](../L2-核心能力/08-并发与可靠性.md#七优雅停机)。K8s 侧需要配对的数字：

```text
terminationGracePeriodSeconds  ≥  preStop 等待  +  应用停机超时  +  余量
            90s                       5s              60~75s         10s
```

- `terminationGracePeriodSeconds` 的倒计时**从 preStop 开始时就已经开始**，preStop 时间不是额外赠送的
- 应用的停机超时（Spring 的 `timeout-per-shutdown-phase`、Uvicorn 的 `--timeout-graceful-shutdown`、Node 自己的强制退出计时器）必须**小于**宽限期减去 preStop
- **SSE / 长流式对话**：宽限期不可能无限长。可靠的做法是应用收到 SIGTERM 后**通知客户端重连**（发送重连事件后关闭），前端基于消息 ID 断点续传到新 Pod；宽限期覆盖"大多数流能自然结束"的时长即可
- **长连接不会自动再均衡**：扩容后，已建立的 SSE / WebSocket / HTTP/2 连接仍停留在旧 Pod 上，新 Pod 只接到新连接。HPA 扩容对存量长连接的负载缓解有延迟，这也是 4.5 中限制缩容速度的原因
- 队列 Worker 的宽限期要覆盖单个任务的最长执行时间，或保证任务可被中断后安全重试（[07 章](../L2-核心能力/07-异步任务与消息队列.md)）

---

## 七、发布与扩缩容

### 7.1 滚动更新与回滚

| 配置 | 行为 | 适合 |
|---|---|---|
| `maxSurge: 25%, maxUnavailable: 25%`（默认） | 边加边减，速度快 | 容量有富余的服务 |
| `maxSurge: 1, maxUnavailable: 0` | 先加后减，容量不下降 | 在线对话服务（本章示例） |
| `maxSurge: 0, maxUnavailable: 1` | 不额外占用资源 | 资源紧张、可接受短暂降容的内部服务 |

```bash
kubectl -n ai-chat rollout status deploy/chat-api --timeout=5m   # 等待并判断发布是否成功
kubectl -n ai-chat rollout history deploy/chat-api
kubectl -n ai-chat rollout undo deploy/chat-api                  # 回到上一版本
kubectl -n ai-chat rollout undo deploy/chat-api --to-revision=7
kubectl -n ai-chat rollout restart deploy/chat-api               # 按滚动策略重启全部 Pod（如轮换已挂载的配置）
```

- 新 Pod 的 readiness 一直不通过时，滚动更新会**卡住而不是继续**（`maxUnavailable: 0` 时旧版本保持服务），这正是 readiness 设计正确的价值
- 使用 GitOps 时，`rollout undo` 只是临时止血，事后必须在 Git 中回退，否则下次同步会把坏版本带回来（[16 章第七节](./16-CICD与发布策略.md#七回滚)）
- 蓝绿、金丝雀等进阶策略见 [16 章](./16-CICD与发布策略.md#六发布策略)

### 7.2 HPA

HPA 的核心公式：`期望副本数 = ceil(当前副本数 × 当前指标值 / 目标值)`。

| 指标类型 | 例子 | 适合 |
|---|---|---|
| Resource | CPU / 内存利用率（相对 requests） | CPU 密集型服务 |
| Pods / Object（自定义指标） | 每个 Pod 的在途请求数、活跃 SSE 连接数 | **AI 对话服务** |
| External | 队列积压长度、云监控指标 | 异步 Worker |

> 🎯 **落地要点**：**AI 对话服务大部分时间在等待模型 API**，CPU 很低，按 CPU 扩容几乎永远不会触发，而此时并发流数量早已打满连接池或事件循环。更合适的指标是**每个 Pod 的在途请求数 / 活跃流数**（通过 Prometheus Adapter 暴露为自定义指标）。文档入库等异步 Worker 按**队列积压**伸缩。

**KEDA**（CNCF 毕业项目）基于事件源驱动伸缩：内置 Redis、Kafka、RabbitMQ、Prometheus、云队列等触发器，并支持**缩容到零**，适合"白天有任务、夜间无任务"的 Worker。原生 HPA 缩容到零的能力仍在通过特性开关逐步推进，生产中目前多用 KEDA。

### 7.3 PDB

PodDisruptionBudget 只防护**主动驱逐**（节点排空、集群升级、自动缩容节点），不防护进程崩溃或节点宕机。

> ⚠️ **注意**：`minAvailable` 等于副本数（如 3 副本配 `minAvailable: 3`）时，节点永远无法被排空，集群升级卡死。用 `minAvailable` 小于副本数，或改用 `maxUnavailable: 1`。

---

## 八、配置与密钥管理

ConfigMap 与 Secret 的两种使用方式行为不同：

| 方式 | 更新后 | 注意 |
|---|---|---|
| 环境变量（`env` / `envFrom`） | **不会更新**，需要重启 Pod | 简单，最常用 |
| 卷挂载 | 文件内容在一段延迟后自动更新 | 应用需要监听文件变化或定期重读；用 `subPath` 挂载的文件**不会更新** |

**Secret 并不加密**：

- Secret 的值只是 **Base64 编码**，任何能 `kubectl get secret -o yaml` 的人都能解码
- 默认在 etcd 中明文存储，需要开启**静态加密**（托管 K8s 通常可对接云 KMS）
- 权限上，能在某个 Namespace 中**创建 Pod** 的人，可以挂载并读取该 Namespace 中的所有 Secret；RBAC 要按 Namespace 收紧
- **不要把 Secret YAML 提交到 Git**，哪怕是私有仓库

| 方案 | 做法 | 适合 |
|---|---|---|
| **External Secrets Operator** | 从云密钥管理服务 / Vault 同步生成 K8s Secret；2025 年曾短暂暂停发版，之后补充维护者恢复活跃 | 已使用云密钥管理服务（**推荐**） |
| **Sealed Secrets** | 用集群公钥加密后的密文可以安全提交 Git，由集群内控制器解密 | 纯 GitOps、没有外部密钥服务的小团队 |
| **Secrets Store CSI Driver** | 以卷的形式直接挂载外部密钥，可不落 K8s Secret | 合规要求密钥不进入 etcd |

密钥轮换、模型 API Key 的分级管理见 [06 章](../L2-核心能力/06-认证授权与Web安全.md)。

---

## 九、Helm、Kustomize 与 GitOps

| | Helm | Kustomize |
|---|---|---|
| 思路 | **模板 + values**，打包为可版本化的 chart | **无模板**，基础 YAML + 按环境的补丁叠加 |
| 适合 | 分发给他人安装的软件；复杂的参数化部署 | 自有服务的多环境差异（dev / staging / prod） |
| 学习成本 | Go 模板语法，可读性随复杂度下降 | 低，`kubectl apply -k` 内置支持 |
| 发布管理 | 有 release 概念，支持升级、回滚、钩子（如迁移 Job） | 只生成 YAML，发布交给 GitOps 工具 |

**Helm 4** 的主要变化：默认使用 **Server-Side Apply**（字段归属由 apiserver 管理，与其他控制器冲突时显式报错而不是静默覆盖）、基于 kstatus 的就绪等待、插件系统重做；`--atomic` 更名为 `--rollback-on-failure`。Helm 3 的 chart 仍可使用，升级前确认 CI 中的参数与插件兼容。

Kustomize 的目录约定：

```text
deploy/
├── base/                    # Deployment、Service、HTTPRoute 等公共部分
│   └── kustomization.yaml
└── overlays/
    ├── staging/             # 副本数、资源、域名的差异补丁
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

**GitOps**（Argo CD / Flux）：集群内的控制器持续把集群状态调和为 Git 仓库中声明的状态，**Git 是唯一事实来源**，变更通过 PR 审查、回滚通过 Git 回退。它把第二节的控制循环思想延伸到了发布流程。流水线如何与 GitOps 衔接见 [16 章](./16-CICD与发布策略.md)。

---

## 十、GPU 工作负载与模型推理部署简述

自建模型推理（vLLM 等）是 AI 团队上 K8s 的常见动因，与普通 Web 服务的差异：

| 方面 | 普通 Web 服务 | 模型推理服务 |
|---|---|---|
| 资源申请 | CPU / 内存 | `limits: { nvidia.com/gpu: 1 }`，由 NVIDIA device plugin / GPU Operator 提供；GPU 不可超卖 |
| 调度 | 任意节点 | GPU 节点通常带污点（taint），Pod 需配 `tolerations` 与节点选择 |
| 启动 | 秒级 | 加载数十 GB 权重需要数分钟：**startupProbe 必须足够宽松**，镜像与权重分离，权重放共享存储或节点本地缓存 |
| 伸缩指标 | CPU / 并发 | 请求排队长度、KV Cache 利用率、首 token 延迟；GPU 节点扩容慢且贵，伸缩要更保守 |
| 路由 | 轮询即可 | 前缀 / KV Cache 感知路由能显著提高缓存命中（Gateway API Inference Extension） |

**DRA（动态资源分配）** 的核心 API 已 GA，并在 1.36、1.37 中继续把优先级备选（"优先 H100，没有就 A100"）、与扩展资源写法兼容等能力推向稳定，正在取代"按整卡计数"的 device plugin 模型。推理引擎选型、显存估算与集群化方案见 [企业级 03 章第三节](../../企业级AI应用开发实战/03-模型层-选型私有化推理与微调.md#三私有化部署)。

---

## 十一、kubectl 排障手册

### 11.1 常用命令

```bash
kubectl config get-contexts && kubectl config current-context   # 先确认操作的是哪个集群
kubectl -n ai-chat get pods -o wide                              # 状态、重启次数、所在节点
kubectl -n ai-chat describe pod chat-api-7d9f-abcde              # 末尾 Events 是第一手线索
kubectl -n ai-chat events --for pod/chat-api-7d9f-abcde          # 只看该对象的事件
kubectl -n ai-chat get events --sort-by=.lastTimestamp | tail -30
kubectl -n ai-chat logs deploy/chat-api -f --since=10m
kubectl -n ai-chat logs chat-api-7d9f-abcde --previous           # 上一次崩溃前的日志（CrashLoopBackOff 必用）
kubectl -n ai-chat exec -it chat-api-7d9f-abcde -- sh            # 镜像有 shell 时
kubectl -n ai-chat debug -it chat-api-7d9f-abcde --image=busybox --target=app   # distroless 镜像：挂临时调试容器
kubectl -n ai-chat port-forward svc/chat-api 8000:80             # 本地直连 Service，绕过 Gateway 定位问题
kubectl -n ai-chat top pod --containers                          # 实际资源用量（需要 metrics-server）
kubectl -n ai-chat get endpointslices -l kubernetes.io/service-name=chat-api   # Service 有没有端点
kubectl -n ai-chat get httproute chat-api -o yaml                # 看 status.parents 中的 Accepted / ResolvedRefs 条件
kubectl auth can-i get secrets -n ai-chat                        # 权限排查
```

> ⚠️ **注意**：多集群用户的头号事故是**在生产集群上执行了以为是测试集群的命令**。用 kubectx / kubens 或在终端提示符中显示当前 context，危险操作前先 `current-context`。

### 11.2 按状态的排查路径

```mermaid
graph TD
    A[Pod 不正常] --> B{STATUS}
    B -->|Pending| C["describe 看 Events<br/>资源不足 / 污点 / PVC 未绑定"]
    B -->|"ImagePullBackOff"| D["镜像名与 tag / 私有仓库凭据 / 限流"]
    B -->|"CrashLoopBackOff"| E["logs --previous<br/>看崩溃前输出与退出码"]
    B -->|"Running 但 0/1 Ready"| F["readiness 失败原因<br/>依赖、端口、路径"]
    B -->|"Running 且 Ready 仍不通"| G["EndpointSlice 是否有端点<br/>Service selector / 端口 / HTTPRoute 状态"]
    E --> H{"Last State 为<br/>OOMKilled?"}
    H -->|是| I[调整内存 limit 或运行时堆参数]
    H -->|否| J[配置缺失 / 依赖不可达 / 探针误杀]
```

| 状态 | 排查命令 | 常见原因 |
|---|---|---|
| **Pending** | `describe pod` 的 Events | `Insufficient cpu/memory`（requests 过大或节点不足）、节点污点无容忍、`nodeSelector` 不匹配、PVC 未绑定 |
| **ContainerCreating** 长时间不动 | `describe pod` | 卷挂载失败（云盘仍挂在旧节点上）、CNI 分配 IP 失败 |
| **CreateContainerConfigError** | `describe pod` | 引用的 ConfigMap / Secret 或其中的 key 不存在 |
| **ImagePullBackOff / ErrImagePull** | `describe pod` 中的拉取错误 | tag 拼错或未推送、私有仓库缺少 `imagePullSecrets`、公共仓库限流、镜像已被删除（如 `bitnami/*`） |
| **CrashLoopBackOff** | `logs --previous`、`describe` 中的 `Last State` 与 `Exit Code` | 配置或环境变量缺失、连不上数据库、端口冲突、liveness 误杀；日志为 `exec format error` 时是**镜像架构不匹配**（[13 章第八节](./13-Docker与Compose.md#八多架构构建apple-silicon-与服务器)） |
| **OOMKilled**（退出码 137） | `describe pod` 的 `Last State: Terminated, Reason: OOMKilled` | 内存 limit 低于实际需要、JVM / Node 堆参数与 limit 不匹配、内存泄漏 |
| **Running 但 READY 0/1** | `describe` 中的 readiness 失败信息；`port-forward` 后手动请求 `/readyz` | 依赖不可达、探针路径或端口写错、应用监听 `127.0.0.1` 而非 `0.0.0.0` |
| **Service 不通 / 网关 503** | `get endpointslices`、`get httproute -o yaml` | selector 与 Pod 标签不一致、`targetPort` 写错、HTTPRoute 未被 Gateway 接受（Namespace 未被允许、跨 Namespace 引用缺授权） |
| **Terminating 卡住** | `get pod -o yaml` 看 `finalizers`；查 preStop | finalizer 对应的控制器不工作、preStop 卡住直到宽限期结束 |
| **Evicted** | `describe pod`、`describe node` 的 Conditions | 节点内存 / 磁盘压力；容器日志或临时文件写满节点磁盘 |
| **重启次数持续增加但状态 Running** | `describe` 看重启原因、监控探针失败率 | liveness 过严或检查了外部依赖、间歇性 OOM |

> 🎯 **落地要点**：排障顺序固定为 **`get` 看状态 → `describe` 看事件 → `logs --previous` 看崩溃前日志 → `get endpointslices` 看流量链路**。绝大多数问题在前三步就能定位。系统化的排障方法论见 [17 章](./17-可观测性与线上排障.md#六排障方法论)。

---

## 十二、配置审查清单

针对 AI 生成的 K8s YAML、Helm values 与 Kustomize 补丁。

### 工作负载

- [ ] 🔴 镜像使用不可变 tag 或 digest，没有 `latest`；镜像架构与节点架构一致
- [ ] 🔴 每个容器都设置了 `resources.requests`（CPU 与内存）和内存 `limits`，数值来自实测而非示例
- [ ] 🔴 `runAsNonRoot: true`、`allowPrivilegeEscalation: false`、`capabilities.drop: ["ALL"]`；没有 `privileged: true` 或 `hostNetwork` / `hostPath`（除非有明确理由）
- [ ] Deployment 的 `selector`、Pod 模板标签、Service `selector` 三者一致
- [ ] 运行时参数与内存 limit 配套：JVM `MaxRAMPercentage`、Node `--max-old-space-size`、Python worker 数显式配置
- [ ] 副本数 ≥ 2，配置了跨可用区的 `topologySpreadConstraints` 或反亲和
- [ ] 启用 HPA 时，GitOps 管理的 Deployment 中不再硬编码 `replicas`

### 探针与停机

- [ ] 🔴 liveness 不检查数据库、Redis、模型 API 等任何外部依赖
- [ ] 启动慢的服务使用 startupProbe，而不是很大的 `initialDelaySeconds`
- [ ] `timeoutSeconds` 不低于 2 秒；健康检查接口绕过鉴权、限流与重量级中间件
- [ ] 配置了 preStop 等待，且 `terminationGracePeriodSeconds` ≥ preStop + 应用停机超时 + 余量
- [ ] 流式对话服务的停机策略包含"通知客户端重连"，宽限期与 HTTPRoute 超时匹配

### 网络与入口

- [ ] 🔴 新写的入口配置使用 Gateway API；没有依赖已退役的 Ingress-NGINX 及其 `nginx.ingress.kubernetes.io/*` 注解
- [ ] Service 类型为 ClusterIP，对外统一经 Gateway；没有为数据库、Redis 创建 LoadBalancer
- [ ] Gateway 的 `allowedRoutes` 限定了可挂载路由的 Namespace
- [ ] 流式路由单独设置了超时，并已在生产链路上实测

### 配置与密钥

- [ ] 🔴 Secret 明文或 Base64 YAML 没有提交到 Git；密钥来自 External Secrets / Sealed Secrets / CSI Driver
- [ ] 通过环境变量注入的配置变更后有重启机制（如 Helm 中基于配置哈希的注解或 `rollout restart`）
- [ ] 没有用 `subPath` 挂载需要热更新的配置文件

### 发布与伸缩

- [ ] 在线服务的滚动更新为 `maxUnavailable: 0`（或经过评估的值），流水线中有 `rollout status` 判定成败
- [ ] HPA 指标与服务瓶颈匹配：AI 对话服务按在途请求 / 活跃流数，Worker 按队列积压，而不是一律 CPU
- [ ] HPA 设置了缩容稳定窗口与速率限制
- [ ] PDB 的 `minAvailable` 小于副本数，不会阻塞节点排空
- [ ] CronJob 设置了 `concurrencyPolicy: Forbid`（或评估过并发影响），任务幂等

---

## 本章小结

- K8s 的价值是**标准化**，代价是复杂度；小团队优先托管容器服务，确需 K8s 时优先托管集群
- 核心心智是**期望状态 + 控制循环**：不要手改控制器管理的对象，"apply 成功"不等于"已生效"
- Pod 是一次性的，流量经 **Service（靠标签选择）→ Gateway API** 进入；Ingress-NGINX 已退役，新入口用 Gateway API
- **内存 limit = request**，CPU 必设 request；JVM、Node、Python 都要显式配置与限额匹配的参数
- 探针按职责拆分：**liveness 只看自身**，readiness 看能否接流量，慢启动用 startupProbe
- 停机时间要配对：宽限期 ≥ preStop + 应用停机超时；长流式连接靠"通知重连"，而不是无限长的宽限期
- AI 对话服务的瓶颈是并发流而非 CPU，HPA 用**在途请求数**，Worker 用**队列积压**（KEDA）
- Secret 只是 Base64；排障按 **get → describe → logs --previous → endpointslices** 的顺序

## 练习

1. 用 kind、k3d 或 OrbStack 在本地起一个集群，安装任一 Gateway API 实现，部署第四节的完整清单（把镜像换成你自己的流式对话服务），通过 HTTPRoute 访问，并用 `curl -N` 验证一个持续 2 分钟的流式回答不被中断
2. 把 liveness 改成检查数据库的反例版本，部署 3 个副本后停掉数据库，观察 Pod 的重启次数与 Service 端点变化；改回正确配置后重复实验，对比两者的故障表现
3. 对服务发起持续的流式请求，同时执行 `kubectl rollout restart`，分别在 `terminationGracePeriodSeconds: 10` 与 `90`、有无 preStop 的组合下统计客户端中断次数
4. 为文档入库 Worker 配置 KEDA，按 Redis 队列长度伸缩并支持缩容到零；投递 500 个入库任务，观察扩容速度与任务完成后的缩容过程
