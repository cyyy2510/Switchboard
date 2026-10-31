## 项目简介

Switchboard 是一个面向复杂客服任务的多 Agent 客服编排运行时。
它不是单个“客服机器人”，也不是简单把几个 prompt 拼在一起，而是一个把意图识别、知识检索、记忆、路由、工具、监控和评测串起来的协同系统。
系统先理解用户问题，再决定由哪个 Agent 主处理、是否需要其他 Agent 辅助、是否应该查知识库、是否应该升级到人工，最后再把结果写回记忆和观测系统。


## 它解决什么问题
真实客服场景并不是简单问答。
用户常常会同时提到：
  - 订单、物流、会员、积分等通用咨询
  - 登录失败、401/500、页面崩溃等技术问题
  - 退款、发票、重复扣款、支付失败等账务问题
  - 转人工、投诉、紧急处理等升级诉求
如果只用一个 Agent，通常会出现三类问题：
1. 分流不准
技术问题被普通客服回答，账单问题被技术 Agent 忽略。
2. 上下文断裂
多轮对话里用户补充订单号、金额、错误码后，系统无法稳定延续。
3. 难以迭代
没有评测、监控和规则热更新，项目只能“看起来能聊”，很难持续优化。
EchoMind 的目标就是把这些能力做成一条完整链路，而不是只做一个能说话的模型壳子。

## 当前架构

Switchboard 不是“单 Agent + prompt 变体”的做法。
它当前是一个 路由驱动的多 Agent 编排架构，核心形态可以概括成：

意图识别 -> 路由决策 -> 单 Agent / 并行多 Agent 执行 -> 响应合并 -> 记忆回写 -> 监控/评测反馈

更具体一点：
- IntentRecognizer 先做三路融合意图识别
- AgentOrchestrator 基于意图、实体、关键词、运行状态做路由
- GeneralAgent / TechnicalAgent / BillingAgent / EscalationAgent 不是同一种 prompt 的简单拷贝，而是有各自角色契约、工具白名单、输入输出边界的运行时角色
- 当请求同时覆盖多个业务域时，编排器会并行派发多个 Agent，再由 ResponseComposer 合并结果
- 运行过程会把 Agent 成功率、延迟、工具质量回写到路由评分里，形成闭环
这意味着，EchoMind 的多 Agent 设计重点不是“Agent 数量”，而是路由、协作、降级和治理。

## 核心技术亮点

**1. 细粒度意图识别**

Switchboard 不只识别“咨询、投诉、技术、账单”这种粗粒度意图，还支持更贴近业务的细粒度分类。
| 细粒度意图        | 归一化意图组 | 示例               |
| ----------------- | ------------ | ------------------ |
| logistics         | query        | 快递什么时候到     |
| refund            | billing      | 退款多久到账       |
| invoice           | billing      | 帮我开发票         |
| payment_issue     | billing      | 为什么重复扣款     |
| technical_login   | technical    | 登录一直报 401     |
| technical_crash   | technical    | 应用一直崩溃       |
| human_handoff     | escalation   | 我要找人工客服     |

意图识别使用三路融合：
- LLM：负责语义理解和上下文判断
- Embedding / 本地哈希向量：负责模板相似度匹配
- Pattern：负责关键词兜底
最终输出的不只是 intent，还包括：
- intent_group
- intent_confidence
- intent_source_scores
- urgency
- entities
这让意图识别不只是分类器，而是后续路由、澄清和升级的结构化输入层。


**2. 结构化多 Agent 路由**


Switchboard 的多 Agent 路由不是简单“命中两个关键词就并行”。
当前实现是一个路由驱动的多 Agent 编排架构：
1. 先通过意图识别得到业务方向
2. 再按意图、关键词和实体打分
3. 选出主 Agent
4. 对足够强的其他领域选择辅助 Agent
5. 必要时并行执行并合并结果
系统内部的打分逻辑并不只是“谁像就选谁”，还会考虑：
- 意图类别
- 关键词命中
- 结构化实体
- 当前 Agent 是否可用
- 在线表现和监控降权


**3. RAG 知识库增强**


Switchboard 使用 ChromaDB 构建知识库，用于存放退款政策、配送说明、技术排障、会员规则等文档。
检索链路包括：
  原始问题
  -> 查询改写
  -> 多子查询并行召回
  -> 合并去重
  -> LLM 重排
  -> Top-K 注入 Agent 上下文
但不是所有请求都会触发 RAG。
系统会先识别意图，只有业务类问题才检索知识库。问候、反馈、转人工、未知意图不会触发 RAG，避免无效检索和上下文干扰。


**4. Redis + ChromaDB 记忆体系**


Switchboard 把记忆拆成三层：
| **记忆类型** | **存储** | **作用** |
| ---- | ---- | ---- |
| 工作记忆 | Redis | 当前会话最近消息 |
| 情景记忆 | ChromaDB `episodic` | 历史对话摘要，支持语义检索 |
| 用户画像 | ChromaDB `user_profile` | 用户偏好和关键实体 |


Redis 读写使用异步客户端，ChromaDB 的同步操作放入线程池，减少主请求链路阻塞。
当前会话消息过多时，系统会压缩旧消息，生成摘要，保留最近几轮对话，避免上下文无限膨胀。


**5. 动态 Skills 注入**


知识库解决的是“业务事实是什么”，Skills 解决的是“客服应该怎么处理”。
例如：
- 技术支持需要先收集错误码、版本、操作步骤
- 账单退款不能承诺立即到账
- 通用客服需要先澄清用户诉求
- 涉及敏感信息时需要提醒用户不要公开密码或验证码
Switchboard 支持从 skills/ 目录加载 Markdown / JSON / TXT 规则文件，并根据 Agent 类型和关键词动态注入到 system prompt。
修改规则后可以通过接口热加载，不需要重启服务。


**6. MCP 工具可靠性治理**
Switchboard 把知识库检索封装成工具，并加入完整的可靠性机制：
- 参数校验
- TTL 缓存
- 超时控制
- 熔断器
- fallback 降级
- 查询改写
- LLM 重排
- 工具成功率和延迟统计
这让工具调用不只是“能调”，还具备可治理、可观测和可降级能力。


**7. Monitor 在线观测和路由降权**


Monitor 会定期采集：
- Agent 成功率
- Agent 平均延迟
- 工具成功率
- 工具平均延迟
- 连续失败次数
- 熔断状态
如果某个 Agent 表现变差，Monitor 会写回 monitor_penalty，影响后续 routing_score。
也就是说，监控不只是展示指标，还会影响后续路由选择。


**8. LLM-as-Judge 端到端评测**
Switchboard 内置 /eval/run 评测入口。
评测内容包括：
- 意图识别 Accuracy
- Macro-F1
- 端到端 Agent 回复质量
- LLM-as-Judge 四维评分
- 回归检测
- 优化建议
LLM-as-Judge 会从四个维度评价回复：
- 相关性
- 准确性
- 完整性
- 有用性
这让项目不只是“能回答”，而是能持续评估回答质量。



## 截图

1. 前端界面展示

![img](
https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20174209.png?Expires=1793440618&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=ZBXo4Qfiq6dRtkmGd4B1AXVpNxw%3D)


![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20174357.png?Expires=1793440660&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=6nZf2mzFodjzA7i4%2B%2ByvBxtu0YM%3D)

![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE2.jpeg?Expires=1793440862&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=7P7eZCUkZmmMwqfvwmd7shZVoGQ%3D)


![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.jpeg?Expires=1793440794&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=SpoCw9nvCM8J0Ks61MWYU45XYpU%3D)


2.项目架构图

![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20173549.png?Expires=1793445383&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=evBBfHoD5UHRn8p6Bv2GFo%2Bapq0%3D)


![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20173622.png?Expires=1793445398&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=T5Z%2B6sV6v0j3a9BmuWgIt23DeC0%3D)


![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20173636.png?Expires=1793445457&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=kS587SspbJmhfI%2FiFIPUoAmq4ws%3D)


![img](https://cyyy25100.oss-cn-beijing.aliyuncs.com/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202026-10-31%20173646.png?Expires=1793445476&OSSAccessKeyId=TMP.3KuALQLpALh9TGgnCH3NdkDar5QmJkYQCuJiPqUtfsZpVHVqgYGo4FTjMkM5UEMv43YGdwkPHn8ZKPe5qNajSkPExoDBDx&Signature=TWz6XFxCMF5S%2BvlIy3l7OtfwvBk%3D)

