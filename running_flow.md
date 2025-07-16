# 🚀 项目执行流程

在用户输入 `prompt` 后，整个项目会按照以下流程执行任务：

---

## 🧠 1. **主入口：main.py**
用户通过命令行或交互式终端输入 prompt：
```python
prompt = args.prompt if args.prompt else input("Enter your prompt: ")
```


- 如果没有提供 prompt，则等待用户手动输入。
- 然后调用 `agent.run(prompt)` 开始执行。

### 🔗 调用链：
```
main.py → Manus.run()
```


---

## 🤖 2. **Manus Agent 初始化与运行**
[Manus](file:///Users/liuhao/Projects/AI/OpenManus/app/agent/manus.py#L17-L164) 是核心代理类（位于 [/app/agent/manus.py](file:///Users/liuhao/Projects/AI/OpenManus/app/agent/manus.py)），它继承自 [BaseAgent](file:///Users/liuhao/Projects/AI/OpenManus/app/agent/base.py#L12-L195) 并实现具体的任务逻辑。

### ✅ 主要行为：
- **初始化 MCP 服务**（用于与其他工具通信）
- **加载浏览器上下文助手**（用于自动化网页操作）
- **运行 [run(prompt)](file:///Users/liuhao/Projects/AI/OpenManus/app/agent/base.py#L115-L153) 方法**

### 🔁 [run()](file:///Users/liuhao/Projects/AI/OpenManus/app/agent/base.py#L115-L153) 方法逻辑：
```python
async def run(self, request: Optional[str] = None) -> str:
    ...
    while self.current_step < self.max_steps and self.state != AgentState.FINISHED:
        self.current_step += 1
        step_result = await self.step()
```


- 最大步数限制为 `max_steps`
- 每一步调用 `step()` 执行具体动作

### 🔗 调用链：
```
Manus.run() → Manus.step() → BaseAgent.step()
```


---

## 🔄 3. **BaseAgent 的 step() 流程**
这是整个代理的核心循环控制方法（位于 `/app/agent/base.py`）。

### 🧩 核心步骤：
1. **获取当前状态**（如浏览器页面、历史消息等）
2. **生成下一步提示词**（Next Step Prompt）
3. **调用 LLM 获取响应**
4. **解析响应并执行工具调用**
5. **更新记忆和状态**

#### 示例代码片段：
```python
async def step(self) -> str:
    # 1. 获取当前状态
    state = await self.get_current_state()

    # 2. 构造 prompt
    next_prompt = self.format_next_step_prompt(state)

    # 3. 调用 LLM
    response = await self.llm.ask(next_prompt)

    # 4. 解析响应
    tool_call = parse_tool_call(response)

    # 5. 执行工具
    result = await tool_call.execute()

    # 6. 更新记忆
    self.memory.add_message(Message(...))

    return result
```


### 🔗 调用链：
```
BaseAgent.step() → LLM.ask() → ToolCall.execute()
```


---

## 🧮 4. **LLM 处理请求**
LLM 类封装了对语言模型的调用（位于 [/app/llm.py](file:///Users/liuhao/Projects/AI/OpenManus/app/llm.py)）。

### ⚙️ 主要功能：
- 支持多模态输入（文本 + 图像）
- 自动 token 计算和限制检查
- 错误重试机制（如网络错误）

### 📦 请求参数示例：
```json
{
  "model": "claude-3-opus",
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "用户输入的 prompt..."}
  ],
  "temperature": 0.0,
  "max_tokens": 8192
}
```


### 🔗 调用链：
```
LLM.ask() → OpenAI API / Anthropic API / Ollama
```


---

## 🛠️ 5. **工具调用（Tool Call）**
根据 LLM 返回的结果，代理将决定是否调用某个工具。

### 📌 常见工具包括：
| 工具名 | 功能 |
|-------|------|
| `BrowserUseTool` | 浏览器自动化操作（点击、输入、搜索等） |
| `PythonExecuteTool` | 执行 Python 脚本 |
| `WebSearchTool` | 执行搜索引擎查询 |
| `FileOperatorTool` | 文件读写操作 |
| `TerminateTool` | 结束任务 |

#### 示例调用格式：
```json
{
  "action": "click_element",
  "index": 10
}
```


### 🔗 调用链：
```
ToolCall.execute() → 具体工具类（如 browser_use_tool.py）
```


---

## 🖥️ 6. **浏览器自动化流程（Browser Use Tool）**
如果任务涉及网页操作，将使用 [BrowserUseTool](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/browser_use_tool.py#L38-L566) 来控制浏览器。

### 🧭 支持的操作：
- `go_to_url`: 导航到指定 URL
- `click_element`: 点击元素
- `input_text`: 输入文本
- `extract_content`: 提取页面内容
- `scroll_down/up`: 滚动页面
- [web_search](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/web_search.py#L0-L0): 使用内置搜索引擎

### 📈 内部逻辑：
- 使用 Playwright 或 Puppeteer 控制浏览器
- 支持 headless 模式或可视化模式
- 可截图上传给 LLM 辅助理解

---

## 📁 7. **文件系统操作**
部分任务需要读写文件（如生成 HTML 页面、保存分析结果）。

### 📂 相关模块：
- [file_operators.py](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/file_operators.py)：封装文件读写逻辑
- [workspace_root](file:///Users/liuhao/Projects/AI/OpenManus/app/config.py#L329-L331)：工作目录路径

---

## 🧪 8. **沙箱环境执行代码**
对于涉及数据处理、脚本执行的任务，会启动沙箱环境执行 Python 代码。

### 🧱 实现方式：
- 使用 Docker 容器隔离执行环境
- 限制资源使用（CPU、内存、超时）
- 支持异步执行命令

---

## 🧾 9. **可视化 & 数据分析**
当用户要求图表展示或数据分析时，会调用以下组件：

### 📊 组件结构：
- [data_visualization.py](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/chart_visualization/data_visualization.py)：负责调用前端 TS 脚本生成图表
- [chartVisualize.ts](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/chart_visualization/src/chartVisualize.ts)：TS 脚本，使用 VChart 渲染图表
- `vmind`：智能洞察模块，自动提取数据趋势、异常点等

---

## 🧨 10. **异常处理与终止**
在整个执行过程中，有完善的异常处理机制：

### ❗ 异常类型：
- [TokenLimitExceeded](file:///Users/liuhao/Projects/AI/OpenManus/app/exceptions.py#L11-L12): 提示词过长
- `OpenAIError`: LLM 接口异常
- [ToolError](file:///Users/liuhao/Projects/AI/OpenManus/app/exceptions.py#L0-L4): 工具调用失败
- `KeyboardInterrupt`: 用户中断执行

### 🚫 终止条件：
- 用户主动输入 `exit` / `q`
- 到达最大步数限制
- 调用 [terminate](file:///Users/liuhao/Projects/AI/OpenManus/app/tool/terminate.py#L0-L0) 工具

---

## 📐 总结图解（调用流程）

```
[User Input Prompt]
        ↓
   main.py (Manus.run())
        ↓
   Manus.step() → BaseAgent.step()
        ↓
   LLM.ask() → 获取模型回复
        ↓
   ToolCall.execute() → 执行工具
        ↓
[Browser / Code Execution / File Op / Search]
        ↓
   Update Memory & Loop
        ↓
[Repeat until done or max steps reached]
```


---
