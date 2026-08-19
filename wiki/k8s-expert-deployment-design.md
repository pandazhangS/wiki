# Matrix 消息桥下游专家 Worker 的 K8s 部署方案

> 状态: 设计稿 v1
> 日期: 2026-08-18
> 范围: 无状态、按配置区分的专家 worker 在 k8s 上的部署、流量路由与自动扩缩

## 1. 背景与目标

Matrix 消息桥的下游是一批**无状态、由特定配置区分**的专家 worker。所有专家共用同一个基座镜像,仅通过挂载不同的配置文件派生为不同专家。

设计目标:

1. **一镜像多专家**: 基座镜像 + 每专家一份配置(ConfigMap),不维护多套镜像。
2. **轻量路由标记**: 消息桥只需极轻量地标记请求去往哪个专家,不感知 k8s 内部拓扑。
3. **弹性伸缩**: 专家 Pod 按负载自动扩缩,空闲专家不长期占用资源。
4. **交付物**: Helm chart,专家列表声明式管理,增删专家只改 values。

## 2. 总体架构

```
                    ┌─────────────────────────── k8s 集群 ───────────────────────────┐
                    │                                                               │
 Matrix 消息桥 ────►│  Higress 网关 (Envoy)                                          │
 (HTTP + header     │    │  X-Expert-Id: <id>  →  HTTPRoute header 匹配              │
  X-Expert-Id)      │    ▼                                                           │
                    │  expert-<id> Service ──► expert-<id> Deployment (N 副本)        │
                    │                              │ 挂载 ConfigMap(expert-<id>-config)│
                    │                              ▼                                  │
                    │                         基座镜像 worker Pod                     │
                    │                                                               │
                    └───────────────────────────────────────────────────────────────┘
```

核心决策:

- **一个专家 = 一个 Deployment + 一个 ConfigMap + 一个 Service + 一个 HPA + 一条 HTTPRoute**。
  Deployment 的所有 Pod 共享同一 Pod 模板,因此"每 Pod 挂不同配置"在原生 Deployment 语义下不可行,必须按专家拆 Deployment。
- **路由层复用现有 Higress 网关(Envoy)**:每个专家一条 Gateway API `HTTPRoute`,按 `X-Expert-Id` header 精确匹配路由到对应 Service。不引入额外的 router 组件,专家列表的增删就是 HTTPRoute 的增删。

## 3. 专家 Pod 设计(一镜像多专家)

### 3.1 资源对象

| 对象 | 命名 | 说明 |
|---|---|---|
| Deployment | `expert-<id>` | 基座镜像,挂载专家配置 |
| ConfigMap | `expert-<id>-config` | 专家专属配置,挂载到 `/etc/expert/` |
| Service | `expert-<id>` | 集群内访问入口,selector 按 `expert-id` 标签 |
| HPA | `expert-<id>` | 每专家独立扩缩 |
| HTTPRoute | `expert-<id>` | Higress 网关路由,按 `X-Expert-Id` header 匹配到本专家 Service |

### 3.2 关键标签

所有专家 Pod 打上 `expert-id: <id>` 标签,作为 Service selector、HPA 目标、监控指标的公共维度。

### 3.3 配置挂载

- 配置以 ConfigMap 形式挂载到固定路径(如 `/etc/expert/config.yaml`),worker 启动时读取。
- 配置变更通过 Helm 升级触发滚动更新:Deployment 模板里注入 ConfigMap 内容 checksum 注解(`checksum/config`),配置一变即滚动重启该专家,不重启其他专家。

### 3.4 优雅退出

- 配置 `terminationGracePeriodSeconds`(默认 30s)+ `preStop` sleep,给 worker 排空在途请求。
- worker 需监听 SIGTERM 并优雅关闭(HTTP server shutdown)。

## 4. 流量路由:消息桥的"轻量标记"

### 4.1 推荐方案:Higress 网关 + HTTPRoute header 匹配

消息桥对**所有专家**请求同一个地址(网关入口),只多带一个请求头:

```
POST /v1/chat
X-Expert-Id: code-reviewer
```

- 消息桥侧契约只有一个 header,不构造 URL、不感知集群 DNS、不维护路由表。
- 路由由现有 Higress 网关承担:每个专家一条 Gateway API `HTTPRoute`,按 `X-Expert-Id` header 精确匹配,`backendRef` 指向该专家的 Service。
- 无匹配路由(未知专家 ID)时网关返回 404,天然兜底。
- 增删专家 = 增删一条 HTTPRoute,网关配置自动生效,无需重启任何组件。

### 4.2 备选方案对比

| 方案 | 桥的标记方式 | 额外组件 | 适用场景 |
|---|---|---|---|
| **A. Higress 原生 header 匹配路由(推荐)** | 一个 header `X-Expert-Id` | 无(复用现有网关) | 已有 Higress;声明式、每专家一条路由、桥不感知拓扑;三种 API 写法见 §4.3 |
| B. 每专家 Service + DNS 命名 | 目标 hostname 即专家 ID(`expert-<id>.expert.svc.cluster.local`) | 无 | 桥在集群内且接受命名约定;零路由配置 |
| C. Higress Ingress path 路由 | URL path 前缀(`/expert/<id>/...`) | Ingress 规则 | 团队更熟 Ingress API;path 即标记 |
| D. KEDA HTTP add-on | Host header / path | KEDA + interceptor | 需要真正的 scale-to-zero(见 §5.3) |

> 若消息桥在集群外,直接走 Higress 网关的对外入口(LB IP/域名)即可,桥侧契约不变。

### 4.3 等价写法(三选一,选一种统一使用)

Higress 原生支持三种 header 匹配路由写法,行为一致,不需要任何插件:

**写法一: Gateway API HTTPRoute(推荐)**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: expert-code-reviewer
spec:
  parentRefs:
    - name: higress-gateway          # Higress 默认网关
      namespace: higress-system
  rules:
    - matches:
        - headers:
            - name: x-expert-id
              value: code-reviewer
      backendRefs:
        - name: expert-code-reviewer
          port: 8080
```

**写法二: Istio VirtualService(Higress 原生兼容,不需要装 Istio)**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: expert-code-reviewer
spec:
  gateways:
    - higress-gateway              # Higress 默认网关
  http:
    - match:
        - headers:
            x-expert-id:
              exact: code-reviewer
      route:
        - destination:
            host: expert-code-reviewer   # 对应 expert-<id> Service
            port:
              number: 8080
```

**写法三: Ingress + header 匹配注解(团队更熟 Ingress API 时用)**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: expert-code-reviewer
  annotations:
    higress.io/exact-match-header-x-expert-id: "code-reviewer"
spec:
  ingressClassName: higress
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: expert-code-reviewer
                port:
                  number: 8080
```

> 注解支持 `exact` / `regex` / `prefix` 三种匹配:`higress.io/exact-match-header-<name>`、`higress.io/regex-match-header-<name>`、`higress.io/prefix-match-header-<name>`(源码 `pkg/ingress/kube/annotations/match.go` 与 e2e 用例 `httproute-match-headers.yaml` 可验证)。

三种写法的行为一致:未匹配 header 的请求由网关返回 404,桥侧契约不变(一个 `X-Expert-Id` header)。选择依据:团队 API 熟练度与现有网关配置风格,不建议混用。

## 5. 自动扩缩

### 5.1 默认方案:每专家 HPA(CPU/内存)

- 每个专家一个 HPA(`autoscaling/v2`),按 CPU 利用率扩缩,`minReplicas`/`maxReplicas` 每专家可配。
- 无额外依赖,原生 k8s 能力,满足大部分流量波动场景。
- 注意:原生 HPA 的 `minReplicas: 0` 对 CPU 指标无效(0 副本时无 Pod 可采样,无法触发扩容),因此默认 `minReplicas ≥ 1`。

**指标来源:不需要 Prometheus**。CPU/内存指标走 Metrics Server(`metrics.k8s.io` API,数据来自 kubelet 内置 cAdvisor),托管集群一般自带,自建集群需确认已安装:

```bash
kubectl get deploy -n kube-system metrics-server
kubectl top nodes        # 能出结果说明 metrics-server 正常
```

只有用业务指标(请求量、队列深度)做扩缩时才需要 Prometheus + prometheus-adapter(或 KEDA 的外部指标源)。

### 5.2 进阶方案:KEDA + 外部指标(scale-to-zero)

- 专家多、空闲时间长、冷启动可接受时,用 KEDA `ScaledObject` 把空闲专家缩到 0。
- 指标来源:网关/worker 暴露的每专家请求指标(Prometheus),或消息桥侧队列深度。
- 每专家一个 ScaledObject,`minReplicas: 0`,空闲超过 `cooldownPeriod` 即缩零。

### 5.3 真正的 scale-to-zero 需要请求保持

HTTP 场景下"缩到 0 再扩容"存在鸡生蛋问题:0 副本时没有 Pod 处理请求,网关转发即 503,而扩容信号恰恰来自流量。两条路:

1. **KEDA HTTP add-on**:interceptor 拦截请求、触发扩容、等待就绪后转发,请求不丢。此时 interceptor 本身可充当路由层(按 Host/path 路由),替代方案 A 的网关路由。
2. **桥侧重试**:网关返回 503 + `Retry-After`,消息桥按指数退避重试。要求桥具备重试能力,标记契约不变。

### 5.4 推荐组合

| 场景 | 组合 |
|---|---|
| 默认(简单可靠) | 方案 A 网关路由 + 每专家 HPA(minReplicas=1) |
| 专家多、空闲多、可接受冷启动 | 方案 D(KEDA HTTP add-on)或方案 A + KEDA Prometheus 缩零 + 桥重试 |

## 6. 示例清单(核心 YAML)

> 以下示例以 `code-reviewer` 专家为例。若走 Helm,这些 YAML 由 `values.experts` 列表渲染生成;若不用 Helm,可直接按此模板手工维护,或用 Kustomize 做参数化。

### 6.0 声明式入口(values 结构,Helm 渲染)

```yaml
image:
  repository: registry.example.com/expert-worker
  tag: latest

# 路由复用现有 Higress 网关,无需额外配置
gateway:
  name: higress-gateway          # HTTPRoute 的 parentRef
  namespace: higress-system

experts:
  - id: code-reviewer
    minReplicas: 1
    maxReplicas: 5
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 500m, memory: 512Mi }
    config: |            # 挂载为 /etc/expert/config.yaml
      persona: 资深代码评审专家
      model: gpt-4o
  - id: translator
    minReplicas: 1
    maxReplicas: 3
    resources:
      requests: { cpu: 100m, memory: 128Mi }
      limits:   { cpu: 500m, memory: 512Mi }
    config: |
      persona: 中英互译专家
      model: gpt-4o-mini
```

增删专家 = 改列表 + 重新渲染/`helm upgrade`,自动生成对应的 Deployment/ConfigMap/Service/HPA/HTTPRoute。

### 6.1 专家 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: expert-code-reviewer-config
  labels:
    app: expert-worker
    expert-id: code-reviewer
data:
  config.yaml: |
    persona: 资深代码评审专家
    model: gpt-4o
```

### 6.2 专家 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: expert-code-reviewer
  labels:
    app: expert-worker
    expert-id: code-reviewer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: expert-worker
      expert-id: code-reviewer
  template:
    metadata:
      labels:
        app: expert-worker
        expert-id: code-reviewer
      annotations:
        # Helm 渲染时注入 ConfigMap 内容哈希,配置变更即触发滚动
        checksum/config: 3f2a1b...
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: worker
          image: registry.example.com/expert-worker:latest
          ports:
            - name: http
              containerPort: 8080
          readinessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 15
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          volumeMounts:
            - name: config
              mountPath: /etc/expert
      volumes:
        - name: config
          configMap:
            name: expert-code-reviewer-config
```

### 6.3 专家 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: expert-code-reviewer
  labels:
    app: expert-worker
    expert-id: code-reviewer
spec:
  selector:
    app: expert-worker
    expert-id: code-reviewer
  ports:
    - name: http
      port: 8080
      targetPort: http
```

### 6.4 专家 HPA(每专家独立扩缩)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: expert-code-reviewer
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: expert-code-reviewer
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 6.5 路由(每专家一条,三选一)

**写法一: Gateway API HTTPRoute(推荐)**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: expert-code-reviewer
  labels:
    app: expert-worker
    expert-id: code-reviewer
spec:
  parentRefs:
    - name: higress-gateway        # 指向现有 Higress 网关
      namespace: higress-system
  rules:
    - matches:
        - headers:
            - name: x-expert-id
              value: code-reviewer
      backendRefs:
        - name: expert-code-reviewer
          port: 8080
```

**写法二: Istio VirtualService(等价,团队更熟 Istio 语法时用)**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: expert-code-reviewer
  labels:
    app: expert-worker
    expert-id: code-reviewer
spec:
  gateways:
    - higress-gateway              # Higress 默认网关
  http:
    - match:
        - headers:
            x-expert-id:
              exact: code-reviewer
      route:
        - destination:
            host: expert-code-reviewer   # 对应 expert-<id> Service
            port:
              number: 8080
```

**写法三: Ingress + header 匹配注解(等价,团队更熟 Ingress API 时用)**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: expert-code-reviewer
  labels:
    app: expert-worker
    expert-id: code-reviewer
  annotations:
    higress.io/exact-match-header-x-expert-id: "code-reviewer"
spec:
  ingressClassName: higress
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: expert-code-reviewer
                port:
                  number: 8080
```

- 消息桥请求网关入口,带 `X-Expert-Id: code-reviewer` header,网关精确匹配后转发到 `expert-code-reviewer` Service。
- 未匹配任何路由的 header(未知专家)由网关返回 404。
- 若桥在集群内,入口地址即网关 Service:`http://higress-gateway.higress-system.svc.cluster.local`;在集群外则用网关对外 LB IP/域名。
- 三种写法行为一致,选一种统一使用,不要混用。

### 6.6 KEDA ScaledObject(可选,scale-to-zero)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: expert-code-reviewer
spec:
  scaleTargetRef:
    name: expert-code-reviewer
  minReplicaCount: 0
  maxReplicaCount: 5
  cooldownPeriod: 300      # 空闲 5 分钟缩到 0
  pollingInterval: 30
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc.cluster.local:9090
        # 按专家维度统计网关/worker 的请求量,具体指标名按实际监控栈调整
        query: sum(rate(expert_requests_total{expert="code-reviewer"}[2m]))
        threshold: "1"
```

> 注意: 缩到 0 后首个请求会 503(网关无可用后端),需配合桥侧重试(503 + `Retry-After`)或改用 KEDA HTTP add-on(interceptor 保持请求并触发扩容)。

## 7. 运维要点

- **配置变更**: 只滚动对应专家,checksum 注解保证;走 Helm 时回滚用 `helm rollback`,手工 manifest 则用 `kubectl rollout undo`。
- **冷启动**: 缩零专家首次请求有延迟;热点专家设 `minReplicas ≥ 1`;worker 镜像保持轻量、启动快。
- **资源隔离**: 每专家独立 requests/limits,避免一个专家打满节点影响其他专家。
- **可观测性**: worker 暴露 Prometheus 指标,按 `expert-id` 标签聚合;网关侧按 header 维度看流量;HPA 状态用 `kubectl describe hpa` 排查。
- **镜像升级**: 只升级基座镜像 tag,所有专家一次滚动;若某专家需固定旧镜像,values 里支持 per-expert `image` 覆盖。
- **路由变更**: 增删专家即增删 HTTPRoute,网关热生效;用 `kubectl get httproute` 确认路由状态(是否被网关接受)。

## 8. 待确认项

1. 消息桥与 worker 的通信协议是否为 HTTP?(gRPC 变体:用 gRPC metadata 传 `expert-id`,Higress 支持 gRPC 路由,但 header 匹配需按 metadata 转换规则配置)
2. 消息桥部署在集群内还是集群外?(集群内走网关 Service DNS,集群外走网关 LB IP/域名)
3. 是否需要 scale-to-zero?(决定是否引入 KEDA 依赖)
4. worker 镜像的监听端口与健康检查路径?(示例里暂用 8080 + `/healthz`,需按实际镜像调整)
5. Higress 网关的 Gateway 资源名与命名空间?(示例里暂用 `higress-gateway` / `higress-system`,需按实际部署调整)