---
title: "[nanobot] 源码分析 - 核心功能与架构设计"
date: 2026-03-09
tags: [源码分析, 开源项目, 技术深度, GitHub, AI, Agent]
subtitle: "深入理解 nanobot 的设计思路与实现细节"
---

## 1. 这个项目的核心功能是什么？

nanobot 是一个**超轻量级的个人 AI 助手框架**，灵感来源于 OpenClaw。它用仅约 4,000 行核心代码实现了完整的 Agent 功能，代码量比 OpenClaw 少 99%。

核心价值：
- **极简设计**：代码清晰易读，易于研究和修改
- **多平台支持**：支持 Telegram、Discord、WhatsApp、飞书、QQ、Slack、钉钉、Email 等 10+ 聊天平台
- **多模型接入**：支持 OpenRouter、Anthropic、OpenAI、DeepSeek、VolcEngine 等 20+ LLM 提供商
- **完整功能**：Agent 循环、工具调用、记忆系统、技能系统、子代理、MCP 支持、定时任务等

## 2. 项目的主要功能和目标是什么？

### 主要功能

**Agent 核心**：
- 完整的 LLM ↔ 工具执行循环
- 内置工具：文件读写、Shell 执行、Web 搜索、消息发送、子代理生成
- MCP（Model Context Protocol）支持，可连接外部工具服务器
- 思考模式（Thinking Mode）支持

**多渠道接入**：
- Telegram、Discord、WhatsApp、Feishu（飞书）
- QQ、DingTalk（钉钉）、Slack、Email
- Matrix（Element）、Mochat（Claw IM）

**记忆与会话**：
- 持久化会话历史
- 内存 consolidation（合并归档）
- 周期性心跳任务（Heartbeat）

**开发友好**：
- Docker 支持
- systemd 服务配置
- 多实例运行（不同配置隔离）

### 项目目标

- **研究就绪**：代码简洁清晰，适合学术研究和教学
- **轻量高效**：最小化资源占用，快速启动
- **易于使用**：一键部署，开箱即用
- **可扩展**：模块化设计，易于添加新的 provider 和 channel

## 3. 代码的整体架构是怎样的？

### 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        用户交互层                              │
│  Telegram  │  Discord  │  WhatsApp  │  Feishu  │  ...     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                      消息总线 (Bus)                          │
│               MessageBus（队列 + 事件）                      │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                    Agent Loop（核心引擎）                     │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  1. 构建上下文（历史 + 记忆 + 技能）                    │ │
│  │  2. 调用 LLM Provider                                  │ │
│  │  3. 执行工具调用（Tool Registry）                       │ │
│  │  4. 发送响应回 Bus                                      │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────┬──────────────────┬──────────────────┬────────────┘
          │                  │                  │
┌─────────▼────────┐ ┌──────▼────────┐ ┌─────▼──────────┐
│  Session Manager │ │  Memory Store  │ │  Skill Loader  │
│  会话管理        │ │  记忆存储      │ │  技能加载       │
└──────────────────┘ └───────────────┘ └────────────────┘
          │                  │                  │
┌─────────▼────────┐ ┌──────▼────────┐ ┌─────▼──────────┐
│    Providers     │ │    Tools       │ │    Cron        │
│  LLM 提供商      │ │  工具注册表    │ │  定时任务      │
└──────────────────┘ └───────────────┘ └────────────────┘
```

### 核心设计模式

1. **消息驱动**：所有交互通过 MessageBus 解耦
2. **注册机制**：Provider、Tool、Channel 都使用 Registry 模式
3. **模块化**：各组件独立，可单独替换
4. **异步优先**：大量使用 asyncio，高并发处理

## 4. 核心模块有哪些，各自职责是什么？

### 1. `agent/loop.py` - AgentLoop（核心引擎）

**职责**：整个系统的核心处理循环

**核心功能**：
- 从 Bus 接收消息
- 构建上下文（历史 + 记忆 + 技能）
- 调用 LLM
- 执行工具调用
- 发送响应回 Bus

**关键类**：
```python
class AgentLoop:
    # 初始化：注册工具、设置会话管理、子代理管理
    # _run_agent_loop(): 执行迭代循环（LLM ↔ 工具）
    # _process_message(): 处理单条消息
    # _consolidate_memory(): 记忆合并归档
```

### 2. `bus/` - 消息总线

**职责**：消息路由和队列管理

**组件**：
- `events.py`: InboundMessage、OutboundMessage 事件定义
- `queue.py`: MessageBus（异步队列）

### 3. `channels/` - 聊天渠道集成

**职责**：对接各聊天平台

**支持的渠道**：
- Telegram、Discord、WhatsApp
- Feishu（飞书）、QQ、DingTalk（钉钉）
- Slack、Email、Matrix
- Mochat（Claw IM）

### 4. `providers/` - LLM 提供商

**职责**：统一的 LLM 调用接口

**关键文件**：
- `base.py`: LLMProvider 基类
- `registry.py`: Provider 注册中心（核心设计！）
- `litellm_provider.py`: LiteLLM 封装（支持 20+ 提供商）
- `azure_openai_provider.py`: Azure OpenAI 特殊处理
- `openai_codex_provider.py`: OpenAI Codex OAuth 支持

**ProviderSpec 设计**：
```python
ProviderSpec(
    name="myprovider",
    keywords=("myprovider", "mymodel"),
    env_key="MYPROVIDER_API_KEY",
    display_name="My Provider",
    litellm_prefix="myprovider",
    skip_prefixes=("myprovider/",),
)
```

添加新 provider 只需 2 步！无需修改 if-elif 链。

### 5. `session/` - 会话管理

**职责**：持久化会话历史

**功能**：
- 会话创建和获取
- 历史消息存储
- 记忆窗口管理

### 6. `agent/tools/` - 工具系统

**职责**：工具注册和执行

**内置工具**：
- `filesystem.py`: ReadFile、WriteFile、EditFile、ListDir
- `shell.py`: Exec（Shell 执行）
- `web.py`: WebSearch、WebFetch
- `message.py`: Message（消息发送）
- `spawn.py`: Spawn（子代理生成）
- `cron.py`: Cron（定时任务管理）
- `mcp.py`: MCP 服务器连接

### 7. `agent/memory.py` - 记忆系统

**职责**：记忆合并和归档

**功能**：
- 自动合并旧会话
- 归档到 memory/ 目录
- 防止上下文过长

### 8. `cron/` - 定时任务

**职责**：Cron 任务管理

**功能**：
- Cron 表达式解析
- 任务调度
- 持久化存储

### 9. `heartbeat/` - 心跳机制

**职责**：周期性唤醒

**功能**：
- 每 30 分钟唤醒一次
- 检查 HEARTBEAT.md
- 执行周期性任务

### 10. `skills/` - 技能系统

**职责**：技能加载和使用

**功能**：
- 从 workspace/skills/ 加载技能
- 自动触发匹配的技能
- 技能元数据解析

## 5. 使用了哪些关键技术栈和框架？

### 核心依赖

| 技术 | 用途 | 版本 |
|------|------|------|
| **Python** | 开发语言 | ≥3.11 |
| **LiteLLM** | 统一 LLM 调用 | ≥1.81.5 |
| **Pydantic** | 数据验证和配置 | ≥2.12.0 |
| **asyncio** | 异步编程 | Python 标准库 |
| **websockets** | WebSocket 支持 | ≥16.0 |
| **httpx** | HTTP 客户端 | ≥0.28.0 |
| **loguru** | 日志 | ≥0.7.3 |
| **Typer** | CLI 框架 | ≥0.20.0 |
| **Rich** | 终端美化 | ≥14.0.0 |
| **MCP** | Model Context Protocol | ≥1.26.0 |

### 渠道特定依赖

- **Telegram**: `python-telegram-bot`
- **Discord**: 自研（通过 HTTP API）
- **WhatsApp**: Node.js bridge（baileys）
- **Feishu**: `lark-oapi`
- **QQ**: `qq-botpy`
- **DingTalk**: `dingtalk-stream`
- **Slack**: `slack-sdk`
- **Matrix**: `matrix-nio`（可选）

### 开发工具

- **pytest**: 测试框架
- **ruff**: 代码 lint
- **hatchling**: 构建系统

## 6. 数据流是如何设计的？

### 完整消息流

```
用户消息
    │
    ▼
[Channel] 接收消息（Telegram/Discord/...）
    │
    ▼
[InboundMessage] 事件对象
    │
    ▼
[MessageBus] 发布到队列
    │
    ▼
[AgentLoop] 消费消息
    │
    ├─► [SessionManager] 获取会话历史
    │
    ├─► [ContextBuilder] 构建提示词
    │      ├─ 系统提示
    │      ├─ 历史消息（memory_window）
    │      ├─ 技能加载
    │      └─ 当前消息
    │
    ├─► [LLMProvider] 调用 LLM
    │      └─ 返回响应（可能包含 tool_calls）
    │
    ├─► 判断：有工具调用？
    │      ├─ 是 ──► [ToolRegistry] 执行工具
    │      │              │
    │      │              └─► 返回工具结果
    │      │              │
    │      │              └─► 回到 LLM 调用（循环，直到 max_iterations）
    │      │
    │      └─ 否 ──► 最终响应
    │
    ├─► [SessionManager] 保存会话
    │
    └─► [OutboundMessage] 响应事件
           │
           ▼
    [MessageBus] 发布响应
           │
           ▼
    [Channel] 发送给用户
```

### 关键点

1. **异步队列**：MessageBus 使用 asyncio.Queue，解耦生产者和消费者
2. **迭代循环**：_run_agent_loop() 支持多轮工具调用（max_iterations=40）
3. **会话持久化**：每条消息后保存到磁盘
4. **记忆窗口**：只保留最近 memory_window（默认 100）条消息
5. **记忆合并**：超过窗口后自动 consolidation，用 LLM 总结旧消息

## 7. 如何处理错误和异常？

### 错误处理策略

| 场景 | 处理方式 |
|------|----------|
| **LLM 返回 error** | 不保存到会话历史（防止污染上下文），返回用户友好提示 |
| **工具调用异常** | 捕获异常，将错误信息作为工具结果返回给 LLM |
| **消息处理异常** | 捕获所有异常，返回 "Sorry, I encountered an error." |
| **MCP 连接失败** | 记录错误，下次消息重试 |
| **任务取消** | 支持 /stop 命令，取消 active tasks 和 subagents |

### 关键代码

```python
# loop.py - 错误响应不保存到历史
if response.finish_reason == "error":
    logger.error("LLM returned error: {}", (clean or "")[:200])
    final_content = clean or "Sorry, I encountered an error calling the AI model."
    break  # 不添加到 messages

# loop.py - 全局异常捕获
except Exception:
    logger.exception("Error processing message for session {}", msg.session_key)
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel, chat_id=msg.chat_id,
        content="Sorry, I encountered an error.",
    ))
```

### 安全措施

- **restrictToWorkspace**: 限制所有工具在 workspace 目录内
- **allowFrom**: 默认空列表拒绝所有，需显式配置或用 ["*"]
- **路径 sanitization**: 文件工具检查路径遍历

## 8. 测试策略和覆盖率如何？

### 测试配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

### 开发依赖

```toml
dev = [
    "pytest>=9.0.0,<10.0.0",
    "pytest-asyncio>=1.3.0,<2.0.0",
    "ruff>=0.1.0",
]
```

### 已知测试文件

```
tests/ 目录存在，但内容未详细分析
```

### 代码质量

- **ruff lint**: line-length=100, target-version=py311
- **类型注解**: 广泛使用 TYPE_CHECKING 和类型提示
- **日志**: loguru 结构化日志

## 9. 部署和运维方案是什么？

### 部署方式

#### 1. 直接运行（开发）

```bash
pip install nanobot-ai
nanobot onboard
nanobot agent          # CLI 模式
nanobot gateway        # 多渠道网关
```

#### 2. Docker

```bash
docker build -t nanobot .
docker run -v ~/.nanobot:/root/.nanobot -p 18790:18790 nanobot gateway
```

#### 3. Docker Compose

```bash
docker compose run --rm nanobot-cli onboard
docker compose up -d nanobot-gateway
```

#### 4. systemd 服务（生产）

```ini
[Unit]
Description=Nanobot Gateway
After=network.target

[Service]
Type=simple
ExecStart=%h/.local/bin/nanobot gateway
Restart=always
RestartSec=10
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=%h

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now nanobot-gateway
loginctl enable-linger $USER  # 保持运行
```

### 配置管理

- **配置文件**: `~/.nanobot/config.json`
- **工作区**: `~/.nanobot/workspace/`
- **多实例**: 通过 `--config` 指定不同配置目录，支持同时运行多个实例

### 日志和监控

- **日志**: loguru 输出到终端
- **systemd 日志**: `journalctl --user -u nanobot-gateway -f`
- **状态命令**: `nanobot status`

## 10. 有哪些值得学习的代码实践？

### 1. Provider 注册机制（registry.py）

**问题**：添加新 provider 需要修改多处代码

**解决方案**：ProviderSpec + Registry 模式

```python
# registry.py
@dataclass(frozen=True)
class ProviderSpec:
    name: str
    keywords: tuple[str, ...]
    env_key: str
    # ... 更多选项

PROVIDERS: tuple[ProviderSpec, ...] = (
    ProviderSpec(name="openrouter", ...),
    ProviderSpec(name="anthropic", ...),
    # ...
)
```

**优点**：添加新 provider 只需 2 步！
1. 在 PROVIDERS 加一个 ProviderSpec
2. 在 config schema 加一个字段
3. 无需修改任何 if-elif 链

### 2. ContextBuilder 模式（context.py）

**问题**：提示词构建逻辑散乱

**解决方案**：专用 ContextBuilder 类

```python
class ContextBuilder:
    def build_messages(self, history, current_message, ...):
        # 统一构建提示词
        pass

    def add_assistant_message(self, ...):
        pass

    def add_tool_result(self, ...):
        pass
```

### 3. 消息总线解耦（bus/）

**问题**：Channel 和 Agent 直接依赖

**解决方案**：MessageBus + 事件对象

```
Channel → InboundMessage → Bus → AgentLoop
                                      ↓
Channel ← OutboundMessage ← Bus ← AgentLoop
```

**优点**：Channel 和 Agent 完全解耦，易于测试和扩展

### 4. 工具注册表模式（agent/tools/registry.py）

**问题**：工具管理混乱

**解决方案**：ToolRegistry + 基类

```python
class ToolRegistry:
    def register(self, tool): ...
    def get_definitions(self): ...  # 返回 OpenAI 格式工具定义
    async def execute(self, name, args): ...
```

**优点**：工具可动态注册，支持 MCP 自动发现

### 5. 异步任务取消（loop.py）

**问题**：长时间运行的任务无法中断

**解决方案**：asyncio.Task 追踪 + /stop 命令

```python
# 追踪 active tasks
self._active_tasks: dict[str, list[asyncio.Task]] = {}

# /stop 命令处理
async def _handle_stop(self, msg):
    tasks = self._active_tasks.pop(msg.session_key, [])
    cancelled = sum(1 for t in tasks if not t.done() and t.cancel())
    # ... 同时取消 subagents
```

### 6. 记忆 consolidation（memory.py）

**问题**：会话历史无限增长，context 溢出

**解决方案**：自动合并 + LLM 总结

```python
# 当 unconsolidated >= memory_window 时触发
if (unconsolidated >= self.memory_window and 
    session.key not in self._consolidating):
    # 异步 consolidation，不阻塞主流程
    _task = asyncio.create_task(_consolidate_and_unlock())
```

### 7. 多实例支持（config 路径推导）

**问题**：无法同时运行多个 bot

**解决方案**：--config + 路径推导

```
--config ~/.nanobot-telegram/config.json
    → 运行时数据: ~/.nanobot-telegram/
    → workspace: 配置中定义或 --workspace 覆盖
```

**优点**：完全隔离，支持同时运行 Telegram + Discord + Feishu

### 8. MCP 懒加载连接（loop.py）

**问题**：启动慢，MCP 连接可能失败

**解决方案**：lazy connect + 重试

```python
async def _connect_mcp(self):
    if self._mcp_connected or self._mcp_connecting or not self._mcp_servers:
        return
    self._mcp_connecting = True
    try:
        # 连接...
        self._mcp_connected = True
    except Exception as e:
        logger.error("Failed to connect MCP (will retry next message): {}", e)
    finally:
        self._mcp_connecting = False
```

**优点**：第一条消息才连接，失败不影响启动，下次重试

## 11. 项目的扩展性和维护性如何？

### 扩展性：优秀 ✅

| 维度 | 评价 | 说明 |
|------|------|------|
| **新 Provider** | ⭐⭐⭐⭐⭐ | 只需 2 步，ProviderSpec 设计极佳 |
| **新 Channel** | ⭐⭐⭐⭐ | Bus 解耦，添加新 channel 只需实现消息收发 |
| **新 Tool** | ⭐⭐⭐⭐⭐ | ToolRegistry 模式，注册即可用 |
| **新 Skill** | ⭐⭐⭐⭐⭐ | 自动从 workspace/skills/ 加载 |
| **MCP** | ⭐⭐⭐⭐⭐ | 原生支持，可连接任意 MCP 服务器 |

### 维护性：优秀 ✅

| 维度 | 评价 | 说明 |
|------|------|------|
| **代码量** | ⭐⭐⭐⭐⭐ | ~4,000 行核心，非常精简 |
| **可读性** | ⭐⭐⭐⭐⭐ | 清晰的模块划分，良好的命名 |
| **类型注解** | ⭐⭐⭐⭐ | 广泛使用 type hints |
| **日志** | ⭐⭐⭐⭐ | loguru 结构化日志 |
| **测试** | ⭐⭐⭐ | 有测试目录，但覆盖率未知 |
| **文档** | ⭐⭐⭐⭐⭐ | README 非常详细，有快速开始和配置示例 |

### 值得改进的地方

1. **测试覆盖**：增加单元测试和集成测试
2. **性能监控**：添加指标暴露（Prometheus）
3. **配置热重载**：支持修改 config.json 无需重启
4. **插件系统**：更正式的插件加载机制（当前已很好，但可更 formal）

## 总结

nanobot 是一个**设计极其精良的轻量级 Agent 框架**：

- **架构清晰**：Bus + AgentLoop + Registry 模式
- **扩展性强**：Provider、Channel、Tool 都易于添加
- **代码精简**：~4,000 行核心实现完整功能
- **文档完善**：README 非常详细，开箱即用
- **生产就绪**：Docker、systemd、多实例都支持

**学习价值极高**，推荐阅读源码！特别是：
- `providers/registry.py` - Provider 注册机制
- `agent/loop.py` - Agent 核心循环
- `bus/` - 消息总线设计
