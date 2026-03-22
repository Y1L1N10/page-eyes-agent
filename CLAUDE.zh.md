# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指引。

## 常用命令

```bash
# 安装依赖
uv sync

# 运行单个测试
pytest tests/test_harmony_agent.py::test_mobile_01 -v -s

# 按平台运行所有测试
pytest tests/test_harmony_agent.py -v -s
pytest tests/test_android_agent.py -v -s
pytest tests/test_ios_agent.py -v -s
pytest tests/test_web_agent.py -v -s
```

测试必须在项目根目录下执行。`pytest.ini` 位于 `tests/` 目录，已配置 `-s` 和 `asyncio_default_fixture_loop_scope=session`。

## 架构说明

PageEyes Agent 是基于 **Pydantic AI** 构建的自然语言驱动 UI 自动化框架。用户用中文或英文写操作指令，agent 负责规划并在真实设备上执行。

### 两种部署模式

通过 `.env` 中的 `AGENT_MODEL_TYPE` 控制：

- **`llm` 模式**（默认）：截图 → **OmniParser** 服务将 UI 元素解析为带 ID 和边界框的结构化列表 → LLM 选择元素 ID 执行操作。需要运行中的 OmniParser 服务（`OMNI_BASE_URL`）。
- **`vlm` 模式**：截图 base64 编码后直接发给多模态大模型，无需 OmniParser。坐标使用 0–999 归一化空间。

`LocationToolParams`、`ClickToolParams`、`InputToolParams` 和 `SYSTEM_PROMPT` 均在模块导入时根据 `default_settings.model_type` 切换行为。

### 三层结构

```
agent.py          ← UiAgent 子类（WebAgent、AndroidAgent、HarmonyAgent、IOSAgent）
tools/            ← AgentTool 子类，每个被 @tool 装饰的方法都会成为 LLM 可调用的工具
device.py         ← 设备封装层（WebDevice=Playwright、AndroidDevice=adb、HarmonyDevice=hdc、IOSDevice=WDA）
```

**`agent.py` 执行流程：**
1. `PlanningAgent` 将用户指令拆分为原子化的 `PlanningStep` 对象
2. 每个步骤由子 agent（`self.agent`）循环调用工具直到步骤完成
3. 每步结束后，若执行过程中未截图则自动补一张截图
4. 结果通过 `report_template.html`（编译后的 Vue 单文件应用）序列化为 HTML 报告

**`tools/_base.py`** 包含：
- `@tool` 装饰器 — 封装工具方法的前置/后置处理、延迟等待、异常捕获，失败时触发 `ModelRetry`
- `AgentTool.get_screen()` — 截图 → OmniParser → 将结果存入 `ctx.deps.context.current_step`
- 标记 `llm=False` 或 `vlm=False` 的工具会根据当前模式从工具列表中排除

**`tools/_mobile.py` → `MobileAgentTool`** 是 Android、鸿蒙、iOS 的共用基类，实现了 `click`、`swipe`、`open_url`、`open_app`（通过子 agent 将 App 名匹配到包名/bundle）。`HarmonyAgentTool` 和 `IOSAgentTool` 用平台专属命令覆盖了 `input` 和 `open_app`。

**各平台设备层差异：**

| 平台 | 调试工具 | 启动 App | 文本输入 |
|---|---|---|---|
| Android | adb (`adbutils`) | `am start` | `adb shell input text` |
| HarmonyOS | hdc (`hdcutils`) | `aa start` | `uitest inputText` |
| iOS | WebDriverAgent (`facebook-wda`) | WDA API | WDA API |
| Web | Playwright | `page.goto()` | Playwright `fill()` |

### 关键文件

- `src/page_eyes/config.py` — 所有配置通过 `pydantic-settings` 管理，从 `.env` 读取，各模块有独立前缀（`AGENT_`、`OMNI_`、`COS_`、`MINIO_`、`BROWSER_`）
- `src/page_eyes/deps.py` — `AgentDeps[DeviceT, ToolT]` 是依赖容器，通过 `RunContext` 在每次工具调用中传递；`StepInfo` 追踪每步状态
- `src/page_eyes/util/hdc_tool.py` — 鸿蒙设备封装；`window_size()` 解析 `hidumper` 输出（同时兼容 `render size:` 和 `render resolution=` 两种格式）
- `src/page_eyes/report_template.html` — 编译后的 Vue 单文件应用；通过字符串替换注入 `window.reportData = {reportData}`；前端实际读取的字段为：`is_success`、`device_size`、`steps[].{step, description, action, is_success, image_url, params.element_id, screen_elements[].{id,type,bbox,content}}`

### 新增平台的步骤

1. 在 `device.py` 中添加 `XxxDevice`（实现 `create()`，存储 `device_size`）
2. 在 `tools/xxx.py` 中添加 `XxxAgentTool`（继承 `AgentTool` 或 `MobileAgentTool`，实现抽象方法）
3. 在 `tools/__init__.py` 中导出
4. 在 `agent.py` 中添加 `XxxAgent(UiAgent)`，包含 `create()` 类方法
5. 在 `tests/conftest.py` 中添加对应 fixture

### 测试 fixtures（`tests/conftest.py`）

设备标识符在文件顶部设置：
- `serial` — Android 设备序列号（留空则使用第一个已连接设备）
- `connect_key` — 鸿蒙设备 key，通过 `hdc list targets` 获取
- `wda_url` — iOS WebDriverAgent 地址
- `platform` — `Platform.KG` / `Platform.QY` 等，影响 `open_url` 中的 URL schema 映射
