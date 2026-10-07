---
title: '别再盲目搞滑窗了：自研 AI Agent 上下文裁剪的思考与工业级实践'
description: '从“越聊越贵越聊越慢”的痛点出发，反思暴力滑窗淘汰与大模型摘要的陷阱，解析工业级工具骨架微压缩方案与极简落地实践。'
slug: 'agent-context-trimming-practice'
date: 2026-10-08T00:00:00+08:00
image: 'cover.svg'
categories:
  - AI 工程
tags:
  - agent
  - 大模型
  - 上下文工程
  - 工程实践
draft: false
---

## 一、引言：Agent 越聊越贵、越聊越慢的真相

在构建自主智能体（AI Agent，特别是基于 ReAct 循环的 Text-to-SQL / 自动化运维智能体）时，很多人写出原型后的第一感觉是惊艳，但只要连续多聊几轮，就会撞上一堵坚硬的墙：

1. **响应越来越慢**：从最初的 1~2 秒，逐渐退化到 10 秒以上；
2. **账单平方级上升**：每一次提问，都在为之前所有轮次的数据重复买单（Token 呈 $O(N^2)$ 膨胀）；
3. **最终撞上限额崩溃**：遇到一次返回多行数据的查询，整个 Agent 直接抛出 `ContextWindowExceeded`。

为了解决这个问题，很多开源教程的第一反应通常是：“**搞个滑动窗口，保留最近 3 轮对话，把旧对话删掉**”或者“**调用大模型把历史对话压缩成一段 Summary**”。

但在真实的严肃业务场景中，**这种做法往往是踩坑的开始**。

---

## 二、三大人认知误区与反思

在摸索上下文裁剪的过程中，我们总结了三个最容易让人走弯路的认知误区：

### 误区 1：混淆了「前端渲染显示」与「模型请求载荷（Payload）」
初学者最常犯的顾虑是：“*如果我把旧对话裁剪淘汰了，那用户在屏幕上、聊天窗口里岂不是看不到了？*”

**真相**：
* **前端 UI / 存储层**：用户的每一次提问、模型的每一次回答，应该 **100% 完整保存在数据库中**，人类想翻看随时可见，一条都不能少。
* **模型 API 请求体**：大模型（LLM）本身是无状态的（Stateless）。所谓“裁剪”，**仅仅是后端在组织向大模型发出的 HTTP 请求数组（`messages`）时，临时做的一层轻量投影视图**。

---

### 误区 2：盲目“滑窗淘汰”，导致模型“目标遗忘与瞎猜（Goal Amnesia）”
假设用户在排查账目：
* **第 1 轮**：*“帮我统计 9 月份增值业务的所有订单总额，按客户归类。”*（核心业务大目标）
* **第 2 轮**：*“刚才少算了退款，把状态非 REFUND 的排除掉重新算。”*（局部纠错）
* **第 3 轮**：*“把排名前 5 的客户列出来。”*（继续下钻）
* **第 4 轮**：*“把这 5 个客户里用过包年套餐的标记出来。”*

如果设置了简单粗暴的“保留最近 2 轮”滑窗淘汰：
到了第 4 轮，第 1 轮的**根目标**（*9月份增值业务*）被彻底滑掉了！模型眼前只剩下“*把这 5 个客户里用过包年套餐的标出来*”，模型根本不知道这 5 个客户从哪来的、是什么业务口径，**当场失去全局上下文，开始陷入幻觉和瞎猜**。

---

### 误区 3：迷信 LLM 摘要（Summary），既贵又丢精度
用大模型把旧历史压缩成一段摘要（例如部分框架的 `ConversationSummaryMemory`）：
1. **丢失数值精度（致命痛点）**：在财务、记账、指标分析场景中，大模型做摘要极容易把具体的金额、订单号抽象为“*之前统计了订单*”，丢掉了最关键的计算凭据；
2. **额外的延迟与账单**：每次超载都要额外多跑一次 LLM 往返，用户感知卡顿，成本反而进一步上升。

---

## 三、工业界的破局解法：Tool Result Clearing（工具结果微压缩）

翻阅目前全球顶级 Agent（如 Anthropic 官方的 **Claude Code**、SWE-bench 刷榜常客 **Aider**、OpenAI 代码解释器）的工程实现，业界公认的最佳实践其实非常纯粹：

> **“用户的消息是高语义价值的‘黄金’，工具吐出的大数据是低语义密度的‘生铁’。永远不要轻易丢弃用户的黄金，只把生铁扔掉。”**

### 算一笔账：Token 的大头到底在哪里？

* **用户与模型的聊天文字**：一问一答通常只有 50~100 个 Token。哪怕聊了整整 **30 轮**，纯文字也才 **1500~2000 个 Token**，在现代模型 128k 甚至更长窗口面前根本不值一提！
* **真正的元凶**：是调用工具（如 SQL 查询、文件读取、Git Diff）吐出来的原始数据！一次 `SELECT` 返回 150 行数据，瞬间吃掉 **3000~4000 个 Token**，占了总上下文的 **80%~90%**！

因此，业界的标准刀法是：**根本不需要急着去删用户说过的话，把刀刃 100% 对准“历史工具的返回结果”即可！**

---

## 四、落地设计：极简的“骨架降维”方案

以 Text-to-SQL Agent 为例，整套裁剪机制遵循以下铁律与设计：

### 1. 铁律：真身与投影彻底分离
* **真身（`self.history`）**：内存/数据库中永远保存最真实、最完整的原始对话和完整的 SQL 结果。日志（如可视化调用追踪 HTML）永远记录全量数据，保证 100% 可审计。
* **投影（`build()` 视图）**：仅在调用模型生成那一瞬间，现场生成一份“轻量版副本”，发完即焚。

### 2. 差异化对待：当轮必须完整，历史就地折叠
* **当轮当前步（Active Step）**：刚刚查出来的 150 条订单数据**必须完整保留**！因为模型马上要基于这些数据进行计算和输出回答。
* **往轮历史（Past Turns）**：上一轮已经分析完毕的数据，模型已经拿到了结论，不再需要每一行明细。此时将其转换为结构化骨架：

```json
// 原始历史数据：整整 3500 Tokens 的冗长表格
[{"id": 1, "order_no": "A001", "amount": 12000.0, ...}, ...]

// 裁剪后的骨架视图：仅约 40 Tokens（立省 98%！）
{
  "_notice": "[历史查询明细已折叠以节省上下文]",
  "row_count": 150,
  "columns": ["id", "order_no", "amount", "customer_name"],
  "sample_first_row": {"id": 1, "order_no": "A001", "amount": 12000.0},
  "hint": "如需全量明细请重新执行相应 SQL"
}
```

---

## 五、核心实现代码（纯 Python，零框架依赖）

不需要引入 LangChain 等重型框架，在上下文装配层只需十来行代码：

```python
from typing import Any
import json

class ContextManager:
    def __init__(self, system_prompt: str):
        self.system_prompt = system_prompt
        self.history: list[dict[str, Any]] = []  # 真实历史，永远完整

    def build(self) -> list[dict[str, Any]]:
        """在发往模型的前一秒，动态生成轻量投影视图。"""
        messages = [{"role": "system", "content": self.system_prompt}]

        # 找到最新一条 tool 消息的位置（只有当轮最新步才保留完整明细）
        last_tool_idx = max(
            (i for i, m in enumerate(self.history) if m.get("role") == "tool"),
            default=-1
        )

        for i, msg in enumerate(self.history):
            # 针对历史 tool 结果：无损瘦身为骨架，其余所有消息（包括用户原话）100% 原样保留
            if msg.get("role") == "tool" and i != last_tool_idx:
                messages.append({
                    **msg,
                    "content": self._make_tool_skeleton(msg.get("content"))
                })
            else:
                messages.append(msg)

        return messages

    @staticmethod
    def _make_tool_skeleton(raw_content: str | None) -> str:
        """纯本地规则快速降维，0 毫秒延时，0 额外开销。"""
        if not raw_content:
            return "[空结果]"
        try:
            data = json.loads(raw_content)
            if isinstance(data, list) and len(data) > 0 and isinstance(data[0], dict):
                return json.dumps({
                    "_notice": "[历史查询明细已折叠以节省上下文]",
                    "row_count": len(data),
                    "columns": list(data[0].keys()),
                    "sample_first_row": data[0],
                }, ensure_ascii=False)
        except Exception:
            pass
        # 兜底截断
        return f"[历史输出已折叠，前100字符: {str(raw_content)[:100]}...]"
```

---

## 六、方案收益对比

| 评测维度 | 传统暴力滑窗 / LLM 摘要 | 工业级工具骨架微压缩（本文方案） |
| :--- | :--- | :--- |
| **额外延迟** | +1.5s ~ 3s（需额外调模型） | **0 毫秒（纯 Python 内存解析）** |
| **额外费用** | 每次压缩均需支付 Token 费用 | **0 元（本地纯计算）** |
| **用户体验** | 聊到第 4 轮开始遗忘最初目标，瞎猜 | **用户说过的每一句话全在，记忆连贯** |
| **Token 削减率** | 削减不稳定，易误伤高价值信息 | **靶向爆破，单次大 SQL 数据直接省 90%+** |
| **协议安全性** | 容易拆散 `tool_calls` 与 `tool` 导致 400 | **保留消息结构与 `tool_call_id`，100% 协议合法** |

---

## 七、结语

在构建 AI Agent 时，我们很容易陷入“**用复杂的架构解决假问题**”的误区。

上下文爆炸的根源从来不是用户聊天的三言两语，而是工具倾泻的大量原始数据。识别真正的矛盾重心，用最克制、最轻量的本地代码解决 90% 的问题，才是工程落地最优雅的姿态。
