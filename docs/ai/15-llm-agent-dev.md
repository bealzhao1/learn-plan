# 大模型 MCP Server / Skills / 工作流 / 智能体开发

## 15.1 MCP (Model Context Protocol) Server

**概念**：MCP 是大模型与外部工具/数据源之间的标准化通信协议。

```
┌─────────────┐       MCP Protocol       ┌─────────────┐
│  AI Client  │ ◄──────────────────────► │  MCP Server │
│ (WorkBuddy) │    JSON-RPC over stdio    │  (工具提供方) │
└─────────────┘       or HTTP/SSE         └──────┬──────┘
                                                  │
                                           ┌──────┴──────┐
                                           │  外部系统    │
                                           │ DB/API/文件  │
                                           └─────────────┘
```

**MCP Server 核心组件**：

| 组件 | 说明 |
|------|------|
| **Tools** | 可调用的函数/能力（如查询数据库、调用API） |
| **Resources** | 可读取的数据源（文件、知识库） |
| **Prompts** | 预定义的提示词模板 |

**开发示例（TypeScript）**：
```typescript
import { McpServer } from "@anthropic-ai/mcp";

const server = new McpServer({
  name: "my-data-server",
  version: "1.0.0",
});

// 注册 Tool
server.tool("query_database", {
  description: "查询业务数据库",
  parameters: {
    sql: { type: "string", description: "SQL 查询语句" },
  },
  handler: async ({ sql }) => {
    const result = await db.query(sql);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  },
});

// 注册 Resource
server.resource("docs://api-reference", {
  description: "API 文档",
  handler: async () => {
    return { content: [{ type: "text", text: apiDocs }] };
  },
});
```

## 15.2 Skills（技能）

**概念**：Skills 是 AI 助手的可复用能力模块，包含指令、工具组合和领域知识。

**结构**：
```
skills/
└── my-skill/
    ├── SKILL.md          # 技能定义（触发条件、指令、流程）
    ├── references/       # 参考文档
    │   └── api-docs.md
    ├── scripts/          # 辅助脚本
    │   └── helper.py
    └── assets/           # 静态资源
```

**SKILL.md 核心结构**：
```markdown
---
title: "数据分析助手"
summary: "根据用户需求进行数据查询和可视化分析"
triggers:
  - "分析数据"
  - "生成报表"
  - "数据可视化"
tools_required:
  - database_query
  - chart_generator
---

# 数据分析助手

## 触发场景
当用户需要数据分析、报表生成、趋势洞察时触发。

## 执行流程
1. 理解用户分析需求
2. 构建查询语句
3. 执行查询获取数据
4. 选择合适的可视化方式
5. 生成图表和分析报告

## 注意事项
- 大数据量时先 COUNT 再决定是否全量拉取
- 敏感数据需脱敏处理
```

## 15.3 工作流 (Workflow)

**概念**：将多个步骤、工具调用、条件判断编排成可复用的自动化流程。

**工作流设计模式**：

```
1. 顺序流 (Sequential)
   Step A → Step B → Step C → Output

2. 并行流 (Parallel)
   ┌→ Step A ─┐
   │          │
   Start ──┼→ Step B ─┼→ Merge → Output
   │          │
   └→ Step C ─┘

3. 条件流 (Conditional)
                    ┌→ Path A → Output A
   Input → Decide ─┤
                    └→ Path B → Output B

4. 循环流 (Loop)
   Input → Process → Check ──→ Output
               ↑        │
               └────────┘ (不满足则重试)

5. 人机协作流 (Human-in-the-loop)
   AI Process → Human Review → AI Continue → Output
```

**工作流编排要素**：

| 要素 | 说明 |
|------|------|
| 触发器 | 定时、事件、手动 |
| 步骤 | 工具调用、LLM 推理、代码执行 |
| 上下文传递 | 步骤间数据流转 |
| 错误处理 | 重试、降级、通知 |
| 状态管理 | 持久化中间状态 |

## 15.4 智能体 (Agent) 开发

**核心架构**：

```
┌────────────────────────────────────────────────┐
│                   Agent Loop                    │
│                                                │
│   ┌──────────┐    ┌──────────┐    ┌────────┐  │
│   │  Observe │───►│  Think   │───►│  Act   │  │
│   │  (感知)   │    │  (推理)   │    │ (行动) │  │
│   └──────────┘    └──────────┘    └────────┘  │
│        ↑                                │      │
│        └────────────────────────────────┘      │
│                  (迭代循环)                      │
│                                                │
│   ┌─────────────────────────────────────────┐  │
│   │              Memory (记忆层)              │  │
│   │  短期记忆 | 长期记忆 | 工作记忆           │  │
│   └─────────────────────────────────────────┘  │
│                                                │
│   ┌─────────────────────────────────────────┐  │
│   │              Tools (工具层)               │  │
│   │  MCP Server | API | 文件系统 | 搜索      │  │
│   └─────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

**智能体关键能力**：

| 能力 | 说明 | 实现方式 |
|------|------|---------|
| 规划 | 拆解复杂任务为子步骤 | CoT / ToT / ReAct |
| 工具使用 | 调用外部工具获取信息或执行操作 | Function Calling / MCP |
| 记忆 | 维护上下文、学习经验 | 向量数据库 / 文件系统 |
| 反思 | 评估结果，调整策略 | Self-Critique / Reflexion |
| 多智能体协作 | 多个 Agent 分工合作 | 消息传递 / 共享任务列表 |

**开发框架对比**：

| 框架 | 特点 | 适用场景 |
|------|------|---------|
| LangChain | 生态最丰富，组件多 | 快速原型、RAG |
| LangGraph | 图结构编排，状态机 | 复杂工作流 |
| CrewAI | 多智能体角色扮演 | 团队协作任务 |
| AutoGen | 微软出品，对话驱动 | 多轮对话代理 |
| Dify | 低代码平台 | 非开发者快速搭建 |
| Coze | 字节出品，可视化编排 | 快速上线 Bot |

**ReAct 模式实现**：
```python
class Agent:
    def __init__(self, llm, tools, memory):
        self.llm = llm
        self.tools = tools
        self.memory = memory
    
    def run(self, task: str) -> str:
        """Agent 主循环"""
        self.memory.add("user", task)
        
        for step in range(max_steps):
            # Think: LLM 推理下一步行动
            response = self.llm.chat(
                messages=self.memory.get_messages(),
                tools=self.tools.schema(),
            )
            
            # Act: 执行工具调用
            if response.tool_calls:
                for call in response.tool_calls:
                    result = self.tools.execute(call.name, call.args)
                    self.memory.add("tool", result)
            
            # Observe: 检查是否完成
            if response.is_final_answer:
                return response.content
        
        return "达到最大步数限制"
```

## 15.5 Prompt Engineering 在智能体中的应用

```markdown
# System Prompt 设计原则

1. **角色定义** — 明确 Agent 的身份和职责边界
2. **能力声明** — 列出可用工具和使用场景
3. **行为约束** — 安全边界、操作限制
4. **输出格式** — 结构化输出要求
5. **示例引导** — Few-shot 示范正确行为

# 关键技巧
- Chain of Thought (CoT): 让模型展示推理过程
- Few-shot: 提供输入-输出示例
- Self-consistency: 多次采样取多数结果
- Tree of Thought: 分支探索多种方案
```

## 15.6 智能体开发最佳实践

1. **最小权限原则** — 只给 Agent 必要的工具和权限
2. **人机协作** — 关键决策设置人工确认节点
3. **可观测性** — 完整的日志、trace、指标
4. **优雅降级** — 工具不可用时有兜底方案
5. **测试驱动** — 为 Agent 行为编写测试用例
6. **成本控制** — 监控 token 消耗，设置预算上限
7. **迭代优化** — 根据实际运行数据持续优化 Prompt 和流程
