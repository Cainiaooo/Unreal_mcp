# Unreal_mcp 测试调研文档

生成日期：2026-01-28

## 目标
说明该 MCP 工具库在实现过程中如何进行代码准确性测试，以及自动化测试与人工测试的分工。

## 结论摘要
- **自动化测试存在但覆盖分层明显**：TS 侧具备单元测试、集成测试与 CI Smoke Test；UE 插件侧主要依赖集成测试与人工验证。
- **必须人工介入的环节仍然较多**：UE 编辑器必须启动并启用插件，很多流程依赖真实 Editor 环境。
- **文档与脚本存在不一致**：`docs/testing-guide.md` 中提到的 `integration-advanced.mjs` 与 `test:advanced` 在当前仓库未找到，实际执行脚本以 `package.json` 为准。

## 自动化测试体系

### 1. 单元测试（Vitest）
- 脚本：`npm run test:unit`
- 说明：只覆盖 TS 侧纯逻辑工具与安全校验。
- 位置：`tests/unit/**`
- 示例：
  - `tests/unit/tools/editor.test.ts`
  - `tests/unit/tools/asset_handlers_security.test.ts`
  - `tests/unit/graphql/*`

### 2. 集成测试（真实 UE 依赖）
- 脚本：`npm test`（等同 `node tests/integration.mjs`）
- 说明：通过 MCP Client 调用 MCP Server，再由 UE 插件执行。
- 要求：必须启动 Unreal Editor，并启用 `McpAutomationBridge` 插件。
- 位置：`tests/integration.mjs`、`tests/test-runner.mjs`
- 方式：
  - `tests/test-runner.mjs` 启动 MCP Server（`dist/cli.js`）
  - 通过 MCP JSON-RPC 调用工具并比对响应
  - 结果写入 `tests/reports/`

### 3. CI Smoke Test（Mock 模式）
- 脚本：`npm run test:smoke`
- 实现：`scripts/smoke-test.ts`
- 核心点：
  - 启动 MCP Server（`dist/cli.js`）
  - 发送 `initialize` 与 `tools/list`
  - 通过 `MOCK_UNREAL_CONNECTION=true` 跳过 UE 依赖
- 用途：保证 MCP Server 基础可运行、工具注册正常

### 4. CI / 质量门禁
- **ESLint**：`.github/workflows/ci.yml` 中运行 `eslint`（TS/JS）
- **CodeQL**：`.github/workflows/codeql.yml` 进行安全与质量分析
- **Smoke Test**：`.github/workflows/smoke-test.yml` 启动 mock 模式测试
- **依赖审查**：`.github/workflows/dependency-review.yml`

## UE 插件侧测试现状

### 自动化覆盖
- UE 插件暂无独立的 C++ 单元测试/自动化测试脚本。
- UE 插件的能力主要通过 **集成测试 + UE 编辑器运行时** 间接验证。

### 人工介入点
- **UE 编辑器必须启动**：集成测试依赖真实 Editor 状态。
- **插件启用与端口确认**：MCP Server 与插件 WebSocket 端口必须匹配。
- **复杂功能验证**：例如资产保存、蓝图编辑、Sequencer、Niagara 等操作仍需要人工确认结果。

## 文档与脚本差异（需要注意）

### `docs/testing-guide.md`
- 提到：`integration-advanced.mjs`、`test:advanced`、`test:all` 覆盖高级阶段测试。
- 实际仓库中：
  - 仅存在 `tests/integration.mjs`
  - `package.json` 中 `test:all` 仍指向 `tests/integration.mjs`
  - 未找到 `integration-advanced.mjs`

结论：文档内容可能滞后，需要以当前脚本为准。

## 当前测试流程的优点与限制

### 优点
- 基础自动化链路完整：单元测试 + 集成测试 + CI Smoke Test。
- Mock 模式使 CI 可运行，不依赖 UE 环境。
- 通过 MCP Client 测试链路覆盖了“客户端→Server→UE 插件”的实际调用路径。

### 限制
- UE 插件缺少独立自动化测试，集成测试依赖人工准备 UE 环境。
- 大量高复杂度功能（蓝图、资产、渲染、关卡）需要人工确认结果。
- 文档与脚本不一致可能导致误用或漏测。

## 结论：是否依赖人工测试
是的，**仍然依赖人工测试**，尤其是：
- 需要 UE 编辑器实时运行的功能；
- 复杂资产/蓝图/关卡等“结果可视化”操作；
- UE 版本差异带来的兼容性验证。

自动化测试主要保障：
- MCP Server 启动可用；
- 工具注册与 JSON-RPC 调用正常；
- TS 侧逻辑与安全校验可靠。

## 关键参考路径
- `docs/testing-guide.md`
- `tests/integration.mjs`
- `tests/test-runner.mjs`
- `tests/unit/**`
- `scripts/smoke-test.ts`
- `.github/workflows/ci.yml`
- `.github/workflows/smoke-test.yml`
- `.github/workflows/codeql.yml`
- `package.json`

