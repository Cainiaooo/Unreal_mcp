# UE MCP 工具库调研文档

生成日期：2026-01-27

## 范围与目标
本调研文档聚焦以下内容：
1. MCP 服务器与 UE 插件的整体架构与实现方式。
2. 与 Unreal Engine 的集成链路与握手流程。
3. 当前已支持的能力范围（工具、资源、GraphQL、WASM）。
4. 在现有代码基础上进行功能扩展的落地步骤与注意事项。

## 架构总览
这是一个“双进程 + WebSocket”架构：
1. TypeScript MCP 服务器（Node.js，JSON-RPC/STDIO）。
2. Unreal Editor 端 C++ 插件（UEditorSubsystem）。
两者通过 WebSocket Automation Bridge 进行能力调用。

### 关键组件（服务端）
- MCP 入口：`src/index.ts`
- 服务器注册与编排：`src/server-setup.ts`
- 工具注册与路由：`src/server/tool-registry.ts`
- 工具定义集合：`src/tools/consolidated-tool-definitions.ts`
- 工具处理集合：`src/tools/consolidated-tool-handlers.ts`
- UE 连接桥：`src/unreal-bridge.ts`
- WebSocket 桥接客户端：`src/automation/bridge.ts`
- 握手与消息处理：`src/automation/handshake.ts`、`src/automation/message-handler.ts`
- 资源注册：`src/server/resource-registry.ts`
- 健康检查与指标：`src/services/health-monitor.ts`、`src/services/metrics-server.ts`
- GraphQL：`src/graphql/server.ts`
- WASM 加速：`src/wasm/index.ts`

### 关键组件（UE 插件）
- 子系统主类：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Public/McpAutomationBridgeSubsystem.h`
- 请求分发：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Private/McpAutomationBridge_ProcessRequest.cpp`
- 连接管理：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Public/McpConnectionManager.h`
- 插件设置：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Public/McpAutomationBridgeSettings.h`
- 各功能处理器：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Private/*Handlers.cpp`

### 关键流程（从 MCP 到 UE）
1. MCP 客户端通过 STDIO 调用工具（JSON-RPC）。
2. `ToolRegistry` 解析请求并路由至 `consolidated-tool-handlers`。
3. 若需 UE 执行：`AutomationBridge.sendAutomationRequest` 发送 `automation_request`。
4. UE 插件侧 `FMcpConnectionManager` 解析消息并回调 `UMcpAutomationBridgeSubsystem`。
5. 子系统在游戏线程内顺序执行处理器（`*Handlers.cpp`），生成响应。
6. UE 侧返回 `automation_response`，TS 侧解析并回传给 MCP 客户端。

### 关键流程（握手与安全）
- 握手消息：`bridge_hello` → `bridge_ack`。
- 可选能力令牌：`MCP_AUTOMATION_CAPABILITY_TOKEN` / `CapabilityToken`。
- UE 端可强制校验令牌（`bRequireCapabilityToken`）。

### 运行时配套能力
- 连接健康与指标：`HealthMonitor` + `/metrics`（Prometheus 风格）。
- GraphQL：可选独立端口，适合复杂查询与聚合。
- WASM：为高消耗操作提供加速并保留 TS 回退。

## 与 Unreal Engine 集成方式
### 插件安装与启用
- 插件位置：`plugins/McpAutomationBridge/`。
- 可复制到项目：`<YourProject>/Plugins/McpAutomationBridge/`。
- 在编辑器中启用插件，并根据功能需要启用相关 UE 内置插件。

### 连接参数
- TS 侧环境变量（示例）：
  - `UE_PROJECT_PATH`
  - `MCP_AUTOMATION_HOST`
  - `MCP_AUTOMATION_PORT`
  - `MCP_AUTOMATION_CAPABILITY_TOKEN`
- UE 侧设置位于 Project Settings ▸ Plugins ▸ MCP Automation Bridge。

### 连接模式
- TS 侧为 WebSocket 客户端，默认连接 `ws://127.0.0.1:8090/8091`。
- UE 侧通过 `FMcpConnectionManager` 管理连接、心跳与重连。
- 插件内部所有请求在游戏线程处理，避免并发修改 UE 编辑器状态。

## 当前能力清单（概览）
### 工具集合（Consolidated Tools）
来源：`src/tools/consolidated-tool-definitions.ts`
- `manage_pipeline`（工具分组与过滤）
- `manage_asset`
- `manage_blueprint`
- `control_actor`
- `control_editor`
- `manage_level`
- `animation_physics`
- `manage_effect`
- `build_environment`
- `system_control`
- `manage_sequence`
- `manage_input`
- `inspect`
- `manage_audio`
- `manage_behavior_tree`
- `manage_lighting`
- `manage_performance`
- `manage_geometry`
- `manage_skeleton`
- `manage_material_authoring`
- `manage_texture`
- `manage_gas`
- `manage_character`
- `manage_combat`
- `manage_ai`
- `manage_inventory`
- `manage_interaction`
- `manage_widget_authoring`
- `manage_networking`
- `manage_game_framework`
- `manage_sessions`
- `manage_level_structure`
- `manage_volumes`
- `manage_navigation`
- `manage_splines`

### 资源（MCP Resources）
来源：`src/server/resource-registry.ts`
- `ue://assets`
- `ue://actors`
- `ue://level`
- `ue://health`
- `ue://automation-bridge`
- `ue://version`

### GraphQL 能力
来源：`docs/GraphQL-API.md`
- 支持资产、Actor、Blueprint 等结构化查询与变更。
- 独立端口与路径配置（`GRAPHQL_HOST` / `GRAPHQL_PORT` / `GRAPHQL_PATH`）。

### UE 端处理器映射
来源：`docs/handler-mapping.md`
- 文档维护了工具动作 → UE C++ 处理器文件/函数的映射关系。
- 用于快速定位与扩展对应 C++ 处理逻辑。

## 扩展方式与落地步骤
### 1. 扩展 MCP 工具（TypeScript 侧）
1. 在 `src/tools/consolidated-tool-definitions.ts` 添加新的工具或 action，并提供输入/输出 schema。
2. 在 `src/tools/consolidated-tool-handlers.ts` 中实现处理逻辑，或在 `src/tools/handlers/*` 新增专用处理器。
3. 若需要 UE 端执行，使用 `executeAutomationRequest` 发送 `automation_request`。
4. 如需资源型能力，扩展 `src/server/resource-registry.ts` 与 `src/handlers/resource-handlers.ts`。
5. 若涉及控制台命令，遵守 `CommandValidator` 安全限制（`src/utils/command-validator.ts`）。

### 2. 扩展 UE 插件能力（C++ 侧）
1. 在 `UMcpAutomationBridgeSubsystem` 中注册新 action：
   - 声明处理器函数（头文件）。
   - 在 `InitializeHandlers()` 注册 action → handler。
2. 在 `*Handlers.cpp` 实现具体逻辑，解析 JSON，调用 UE API。
3. 通过 `SendAutomationResponse` / `SendAutomationError` 返回结果。
4. 注意 UE 5.7 安全规范：
   - 禁用 `UPackage::SavePackage()`，使用 `McpSafeAssetSave`。
   - SCS 组件创建需使用 `SCS->CreateNode()`。
   - 禁用 `ANY_PACKAGE`，使用 `nullptr` 查找。
5. 避免阻塞游戏线程；必要时使用异步任务并回调。

### 3. 文档与测试
- 更新 `docs/handler-mapping.md` 以映射新 action。
- 集成测试集中在 `tests/integration.mjs`。
- 单元测试按需与源码同目录放置 `.test.ts`。

## 重要实现细节与注意事项
- TypeScript 侧严格零 `any`；使用 `unknown` 或接口。
- 运行时输出必须保持 JSON-RPC 纯净（默认将日志重定向到 stderr）。
- Automation Bridge 支持请求排队与最大并发控制。
- 支持离线模式：`MOCK_UNREAL_CONNECTION=true`。
- 多端口与心跳策略可配置，避免连接不稳定时的阻塞。

## 关键参考路径
- 架构入口：`src/index.ts`
- 连接桥：`src/automation/bridge.ts`
- 工具定义：`src/tools/consolidated-tool-definitions.ts`
- 工具处理：`src/tools/consolidated-tool-handlers.ts`
- UE 子系统：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Public/McpAutomationBridgeSubsystem.h`
- UE 请求分发：`plugins/McpAutomationBridge/Source/McpAutomationBridge/Private/McpAutomationBridge_ProcessRequest.cpp`
- Handler 映射：`docs/handler-mapping.md`


## 待补充内容（已整理）

### 1. 按工具分组的 action 概览（示例）
说明：完整映射见 `docs/handler-mapping.md`。

- manage_asset：list、import、duplicate、rename、move、delete、create_folder、search_assets、get_dependencies、create_material、create_material_instance、create_render_target、add_material_node、connect_material_pins、remove_material_node 等。
- manage_blueprint：create、get_blueprint、compile、add_component、set_default、modify_scs、get_scs、create_node、delete_node、connect_pins、break_pin_links、set_node_property 等。
- control_actor：spawn、spawn_blueprint、delete、duplicate、apply_force、set_transform、get_transform、set_visibility、add_component、add_tag、find_by_tag、list、attach、detach 等。
- control_editor：play、stop、set_camera、console_command、screenshot、create_bookmark、simulate_input 等。
- manage_level：load、save、create_level、export_level、import_level、load_cells、set_datalayer 等。
- animation_physics：create_animation_bp、play_montage、setup_ragdoll、configure_vehicle 等。
- manage_effect：niagara、spawn_niagara、debug_shape、create_niagara_system、create_niagara_emitter、add_niagara_module 等。
- build_environment：create_landscape、sculpt、paint_foliage、add_foliage_instances、get_foliage_instances、remove_foliage、create_procedural_terrain 等。
- system_control：execute_command、console_command、run_ubt、run_tests、subscribe、unsubscribe、spawn_category、start_session 等。
- manage_sequence：manage_sequence、add_keyframe、manage_track、add_camera_track、add_animation_track、add_transform_track 等。
- manage_input：create_input_action、create_input_mapping_context、add_mapping、remove_mapping 等。
- manage_audio / manage_behavior_tree / manage_lighting / manage_performance / manage_geometry / manage_skeleton / manage_material_authoring / manage_texture / manage_gas 等请以映射文档为准。

### 2. 时序与数据流（文字版）
- MCP 客户端发起工具调用（STDIO / JSON-RPC）。
- 服务端 `ToolRegistry` 校验参数并路由到工具处理器。
- 需要 UE 执行时，`AutomationBridge` 发送 `automation_request` 到插件。
- 插件 `FMcpConnectionManager` 解析消息，回调 `UMcpAutomationBridgeSubsystem`。
- 子系统在游戏线程顺序执行处理器，生成 `automation_response`。
- 服务端解析响应并回传给 MCP 客户端。

### 3. 新增工具的最小改动模板
TypeScript 侧：
1. 在 `src/tools/consolidated-tool-definitions.ts` 新增工具或 action，补齐输入/输出 schema。
2. 在 `src/tools/consolidated-tool-handlers.ts` 绑定处理逻辑，或拆到 `src/tools/handlers/*`。
3. 需要 UE 执行时通过 `executeAutomationRequest` 发送请求。

C++ 侧：
1. 在 `UMcpAutomationBridgeSubsystem` 声明新 handler 函数并注册 action。
2. 在 `*Handlers.cpp` 解析 JSON 并调用 UE API。
3. 使用 `SendAutomationResponse` / `SendAutomationError` 返回结果。

### 4. GraphQL 扩展与示例
- 启用：设置 `GRAPHQL_ENABLED=true`，端口由 `GRAPHQL_PORT` 控制。
- 适用场景：多对象聚合查询、嵌套关系查询。
- 示例：按类筛选资产并返回依赖关系与标签。

```graphql
query {
  assets(filter: { class: "Material" }) {
    edges {
      node {
        name
        path
        dependencies { name path }
        tags
      }
    }
    totalCount
  }
}
```

