<div align="center">

# 🌍 多智能体隔离：基于本地 Dify 的叙事引擎 MVP

[![GitHub stars](https://img.shields.io/github/stars/H2COCH2/dify-multi-agent-isolation?style=social)](https://github.com/H2COCH2/dify-multi-agent-isolation)
[![GitHub license](https://img.shields.io/github/license/H2COCH2/dify-multi-agent-isolation)](https://github.com/H2COCH2/dify-multi-agent-isolation/blob/main/LICENSE)
[![Dify](https://img.shields.io/badge/Platform-Dify-blue)](https://dify.ai)
[![Python](https://img.shields.io/badge/Language-Python-3.10+-green)](https://www.python.org)

> **一个基于本地部署 Dify 的多 Agent 协作叙事引擎可行性验证项目。**
> 核心目标是解决单 Agent 架构下多角色场景的“信息隔离失效”（即 NPC 出现“上帝视角”）问题。

</div>

---

## 📖 项目背景

在体验了市面上对标 SillyTavern（酒馆）的 AI 文字角色扮演产品后，我发现单 Agent 架构驾驭宏大叙事时存在一个致命问题：**NPC 会拥有上帝视角**。

一个 LLM 既要扮演 A 角色，又要扮演 B 角色，它必然会读取到所有上下文，导致 B 角色知道了 A 角色的内心戏，或者 NPC 知道了玩家尚未公开的隐秘行动。为了验证多角色信息隔离的可行性，我选择在本地 Dify 上快速搭建了一个 MVP。它虽然受限于 Dify 的能力，只能像回合制游戏一样按顺序推进，但**完美验证了动态上下文路由、代码层标签过滤和结构化输出容错三项核心机制**。

## 📌 项目演进：从单 Agent 到多 Agent

本项目并非一蹴而就，而是经历了明确的架构演进：

*   **第一阶段：单 Agent 提示词引擎（探索期）**
    最初，我尝试用极度复杂的单 Agent 提示词来实现多角色叙事。
    *   **V1 版本**（重规则）：定义了严格的优先级仲裁、双段自检、知悉库与数值模型（[查看 `prompt-v1-strict-rules.md`](docs/prompts/prompt-v1-strict-rules.md)）。
    *   **V2 版本**（重思维链）：引入“底噪清除令”与强制第一人称视角，大幅提升沉浸感（[查看 `prompt-v2-cot-injection.md`](docs/prompts/prompt-v2-cot-injection.md)）。
    *   **瓶颈**：随着规则复杂度上升，单 Agent 出现规则冲突、遗忘，且“上帝视角”无法从根本上在代码层被隔离。
*   **第二阶段：多 Agent 架构（重构期）**
    为彻底解决上述问题，将架构重构为多 Agent 协同（Dify 工作流）。把原本由单 Agent 承担的全部职责，拆分为 6 类独立 Agent 节点，并引入 **Python 代码层**进行标签过滤与动态上下文路由，从数据源头切断越权信息的获取。

> 注：单 Agent 的详细设计文档已被完整保留在仓库中，作为这段技术探索的真实记录。

## 🧠 核心架构（6 类 Agent 分工）

系统由 6 类 LLM/节点组成，职责单一，通过 JSON 与状态变量解耦，尽量做到“高内聚、低耦合”：

| Agent 节点 | 核心职责 |
| :--- | :--- |
| 🎬 **环境 Agent** | 维护客观世界的舞台布景，推演时间流逝，剥离玩家内心戏，只保留外部可见动作。 |
| 🎥 **导演 Agent** | 判定玩家所处区域状态，决定 NPC 的反应顺序，精准派发玩家的心理暗示。 |
| 🧑‍🤝‍🧑 **角色 Agent ×2** | 独立的 NPC 扮演者。拥有各自的内心戏、外部表现、记忆、好感度、占有欲和“意欲”。 |
| 📝 **叙事 Agent** | 将碎片化的环境、玩家动作和 NPC 反应无缝融合成沉浸式的小说正文。 |
| 📜 **历史 Agent** | 压缩世界编年史，维护长期记忆，防止上下文窗口爆炸。 |
| 🕵️ **支线/异步 Agent** | 推演主视野之外的 NPC 自主行动与支线剧情，丰富世界观运转的真实感。 |

## 🔐 核心亮点一：信息权限隔离机制

这是本项目最有价值的工程实践，也是可迁移至企业知识库 RAG 权限控制、多角色智能客服的核心逻辑。

*   **状态机路由**：根据玩家输入和 NPC 意欲，动态判定玩家当前区域状态（独处 / 与一人 / 与多人 / 独处他人同行）。
*   **行为标签过滤**：强制所有 NPC 输出附带行为标签，通过 Python 代码节点进行标签切分与过滤。
    *   `【全局可见】`：所有人都能看到、听到的动作或对话。
    *   `【玩家不知晓】`：只有玩家看不到（但其他 NPC 能看到）。
    *   `【其他角色不知晓】`：只有另外一个 NPC 看不到（但玩家能看到）。
    *   `【个人隐秘】`：除了自己，玩家和其他 NPC 都不知晓。

## ⚙️ 核心亮点二：动态上下文路由与分发

代码层实现 `NPC上下文分发器`，完全通过状态机逻辑，动态生成每个角色的专属上下文，而不是把全局信息一股脑塞给大模型：

*   **环境打包与隔离**：如果玩家处于独处，NPC 拿不到玩家当前环境，只能拿到地点库进行“自主行动”；如果玩家与 NPC1 在一起，则 NPC1 拿到“与玩家一起”的状态和当前环境，而 NPC2 依然拿到“自主行动”的状态。
*   **历史动作定向分发**：解析上一轮 `last_events` 提取行动者名字，仅将玩家亲眼所见或同行 NPC 的动作分发给对应角色，避免 NPC 产生“上帝视角”。
*   **专属上下文组装**：根据角色是否在场，动态拼接不同的数据包（当前环境、已知地点库、同行 NPC 动作、个人存档数据）。

> 💡 代码片段：
> ```python
> # 玩家与一人状态
> elif "玩家与一人" in player_area_status:
>     match = re.search(r'玩家与一人[（\(](.*?)[）\)]', player_area_status)
>     if match:
>         target_name = match.group(1).strip()
>         if target_name == focus_npc:
>             env_for_npc1 = current_env
>             status_for_npc1 = "与玩家一起"
>         elif target_name == focus_npc2:
>             env_for_npc2 = current_env
>             status_for_npc2 = "与玩家一起"
> # 最终返回各自独立的上下文
> return {
>     "npc1_context": build_context(...),
>     "npc2_context": build_context(...),
>     "npc1_status": status_for_npc1,
>     "npc2_status": status_for_npc2
> }

🛠 技术栈与工程实现

· 工作流编排：Dify（本地部署）

· 底层模型：DeepSeek API

· 后端与接口：Python、FastAPI、Flask

· 前端与交互：Streamlit（最小可运行前端）

· 数据治理：所有 LLM 强制 JSON 输出，Python 代码节点进行清洗、解析与异常兜底，单节点解析失败不影响整体工作流。

⚠️ 当前局限与反思（MVP 的边界）

本项目属于可行性验证 MVP，并非生产环境系统，存在明确的局限性：

1. 缺乏真正的状态管理（无 State）：Dify 无法实现真正的同步协作，目前只能做成“回合制游戏”。NPC 与玩家无法在同一时间线上真正并行行动。
2. 行动顺序固化与调度开销：Dify 本身不支持让一个 LLM 持续扮演同一 NPC（否则先行动的永远是那个 NPC）。因此我设计了导演 Agent 进行焦点分析来动态决定谁先手，但这增加了额外的调度开销与延迟。
3. 记忆连贯性与上下文瓶颈：受限于 Dify 的能力，NPC 记忆只能依赖大模型在动态数据库中进行自主压缩与存储，缺乏专业的短期/长期记忆调度机制，长上下文推演中易丢失细节。
4. API Key 锁定：Dify 的工作流 Key 是固定的，无法让用户使用自己的 API Key（导致用户无法自备算力降低服务器成本）。
5. NPC 数量限制：当前仅支持 2 个 NPC 用于验证多角色并行逻辑，增加更多 NPC 会由于缺少 State 导致调度复杂度指数级上升。

🚀 后续重构计划（LangGraph）

针对 Dify 的局限性，下一步的核心计划是全面转向 LangGraph 重构：

☐ 引入 StateGraph 和 Checkpointer，实现多 Agent 的真正的状态同步与持久化

☐ 利用 Conditional Edges 动态路由 NPC 行动，取消硬编码的回合制限制

☐ 引入 ChromaDB / 向量数据库，把当前基于代码的标签过滤升级为基于元数据检索的企业级 RAG 权限控制

📦 快速开始

1. 本地部署 Dify（参考官方文档）
2. 在 Dify 中创建应用，选择“导入 DSL 文件”
3. 上传 flows/world-engine.yml （本文件已脱敏，不含任何 API Key）
4. 在 Dify 模型设置中配置你自己的 DeepSeek API Key
5. 运行并体验多 Agent 协作与信息隔离机制

📂 目录结构参考

```text
dify-multi-agent-isolation/
├── flows/
│   └── world-engine.yml               # Dify 多Agent工作流
├── docs/
│   ├── prompts/                       # 单Agent演进过程文档
│   │   ├── prompt-v1-strict-rules.md  # V1: 重规则版
│   │   └── prompt-v2-cot-injection.md # V2: 重思维链版
│   └── screenshots/                   # 项目截图 (持续更新)
├── README.md
└── LICENSE
```

👤 作者

· GitHub：@H2COCH2

· 简历定位：2027届计算机科学与技术专业，求职方向 AI 应用开发

· 声明：本项目为个人独立架构与开发，用于技术可行性验证与学习交流。

---

<div align="center">

如果这个项目对你有启发，欢迎点个 ⭐ Star 支持一下！

</div>
```
