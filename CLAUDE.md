# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run a single test
pytest tests/test_harmony_agent.py::test_mobile_01 -v -s

# Run all tests for a platform
pytest tests/test_harmony_agent.py -v -s
pytest tests/test_android_agent.py -v -s
pytest tests/test_ios_agent.py -v -s
pytest tests/test_web_agent.py -v -s
```

Tests must be run from the project root. `pytest.ini` lives in `tests/` and sets `-s` and `asyncio_default_fixture_loop_scope=session`.

## Architecture

PageEyes Agent is a natural-language-driven UI automation framework built on **Pydantic AI**. The user writes instructions in plain Chinese/English; the agent plans and executes them on a real device.

### Two deployment modes

Controlled by `AGENT_MODEL_TYPE` in `.env`:

- **`llm` mode** (default): Screenshot → **OmniParser** service parses UI elements into a structured list with IDs and bounding boxes → LLM picks element IDs to act on. Requires a running OmniParser service (`OMNI_BASE_URL`).
- **`vlm` mode**: Screenshot is base64-encoded and sent directly to a multimodal LLM. No OmniParser needed. Coordinates use a 0–999 normalized space.

`LocationToolParams`, `ClickToolParams`, `InputToolParams`, and `SYSTEM_PROMPT` all switch behavior at import time based on `default_settings.model_type`.

### Three-layer structure

```
agent.py          ← UiAgent subclasses (WebAgent, AndroidAgent, HarmonyAgent, IOSAgent)
tools/            ← AgentTool subclasses, each method decorated with @tool becomes an LLM-callable tool
device.py         ← Device wrappers (WebDevice=Playwright, AndroidDevice=adb, HarmonyDevice=hdc, IOSDevice=WDA)
```

**`agent.py` execution flow:**
1. `PlanningAgent` breaks the user prompt into atomic `PlanningStep` objects
2. For each step, a sub-agent (`self.agent`) calls tools until the step completes
3. After each step, a screenshot is taken if none was captured during execution
4. Results are serialized to an HTML report via `report_template.html` (compiled Vue app)

**`tools/_base.py`** contains:
- `@tool` decorator — wraps tool methods with pre/post handlers, delays, error catching, and `ModelRetry` on failure
- `AgentTool.get_screen()` — screenshots → OmniParser → stores result in `ctx.deps.context.current_step`
- Tools tagged `llm=False` or `vlm=False` are excluded from the tool list depending on mode

**`tools/_mobile.py` → `MobileAgentTool`** is the shared base for Android, Harmony, and iOS, implementing `click`, `swipe`, `open_url`, `open_app` (uses a sub-agent to match app name to package/bundle). `HarmonyAgentTool` and `IOSAgentTool` override `input` and `open_app` with platform-specific commands.

**Device layer differences:**

| Platform | Tool | App launch | Text input |
|---|---|---|---|
| Android | adb (`adbutils`) | `am start` | `adb shell input text` |
| HarmonyOS | hdc (`hdcutils`) | `aa start` | `uitest inputText` |
| iOS | WebDriverAgent (`facebook-wda`) | WDA API | WDA API |
| Web | Playwright | `page.goto()` | Playwright `fill()` |

### Key files

- `src/page_eyes/config.py` — all config via `pydantic-settings`, reads `.env` with prefixes (`AGENT_`, `OMNI_`, `COS_`, `MINIO_`, `BROWSER_`)
- `src/page_eyes/deps.py` — `AgentDeps[DeviceT, ToolT]` is the dependency container passed through every tool call via `RunContext`; `StepInfo` tracks per-step state
- `src/page_eyes/util/hdc_tool.py` — HarmonyOS device wrapper; `window_size()` parses `hidumper` output (supports both `render size:` and `render resolution=` formats)
- `src/page_eyes/report_template.html` — compiled single-file Vue app; data injected via `window.reportData = {reportData}` string replacement; fields consumed: `is_success`, `device_size`, `steps[].{step, description, action, is_success, image_url, params.element_id, screen_elements[].{id,type,bbox,content}}`

### Adding a new platform

1. Add `XxxDevice` in `device.py` (implement `create()`, store `device_size`)
2. Add `XxxAgentTool` in `tools/xxx.py` (extend `AgentTool` or `MobileAgentTool`, implement abstract methods)
3. Export from `tools/__init__.py`
4. Add `XxxAgent(UiAgent)` in `agent.py` with `create()` classmethod
5. Add fixture in `tests/conftest.py`

### Test fixtures (`tests/conftest.py`)

Device identifiers are set at the top of the file:
- `serial` — Android device serial (blank = first connected device)
- `connect_key` — HarmonyOS device key from `hdc list targets`
- `wda_url` — iOS WebDriverAgent URL
- `platform` — `Platform.KG` / `Platform.QY` etc., affects URL schema mapping in `open_url`
