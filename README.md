# Kuikly AI Skills

一组面向 Kuikly DSL 与 Kotlin Multiplatform（KMP）开发的可复用 AI Skills。目前聚焦两类实际问题：将既有宿主能力接入 Kuikly，以及定位页面交互与响应式状态异常。

本仓库由社区维护，**不是 Kuikly 官方仓库**。Skill 提供排查路径、决策依据和验收方法，不替代目标项目锁定版本的 Kuikly 源码、宿主协议或真机验证。

## Skills

| Skill | 适合处理 | 不适合处理 |
| --- | --- | --- |
| [kuikly-bridge-adaptation](skills/kuikly-bridge-adaptation/SKILL.md) | 从既有 Web/Native Bridge 追踪真实契约，判断 KMP 与 Kuikly Module 的边界，核对 Android、iOS、HarmonyOS 的注册、参数、回调和导航行为 | 凭空设计统一业务协议，或仅凭方法名推断宿主实现 |
| [kuikly-dsl-interaction-state](skills/kuikly-dsl-interaction-state/SKILL.md) | 排查点击后不刷新、列表旧内容、输入回写、手势冲突、异步回包覆盖与页面恢复重复加载 | Kuikly Compose DSL、纯视觉样式还原或无证据的整页重构 |

每个 Skill 都以 `SKILL.md` 为入口，按需阅读同目录的 `references/`。建议先从问题对应的 Skill 入手，不必一次加载全部参考文件。

```text
skills/
├── kuikly-bridge-adaptation/
│   ├── SKILL.md
│   └── references/       # 契约映射、Module API、宿主验收
└── kuikly-dsl-interaction-state/
    ├── SKILL.md
    └── references/       # 响应式渲染、输入/手势/竞态
```

## 使用方式

将所需的**完整 Skill 目录**放到你使用的 AI Agent 支持的技能目录，或通过该工具的导入功能指定本仓库中的对应目录。不要只复制 `SKILL.md`，否则其中的 `references/` 相对链接会失效。技能发现位置与调用语法由具体工具决定；目录组织参考 [Agent Skills 规范](https://agentskills.io/specification)。

安装后可直接在需求中点名，例如：

> 使用 `kuikly-bridge-adaptation`，核对现有 Web Bridge 方法迁入 Kuikly 后的入参、回调和页面关闭行为。

> 使用 `kuikly-dsl-interaction-state`，定位 Tab 状态已更新但界面仍显示旧内容的问题。

## 使用边界

- 先确认项目使用的 Kuikly 版本，并以该版本源码和目标宿主实现为准；Skill 中的 API 片段不是可直接粘贴的完整业务代码。
- Bridge 的通道、封包、错误码和超时属于目标项目契约，本仓库不规定统一值。
- 编译通过、Demo 模拟通过和真实宿主三端验证是不同的结论；报告中应分别说明。
- 公开问题或示例时，请移除 Token、密钥、账户信息和内部地址。

参考资料：[KuiklyUI](https://github.com/Tencent-TDS/KuiklyUI)、[Kotlin Multiplatform 文档](https://kotlinlang.org/docs/multiplatform.html)、[Agent Skills 规范](https://agentskills.io/specification)。本仓库的工作流与文字为独立整理，不代表上述项目的官方建议。
