

这里整理了 GitHub 上高星、高口碑的 Agent 教程开源项目，按**零基础入门、原理拆解、主流框架、多智能体、实战案例**分类，覆盖从新手入门到进阶落地的全学习路径，所有项目均持续维护、配套完整代码与文档。

### 一、零基础保姆级入门教程（中文友好，从 0 到 1）

#### 1. Hello-Agents（Datawhale 开源）

- 项目地址：[https://github.com/datawhalechina/hello-agents](https://github.com/datawhalechina/hello-agents)
    
- Star：6k+
    
- 核心亮点：国内顶级开源社区出品的全中文系统化 Agent 教程，从智能体核心定义、发展史、ReAct/Plan-and-Solve 等经典范式讲起，带你**不依赖现成框架，用原生 API 从零手写 Agent 核心框架**，彻底吃透底层逻辑；同时覆盖 Coze/Dify 低代码平台、LangGraph 主流框架，还有多智能体、RAG、记忆系统、MCP 协议等进阶内容，配套完整电子书、可运行代码和面试题汇总，兼顾理论与实战，新手友好度拉满。
    
- 适合人群：零基础新手、想系统学习 Agent 全栈知识的开发者、学生、AI 产品经理。
    

#### 2. ai-agents-for-beginners（微软官方）

- 项目地址：[https://github.com/microsoft/ai-agents-for-beginners](https://github.com/microsoft/ai-agents-for-beginners)
    
- Star：20k+
    
- 核心亮点：微软官方出品的全球标杆级 Agent 零基础课程，12 节系统化课程，从基础概念、工具调用、记忆管理，到多智能体协作、RAG 集成、生产环境最佳实践，每节课都有完整代码示例、原理图解和配套作业，内容权威严谨，配套视频讲解，是 Agent 入门的黄金教材。
    
- 适合人群：零基础新手、偏好官方权威教程、想学习国际标准 Agent 开发规范的开发者。
    

### 二、核心原理拆解：从零手搓 Agent（无框架黑盒，吃透本质）

#### 1. learn-claude-code

- 项目地址：[https://github.com/shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
    
- Star：54.7k+
    
- 核心亮点：爆火的极简 Agent 教程，用几百行核心代码剥离所有重型框架的封装黑盒，直击 Agent 的核心本质 ——**一个核心循环 + 终端执行工具，就是一个完整的智能体**。项目拆分为 12 节渐进式课程，每节只新增一个机制（工具调用、上下文压缩、多智能体协作等），全程不改动核心循环代码，全中文文档，完全不会被复杂概念劝退。
    
- 适合人群：被 LangChain 等框架搞晕、想彻底吃透 Agent 底层运转逻辑的开发者。
    

#### 2. ai-agents-from-scratch

- 项目地址：[https://github.com/pguso/ai-agents-from-scratch](https://github.com/pguso/ai-agents-from-scratch)
    
- 核心亮点：纯原生代码从零构建 Agent，无任何第三方框架黑盒依赖。10 节完整课程，从本地 LLM 加载、基础 Prompt 交互，到 Function Calling、记忆系统、ReAct 范式完整实现，甚至带你重写 LangChain/LangGraph 的核心逻辑，同时支持 Python 和 JS 双版本，完美支持本地大模型，完全离线可运行。
    
- 适合人群：想深入理解框架底层、偏好原生代码实现、需要本地离线运行 Agent 的开发者。
    

#### 3. BabyAgent（Go 语言版）

- 项目地址：[https://github.com/baby-llm/baby-agent](https://github.com/baby-llm/baby-agent)
    
- 核心亮点：专为后端工程师设计的 Go 语言 Agent 教学项目，跳过复杂的数学推导，纯从工程实现视角拆解 Agent 核心原理，用可运行的极简代码讲透 Tool Calling、Agent Loop、MCP、Memory、RAG 等核心机制，注释详尽，完全贴合后端开发者的技术思维。
    
- 适合人群：有 Go 基础、无 LLM 背景的后端工程师，想从工程视角理解 Agent 实现。
    

### 三、主流 Agent 框架专属教程 & 实战项目

#### 1. LangGraph 全体系教程合集

- 官方示例仓库：[https://github.com/langchain-ai/langgraph-example](https://github.com/langchain-ai/langgraph-example)
    
- 完整进阶指南：[https://github.com/mkassaf/langgraph-complete-guide](https://github.com/mkassaf/langgraph-complete-guide)
    
- 新手一键脚手架：[https://github.com/shanevcantwell/langgraph-agentic-scaffold](https://github.com/shanevcantwell/langgraph-agentic-scaffold)
    
- 核心亮点：LangGraph 是当前构建复杂有状态 Agent 的绝对主流框架。官方仓库提供了从基础到进阶的全量可运行示例；进阶指南覆盖路由 Agent、工具调用、状态管理、多智能体协作、记忆持久化等全场景，带完整代码和详细注释；脚手架项目提供了一键启动的 Agent 模板，5 分钟就能跑通第一个 LangGraph 项目，对新手极度友好。
    
- 适合人群：想学习 LangGraph 框架、构建复杂工业级 Agent 的开发者。
    

#### 2. Anthropic Courses（Claude 官方）

- 项目地址：[https://github.com/anthropics/courses](https://github.com/anthropics/courses)
    
- 核心亮点：Claude 官方出品的 Agent 权威教程，聚焦于 Claude 的工具调用、Skills 系统、Agent 工作流构建，从 Prompt 工程到完整 Agent 系统实现，配套完整代码和生产级最佳实践，是学习 Anthropic 生态 Agent 开发的唯一官方资料。
    
- 适合人群：使用 Claude 模型、想学习官方 Agent 开发规范的开发者。
    

### 四、多智能体（Multi-Agent）专项教程

#### 1. Hugging Multi-Agent（Datawhale）

- 项目地址：[https://github.com/datawhalechina/Hugging-Multi-Agent](https://github.com/datawhalechina/Hugging-Multi-Agent)
    
- 核心亮点：国内开源社区出品的多智能体全中文教程，从多智能体核心理论、通信机制、角色设计，到 AutoGen、CrewAI、MetaGPT 等主流多智能体框架实战，覆盖科研、金融、教育、研发等多个场景的完整项目案例，配套可运行代码和详细文档，是国内多智能体学习的标杆项目。
    
- 适合人群：想学习多智能体协作、构建多角色 Agent 系统的开发者。
    

#### 2. 500-AI-Agents-Projects

- 项目地址：[https://github.com/ashishpatel26/500-AI-Agents-Projects](https://github.com/ashishpatel26/500-AI-Agents-Projects)
    
- Star：18k+
    
- 核心亮点：超全的 AI Agent 落地案例合集，收录 500 + 个 Agent 项目，按医疗、金融、教育、DevOps、营销等垂直领域分类，覆盖 CrewAI、AutoGen、LangGraph 等所有主流框架的实际应用，每个案例都有开源代码和实现思路，既是多智能体学习手册，也是行业落地参考指南。
    
- 适合人群：想找垂直场景 Agent 落地参考、学习不同框架实战用法的开发者、产品经理。
    

### 五、实战案例 & 全场景应用合集

#### 1. awesome-llm-apps

- 项目地址：[https://github.com/Shubhamsaboo/awesome-llm-apps](https://github.com/Shubhamsaboo/awesome-llm-apps)
    
- Star：68k+
    
- 核心亮点：全球顶级的 LLM 应用 & Agent 开源合集，包含 50 + 个完整可运行的 Agent 项目，从旅行 Agent、客服 Agent、科研文献 Agent，到 RAG+Agent、多模态 Agent、自动化办公 Agent，每个项目都有完整代码、依赖配置和运行教程，开箱即用，既能用于学习，也能直接二次开发。
    
- 适合人群：想快速上手 Agent 实战、找可复用项目模板的开发者。
    

#### 2. GenericAgent

- 项目地址：[https://github.com/lsdefine/GenericAgent](https://github.com/lsdefine/GenericAgent)
    
- Star：4.4k+
    
- 核心亮点：3000 行代码实现的可自我进化的 Agent，核心设计理念是「最小核心 + 自我进化」，不预设固定技能，通过用户使用不断生长专属技能树，整个仓库从 git init 到每一条 commit message 全由 Agent 自己完成，代码极简，设计思路极具启发性，非常适合学习 Agent 的自我迭代和长期记忆系统设计。
    
- 适合人群：想研究 Agent 自我进化、长期记忆系统的进阶开发者。
    

---

### 重要提示

1. 所有项目均需遵守对应开源协议，商用请提前查看仓库 LICENSE 文件；
    
2. 涉及大模型 API 调用的项目，需要自行准备对应模型的 API Key；
    
3. 本地运行项目，建议使用 Python 虚拟环境（venv/conda），避免依赖版本冲突。
    

需要我根据你的技术栈（Python/Go 等）和学习目标（入门 / 多智能体 / 生产落地），帮你筛选最合适的项目，并给出一周学习路线吗？