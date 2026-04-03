# Hello-Agents 项目 Code Wiki 文档

## 1. 项目整体架构
本项目是由 Datawhale 社区发起的《从零开始构建智能体》开源教程及实战项目库。
整体架构由四大部分组成：
1. **文档教程区 (`docs/`)**：包含从第一章到第十六章的完整理论与实战教程文档。
2. **源码实战区 (`code/`)**：对应各章节的配套代码，涵盖基础 Agent 范式、底层自研框架 HelloAgents 的实现、高级检索与记忆系统，以及后续综合实战（如智能旅行助手、赛博小镇等）。
3. **社区共创区 (`Co-creation-projects/`)**：社区开发者基于 HelloAgents 框架等技术构建的实战应用项目（如 CodeAgentCli、DeepCastAgent、ProgrammingTutor 等）。
4. **扩展阅读区 (`Extra-Chapter/`)**：收录关于面试问题、Dify 保姆级教程、环境配置、GUI Agent 科普等进阶分享。

## 2. 主要模块职责
- **`docs/`**：核心理论教材，按章节划分，讲解大语言模型基础、经典范式、记忆与检索、上下文工程及高级 RL。
- **`code/chapter4/`**：智能体经典范式的初级代码实现（ReAct、Plan-and-Solve、Reflection）。
- **`code/chapter7/`**：**HelloAgents 框架的孵化地**。从零构建出一个轻量级、教学友好、功能完备的 Agent 框架，定义了统一的 LLM 调用接口、消息系统、Agent 基类和工具系统。
- **`code/chapter13/ - chapter15/`**：进阶实战案例代码。分别实现了智能旅行助手（多智能体协作）、自动化深度研究智能体（DeepResearch）、赛博小镇（游戏与多智能体的结合）。
- **`Co-creation-projects/`**：存放各类垂直领域的智能体创新应用。展示了如何利用框架进行二次开发。

## 3. 关键类与函数说明 (核心：HelloAgents 框架)
项目中最核心的部分在于其自研的 **HelloAgents 框架**。以下是其关键类及职责拆解：

### 3.1 核心通信与数据结构
- **`HelloAgentsLLM`**：统一的大模型调用中枢。支持自动检测供应商（OpenAI、ModelScope、Zhipu 等）以及本地部署（VLLM、Ollama）。通过内部的 `_auto_detect_provider` 和 `_resolve_credentials` 函数智能推断环境配置。
- **`Message`**：统一消息格式数据类。约束 `role` 为 user、assistant、system 或 tool，支持 `to_dict()` 方法将其转换为兼容 OpenAI API 的格式。
- **`Config`**：框架配置类。提供中心化配置管理，提供 `from_env()` 类方法支持从环境变量中快速初始化。

### 3.2 智能体体系 (Agent 体系)
- **`Agent` (抽象基类)**：定义了智能体的核心属性（`name`、`llm`、`system_prompt`、`_history`）与抽象的 `run(input_text)` 方法规范。
- **`SimpleAgent`**：基础对话智能体，原生支持可选工具调用。核心在于 `_run_with_tools` 方法，通过正则匹配 `[TOOL_CALL:工具名:参数]` 进行多轮次的 LLM 与工具交互。
- **`ReActAgent`**：实现了 "Thought-Action-Observation" 推理循环。在自定义的系统提示词中强制约束行动格式，并结合 `ToolRegistry` 在限定步数 (`max_steps`) 内循环执行复杂任务。
- **`ReflectionAgent`**：实现了 "执行-反思-优化" 的循环。利用多套 prompt（initial、reflect、refine）对大模型的初次输出进行自我批判和二次优化。
- **`PlanAndSolveAgent`**：将复杂问题分解为子步骤（由 Planner 规划出 Python 列表），随后由 Executor 逐一解决并聚合上下文。
- **`FunctionCallAgent`**：利用 OpenAI 原生的 Function Calling 机制实现的 Agent，其核心包含 `_build_tool_schemas` 等方法，相较于纯 Prompt 约束具有更高的鲁棒性。

### 3.3 工具系统 (Tool System)
- **`Tool` (抽象基类)**：所有工具的基础抽象，必须实现 `run(parameters)` 核心执行逻辑与 `get_parameters()` 获取参数说明。
- **`ToolRegistry`**：工具注册表。支持对象级别注册 (`register_tool`) 和普通函数级别快速注册 (`register_function`)。内部提供 `to_openai_schema()` 方法，能将任意工具转换为原生 function calling 所需 JSON Schema。
- **`ToolChain` & `ToolChainManager`**：高级特性——工具链管理系统。支持添加 `steps` 并按顺序组合多个工具步骤执行，将前一步结果作为后一步输入（例如先 Search 再 Calculate）。
- **`AsyncToolExecutor`**：高级特性——基于 Python 线程池 (`ThreadPoolExecutor`) 提供异步的并行工具调度功能 (`execute_tools_parallel`)。

## 4. 依赖关系
- **基础运行环境**：Python 3.10+
- **核心依赖包**：
  - `hello-agents`：项目的主核心库。
  - `openai`：作为标准的底层大模型网络请求客户端库。
  - `pydantic`：用于实现 `Message`、`Config` 和 `ToolParameter` 等类的数据验证。
  - `python-dotenv`：用于加载本地 `.env` 环境变量配置。
- **可选依赖包 (取决于启用的具体工具)**：
  - `tavily-python`、`google-search-results` (SerpAPI)：用于支持多源智能搜索工具。
  - `vllm` / `ollama`：用于高性能本地大模型部署。

## 5. 项目运行方式
1. **克隆与环境准备**：
   下载项目并建立 Python 3.10 以上版本的虚拟环境。
2. **安装核心框架**：
   根据你想要运行的子项目，通过 `pip install "hello-agents>=0.2.8"` 进行安装。
3. **配置环境变量**：
   在运行目录（如 `code/chapter7/` 或 `Co-creation-projects/...` 下）创建 `.env` 文件。
   至少配置以下信息：
   ```env
   LLM_API_KEY="your-api-key"
   LLM_BASE_URL="https://api-inference.modelscope.cn/v1/"
   ```
4. **运行案例**：
   - 框架测试：进入 `code/chapter7/` 运行 `python test_simple_agent.py` 体验框架基础能力。
   - 综合应用：进入对应章节如 `code/chapter13/helloagents-trip-planner/backend`，运行 `python run.py` 等。

## 6. HelloAgents 框架实现拆解总结
HelloAgents 框架的设计理念是 **“轻量级、教学友好且万物皆为工具”**。
- **高度解耦**：底层通过 `HelloAgentsLLM` 兼容各种模型厂商，并统一转为标准 OpenAI 格式。
- **标准化与规范化**：上层的各类 Agent（Simple, ReAct, Reflection 等）不再是一团乱麻的面条代码，而是全部继承自 `Agent` 基类，并通过强化版 Prompt 控制输出格式。
- **灵活的工具拓展**：不管是普通函数还是封装完备的类，都能被 `ToolRegistry` 接管。通过文本正则匹配 `[TOOL_CALL:...]` 或者原生 API `to_openai_schema` 方式抛给模型，极大降低了用户从零开发智能体应用的门槛。
