# Elasticsearch 配置下发流程（从“改配置”到“各节点生效”）

本文描述 Elasticsearch 集群配置从发起变更到节点生效的端到端流程，重点覆盖动态配置（`/_cluster/settings`）的下发链路，并补充静态配置的差异。

## 1. 端到端主流程图

```mermaid
flowchart TD
    A["运维/平台发起配置变更\n(API/控制台/自动化任务)"] --> B["入口节点接收请求\nPUT /_cluster/settings"]
    B --> C["请求路由到 Master 节点"]
    C --> D["Master 执行预检\n- 权限校验\n- setting 名称/类型校验\n- dynamic/static 属性校验\n- 取值范围/依赖关系校验"]
    D -->|校验失败| E["返回 4xx/错误原因\n(不进入发布)"]
    D -->|校验通过| F["提交 ClusterStateUpdateTask"]
    F --> G["在 Master 单线程更新集群状态\n生成新 cluster state(version+1)"]
    G --> H["持久化/记录元数据\n(按实现写入集群状态存储)"]
    H --> I["Master 发起集群状态发布\npublish new cluster state"]
    I --> J["各节点接收并反序列化 state diff/完整 state"]
    J --> K["节点本地二次校验与应用前检查\n- 版本连续性\n- 兼容性"]
    K -->|异常| L["节点拒收/请求重传/等待下一轮发布"]
    K -->|通过| M["触发 Setting Update Consumer\n将新值应用到模块"]
    M --> N["模块生效\n- 路由/恢复策略\n- 断路器/限流\n- 缓存/查询相关动态参数等"]
    N --> O["节点向 Master ACK"]
    O --> P{"Master 收齐 ACK?\n(法定多数/超时策略)"}
    P -->|是| Q["发布完成\nAPI 返回 acknowledged:true"]
    P -->|否| R["超时/部分成功\nAPI 可能返回 acknowledged:false"]
    Q --> S["运维侧验收\nGET _cluster/settings\nGET _nodes/settings\n业务指标/日志观察"]
    R --> S
```

## 2. 动态配置与静态配置分支

```mermaid
flowchart TD
    A["变更请求进入平台"] --> B{"配置类型?"}

    B -->|动态配置 Dynamic| C["调用 Cluster Settings API"]
    C --> D["Master 校验并更新 Cluster State"]
    D --> E["发布到所有节点"]
    E --> F["节点在线应用（无需重启）"]
    F --> G["ACK + 验收"]

    B -->|静态配置 Static| H["修改 elasticsearch.yml/JVM 参数"]
    H --> I["灰度重启节点\n(滚动重启策略)"]
    I --> J["节点启动时加载新配置"]
    J --> K["加入集群并恢复分片"]
    K --> L["继续下一节点，直至全量完成"]
    L --> M["最终验收"]
```

## 3. 关键时序图

```mermaid
sequenceDiagram
    autonumber
    participant U as 运维/平台
    participant C as 协调节点
    participant M as Master节点
    participant D1 as 数据节点A
    participant D2 as 数据节点B

    U->>C: PUT /_cluster/settings (新配置)
    C->>M: 转发配置更新请求
    M->>M: 校验(权限/类型/动态属性/范围)
    alt 校验失败
        M-->>C: 4xx + error reason
        C-->>U: 失败响应
    else 校验通过
        M->>M: ClusterStateUpdateTask 更新状态(version+1)
        M->>D1: publish cluster state(diff/full)
        M->>D2: publish cluster state(diff/full)
        D1->>D1: 应用 setting consumer 到本地模块
        D2->>D2: 应用 setting consumer 到本地模块
        D1-->>M: ACK
        D2-->>M: ACK
        M-->>C: acknowledged=true/false
        C-->>U: 返回结果
    end
```

## 4. 关键节点说明

- **校验阶段（Master）**：决定“能不能改”，常见失败包括参数拼写错误、类型不匹配、配置不支持动态更新。
- **Cluster State 更新**：决定“改了什么”，成功变更会形成新的状态版本。
- **发布阶段（Publish）**：决定“怎么下发”，Master 向各节点广播新状态（增量或全量）。
- **应用阶段（Consumer）**：决定“哪里生效”，节点通过对应模块的更新回调应用新配置。
- **ACK 与验收**：决定“是否形成闭环”，需结合 API 返回、节点视角和业务指标共同确认。

## 5. 变更与验收建议

1. 变更前确认目标参数是否为动态配置。
2. 变更后执行：
   - `GET /_cluster/settings?include_defaults=true`
   - `GET /_nodes/settings`
3. 观察关键日志与指标（延迟、拒绝率、恢复速度、资源利用率）。
4. 大集群采用分批灰度策略，避免一次性全量变更。
5. 每次变更保留回滚值，出现异常时快速恢复。
