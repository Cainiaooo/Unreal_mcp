---
name: unreal-mcp-usage
description: "指导在 UE 工程开发中初始化并使用 Unreal_mcp：检查 UE 插件、配置 AI 工具的 NPX MCP、用 NPX 安装/启动服务器、验证连接，并在初始化失败时输出明确不可用错误。"
---

# Unreal_mcp 使用指导

## 目标
在调用 Unreal_mcp 之前完成初始化流程，并确保连接成功。任何一步失败都必须停止，并输出“无法使用 Unreal_mcp 工具”的错误。

## 初始化流程（必须按顺序执行）

### 1. 检测 UE 插件是否存在并启用
执行以下检查之一即可：
- 项目目录存在：`<UE项目>/Plugins/McpAutomationBridge/`
- 或 `.uproject` 中声明并启用该插件

若检测失败，直接进入“失败处理”。

### 2. 配置 AI 工具的 MCP（NPX 方式）
在 AI 工具的 MCP 配置中添加 Unreal_mcp。示例：

```json
{
  "mcpServers": {
    "unreal-engine": {
      "command": "npx",
      "args": ["unreal-engine-mcp-server"],
      "env": {
        "UE_PROJECT_PATH": "C:/Path/To/YourProject/YourProject.uproject",
        "MCP_AUTOMATION_HOST": "127.0.0.1",
        "MCP_AUTOMATION_PORT": "8091"
      }
    }
  }
}
```

要求：
- 必须设置 `UE_PROJECT_PATH` 为真实 `.uproject` 绝对路径。
- 若 UE 插件监听端口不同，必须同步修改 `MCP_AUTOMATION_PORT`。

### 3. 使用 NPX 安装并启动服务器
执行或触发 AI 工具启动：
- `npx unreal-engine-mcp-server`

注意：
- 首次运行会自动下载依赖。
- UE 编辑器必须已启动，且插件已启用。

### 4. 验证准备完成
满足以下任意一项即可：
- MCP 客户端显示服务器已就绪并连接到 UE。
- 资源 `ue://health` 可读取且状态正常。

只有在验证成功后，才允许调用 Unreal_mcp 工具。

## 失败处理（必须输出明确错误）
任一步骤失败都必须输出以下格式，并停止后续动作：

```
无法使用 Unreal_mcp 工具：<失败原因>。
请确认：UE 插件已安装并启用、AI 工具已配置 NPX、Unreal_mcp 服务器可启动且端口匹配。
```

常见失败原因示例：
- 未找到 `McpAutomationBridge` 插件。
- MCP 配置缺少 `UE_PROJECT_PATH`。
- `npx unreal-engine-mcp-server` 启动失败。
- UE 编辑器未启动或插件未启用。

## 使用前置条件（简要）
- UE 编辑器已运行并启用 MCP 插件。
- MCP 配置使用 NPX 且环境变量正确。
- Unreal_mcp 服务器已启动并可连接。

