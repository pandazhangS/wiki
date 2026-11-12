# Higress Wasm 插件内存治理方案

> 状态: 调研 + 治理方案 v1
> 日期: 2026-11-12
> 范围: Higress 网关 Wasm 插件引起的 Envoy 驻留内存（RSS）增长的机制、观测、版本修复与治理清单
> 基准: 代码证据基于 Higress v2.1.9（HEAD `ec5031c2`）；发布信息与 issue 结论核对到 Higress v2.2.6

## 1. 结论

官方已有明确结论和一组可落地的缓解手段，但没有"一键解决"的开关。

- **官方定性**: Wasm VM 的线性内存按设计单调增长，free 掉的内存只在 VM 内复用，不归还操作系统；只有重建 VM 时才归还。旧 VM 被 in-flight 请求或长连接持有时，重建也无法回收。对应 issue 按 mechanism-level（机制层面）关闭。
- **规模关系**（粗略）: `RSS ≈ worker 线程数 × 启用插件数 × 单 VM 堆大小`，重建期间还会短时出现新旧两代 VM 叠加。
- **治理路径**: 先量化（`active/created/rebuild/memory_size`），再升级到包含重建优化的版本、降低单 VM 堆（body 缓冲）、保证旧代能排空、控制 VM 数量，并按双代峰值做容量预留。

## 2. 机制与官方定性

维护者 johnlanni 在 [higress#3920](https://github.com/higress-group/higress/issues/3920) 的关闭说明（2026-09-28）中给出的结论：

> Wasm VM 的堆按设计单调增长: 释放的内存只在 VM 内部复用，不会归还给操作系统。只有重建 VM 时内存才会归还。如果 VM 被 in-flight 请求长期持有，重建也无法回收。这是 Wasm 内存模型的固有限制，网关无法完全绕过。

补充事实：

- Envoy 的 Wasm VM 按 worker 线程创建、相互隔离；Higress 按插件维度各有一组 VM，指标中体现为 `plugin.<namespace>.<name>.worker_N.memory_size`。
- 单个 VM 的线性内存在 issue 中被观测到达到 `1073741824`（1 GiB），对应大请求体或长期未回收的堆。这个数值是排查信号，不是所有插件的常态。
- VM 重建时旧代不会立刻销毁: 旧代要等在途请求/流结束。因此重建期间新旧两代共存，RSS 会短时上升。
- RSS 在流量高峰后不回落是预期行为，不能单独作为"泄漏"的判据。官方给出的反例边界是: 空闲网关上重建完成后内存仍不下降，才按具体泄漏另开 issue。

需要区分两类问题：

1. **机制性增长**: VM 堆只增不减，属于本文治理范围。
2. **真实泄漏**: 例如 2.1.6 修复的 `proxy-wasm-cpp-host` 内存泄漏（[#2662](https://github.com/higress-group/higress/pull/2662)），升级即可消除。

## 3. 版本与修复时间线

| 版本 | 变更 | 对内存的意义 |
|---|---|---|
| 2.1.6 | 修复 `proxy-wasm-cpp-host` 内存泄漏 | 消除真实泄漏路径 |
| 2.1.9 | 新增按插件、按 worker 的 `memory_size` 指标 | 可定位到具体插件与线程 |
| 2.2.3 | 移除 request-count 触发的重建（[#3923](https://github.com/higress-group/higress/pull/3923)）；MCP filter 高内存时重建；合入 Envoy 代际感知重建优化（[envoy#23](https://github.com/higress-group/envoy/pull/23)） | 减少重建风暴，允许旧代存活时重建当前代，避免配置更新绕过双代限制堆积 VM |
| 2.2.6 | 当前最新版本，Envoy 升级到 1.36.12 | 包含上述修复与后续稳定性改进 |

插件侧：`ai-proxy` 2.0.0 起已去掉 `WithRebuildAfterRequests(1000)`；v2.1.9 的 `ai-proxy`、`ai-statistics`、`transformer` 仍带每 1000 请求重建的触发，高并发下会产生重建风暴。

## 4. 观测与分型

### 4.1 指标

在网关容器内执行：

```bash
curl -s 'http://127.0.0.1:15000/stats?filter=wasm' \
  | egrep 'active|created|rebuild|recover|crash|memory_size'
curl -s 'http://127.0.0.1:15000/memory'
ps -o pid,rss,vsz,cmd -p <envoy-pid>
```

Prometheus 侧的指标名会把点替换为下划线，例如 `wasm_envoy_wasm_runtime_v8_active`。

| 指标 | 含义 | 判断 |
|---|---|---|
| `wasm.envoy.wasm.runtime.v8.active` | 当前存活 VM 数 | 大致等于 worker 数 × 启用插件数；持续上涨且不回落说明旧代堆积 |
| `wasm.envoy.wasm.runtime.v8.created` | 累计创建 VM 数 | 持续增长说明重建频繁 |
| `...plugin.<ns>.<name>.rebuild_total` | 插件重建次数 | 与 `created` 同步增长 |
| `...plugin.<ns>.<name>.recover_total` / `crash_total` / `recover_error` | 崩溃与恢复 | 非 0 需结合插件日志排查 |
| `...plugin.<ns>.<name>.worker_N.memory_size` | 单 worker 的 VM 线性内存 | 常见值在 24–96 MiB；接近 1 GiB 重点查大 body 与回收阻塞 |
| `...plugin.<ns>.<name>.main_thread.memory_size` | 主线程/singleton VM | 一般较小 |

### 4.2 日志

```bash
egrep 'wasm vm requested rebuild|wasm vm rebuild success|proactive rebuild skipped|remaining active|Plugin reached rebuild|RuntimeError|~Wasm' /var/log/higress/gateway.log
```

重点日志：

- `wasm vm requested rebuild, try to rebuild` / `wasm vm rebuild success`: 发生过重建。
- `wasm vm proactive rebuild skipped: current generation has N active streams`: 长连接或 in-flight 请求阻止了旧代回收。
- `~Wasm ... remaining active`: 旧代仍未释放。
- `Plugin reached rebuild threshold after 1000 requests`: 旧版本插件的请求数重建触发。
- `Plugin reached rebuild memory threshold`: 内存阈值重建触发。

### 4.3 分型

| 现象 | 判断 | 处置 |
|---|---|---|
| `active` 稳定在"worker 数 × 插件数"附近，RSS 阶梯上涨后进入平台期 | 机制性增长 | 按峰值做容量规划 |
| `active` 持续上涨，`created` 也涨，日志中旧代释放很少 | 旧代被长连接或 in-flight 请求持有 | 限制流/连接时长，让旧代排空 |
| `rebuild_total` 快速增长 | 重建频繁 | 升级插件与网关，去掉 request-count 重建 |
| 单 worker `memory_size` 接近 1 GiB | 大请求体/大响应体在 VM 内堆积 | 启用轻量模式、限制 body，避免整体缓冲 |

## 5. 治理手段

### 5.1 升级到包含重建优化的版本

- 网关升级到 2.2.3 及以上，当前可直接以 2.2.6 为目标。
- 同步升级 `WasmPlugin` 引用的插件镜像，确认 `ai-proxy` 为 2.0.0 及以上。
- 升级后观察 `active` 是否回落到"worker 数 × 插件数"附近，以及 `created` 增速是否下降。

### 5.2 降低单 VM 堆

最有效的是减少插件对大 body 的整体缓冲：

- `ai-statistics` 生产环境开启轻量模式 `use_default_response_attributes: true`，不缓冲完整请求体和流式响应体。
- 设置 `max_request_body_bytes`（默认 100 MiB，上限 200 MiB，对应插件 VM 的重建上限），避免 multipart、base64 大图进入插件缓冲。
- 插件代码避免在请求生命周期内长期持有大块数据。Go 编译的 Wasm 使用保守式 GC，大块非 UTF-8 数据处理容易出现"看起来像泄漏"的增长，社区在 [#1714](https://github.com/higress-group/higress/issues/1714) 有讨论；重数据场景可考虑 Rust/C++ 插件。
- 必要时收紧 Envoy 的连接/请求缓冲区上限，降低峰值堆占用。

### 5.3 保证旧代能回收

重建被长连接阻塞时，内存阈值触发也不会生效。官方建议通过 EnvoyFilter 下发最大连接维持时间：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: wasm-vm-drain
  namespace: higress-system
spec:
  workloadSelector:
    labels:
      app: higress-gateway   # 按实际 gateway pod 标签调整
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: MERGE
      value:
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          common_http_protocol_options:
            max_stream_duration:
              seconds: 1800
            max_connection_duration:
              seconds: 3600
```

注意事项：

- 阈值必须大于业务最长合法流；超时的 SSE / WebSocket 会被主动断开。
- 时长上限只负责让旧代排空，重建仍需要触发源: 插件内存阈值、配置变更或崩溃恢复。两者要配合使用。
- 对自研插件，在 wasm-go wrapper 中设置内存阈值重建，例如 `wrapper.WithRebuildMaxMemBytes[config.PluginConfig](200 * 1024 * 1024)`；官方 `ai-proxy` / `ai-statistics` 当前使用 200 MiB。

### 5.4 控制 VM 数量

- 减少启用的插件实例；多个小插件可以合并成一个插件。
- 一个 `WasmPlugin` 实例通过 `matchRules` 在多条路由上生效时仍是一份 VM；拆成多个 `WasmPlugin` 实例才会增加 VM。
- VM 数随 Envoy worker 线程数线性增长（`--concurrency`，默认跟随 CPU）。在插件数量不变时，可以改用"更少 CPU 的单副本 + 更多副本"的水平扩展方式，用副本数换吞吐。
- Higress 的 `WasmPlugin` CR 只暴露 `vmConfig.env`，无法通过 CR 配置运行时或 `vm_id`。Envoy 的跨插件共享 VM 要求 `vm_id/runtime/code/configuration` 完全一致，实际可操作的手段仍是减少插件数量。

### 5.5 容量与告警

- Pod 内存 limit 按 **峰值 × 2** 预留，覆盖重建期间两代 VM 共存。
- 对以下信号建告警: `active` 持续上涨、`rebuild_total` 速率异常、单 worker `memory_size` 接近 1 GiB、`crash_total`/`recover_error` 非 0。
- 滚动重启只作为最后手段，会断开连接；常规治理不应依赖重启。

### 5.6 长期选项：WAMR

官方博客介绍过从 V8 切换到 WAMR 的运行时方案：AOT 模式下性能更好，且 WAMR 的资源开销显著低于 V8，官方称可在约 100 KB 内存中运行单个 Wasm 实例。

需要注意：运行时属于数据面镜像/构建层面的选择，不是 `WasmPlugin` CR 的字段。采用前先确认所用 Higress 网关镜像是否内置 WAMR，并验证插件 AOT 文件与现有插件生态的兼容性。

参考: [Higress 全新 Wasm 运行时，性能大幅提升](https://higress.ai/blog/higress-gvr7dx_awbbpb_rggp2v9saa6gehsg/)

## 6. 最小落地清单

1. 网关升级到 2.2.3+（目标 2.2.6），插件镜像确认 `ai-proxy` 2.0.0+。
2. `ai-statistics` 开启 `use_default_response_attributes: true`，设置 `max_request_body_bytes`。
3. 给长连接/SSE 场景配置 `max_stream_duration` 与 `max_connection_duration`。
4. 建立 `active`、`rebuild_total`、单 worker `memory_size` 的监控与告警。
5. 按双代峰值重新规划网关 Pod 的内存 request/limit。

## 7. 证据来源

- [higress#3920: envoy 进程内存持续增长](https://github.com/higress-group/higress/issues/3920) — 官方机制性结论、指标与日志排查方法
- [higress#3923: 移除 request-count rebuild](https://github.com/higress-group/higress/pull/3923) — 重建触发调整
- [envoy#23: guard proactive rebuilds by active generation](https://github.com/higress-group/envoy/pull/23) — 代际感知重建优化
- [higress#2662](https://github.com/higress-group/higress/pull/2662) — `proxy-wasm-cpp-host` 泄漏修复（2.1.6）
- [Wasm 插件原理](https://higress.ai/docs/latest/user/plugins/wasm-dev/wasm15/) — VM 按线程隔离、VM 共享条件、运行时类型
- [ai-statistics README](https://github.com/higress-group/higress/blob/main/plugins/wasm-go/extensions/ai-statistics/README.md) — 轻量模式与 body 上限
- [higress#1714](https://github.com/higress-group/higress/issues/1714) — Go Wasm 保守式 GC 讨论
