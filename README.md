# Nacos Service Monitor by HTTP（Zabbix 7.0 模板）

给 Nacos 注册中心/配置中心做一层“服务级健康看门狗”。模板本身只走 HTTP，服务发现由两个几十行的 `bash + wget` 脚本完成，不装 exporter、不改 Nacos 配置，默认（未开鉴权）的 Nacos 不需要账号。

适合：Spring Cloud / Dubbo 把服务注册在 Nacos，但不想为监控再引一套 Prometheus + Exporter；或者 Zabbix 已经通过 proxy 管着内网节点，希望采集跟着 proxy 走。

## 和常见 Nacos 监控方案的区别（先看这段）

- **官方没有 Nacos 模板**。社区常见做法是 agent 上装 Python 脚本、用 `UserParameter` 每个服务轮询一次，或者上 Prometheus + Nacos exporter + Grafana。这套走 Zabbix HTTP agent，不增加新的服务进程。
- **零侵入**：不写 Nacos、不重启、不要求开鉴权。默认 `{$NACOS.URL}` 指向 Nacos 即可；真开了鉴权时把 `{$NACOS.AUTH}` 填成 `username=..&password=..` 或 `accessToken=..`。
- **服务目录是 O(1) 请求**：每分钟一次 `catalog/services` 拿回全部服务，再用依赖项拆字段。只有“实例明细”是每个服务一次请求，大注册中心可以关掉它。
- **group + service 双维度**：同名服务在 `DEFAULT_GROUP` 和 `DUBBO_GROUP` 各注册一份时不会被合并，也不会只取第一条。
- **不只查一个 health 数**：服务注册健康、具体不健康实例的 IP、服务端 status/readiness/客户端连接数、配置中心条数、宿主 CPU/load 都在模板里。
- **告警分级**：High 发邮件，Warning 只进日报；事件名直接带不健康实例 IP；阈值全部是宏。
- **能跑在瘦 proxy**：两个发现脚本只依赖 `bash`、`wget`、`sed/grep`，Alpine 版 zabbix-proxy 容器里没有 `curl/python3/jq` 也能用。
- **短板也写清楚**：Nacos 2.2.1 的 `mem` 指标恒为 1（见“已知限制”），模板不拿它做告警；不理解成完整 Nacos 监控，集群/raft/JVM 需要另配。

## 采集方式

```
catalog/services (1m) ─┬─ nacos.total.services
                       └─ LLD(服务/组) ─┬─ healthy / instances / healthy_pct / trigger_flag
                                        └─ instance/list (1m/服务) → hosts[] → unhealthy IP 列表

operator/metrics (1m)  → status / services / instances / clients / subscribes / cpu / load / mem
console/server/state (5m) → version / mode
console/health/readiness (1m)
console/namespaces (5m) → LLD(非 public 命名空间) → 每个命名空间的配置条数
```

`catalog/services` 只需要一次请求就能覆盖全部服务；`instance/list` 用于把“有人不健康”落到具体 IP，历史只保留 1 天。

## 监控内容

| 层面 | 监控项 | 告警 |
|---|---|---|
| 服务注册（按 `group/service` 发现） | 实例数、健康实例数、健康占比、保护阈值标记、实例明细、不健康实例 IP | 无健康实例（High）、健康占比低于阈值（High）、triggerFlag=true（Warning/日报）、有实例不健康（High，事件带 IP） |
| 服务目录 | 服务总数 | 比上次减少（Warning/日报）、减少超过 `{$NACOS.SERVICE.DROP}`（High）、超过 5m 无数据（High） |
| 服务端 | status、services、instances、clients、subscribes、version、mode、readiness | status 不是 UP（High）、clients 低于阈值（High）、readiness/metrics/state 无数据（High 或 Warning） |
| 配置中心 | public 配置总数；非 public 命名空间的配置数 | 配置总数为 0（High）、单次采集下降超过 `{$NACOS.CONFIG.DROP}`（Warning/日报） |
| 宿主资源 | 整机 CPU、1 分钟 load average；mem 只采集 | CPU/load 各有 Warning（日报）和 High（邮件）两档；mem 不做告警 |
| 采集自监控 | 各 HTTP 主项 | 主项无数据 |

## 前置要求

- Zabbix **7.0**（本模板在 7.0.30 验证；用到了 YAML 导出格式和 7.0 的预处理行为）。
- Nacos **2.x**（在 2.2.1 standalone 验证；catalog/operator/console 都是 v1 OpenAPI）。
- Zabbix server 或 proxy 能访问 Nacos 的 8848 端口。
- 两个发现脚本必须放在**实际执行采集的那一侧**：主机由 server 直采就放 server 的 `externalscripts`，由 proxy 采就放 proxy 的 `externalscripts`。

## 安装

1. 安装发现脚本（文件名必须和模板里的 key 完全一致）：

   ```bash
   install -m 755 scripts/nacos.service.discovery   /usr/lib/zabbix/externalscripts/
   install -m 755 scripts/nacos.namespace.discovery /usr/lib/zabbix/externalscripts/
   ```

   如果主机由 Docker 里的 proxy 采集，把两个文件放到 proxy 容器 `externalscripts` 对应的宿主 bind mount 里（例如 `/usr/local/share/zbx-proxy-scripts/externalscripts/`），容器内路径仍是 `/usr/lib/zabbix/externalscripts/`。

2. 导入 `template_nacos_service_monitor_by_http.yaml`（Data collection → Templates → Import）。导入后模板在 **Templates/Applications** 组。

3. 创建主机并链接模板。主机需要有一个接口（Zabbix 的要求，HTTP agent 不会去连它），把宏改成实际地址：

   ```
   {$NACOS.URL}       = http://nacos-host:8848
   {$NACOS.NAMESPACE} =          # 留空 = public
   ```

4. LLD 默认 1 小时跑一次。首次验证可以临时把 `{$NACOS.DISCOVERY.DELAY}` 改成 `1m`，然后在 server/proxy 侧重载配置缓存（`zabbix_server -R config_cache_reload` / `zabbix_proxy -R config_cache_reload`），看到项和触发器后再改回 `1h`。

5. 可以直接用下面的请求确认端点和字段（不需要鉴权）：

   ```bash
   curl -s 'http://nacos-host:8848/nacos/v1/ns/operator/metrics?onlyStatus=false'
   curl -s 'http://nacos-host:8848/nacos/v1/ns/catalog/services?hasIpCount=true&withInstances=false&pageNo=1&pageSize=1000'
   ```

## 宏

| 宏 | 默认 | 说明 |
|---|---|---|
| `{$NACOS.URL}` | `http://127.0.0.1:8848` | Nacos 地址，结尾不要带斜杠 |
| `{$NACOS.NAMESPACE}` | 空 | 命名空间 ID，空 = public |
| `{$NACOS.AUTH}` | 空 | 鉴权附加查询串，如 `username=..&password=..` 或 `accessToken=..` |
| `{$NACOS.DELAY}` | `1m` | 实例级指标采集间隔 |
| `{$NACOS.DISCOVERY.DELAY}` | `1h` | 服务/命名空间发现间隔 |
| `{$NACOS.TIMEOUT}` | `10s` | HTTP 请求超时 |
| `{$NACOS.EXCLUDE}` | `^$` | 发现时排除的服务名正则 |
| `{$NACOS.SERVICE.DROP}` | `2` | 单次采集服务数下降多少算异常 |
| `{$NACOS.CONFIG.DROP}` | `10` | 单次采集配置数下降多少算异常 |
| `{$NACOS.CLIENT.MIN}` | `1` | 客户端连接数告警下限 |
| `{$NACOS.HEALTHY.MIN.PCT}` | `50` | 健康实例占比告警下限（%） |
| `{$NACOS.UNHEALTHY.WARN}` | `5m` | 健康实例持续为 0 多久告警；可按服务覆盖 |
| `{$NACOS.CPU.WARN}` | `0.6` | 整机 CPU 提醒线（0-1，15 分钟 min），只进日报 |
| `{$NACOS.CPU.CRIT}` | `0.8` | 整机 CPU 告警线（0-1，10 分钟 min），发邮件 |
| `{$NACOS.LOAD.WARN}` | `4.0` | 1 分钟负载提醒线（默认按 4 核），只进日报 |
| `{$NACOS.LOAD.CRIT}` | `6.0` | 1 分钟负载告警线（4 核的 1.5 倍），发邮件 |
| `{$NACOS.NODATA.PERIOD}` | `5m` | 主项无数据判定窗口 |

**换到核数不同的机器时，`{$NACOS.LOAD.WARN}` / `{$NACOS.LOAD.CRIT}` 要按核数重新算。**

## 告警分级

- **High**：发邮件（模板对象带 `svc=nacos` 标签，需配合你自己的告警动作，按 `svc=nacos` 过滤）。
- **Warning**：只进日报/看板，不发邮件（带 `scope=notice`）。
- 不健康实例的事件名会直接渲染 IP 列表，例如 `Service DEFAULT_GROUP/order-service has unhealthy instance(s) ["192.0.2.10"]`（示例使用文档保留地址）。

## 性能与规模

每个服务固定是 6 个项（健康数、实例数、健康占比、保护阈值、实例明细、不健康 IP）和 4 条触发器；模板另有 18 个固定项、14 条固定触发器，每个非 public 命名空间 1 个项和 1 条触发器。20 个服务时主机上约 140 个项、96 条触发器，HTTP 请求约 23 次/分钟。

服务数上到 200 时大约 1200 个项。需要压缩时可以：

- 把 `{$NACOS.DELAY}` 从 `1m` 调到 `2m/5m`；
- 关掉“实例明细”和“不健康实例”两个原型（它们才是每服务一次 HTTP 请求的项）；
- 保留 catalog 主项和健康数/实例数/占比，服务级健康仍然是 O(1) 请求。

## 已知限制

- **`mem` 不能告警**：Nacos 2.2.1 的 `EnvUtil.getMem()` 是 `1 - 整数除法(free/total)`，free < total 时永远返回 1.0。实测在 free 远小于 total 时仍返回 1.0，连续采样也全是 1.0。内存请用宿主机 agent 的 `vm.memory.utilization`、`vm.memory.size[available]` 和 OOM 日志。
- **没有集群/raft/JVM 指标**：只看 standalone 对外表现；多节点可以每个节点挂一次模板，但模板不做集群聚合，也不采堆、GC、线程池。
- **依赖 Nacos v1 OpenAPI**：`/nacos/v1/...` 在 2.x 可用，3.x 有移除风险；升级 Nacos 前先在测试环境验证这几个端点。
- **鉴权未在真实开启鉴权的 Nacos 上验证过**：`{$NACOS.AUTH}` 已留好，但填值后需要自行确认各端点都接受同一种写法。
- **服务消失只有总数维度**：任意一次减少会给 Warning，减少超过 `{$NACOS.SERVICE.DROP}` 给 High；没有做到“某个服务消失”的逐服务事件。
- **命名空间 LLD 只数配置条数**，不发现命名空间里的服务。
- 两个外部脚本是硬依赖；proxy 镜像里没有 `bash/wget` 时脚本会失败，需要在镜像或宿主里补。

## 测试环境

- Zabbix 7.0.30（server + Zabbix proxy）
- Nacos 2.2.1 standalone（JDK 17）
- 被测环境包含同名服务跨 group 注册的场景，用于验证 group 维度

## 许可证

- 模板与 README：GNU General Public License v2.0（见 `LICENSE`）
- 两个发现脚本：MIT（见 `scripts/LICENSE`）
