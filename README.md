# SRE 可观测性演示平台 (sre-observability-demo)

本项目以 Infrastructure as Code 的方式管理一套部署在 Kubernetes (v1.37.1) 上的完整可观测性平台，包含链路追踪 (Tempo)、指标监控 (Prometheus)、黑盒健康探测与邮件告警 (Blackbox Exporter + Grafana Alerting)、日志采集 (Loki + OpenTelemetry Collector)、可视化 (Grafana)，以及一个带 OpenTelemetry 埋点和 JSON 结构化日志的 Python 演示应用 (demo-api)。

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
                        │           + Loki            │
                        └──┬──────────┬──────────┬────┘
                           │          │          │
          ┌────────────────▼──┐ ┌─────▼──────┐ ┌─▼──────────────────┐
          │ Prometheus v3.5.0 │ │ Tempo 2.9.0│ │ Loki 3.4.2 (10Gi)  │
          │ (PVC 10Gi)        │ │ OTLP:4317/8│ │ OTLP摄入: 3100/otlp│
          │ 抓取 cAdvisor     │ │ 查询: 3200 │ │ LogQL 查询: 3100   │
          └────────▲──────────┘ └─────▲──────┘ └─▲──────────────────┘
                   │                  │          │ OTLP/HTTP (logs)
                   │                  │ OTLP     │
                   │                  │ (traces) │
          ┌────────┴──────────────────┴──────────┴─────────────┐
          │        demo-api v2 (Flask + OpenTelemetry)          │
          │        http://<节点IP>:30080                        │
          │   /api/process 内部调用 method_a + method_b         │
          │   JSON 日志输出到 stdout (含 method/n/duration 等)   │
          └────────┬───────────────────┬────────────────────────┘
                   │ 容器日志落盘       │ HTTP /healthz 探测
                   │ /var/log/pods     │
   ┌───────────────▼───────────────┐   │
   │ OpenTelemetry Collector (DS)  │   │   ┌────────────────────────────┐
   │ filelog 采集 -> CRI/JSON 解析  │   │   │ Blackbox Exporter v0.27.0  │
   │ 提取 K8s 元数据 -> 发送 Loki    │   └──>│ http_2xx 探测 /healthz     │
   └───────────────────────────────┘       └──────────┬─────────────────┘
                                                      │ probe_success 等
                                          ┌───────────▼─────────────┐
                                          │ Prometheus 抓取探测指标   │
                                          └───────────┬─────────────┘
                                                      │ Grafana Alerting:
                                                      │ probe_success < 1 持续 1m
                                                      │ -> 邮件 yschen0925@sina.com
```

日志链路: demo-api 打印单行 JSON 日志到 stdout -> containerd 落盘 `/var/log/pods/` ->
OpenTelemetry Collector (DaemonSet, filelog receiver) 解析 CRI 头与 JSON ->
通过 OTLP/HTTP 发送到 Loki 原生摄入端点 (`/otlp/v1/logs`) -> Grafana 日志面板展示。

健康探测与告警链路: Prometheus (blackbox-demo-api job) 调用 Blackbox Exporter 的
`/probe` 接口, 探测 demo-api 的 `/healthz` -> 指标 `probe_success`/`probe_http_status_code`/`probe_duration_seconds`
-> Grafana "Demo API 健康状态" dashboard 展示; Grafana 告警规则 (probe_success < 1
持续 1 分钟) 触发后通过 SMTP 发送告警邮件到 yschen0925@sina.com。

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
├── loki/                # 日志存储与查询
│   ├── configmap.yaml   # 单体模式配置, TSDB + filesystem, OTLP 摄入开启
│   ├── pvc.yaml         # 10Gi
│   ├── deployment.yaml
│   └── service.yaml
├── otel-collector/      # OpenTelemetry Collector 日志采集 (DaemonSet)
│   ├── rbac.yaml        # ServiceAccount
│   ├── configmap.yaml   # filelog 采集管道 (CRI 解析/元数据/OTLP 发送 Loki)
│   └── daemonset.yaml
├── blackbox/            # Blackbox Exporter 黑盒健康探测
│   ├── deployment.yaml
│   └── service.yaml
├── prometheus/          # 指标监控
│   ├── rbac.yaml        # ServiceAccount + ClusterRole (抓取 kubelet/cAdvisor)
│   ├── configmap.yaml   # 抓取配置 (含 blackbox-demo-api 探测 job)
│   ├── pvc.yaml         # 10Gi
│   ├── deployment.yaml
│   └── service.yaml
├── grafana/             # 可视化与告警
│   ├── configmap.yaml   # Prometheus + Tempo + Loki 数据源预配置
│   ├── dashboard.yaml   # Dashboard as Code (Demo API 日志 + Demo API 健康状态)
│   ├── alerting.yaml    # 告警 as Code (邮件 contactPoint + 策略 + 告警规则)
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

### loki/ (日志存储与查询)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| loki/configmap.yaml | ConfigMap `loki-config` | 单体模式; HTTP 端口 3100; schema v13 + TSDB 索引 + filesystem 对象存储 (chunks/rules 落 `/var/loki`); `allow_structured_metadata: true` 支持原生 OTLP 摄入; 拒绝 7 天以上旧样本 |
| loki/pvc.yaml | PVC `loki-data` | 10Gi, RWO, StorageClass `local-path` |
| loki/deployment.yaml | Deployment | 镜像 `loki:3.4.2`; `strategy: Recreate`; initContainer `fix-perm` 属主改为 10001:10001; 参数 `-config.file=/etc/loki/loki.yaml`; readiness 探针 `/ready` |
| loki/service.yaml | Service (ClusterIP) | 3100, 供 otel-collector 摄入 (`/otlp/v1/logs`) 与 Grafana 查询 (LogQL) |

### otel-collector/ (OpenTelemetry 日志采集)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| otel-collector/rbac.yaml | ServiceAccount | 最小 SA (采集走节点文件路径解析元数据, 不访问 K8s API) |
| otel-collector/configmap.yaml | ConfigMap `otel-collector-config` | filelog receiver 采集 `/var/log/pods/*/*/*.log`; operators 管道: 解析 CRI 头 (时间戳/流) -> 提升日志行到 body -> 从文件路径提取 namespace/pod/uid/container -> 写入 resource 属性 (Loki 标签, 容器名映射为 `service.name`) -> 解析 JSON 日志字段 -> severity_parser 设置日志级别; exporter `otlphttp/loki` 指向 `http://loki.monitoring.svc.cluster.local:3100/otlp`; health_check 监听 `0.0.0.0:13133` |
| otel-collector/daemonset.yaml | DaemonSet | 镜像 `opentelemetry-collector-contrib:0.127.0`; hostPath 只读挂载节点 `/var/log/pods`; readiness 探针 13133 |

### blackbox/ (黑盒健康探测)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| blackbox/deployment.yaml | Deployment | 镜像 `blackbox-exporter:v0.27.0`; 内置 `http_2xx` 模块 (HTTP 2xx 视为成功); 端口 9115; readiness 探针 `/-/healthy` |
| blackbox/service.yaml | Service (ClusterIP) | 9115, 供 Prometheus 抓取 `/probe` 探测接口 |

探测指标说明 (由 prometheus/configmap.yaml 的 `blackbox-demo-api` job 生成):
`probe_success` (1=健康, 0=异常), `probe_duration_seconds` (探测耗时),
`probe_http_status_code` (HTTP 状态码), 探测目标 `http://demo-api.demo.svc.cluster.local/healthz`。

### prometheus/ (指标监控)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| prometheus/rbac.yaml | ServiceAccount / ClusterRole / ClusterRoleBinding | SA `prometheus`; 授权 nodes, nodes/proxy, services, endpoints, pods 读取及 `/metrics` 非资源 URL, 供 apiserver proxy 抓取使用 |
| prometheus/configmap.yaml | ConfigMap `prometheus-config` | `prometheus.yml`: 全局抓取间隔 30s; 四个 job — `prometheus` (自身 9090), `kubernetes-nodes-cadvisor` (https 经 apiserver proxy 抓 `/api/v1/nodes/<node>/proxy/metrics/cadvisor`, 使用 SA token, `insecure_skip_verify`), `kubernetes-pods` (按注解 `prometheus.io/scrape: "true"` 自动发现), `blackbox-demo-api` (经 blackbox-exporter 探测 demo-api /healthz, relabel 注入 target 参数) |
| prometheus/pvc.yaml | PVC `prometheus-data` | 10Gi, RWO, StorageClass `local-path` |
| prometheus/deployment.yaml | Deployment | 镜像 `prom/prometheus:v3.5.0`; `strategy: Recreate`; initContainer `fix-perm` 将数据目录属主改为 65534:65534; 参数 `--storage.tsdb.retention.time=15d` (保留 15 天); 端口 9090 |
| prometheus/service.yaml | Service (ClusterIP) | 9090, 供 Grafana 数据源访问 |

### grafana/ (可视化)

| 文件 | 资源 | 配置说明 |
|---|---|---|
| grafana/configmap.yaml | ConfigMap `grafana-datasources` | 预置三个数据源: `Prometheus` (uid 固定 `prometheus`, 默认, url `http://prometheus.monitoring.svc.cluster.local:9090`), `Tempo` (uid 固定为 `tempo`, url `http://tempo.monitoring.svc.cluster.local:3200`) 与 `Loki` (uid 固定为 `loki`, url `http://loki.monitoring.svc.cluster.local:3100`) |
| grafana/dashboard.yaml | ConfigMap `grafana-dashboards` | Dashboard as Code: `dashboards.yaml` (provider, folder `SRE Demo`, 30s 自动刷新文件) + `demo-api-logs.json` (Logs 面板 + 日志量曲线) + `demo-api-health.json` (uid `demo-api-health`: 服务状态 UP/DOWN Stat、近5分钟失败次数 Stat、探测耗时与 HTTP 状态码曲线) |
| grafana/alerting.yaml | ConfigMap `grafana-alerting` | 告警 as Code, 三个 provisioning 文件 — `contact-points.yaml` (email 接收人 yschen0925@sina.com), `policies.yaml` (根路由 -> demo-api-email), `alert-rules.yaml` (规则组 `demo-api-health`: 规则 "Demo API 无法访问", 条件 `probe_success{job="blackbox-demo-api"} < 1` 持续 1m 触发, noDataState=Alerting, severity=critical) |
| grafana/deployment.yaml | Deployment | 镜像 `grafana:12.2.0`; `strategy: Recreate`; initContainer `fix-perm` 属主改为 472:472; 环境变量 `GF_SECURITY_ADMIN_USER=admin` / `GF_SECURITY_ADMIN_PASSWORD=admin`; SMTP 告警发件配置 `GF_SMTP_ENABLED=true`, `GF_SMTP_HOST=smtp.sina.com:465`, 发件账号/授权码为占位符 `REPLACE-WITH-SENDER@sina.com` / `REPLACE-WITH-SMTP-AUTH-CODE` (替换后重启生效); PVC 挂 `/var/lib/grafana`, 数据源挂 `/etc/grafana/provisioning/datasources`, 告警配置挂 `/etc/grafana/provisioning/alerting`; readiness 探针 `/api/health` |
| grafana/pvc.yaml | PVC `grafana-data` | 10Gi, RWO, StorageClass `local-path` |
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
kubectl apply -f loki/
kubectl apply -f otel-collector/
kubectl apply -f blackbox/
kubectl apply -f prometheus/
kubectl apply -f grafana/
kubectl apply -f demo-api/
```

demo-api 的镜像 `demo-api:v2` (含 JSON 结构化日志) 由源码仓库 python-observability-demo 构建, 并导入集群节点 containerd (`ctr -n k8s.io images import`), 部署清单中使用 `imagePullPolicy: IfNotPresent`。

## 访问入口

| 组件 | 地址 | 说明 |
|---|---|---|
| Grafana | http://<节点IP>:30300 | 账号 admin / admin; dashboard: SRE Demo -> Demo API 日志 |
| demo-api | http://<节点IP>:30080 | RESTful API |
| Prometheus | ClusterIP:9090 | 集群内访问 (Grafana 数据源) |
| Tempo | ClusterIP:3200/4317/4318 | 集群内访问 (查询 + OTLP 接收 traces) |
| Loki | ClusterIP:3100 | 集群内访问 (LogQL 查询 + OTLP 摄入 logs) |

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

## 验证日志采集

1. 调用应用接口产生日志:

```bash
curl "http://<节点IP>:30080/api/process?n=5&text=loki-test"
```

2. 查询 Loki (确认日志已入库, 从集群内访问):

```bash
kubectl -n monitoring port-forward --address 127.0.0.1 svc/loki 3100:3100 &
curl -s 'http://127.0.0.1:3100/loki/api/v1/query_range?query=%7Bservice_name%3D%22demo-api%22%7D&limit=5'
```

3. 打开 Grafana (http://<节点IP>:30300) -> Dashboards -> SRE Demo -> "Demo API 日志": Logs 面板显示 demo-api 的 JSON 日志 (含 method_a/method_b 的开始与完成、耗时等中文消息), 下方 Timeseries 面板显示每分钟日志行数 (按级别分组)。

## 验证健康探测与告警

1. 验证探测指标 (从集群内查询 Prometheus):

```bash
kubectl -n monitoring port-forward --address 127.0.0.1 svc/prometheus 9090:9090 &
curl -s 'http://127.0.0.1:9090/api/v1/query?query=probe_success%7Bjob%3D%22blackbox-demo-api%22%7D'
# 正常返回 value "1"; demo-api 宕机时返回 "0"
```

2. 打开 Grafana -> Dashboards -> SRE Demo -> "Demo API 健康状态": 服务状态 Stat 显示 "UP 正常" (绿色), 近5分钟失败次数、探测耗时与 HTTP 状态码曲线。

3. 告警链路验证 (down 场景): 将 demo-api 缩容到 0 (`kubectl -n demo scale deploy/demo-api --replicas=0`), 约 2 分钟后 Grafana 告警 "Demo API 无法访问" 进入 Alerting 状态并尝试发送邮件到 yschen0925@sina.com; 验证完恢复 `--replicas=2`, 告警回到 Normal。

4. 查看告警状态:

```bash
curl -s -u admin:admin "http://<节点IP>:30300/api/prometheus/grafana/api/v1/alerts"
```

注意: 邮件实际发送需要先在 grafana/deployment.yaml 中将 SMTP 占位符替换为真实的
新浪邮箱发件账号与授权码, 再执行 `kubectl -n monitoring rollout restart deploy/grafana`。

## 镜像与版本

| 组件 | 版本 | 镜像来源 |
|---|---|---|
| Kubernetes | v1.37.1 | registry.aliyuncs.com/google_containers |
| Prometheus | v3.5.0 | docker.m.daocloud.io/prom/prometheus |
| Blackbox Exporter | v0.27.0 | docker.m.daocloud.io/prom/blackbox-exporter |
| Grafana | 12.2.0 | docker.m.daocloud.io/grafana/grafana |
| Tempo | 2.9.0 | docker.m.daocloud.io/grafana/tempo |
| Loki | 3.4.2 | docker.m.daocloud.io/grafana/loki |
| OpenTelemetry Collector | 0.127.0 (contrib) | docker.m.daocloud.io/otel/opentelemetry-collector-contrib |
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
- otel-collector operators 的 `on_error` 只允许 `send`/`drop` (没有 `continue`); OTTL 表达式中正则 `^{` 无需转义 (`^\{` 会报 invalid char escape)
- health_check 扩展必须配置 `endpoint: 0.0.0.0:13133`, 默认仅监听 localhost 会导致 K8s 探针失败 (Pod 卡在 0/1 Running)
- 修改 grafana/configmap.yaml 或 dashboard.yaml 后需重启 Grafana (`kubectl -n monitoring rollout restart deploy/grafana`) 才能生效
- Loki 标签: `service_name` 来自容器名, 与 Tempo 的 `service.name` 一致; `k8s_pod_name`/`k8s_namespace_name` 等来自文件路径解析; JSON 字段 (level/method/duration_ms) 存于 structured metadata
- Grafana 12 告警 provisioning: email contactPoint 的 `settings.addresses` 是**字符串** (多个收件人分号分隔), 写成数组会启动崩溃; 告警规则引用的数据源需要固定 uid (datasources.yaml 中 `uid: prometheus`), 且修改 uid 后旧库中同名的自动 uid 数据源会冲突导致启动失败 ("data source not found"), 需删除重建 PVC
- ConfigMap 的 block scalar (`|-`) 中顶格 `---` 会终止字符串: 多个 provisioning 文档必须拆成不同的 key, 不能在一个 key 内用 `---` 分隔
- SMTP 邮件告警: 发件配置在 grafana/deployment.yaml 环境变量 (当前为占位符); 告警收件人为 yschen0925@sina.com (grafana/alerting.yaml); 告警规则状态可用 `/api/prometheus/grafana/api/v1/alerts` API 查询
