# actr 开发生态系统：综合设计说明

### 1. 综述与设计哲学

可以把 actor-rtc（简称 actr）理解为一款基于 Actor 模型的微服务框架。它以 WebRTC 作为通信底座，在此之上构建了高效且安全的点对点通信机制，让各种形式的连接和对话简洁而自然。同时，借助以 Protobuf 为基础构建的服务契约体系，配以 actr 的脚手架生成工具，简单准备，开发者就可以得到拥有完整类型系统支持的本地编程体验。

我们的核心设计哲学是以“协议优先 + 运行时闭环”为基础，通过管理服务间的契约而非代码包，构建可演进、可治理的服务网络。

这个生态系统由三个协同工作的核心支柱构成，它们共同为开发者提供了一个从项目初始化到生产部署的无缝体验。

### 2. 生态系统的三大支柱

| 支柱                      | 角色                 | 核心职责                                                                                                                                                                                                                                                                                                                                   |
| :------------------------ | :------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **开发者**                | **业务创造者**       | • **定义 `.proto` 契约**：描述服务提供的接口和依赖的接口。<br>• **实现业务逻辑**：在框架提供的生命周期和消息处理回调中编写代码。<br>• **声明 Actor 意图**：通过 `Actr.toml` 配置文件描述 Actor 的身份、能力和依赖。                                                                                                                        |
| **`actr` CLI**            | **开发环境的赋能者** | • **项目脚手架 (`init`)**：创建结构良好、遵循最佳实践的项目模板。<br>• **依赖管理 (`install`)**：解析依赖，生成 `actr.lock.toml`，确保构建的一致性。<br>• **代码生成器 (`gen`)**：基于 proto 文件和配置调用代码生成器，生成业务接口代码。<br>• **脚本执行器 (`run`)**：作为用户自定义命令的入口，启动 Actr 进程。                          |
| **Actr 框架及运行时系统** | **运行时的基石**     | • **运行时组装**: 提供构建器 (Builder) API，允许开发者在 `main` 函数中，通过编程方式（例如，从解析过的配置文件中）注入信令、Actor 等核心组件。<br>• **提供运行时能力**：通过 `Context` 对象向业务代码暴露通信、服务发现、日志等核心功能。<br>• **管理 Actor 生命周期**：负责底层的网络连接、状态管理，并在关键事件时调用开发者的业务逻辑。 |

这三个支柱形成了一个完美的闭环：开发者通过 `.proto` 和 `.toml` **声明意图**，CLI 将这些意图转化为**可执行的环境和代码**，SDK 则在这个环境中为开发者的业务逻辑**提供动力**。

这个分工清晰的模型是理解 `actr` 生态系统工作方式的关键。CLI 负责准备好**构建时**的一切（`.lock` 文件，生成的代码，服务指纹），并充当**启动**的入口；SDK 则提供运行时环境，并响应由开发者组装和初始化的系统。

### 3. 目录与如何阅读

总体导航（你可以从很小开始，也可以逐步深入）

- 如果你刚刚接触 Actr：从“介绍”和“概念与架构”开始，建立心智模型。
- 如果你准备上手开发：直接看“开发者指南”，跟着 CLI 与生成一步步做。
- 如果你要理解系统实现细节：进入“实现内幕”和“协议与规范”。
- 如果你只想查名词或策略：去“附录”。

顶层入口

- [0-ecosystem-overview.zh.md](0-ecosystem-overview.zh.md)
  - 总览 Actr 的目标、核心理念（Actor 为本、协议优先、双路径）、开发—生成—运行的闭环。
  - 你将了解到：为什么选择 Actr、它适合谁、如何渐进式采用。
  - 适合谁：初次了解与决定是否采用的读者。

概念与架构（建立心智模型）

- [1-concepts-and-architecture.zh.md](1-concepts-and-architecture.zh.md)
  - 用通俗语言总结 Actr 的核心理念与系统边界。
  - 你将了解到：Actor/消息/上下文、State Path 和 Fast Path 的分工。
- [1.1-actorsystem-and-actor.zh.md](1.1-actorsystem-and-actor.zh.md)
  - 解构 Actor 与 ActorSystem 的职责。
  - 你将了解到：调度、邮箱（Mailbox）、背压、生命周期、并发治理。
- [1.2-framework-internal-protocols.zh.md](1.2-framework-internal-protocols.zh.md)
  - 框架内协议的角色与设计原则。
  - 你将了解到：内部消息格式、路由、兼容性要点。
- [1.3-interacting-with-the-outside-world.zh.md](1.3-interacting-with-the-outside-world.zh.md)
  - Actr 与外部系统的边界与适配方式。
  - 你将了解到：信令、发现、ACL，以及与第三方系统的交互策略。
- [1.4-inter-actor-communication-patterns.zh.md](1.4-inter-actor-communication-patterns.zh.md)
  - Actor 间通信的常见模式。
  - 你将了解到：请求-响应、订阅/发布、流式处理、错误与重试等。

开发者指南（动手做一个，从小到大）

- [2-developer-guide.zh.md](2-developer-guide.zh.md)
  - 你的第一站。把 .proto、Actr.toml、代码生成和运行串起来。
  - 你将了解到：如何用 CLI 完成 init/install/gen/run 的最小闭环。
- [2.1-media-sources-and-tracks.zh.md](2.1-media-sources-and-tracks.zh.md)
  - 媒体源与 Track 的开发范式。
  - 你将了解到：在 State Path 定义控制面，在 Fast Path 处理流数据。
- [2.2-actor-cookbook.zh.md](2.2-actor-cookbook.zh.md)
  - 实用菜谱与范式示例。
  - 你将了解到：常见任务的最佳实践（路由、订阅、回调注册、上下文使用）。
- [2.3-testing-your-actors.zh.md](2.3-testing-your-actors.zh.md)
  - 测试策略与工具。
  - 你将了解到：如何回放 State Path、如何验证 Fast Path 回流的一致性。
- [2.4-project-manifest-and-cli.zh.md](2.4-project-manifest-and-cli.zh.md)
  - Actr.toml、锁文件与 CLI 命令一览。
  - 你将了解到：依赖锁定、生成策略、版本与指纹管理。

实现内幕（理解“为什么这样设计”和“系统如何跑起来”）

- [3-how-it-works.zh.md](3-how-it-works.zh.md)
  - 总述运行机制，连接各专题的中心页。
  - 你将了解到：Attach 与代码生成如何拼装、运行时如何兑现能力。
- [3.1-attach-and-codegen.zh.md](3.1-attach-and-codegen.zh.md)
  - Attach 生命周期与生成物如何接入系统。
  - 你将了解到：Builder/Attach、Trait 生成、回调绑定。
- [3.2-the-context-philosophy.zh.md](3.2-the-context-philosophy.zh.md)
  - Context 的设计哲学。
  - 你将了解到：把复杂性藏在上下文里，给业务“即取即用”的能力。
- [3.3-application-proto-contract.zh.md](3.3-application-proto-contract.zh.md)
  - 应用层 .proto 契约的组织方式。
  - 你将了解到：消息/服务的布局与演进。
- [3.4-signaling.zh.md](3.4-signaling.zh.md)
  - 信令的职责与流程。
  - 你将了解到：握手、预检、穿透与适配器设计。
- [3.5-input-handler-internals.zh.md](3.5-input-handler-internals.zh.md)
  - 输入处理器的内部机制。
  - 你将了解到：消息进入系统后的分发与治理。
- [3.6-the-persistent-mailbox.zh.md](3.6-the-persistent-mailbox.zh.md)
  - 持久化邮箱的设计。
  - 你将了解到：可靠性、顺序性与回放。
- [3.7-state-path-scheduling.zh.md](3.7-state-path-scheduling.zh.md)
  - State Path 的调度策略。
  - 你将了解到：顺序执行、并发控制、背压传播。
- [3.8-fast-path-internals.zh.md](3.8-fast-path-internals.zh.md)
  - Fast Path 内部原理。
  - 你将了解到：直达回调、数据面并行、与 State Path 的回流协作。
- [3.9-automatic-stream-binding.zh.md](3.9-automatic-stream-binding.zh.md)
  - 自动流绑定机制。
  - 你将了解到：如何把协议定义的流自动绑定到回调。
- [3.10-fast-path-lifecycle.zh.md](3.10-fast-path-lifecycle.zh.md)
  - Fast Path 生命周期管理。
  - 你将了解到：注册/注销、资源守恒、错误处理。
- [3.11-production-readiness.zh.md](3.11-production-readiness.zh.md)
  - 生产化清单。
  - 你将了解到：观测性、扩容、容错、升级策略。
- [3.12-service-discovery-and-compatibility.zh.md](3.12-service-discovery-and-compatibility.zh.md)
  - 发现与兼容性协商。
  - 你将了解到：指纹缓存、版本协商、多版本共存。
- [3.13-dynamic-discovery-and-acls.zh.md](3.13-dynamic-discovery-and-acls.zh.md)
  - 动态发现与访问控制。
  - 你将了解到：多租户、安全域、纵深防御设计。

协议与规范（当你需要“精确、可实现”的定义）

- [4.1-overall.zh.md](4.1-overall.zh.md)
  - 协议族总览。
  - 你将了解到：各子协议之间的关系与边界。
- [4.2-actr-protocol.zh.md](4.2-actr-protocol.zh.md)
  - Actr 核心协议定义。
  - 你将了解到：消息格式、服务发现、指纹、兼容性规则。
- [4.3-actr-version.zh.md](4.3-actr-version.zh.md)
  - 版本与兼容性模型。
  - 你将了解到：语义、协商、退让策略。
- [4.4-actr-config.zh.md](4.4-actr-config.zh.md)
  - 配置规范（Actr.toml 等）。
  - 你将了解到：身份、依赖、能力的声明方式。
- [4.5-actr-framework.zh.md](4.5-actr-framework.zh.md)
  - 框架层规范。
  - 你将了解到：运行时能力的边界与抽象。
- [4.6-actr-runtime.zh.md](4.6-actr-runtime.zh.md)
  - 运行时约定。
  - 你将了解到：线程/调度/资源模型的可见契约。
- [4.7-actr-cli.zh.md](4.7-actr-cli.zh.md)
  - CLI 规范与命令语义。
  - 你将了解到：init/install/gen/run 的可预期行为。
- [4.8-signaling-server.zh.md](4.8-signaling-server.zh.md)
  - 信令服务器规范。
  - 你将了解到：接口、状态机与安全策略。

工程与生成（让“声明即生成”真正落地）

- [appendix-codegen-pipeline.zh.md](appendix-codegen-pipeline.zh.md)
  - 代码生成流水线的端到端说明。
  - 你将了解到：从 .proto 到可用 API/回调绑定/路由的全过程。

附录（当参考百科来用）

- [appendix-glossary.zh.md](appendix-glossary.zh.md)
  - 术语表。
  - 你将了解到：快速对齐名词与上下文。
- [appendix-crate-layering-principles.zh.md](appendix-crate-layering-principles.zh.md)
  - crate 分层原则。
  - 你将了解到：模块化与依赖治理的设计准则。
- [appendix-lane-selection-strategy.zh.md](appendix-lane-selection-strategy.zh.md)
  - Lane 选择策略（流/并发相关）。
  - 你将了解到：选择与调优的考虑。
- [appendix-runtime-design.zh.md](appendix-runtime-design.zh.md)
  - 运行时设计补充。
  - 你将了解到：实现层的更多背景与权衡。

推荐阅读路径

- 快速了解：0（介绍）→ 1（概念与架构）
- 上手开发：2（开发者指南）→ 2.4（CLI）→ 回到 2.2（菜谱）完善实现
- 深入机制：3（实现内幕）→ 3.7/3.8（State/Fast Path）→ 3.12/3.13（发现/兼容性/ACL）
- 规范化对齐：4（协议与规范）→ 4.2/4.3（核心）→ 4.8（信令）
- 查缺补漏：附录（术语/分层/运行时）