---
name: kuikly-bridge-adaptation
description: 将已有 Web/Native Bridge 能力接入 Kuikly DSL 页面，或排查 Kuikly Module 与 Android、iOS、HarmonyOS 宿主的参数、回调和导航契约。适用于既有协议迁移与跨端联调；不预设任何业务通道、封包或错误码。
---

# Kuikly Bridge 适配

目标是复用**目标应用已经实现的能力和协议**，让 Kuikly 页面只依赖一层有类型的调用接口。先查证旧调用和宿主实现，再决定是否需要 Module；不要因为旧页面使用 Bridge，就把纯 Kotlin 逻辑也搬到宿主。

## 先判定能力归属

| 能力 | 优先落点 | 判断依据 |
| --- | --- | --- |
| 纯数据变换、可跨端实现的协议解析 | KMP `commonMain` | 不需要设备或宿主状态；有可用的多平台依赖 |
| 平台 API，但没有既有宿主协议 | 平台 source set 的接口实现、必要时 `expect/actual` | 实现能随共享产物交付；所有目标平台都有对应实现 |
| 登录态、路由、权限、支付等既有宿主能力 | Kuikly Module + 宿主适配 | 生命周期、权限或协议所有权在宿主；不能复制一套状态 |

上述是选型，不是互斥的架构层级。一个调用可在 `commonMain` 整理参数、通过 Module 取宿主状态、再在 `commonMain` 解析结果。KMP 边界按[官方互操作说明](https://kotlinlang.org/docs/multiplatform/multiplatform-connect-to-apis.html)与[expect/actual 说明](https://kotlinlang.org/docs/multiplatform/multiplatform-expect-actual.html)核对；尤其不要让 `commonMain` 直接引用单一平台的类。

## 迁移与排障工作流

1. **盘点一条真实调用链。** 从旧页面调用点追到旧 Bridge 实现和目标宿主处理器，记录方法名、每层参数、返回、错误/取消、是否同步、回调次数和导航作用。不能从函数名推断协议。填写[契约映射卡](references/contract-mapping.md)。
2. **缩小适配面。** 保留目标应用已有的通道与封包；页面调用一个职责型 Kotlin API，序列化和旧 selector 留在边界层。若旧能力可直接在 KMP 完成，跳过 Module。
3. **实现真实 API。** 要增加 Kuikly Module 时，先看[Module API 与注册要点](references/module-api.md)，再对照项目锁定的 Kuikly 版本。异步回调、一次性/常驻回调、JSON 字符串层级与线程归属分别确认。
4. **逐端接线。** 每个目标平台核对 Module 注册、宿主方法分发、参数类型、所有终态回调、页面关闭后的行为。使用[宿主契约与验收](references/host-contract-checks.md)记录证据；没有该平台就不虚构实现。
5. **回归两条路径。** 用相同脱敏输入比较旧调用与 Kuikly 调用的宿主实际入参和结果；同时回归原 Web 页面，确认新接入没有改坏共享处理器。

第一次为某个宿主接线时，可按[端到端示例](references/end-to-end-example.md)逐层核对“页面 → Module → 宿主分发 → 回包 → 页面状态”；其中的方法名和结果格式只是示意，必须替换成目标应用查证后的契约。

## 不可跳过的契约边界

- 不发明“通用”请求头、业务码、超时秒数或导航语义；这些必须由目标协议和产品要求决定。
- 不把异步原生调用包装成页面线程阻塞等待。若业务需要超时，明确由哪层计时、取消是否可达宿主、迟到回包如何丢弃。
- 区分 Kuikly 内部页栈、宿主容器栈和旧 Web 容器；名称类似的 `close` 不保证弹出同一层。
- 日志和测试样本脱敏；Token、密钥、账户号不能进入公开 Skill、示例或诊断输出。

交付时给出契约映射、变更的 Kuikly/宿主位置，以及“源码核对、转换样本、平台编译、真实宿主交互”各自的验证状态。只测一端不能声称三端通过。
