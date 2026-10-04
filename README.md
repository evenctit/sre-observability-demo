# SRE 可观测性演示平台 (sre-observability-demo)

本项目以 Infrastructure as Code 的方式管理一套部署在 Kubernetes (v1.37.1) 上的完整可观测性平台，包含链路追踪 (Tempo)、指标监控 (Prometheus)、可视化 (Grafana)，以及一个带 OpenTelemetry 埋点的 Python 演示应用 (demo-api)。

## 架构

```
                        ┌─────────────────────────────┐
                        │   用户浏览器                  │
                        │   http://<节点IP>:30300      │
                        └──────────────┬──────────────┘
                                       │ NodePort
                        ┌──────────────▼──────────────┐
                        │   Grafana 12.2.0 (PVC 10Gi) │
                        │   数据源: Prometheus + Tempo │
                        └──────┬───────────────┬──────┘
                               │               │
                ┌──────────────▼───┐   ┌───────▼─────────────┐
                │ Prometheus v3.5.0│   │ Tempo 2.9.0         │
                │ (PVC 10Gi)       │   │ OTLP: 4317/4318     │
                │ 抓取 cAdvisor    │   │ 查询 API: 3200      │
                └────────▲─────────┘   └───────▲─────────────┘
                         │                     │ OTLP/HTTP 上报 traces
                ┌────────┴─────────────────────┴─────────────┐
                │        demo-api (Flask + OpenTelemetry)     │
                │        http://<节点IP>:30080                │
                │   /api/process 内部调用 method_a + method_b │
                └─────────────────────────────────────────────┘
```

## 目录结构 (按组件划分)

```
sre-observability-demo/
├── namespaces/          # 命名空间 (demo + monitoring)
│   └── namespace.yaml
├── storage/             # local-path 存储供应器, 为 PVC 提供动态存储
│   ├── rbac.yaml
│   ├── configmap.yaml   # 含 setup/teardown 脚本 (使用 VOL_DIR 变量)
│   ├── deployment.yaml
│   └── storageclass.yaml
├── tempo/               # 链路追踪后端
│   ├── configmap.yaml   # OTLP 4317/4318 + 查询 3200
│   ├── deployment.yaml
│   └── service.yaml
├── prometheus/          # 指标监控
│   ├── rbac.yaml        # ServiceAccount + ClusterRole (抓取 kubelet/cAdvisor)
│   ├── configmap.yaml   # 抓取配置
│   ├── pvc.yaml         # 10Gi
│   ├── deployment.yaml
│   └── service.yaml
├── grafana/             # 可视化
│   ├── configmap.yaml   # Prometheus + Tempo 数据源预配置
│   ├── pvc.yaml         # 10Gi
│   ├── deployment.yaml
│   └── service.yaml     # NodePort 30300
└── demo-api/            # Python 应用部署清单 (源码在 python-observability-demo 仓库)
    ├── deployment.yaml
    └── service.yaml     # NodePort 30080
```

## 各组件 YAML 配置明细

### namespaces/ (命名空间)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| namespace.yaml | Namespace x2 | `demo` (demo-api 应用), `monitoring` (全部可观测性组件) |

### storage/ (local-path 存储供应器)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| storage/rbac.yaml | ServiceAccount / ClusterRole / ClusterRoleBinding | SA `local-path-provisioner-service-account`; 角色授权 nodes/pvcs/configmaps 读取, persistentvolumes/pods 全权, events 创建, storageclasses 读取 |
| storage/configmap.yaml | ConfigMap `local-path-config` | `config.json`: PV 根目录 `/var/local-path-provisioner`; `setup`/`teardown` 脚本使用 provisioner 注入的 `VOL_DIR` 变量; `helperPod.yaml` 使用 busybox:1.36 |
| storage/deployment.yaml | Deployment | 镜像 `local-path-provisioner:v0.0.31`; 启动参数 `--debug start --config /etc/config/config.json`; 环境变量 `POD_NAMESPACE`; 挂载 ConfigMap 到 `/etc/config/` |
| storage/storageclass.yaml | StorageClass `local-path` | 标注为集群**默认** StorageClass (`is-default-class: "true"`); 绑定模式 `WaitForFirstConsumer`; 回收策略 `Delete` |

### tempo/ (链路追踪后端)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| tempo/configmap.yaml | ConfigMap `tempo-config` | `tempo.yaml`: 查询端口 3200; OTLP 接收 gRPC 4317 / HTTP 4318; trace 存储后端 `local` (`/var/tempo/traces`), WAL 路径 `/var/tempo/wal` |
| tempo/deployment.yaml | Deployment | 镜像 `tempo:2.9.0`; 启动参数 `-config.file=/etc/tempo/tempo.yaml`; 暴露端口 3200/4317/4318; readiness 探针 `/ready`; trace 数据用 emptyDir (当前未持久化) |
| tempo/service.yaml | Service (ClusterIP) | `tempo.monitoring.svc.cluster.local` 的 3200 (Grafana 查询) / 4317+4318 (demo-api 上报 traces) |

### prometheus/ (指标监控)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| prometheus/rbac.yaml | ServiceAccount / ClusterRole / ClusterRoleBinding | SA `prometheus`; 授权 nodes, nodes/proxy, services, endpoints, pods 读取及 `/metrics` 非资源 URL, 供 apiserver proxy 抓取使用 |
| prometheus/configmap.yaml | ConfigMap `prometheus-config` | `prometheus.yml`: 全局抓取间隔 30s; 三个 job — `prometheus` (自身 9090), `kubernetes-nodes-cadvisor` (https 经 apiserver proxy 抓 `/api/v1/nodes/<node>/proxy/metrics/cadvisor`, 使用 SA token, `insecure_skip_verify`), `kubernetes-pods` (按注解 `prometheus.io/scrape: "true"` 自动发现) |
| prometheus/pvc.yaml | PVC `prometheus-data` | 10Gi, RWO, StorageClass `local-path` |
| prometheus/deployment.yaml | Deployment | 镜像 `prom/prometheus:v3.5.0`; `strategy: Recreate`; initContainer `fix-perm` 将数据目录属主改为 65534:65534; 参数 `--storage.tsdb.retention.time=15d` (保留 15 天); 端口 9090 |
| prometheus/service.yaml | Service (ClusterIP) | 9090, 供 Grafana 数据源访问 |

### grafana/ (可视化)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| grafana/configmap.yaml | ConfigMap `grafana-datasources` | 预置两个数据源: `Prometheus` (默认, url `http://prometheus.monitoring.svc.cluster.local:9090`) 与 `Tempo` (uid 固定为 `tempo`, url `http://tempo.monitoring.svc.cluster.local:3200`) |
| grafana/pvc.yaml | PVC `grafana-data` | 10Gi, RWO, StorageClass `local-path` |
| grafana/deployment.yaml | Deployment | 镜像 `grafana:12.2.0`; `strategy: Recreate`; initContainer `fix-perm` 属主改为 472:472; 环境变量 `GF_SECURITY_ADMIN_USER=admin` / `GF_SECURITY_ADMIN_PASSWORD=admin`; PVC 挂 `/var/lib/grafana`, 数据源挂 `/etc/grafana/provisioning/datasources`; readiness 探针 `/api/health` |
| grafana/service.yaml | Service (NodePort) | 3000 -> nodePort **30300**, 对外提供 HTTP 访问 |

### demo-api/ (Python 演示应用)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| demo-api/deployment.yaml | Deployment | 镜像 `demo-api:v1` (源码在 python-observability-demo 仓库); 2 副本; 端口 5000; 环境变量 `OTEL_SERVICE_NAME=demo-api`, `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://tempo.monitoring.svc.cluster.local:4318/v1/traces`; 资源 requests 100m/128Mi, limits 500m/256Mi |
| demo-api/service.yaml | Service (NodePort) | 80 -> 5000, nodePort **30080**, 对外提供 `/api/process` 等接口 |

## 部署方式

按依赖顺序 apply (storage 必须先于其它组件, 因为 PVC 依赖其 StorageClass):

```bash
kubectl apply -f namespaces/namespace.yaml
kubectl apply -f storage/
kubectl apply -f tempo/
kubectl apply -f prometheus/
kubectl apply -f grafana/
kubectl apply -f demo-api/
```

demo-api 的镜像 `demo-api:v1` 由源码仓库 python-observability-demo 构建, 并导入集群节点 containerd (`ctr -n k8s.io images import`), 部署清单中使用 `imagePullPolicy: IfNotPresent`。

## 访问入口

| 组件 | 地址 | 说明 |
|---|---|---|
| Grafana | http://<节点IP>:30300 | 账号 admin / admin |
| demo-api | http://<节点IP>:30080 | RESTful API |
| Prometheus | ClusterIP:9090 | 集群内访问 (Grafana 数据源) |
| Tempo | ClusterIP:3200/4317/4318 | 集群内访问 (查询 + OTLP 接收) |

## 验证链路追踪

1. 调用应用接口产生 trace:

```bash
curl "http://<节点IP>:30080/api/process?n=10&text=hello"
```

返回体中包含 `method_a` (斐波那契计算) 与 `method_b` (文本处理) 两个方法的结果。

2. 打开 Grafana (http://<节点IP>:30300) -> Explore -> 选择 Tempo 数据源 -> 输入 TraceQL 查询:

```
{resource.service.name="demo-api"}
```

3. 点击任一 trace, 瀑布图中可见 `GET /api/process` 请求 span 及其子 span `method_a`、`method_b`。

## 镜像与版本

| 组件 | 版本 | 镜像来源 |
|---|---|---|
| Kubernetes | v1.37.1 | registry.aliyuncs.com/google_containers |
| Prometheus | v3.5.0 | docker.m.daocloud.io/prom/prometheus |
| Grafana | 12.2.0 | docker.m.daocloud.io/grafana/grafana |
| Tempo | 2.9.0 | docker.m.daocloud.io/grafana/tempo |
| local-path-provisioner | v0.0.31 | docker.m.daocloud.io/rancher/local-path-provisioner |

说明: 控制面组件镜像统一走阿里云官方镜像源; 第三方镜像走国内 DaoCloud 代理 (阿里云不提供 docker.io 全量公共代理)。集群节点 containerd 已在 `/etc/containerd/certs.d/` 配置 docker.io 与 ghcr.io 的国内代理。

## 日常维护 (IaC 工作流)

所有配置改动均通过修改本仓库的 YAML 完成:

```bash
# 1. 修改本地对应组件目录下的 yaml
# 2. 应用到集群
kubectl apply -f <组件目录>/
# 3. 提交并同步 GitHub
git add . && git commit -m "..." && git push
```

关键配置备注:

- storage/configmap.yaml 中的 setup/teardown 脚本使用 provisioner (v0.0.31) 注入的 `VOL_DIR` 环境变量, 不要改回旧版 `VOLUME_DIR`
- Prometheus 通过 apiserver proxy (`/api/v1/nodes/<node>/proxy/metrics/cadvisor`) 抓取容器指标, 依赖 prometheus/rbac.yaml 中的 `nodes/proxy` 权限
- PVC 数据落在节点 `/var/local-path-provisioner/` 目录; Prometheus 数据保留 15 天
- demo-api 通过环境变量 `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` 指向 Tempo 的 OTLP/HTTP 接口
