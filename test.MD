# Agent 评测平台完整详细方案文档

**文档版本**：1.0  
**撰写日期**：2026 年 3 月  
**文档背景**：本文档基于与 Grok 的完整对话记录整理而成，涵盖了从最初方案设计到最终落地细节的所有讨论内容，包括 ReAct + Skill + Subagent 架构的黑盒 Agent 评测难点、多接口处理、返回内容解析、工具/Skill 调用评测（含动态加载）、DeepEval 集成、页面配置化实现、多段响应处理、系统整体架构、非侵入式平台的优缺点、多 Agent 协作扩展、参考平台对比等全部细节。文档采用连贯叙述形式，力求详尽、可直接作为产品方案、开发文档、团队 Wiki 或对外汇报材料使用。

## 1. 项目背景与核心挑战

当前要评测的 Agent 采用 **ReAct + Skill + Subagent** 架构，是一个完全黑盒的系统，可能使用非 Python、非 LangChain 框架开发，且对外暴露多个接口（create、stop、session、chat 等）。Agent 返回内容可能是多段式的（包含 thought、action、tool call、observation、final answer 等），Skill 可能在运行时动态加载，Tool/Skill 调用正确性与效率的标准由 Agent 侧定义而非平台硬编码。

传统单一模型 LLM-as-Judge 方式在这种场景下容易出现幻觉、不一致、偏见等问题，无法可靠处理多接口生命周期、复杂解析、动态 Skill、协作链路等问题。因此，需要构建一个**非侵入式、配置化、网页驱动的黑盒评测平台**，以 DeepEval 为核心引擎，实现端到端质量保障。

平台的核心定位是：**配置驱动的黑盒端到端 Agent 质量平台**。用户（产品、测试、Agent 开发者）通过网页自行配置接口、输入字段、输出提取规则、参考标准、指标等，一键运行评测并获得详细报告。平台不对被测 Agent 做任何侵入式修改（无 SDK、无 instrumentation、无代码权限要求），完全通过公开 API 接口进行评测。

## 2. 整体方案思路与框架改造

针对黑盒 + 多接口 + 多段响应 + 动态 Skill 的复杂性，方案采用**分层评测 + 自动化 Harness + 配置化**的混合架构，核心是构建 **Test Harness（评测主程序）**，将评测拆分为四个独立维度，先用确定性方法覆盖大部分硬指标，再用 LLM 处理主观部分。

**详细拆解的四个维度**：

- **接口层**：处理 create/stop/session 等多接口的生命周期管理、并发、超时、资源释放。实现方式：异步 AgentClient 类 + 状态机验证，确保每次测试都完整走 create → 多轮 chat → stop 流程，并检查资源是否正确释放。
- **解析层**：解决返回内容格式漂移问题。核心采用 Pydantic Schema 强制校验作为第一道防线，如果失败则 fallback 到 LLM 结构化提取或正则兜底。针对多段响应，新增 segments 列表收集机制，后续通过配置化的提取规则解析出 final_answer、tools_called、thoughts 等字段。
- **工具/Skill 调用层**：重点评测调用正确性与效率。由于标准由 Agent 侧定义，平台提供参考注入机制（测试用例中允许用户上传 expected_tools JSON，包含 name、params_schema、order）。平台负责从响应中提取实际调用列表，然后进行精确匹配（名字 + 参数 + 可选顺序），计算 Precision/Recall/F1 以及 Redundancy Rate、Call Count Penalty 等效率指标。
- **整体效果层**：使用多模型投票（GPT-4o + Claude-3.5 + DeepSeek 等）进行 Task Completion、Answer Relevancy 等主观评估，同时支持路径相似度对比（与人工标注的 Golden Trajectory 比较，使用编辑距离或 LLM 语义相似度）。

**动态 Skill 专项评测**重点关注：Skill Selection Precision/Recall、Loading Timing Score、Loaded vs Used Ratio、Redundancy Rate、Negative Impact Rate，以及 Δ 对比实验（同一任务集分别跑“有动态 Skill”“固定 Skill”“无 Skill”版本，量化成功率、步数、token 消耗的提升或退化）。

## 3. 系统整体架构

平台采用前后端分离的 **配置驱动 Eval Harness** 架构：

- **前端层**（React + Ant Design）：提供完整配置化页面，包括 Agent 接口配置、输入字段动态添加、多段响应提取规则配置、测试用例管理（支持 expected_tools 注入）、指标选择、协作链路配置。
- **后端层**（FastAPI + Celery + Redis）：包含 Agent Client（生命周期管理）、Multi-Segment Collector（收集多段响应）、配置化 Parser（根据页面 JSON 动态提取）、Orchestrator（多 Agent 协作链路）、DeepEval Scorer（对比参考标准）、报告生成模块。
- **存储层**：PostgreSQL 保存测试用例、提取模板、历史报告。
- **执行层**：异步任务队列 + 并发限流 + 缓存机制。

整个架构完全黑盒、非侵入式，所有逻辑都在平台侧，通过配置驱动实现。

## 4. DeepEval 集成细节

平台以 DeepEval 为核心评测引擎。Agent 通过编写 runner 函数（或 EvaluationDataset + agent_runner）接入。自定义指标通过继承 BaseMetric 或使用 GEval 实现。针对多段响应，runner 先收集 segments 列表，再通过配置化提取逻辑生成 LLMTestCase。

## 5. 页面配置化与多段响应处理

页面支持完整配置化，包括输入字段动态列表、多段响应提取规则（JSONPath + Regex + 优先级列表）、提取规则测试面板。多段响应处理流程为：采集 → 配置化提取 → 评测（final_answer 用于正确性、tools_called 用于 Tool 对比、全段用于轨迹分析）→ 报告（提供原始多段展开面板）。

## 6. 多 Agent 协作扩展

当前平台默认支持单一 API 接口，但多 Agent 协作需要平台侧维护协作关系。推荐端到端黑盒 orchestrator（新增协作链路配置模块）或子 Agent 独立 + 链路组合两种方式，均保持非侵入式。

## 7. 非侵入式平台的优势与劣势

**优势**：接入零成本、兼容性极强、隐私安全最高、验证速度快、协作友好、配置化强。  
**劣势**：诊断深度有限、配置复杂度较高、完全不暴露 tool call 时精度降级、协作关系需平台侧维护、生产实时监控较弱。

## 8. 落地路线图与技术栈建议

**MVP（1-2 周）**：接口生命周期 + 多段收集 + 配置化 Parser + 基础 DeepEval。  
**v1（3-6 周）**：参考标准注入、动态 Skill Δ、协作链路、A/B 测试。  
**成熟阶段**：在线监控、Adversarial 测试。  
技术栈：React 前端、FastAPI + Celery 后端、DeepEval 引擎、PostgreSQL 存储。

## 9. 参考平台对比

LangChain/LangSmith 偏向白盒 trace，适合内部集成场景；Confident AI 强调 eval + observability 一体化、DeepEval 引擎、团队协作、低工程负担，是黑盒平台的优秀参考。

## 10. FAQ（完整问题与答案）

### 10.1 整体方案思路与框架改造

**Q1：我现在在写一个包含 ReAct + Skill + Subagent 架构的 Agent 评测方案，目前可能是单一模型评测，但难点是多接口、返回内容解析、工具调用等，该怎么设计方案？**  
**A**：推荐采用“分层 + 自动化 + 多维度评测”混合方案，而不是纯 LLM-as-Judge。把评测拆成接口层、解析层、工具调用层、整体效果层。先用确定性检查（Pydantic Schema、状态机、Tool Sandbox）覆盖 70% 硬指标，再让 LLM 只负责主观合理性。核心是构建 Test Harness（AgentClient + Session Manager + Trace Collector + Scorer），支持完整生命周期（create → chat → stop）。

**Q2：我的 Agent 是黑盒、非 Python、非 LangChain、多接口，该怎么评测？框架要做什么改造？**  
**A**：把评测框架升级为 API-driven Evaluation Harness。核心改造：新增 AgentClient（封装多接口）、Session Lifecycle Manager、Response Parser（jsonpath/regex）、Tool/Skill Executor 沙盒。推荐以 DeepEval 为核心引擎，结合自定义 runner，实现端到端黑盒评测。无需改动被测 Agent。

### 10.2 Skill 评测（静态 + 动态加载）

**Q3：方案中 Skill 评测的内容是什么？路径相似度到底和谁比较？**  
**A**：Skill 评测拆成 4 个子维度：Selection Accuracy、Invocation Correctness、Necessity/Efficiency、Outcome Quality。路径相似度（Trajectory Similarity）主要和 Golden/Expert Trajectory（人工标注的标准路径）比较，也可与 Reference Trajectory 或 Optimal Trajectory 对比。动态加载 Skill 额外关注：加载准确率、时机合理性、Loaded vs Used Ratio、Δ 对比（有动态 vs 无动态）。

**Q4：Agent 执行过程中会动态加载 Skill，这种一般要怎么评测？**  
**A**：重点测“Δ”和“选择质量”。主流方式：端到端成功率对比（有 Skill vs 无 Skill）、Trajectory-level 评估（Precision/Recall@K、Loading Timing）、LLM Judge 专项 prompt、A/B 测试。推荐加 Skill Selection Precision、Redundancy Rate、Negative Impact Rate 指标。

**Q5：Tool / Skill 调用正确性 & 效率这部分是定义在被测 Agent 中的，而不是平台上的，该怎么处理？**  
**A**：平台角色是“可配置的比较器”。让用户/Agent 提供者在测试用例中注入 expected_tools（name + params_schema + order）、tool_efficiency_rules（max_calls 等）。平台只负责从响应中提取实际调用，然后对比计算分数。报告中明确标注“参考标准由 Agent 提供者定义”。

### 10.3 DeepEval 集成与 Agent 注册

**Q6：在 DeepEval 上如何注册一个新的接口？**  
**A**：DeepEval 没有“注册接口”的概念，而是通过写 自定义 Metric（继承 BaseMetric）或 GEval 来实现新评估逻辑。写完类或 GEval 后直接丢进 evaluate() 即可使用，无需额外注册步骤。

**Q7：那我要怎么注册我的待评测的 Agent？**  
**A**：不需要在 DeepEval 里注册 Agent，而是写一个 runner 函数（或 EvaluationDataset + agent_runner）。runner 负责调用你的黑盒接口，拿到 actual_output，然后包装成 LLMTestCase。平台只需提供调用逻辑即可。

### 10.4 页面配置化实现

**Q8：这部分代码是用户来写的，我如果期望在我的页面上实现配置化（定义输入字段和输出），可以实现吗？**  
**A**：完全可以。推荐前后端分离：前端 React 表单配置接口、输入字段、输出提取规则、测试用例、指标；后端 FastAPI + DeepEval 执行。支持动态添加输入字段、CSV 上传、提取规则配置（jsonpath/regex）。

**Q9：这里的 Agent 要如何定义输入和输出？我要添加一个接口的映射吗？**  
**A**：不需要改 Agent 本身。在平台后端加一层 规范化层（输出映射），用户在页面配置 actual_output_path（jsonpath）、tools_called_path 等。最终塞给 LLMTestCase 的字段是标准化后的。

**Q10：C 的页面可以返回给我一个原型配置吗？**  
**A**：已提供完整 React + Ant Design 原型代码（接口配置、输入字段列表、提取规则、测试用例、指标选择）。核心 payload JSON 结构也已给出，可直接用于开发。

**Q11：有没有相关已经成熟的页面链接让我参考？**  
**A**：推荐参考 Confident AI（deepeval.com）、Langfuse demo（langfuse.com/docs/demo）、Arize Phoenix、Helicone 等平台的 dashboard 截图和交互。

### 10.5 多段响应处理与提取配置化

**Q12：Agent 返回的响应可能是多段的，平台侧要怎么做？**  
**A**：把响应收集成 segments 列表，然后写 extract_final、extract_tools、extract_thoughts 函数。评测时 final_answer 用于正确性，tools_called 用于 Tool 评测，全段用于轨迹分析。

**Q13：结构化提取函数（多段专属）我想做成配置化的内容，要怎么做？**  
**A**：推荐 JSONPath + Regex + 优先级列表 配置方案。用户在页面配置 multi_segment_extraction JSON（priority、array_path、regex 等），后端用动态解析器执行。还可加实时测试面板（粘贴样例 → 预览提取结果）。

### 10.6 平台形态、架构、优缺点

**Q14：那我的平台具体是一个怎么样的形态？非侵入式的一个评测平台，优缺点在哪儿？**  
**A**：形态是 配置驱动的黑盒端到端 Agent 质量平台（非侵入式）。用户通过网页配置接口、解析规则、测试用例、参考标准，一键跑评测。优势：接入门槛极低、兼容任意语言/框架、隐私安全高、协作友好、验证速度快。劣势：诊断深度中等（无法看到内部 reasoning）、配置复杂时学习曲线陡、完全不暴露 tool call 时只能靠反推。

**Q15：系统的整体架构内容是什么？**  
**A**：核心架构为 Eval Harness（Agent Client + Session Manager + Response Parser + Tool Executor + Scorer），以 DeepEval 为引擎。前端配置页面，后端异步执行 + 报告 dashboard。支持黑盒多接口、多段响应、参考标准注入。

**Q16：为什么我的平台适合多 Agent 协作场景？**  
**A**：因为完全黑盒、非侵入、标准外置、端到端任务级评估。协作关系由平台维护 orchestrator 或链路配置即可，无需子 Agent 改动，责任清晰，最适合异构、多团队场景。

**Q17：为什么适合“快速验证效果”而非“深度 debug 内部逻辑”的团队？**  
**A**：平台牺牲了部分诊断深度，换来了极低的接入门槛和极快的验证速度（配置几分钟就能跑出整体效果）。正好匹配早期 PoC、MVP、效果导向的团队节奏。

**Q18：目前我的平台只能评测单一暴露 API 接口，多 Agent 的协作关系我需要自己维护吗？**  
**A**：是的，需要你（或用户）维护协作关系。推荐先实现端到端黑盒 orchestrator（写调用顺序）或链路配置模板。短期内协作关系由平台侧维护，后续可扩展子 Agent 独立评测 + 链路组合。

### 10.7 参考平台（LangChain / Confident AI）

**Q19：LangChain 评测框架是什么样的？**  
**A**：以 LangSmith 为核心，重点在 tracing + agentevals（轨迹评估）。适合白盒场景，对黑盒支持较弱。

**Q20：Confident AI 平台的核心思想和亮点是什么？**  
**A**：核心思想是“让 LLM 质量工程化、可协作、可规模化”（Pytest for LLMs 的团队版）。亮点：DeepEval 引擎 + eval + observability 一体化 + 红队安全 + 低工程负担。

**Q21：评估 + 可观测性一体化是什么意思？怎么打通的？**  
**A**：意思是把开发阶段的 eval 和生产阶段的 tracing 放在同一个平台、同一套 trace、同一套 metric 里。打通方式：统一 trace（@observe）、online evals（metric 直接跑在生产 trace 上）、trace 自动转 dataset 形成闭环。

---

**文档结束**  
本文档已完整、详细地涵盖了从方案思路到具体实现、架构、优缺点、多 Agent 扩展、DeepEval 细节、页面配置、多段响应处理、FAQ 等所有讨论内容。你可以直接复制保存为 .md 文件，或导入到 Notion / Typora / Word 中使用。如需进一步补充图表、具体代码仓库模板或调整章节顺序，请随时告知。