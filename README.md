# LyClaw · Java AI Agent 框架

> **Java 21 + Spring Boot 3 + WebFlux** 的多 Agent 协作框架：声明式 Agent 定义、ReAct 推理-行动循环、Agent Mesh 消息路由、子 Agent 委派、可插拔 Pipeline、27 个生命周期 Hook、上下文自动压缩、容器沙箱与 SSE 流式推送。

[![Java](https://img.shields.io/badge/Java-21-orange)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.14-green)](https://spring.io/projects/spring-boot)
[![Modules](https://img.shields.io/badge/Maven-6%20modules-blue)](#模块结构)
[![Tests](https://img.shields.io/badge/Tests-40%20classes-lightgrey)](#测试)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

---

## 一句话概括

把"多个 LLM Agent 各自有专长、能互相发消息、能派生子 Agent、能被工具和 Hook 约束"这套东西，做成一套**可嵌入任意 Spring Boot 应用**的框架——用注解声明 Agent，用 Mesh 路由消息，用 ReAct 引擎跑循环，用 SSE 把过程流式推给前端。

---

## 1. 三个核心抽象

| 抽象 | 角色 |
| --- | --- |
| **AgentMesh** | 多 Agent 系统的**注册中心 + 消息路由器 + 事件总线** |
| **AgentInstance** | 消息驱动的执行单元，三种实现：`LLMAgent` / `ToolAgent` / `ProxyAgent` |
| **ReActEngine** | LLM 多轮「思考 → 调用工具/子 Agent → 观察 → 继续」循环引擎，流式优先 |

```
用户请求 → AgentMesh（路由/发现）→ LLMAgent（ReAct 循环）
                                      │
                    思考 → 工具调用 / 子 Agent 委派 → 观察结果 → 继续
                                      │
                              SSE 事件流 → 前端
```

**三种 Agent 类型**

| 类型 | 说明 |
| --- | --- |
| `LLMAgentInstance` | 完整 AI Agent：system prompt + tools + model，内部跑 ReAct |
| `ToolAgentInstance` | 无状态工具 Agent，封装单个 Tool 的执行 |
| `ProxyAgentInstance` | 包装 `@Agent` 注解接口的 JDK 动态代理（向后兼容） |

---

## 2. 模块结构

```
lyclaw-ai-framework
├── lyclaw-framework       SPI + 注解 + 数据模型（零 Spring 依赖）          263 个 Java 文件
├── lyclaw-autoconfigure   Spring Boot 自动配置 + BeanPostProcessor 扫描      37
├── lyclaw-action          默认实现（ToolRegistry / SkillExecutor / Sandbox） 43
├── lyclaw-starter         一站式 Starter POM（只聚合依赖）
├── lyclaw-web             独立部署入口 + REST 控制器 + SSE                   10
└── lyclaw-ui              Vue 3 + Vite 前端
```

> 设计原则：**核心零框架依赖**，Spring 只在 `autoconfigure` 层出现；因此 `lyclaw-framework` 可脱离 Spring 独立使用。

---

## 3. 核心能力

| 能力 | 说明 |
| --- | --- |
| **Agent Mesh** | 动态注册、按能力发现（`findByCapability`）、点对点发送、广播、流式订阅 |
| **ReAct 循环** | 多轮工具调用，最大轮数可配；支持工具审批（前端弹窗，超时自动拒绝） |
| **子 Agent（Subagent）** | 运行期生成、委派模式（`suggest` / `prefer`）、并发上限、嵌套深度限制、超时归档 |
| **Pipeline 管线** | 可插拔 Stage：`ContextBuild → SecurityCheck → PlanExecution → Respond → Metrics` |
| **工具系统（三种写法）** | ① `@Tool` 注解 ② 实现 `Tool` 接口 ③ 动态 `ToolProvider` |
| **Hook 系统** | **27 个生命周期钩子**（Agent 前后、工具前后、LLM 前后、压缩前后…） |
| **声明式 Agent** | `@Agent` 注解声明接口，自动生成动态代理 |
| **上下文压缩** | token 预算 + 保留最近 N 轮 + QualityGuard 质检重试 + 中途预检查 + 结果修剪 |
| **模型抽象** | OpenAI 兼容 / Anthropic / Ollama / Gemini；多 Provider、路由、降级链、熔断、重试退避 |
| **沙箱隔离** | `DIRECT` / `SANDBOX`（守护线程+临时目录）/ `PROCESS`（独立进程）/ Docker / Podman |
| **流式增强** | 块流式输出（按段落/句子断块）、人类打字延迟模拟、typing indicator |
| **任务规划** | 规则 + LLM 混合规划器（置信度阈值决定走 DAG 规则还是 LLM 分解） |
| **其他** | Agent 路由（按频道/@提及）、身份系统（IDENTITY.md）、心跳任务、引导文件（AGENTS/TOOLS/MEMORY.md） |

---

## 4. 快速开始

### 4.1 环境

| 组件 | 版本 |
| --- | --- |
| JDK | **21+** |
| Maven | 3.8+ |
| Node.js | 20+（前端） |

### 4.2 配置

最小配置（`lyclaw-web/src/main/resources/application.yml`）：

```yaml
lyclaw:
  chat:
    default-provider: deepseek
    default-model: deepseek-v4-flash
    models:
      deepseek:
        provider: openai-protocol          # openai-protocol | anthropic | ollama | gemini
        base-url: https://api.deepseek.com
        api-key: ${DEEPSEEK_API_KEY:}      # 环境变量注入
        model: deepseek-v4-flash
        retry: { max-attempts: 3, backoff: exponential }
        fallback: []                        # 降级链
```

关键环境变量：

| 变量 | 说明 |
| --- | --- |
| `DEEPSEEK_API_KEY` | 模型密钥（可配多个 Provider） |
| `TAVILY_API_KEY` | `web_search` 工具（不配则该工具不可用） |
| `MAIL_PASSWORD` | 若启用邮件通知 |
| `LYCLAW_SQLITE_PATH` | 会话库路径（默认 `./data/lyclaw-sessions.db`） |

### 4.3 构建与启动

```bash
# 构建全部模块
mvn -DskipTests clean package

# 启动独立服务（默认 8082）
java -jar lyclaw-web/target/lyclaw-web-*.jar
# 或
mvn -pl lyclaw-web spring-boot:run

# 前端
cd lyclaw-ui && npm install && npm run dev
```

启动后：

| 入口 | 地址 |
| --- | --- |
| 服务端口 | `http://localhost:8082` |
| Swagger UI | `http://localhost:8082/swagger-ui.html` |
| OpenAPI JSON | `http://localhost:8082/v3/api-docs` |
| Prometheus 指标 | `http://localhost:8082/actuator/prometheus` |
| 健康检查 | `http://localhost:8082/actuator/health` |

### 4.4 作为依赖嵌入已有项目

```xml
<dependency>
    <groupId>lyjew.com</groupId>
    <artifactId>lyclaw-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

---

## 5. 使用示例

### 5.1 声明一个 Agent

```java
@Agent(
    id = "code-reviewer",
    name = "代码审查员",
    description = "审查代码质量、安全与可维护性",
    systemPromptOverride = "你是一位资深代码审查员，关注质量、安全与可维护性。",
    model = "deepseek-v4-flash"
)
public interface CodeReviewer {
    @Tool(description = "审查指定路径的代码")
    String review(@ToolParam(description = "代码路径") String path);
}
```

### 5.2 通过 Mesh 调用

```java
AgentRef ref = mesh.register(AgentSpec.builder()
        .agentId("code-reviewer")
        .capability("code-review")
        .model("deepseek-v4-flash")
        .systemPrompt("你是一个代码审查员…")
        .build());

AgentMessage resp = mesh.send(AgentMessage.builder()
        .to("code-reviewer")
        .payload("审查这个 PR")
        .build()).join();

Flux<AgentMessage> stream = mesh.sendStream(message);   // 流式
List<AgentRef> reviewers = mesh.findByCapability("code-review");
```

### 5.3 工具三种写法

```java
// ① 注解
@Tool(description = "返回天气")
String weather(@ToolParam(description = "城市名") String city);

// ② 实现接口（完全掌控 schema / 超时）
public class ShellTool implements Tool { ... }

// ③ 动态注册
toolRegistry.register(ToolProvider.of("custom", spec, fn));
```

---

## 6. 项目结构

```
Lyjew-Claw/
├── pom.xml                       # 聚合 POM（Java 21 / Spring Boot 3.5.14）
├── lyclaw-framework/             # 核心 SPI：Agent / Tool / Hook / Pipeline / Model
│   └── src/main/java/lyjew/com/lyclaw/
│       ├── agent/  mesh/  react/  tool/  hook/  pipeline/
│       ├── model/  plan/  compact/  sandbox/  streaming/
│       ├── routing/  identity/  heartbeat/  bootstrap/  extension/
│       └── ...
├── lyclaw-autoconfigure/         # @Configuration + 配置属性绑定
├── lyclaw-action/                # 默认实现与内置工具
├── lyclaw-starter/               # 依赖聚合
├── lyclaw-web/                   # 独立服务：controller + application.yml
├── lyclaw-ui/                    # Vue 3 前端（2.0.0）
├── docs/                         # 架构与演进文档（8 篇）
├── sql/                          # 建表脚本
└── tools/                        # loki / prometheus 监控配置
```

---

## 7. 测试

```bash
mvn test                                  # 全量（40 个测试类）
mvn -pl lyclaw-framework test             # 只跑核心模块
```

覆盖方向：ReAct 循环与事件、工具管道与沙箱、上下文压缩与质量守卫、Mesh 路由、Hook 触发顺序、模型降级/熔断/重试。

---

## 8. 可观测性

- **Actuator**：`health / info / metrics / prometheus`
- **Prometheus 指标**：LLM 调用次数与延迟、工具调用统计、ReAct 轮数、压缩触发次数
- **Loki + Grafana**：`tools/loki`、`tools/prometheus` 提供现成配置与看板 JSON
- **SSE 事件流**：思考、工具调用、工具结果、子 Agent 公告、压缩通知

---

## 9. 配置速查（按模块）

| 前缀 | 作用 | 常用项 |
| --- | --- | --- |
| `lyclaw.chat` | 模型与调用 | `models.*`、`default-provider`、`fallback-chain`、`circuit-breaker` |
| `lyclaw.tool` | 工具执行 | `default-timeout-ms`、`max-rounds`、`sandbox-level` |
| `lyclaw.agent` | Agent 行为 | `default-mode`、`max-tool-rounds`、`approval-timeout-seconds` |
| `lyclaw.subagent` | 子 Agent | `delegation-mode`、`max-concurrent`、`max-spawn-depth` |
| `lyclaw.plan` | 任务规划 | `max-nodes`、`time-budget-ms`、`hybrid-confidence-threshold` |
| `lyclaw.compaction` | 上下文压缩 | `reserve-tokens`、`keep-recent-tokens`、`quality-guard` |
| `lyclaw.sandbox` | 容器沙箱 | `backend`(DOCKER/PODMAN)、`memory-limit-mb` |
| `lyclaw.streaming` | 流式增强 | `block.*`、`human-delay.*`、`typing-indicator.*` |
| `lyclaw.heartbeat` | 心跳任务 | `every`、`active-hours`、`prompt` |
| `lyclaw.retry` | 运行重试 | `base`、`max`、`default-strategy` |

完整带注释的配置样板见 `lyclaw-web/src/main/resources/application.yml`。

---

## 10. 已知限制

| 项 | 说明 |
| --- | --- |
| 会话存储 | 默认 SQLite 单文件，多实例部署需换成共享存储 |
| 沙箱强度 | `PROCESS` 级为独立进程，非内核级隔离；强隔离需 Docker/Podman |
| 子 Agent 深度 | 默认 `max-spawn-depth=1`（不允许递归生成），可配但需注意成本 |
| 上下文压缩 | 压缩依赖 LLM 调用，长会话会增加额外 token 开销 |
| 前端完整度 | UI 2.0 以会话/调试为主，管理端功能有限 |

## 11. Roadmap

- [ ] 会话存储抽象（SQLite / JDBC / Redis 多实现）
- [ ] 工具结果向量化缓存，降低重复调用
- [ ] Agent 评估集与回归基准（多 Agent 任务成功率）
- [ ] 多租户隔离与配额
- [ ] 前端 Agent 拓扑可视化

---

## 12. License

[MIT License](LICENSE) © 2026 Lyjew
