---
name: kuikly-dsl-interaction-state
description: 排查 Kuikly DSL 点击后不刷新、列表同 ID 旧内容、输入框旧值回写、横纵手势冲突、页面恢复重复加载和异步结果覆盖新状态。按事件到视图的状态链定位；不用于 Compose DSL 或纯样式还原。
---

# Kuikly DSL 交互状态排障

本 Skill 处理“操作已经发生，最终可见内容却不对”。它补充的是排障路径，不是完整 Kuikly API 手册。先确认目标项目使用的是 Kuikly DSL 而非 Compose DSL，再核对项目锁定版本的 `observable`、指令、事件与生命周期签名；不要把 Web/Compose 的整树重组习惯套进 Kuikly DSL。

## 先定位断点

```text
输入命中 → 事件回调 → 状态提交 → 响应式依赖读取 → 属性更新/节点变动 → 可见结果
```

用同一个操作 ID 串联少量日志，例如 `op=17 event=tabClick`、`op=17 state=tabB`、`op=17 visible=tabB`。只补断开的那一段；不要仅凭“点击有日志”就重写页面，也不要用全局版本号强刷来掩盖依赖断开。临时高频日志在问题解决后移除。

按首次不一致处决定下一步：没有事件就查命中与父子事件；有事件但状态没变就查提交条件；状态已变但节点没变就查响应式读取/结构指令；节点已变但屏幕仍旧就查覆盖层、滚动位置或运行中的旧产物；先正确后回退就查旧异步回包/动画完成回调。一次只追一条可复现操作，不同时改五层。

| 现象 | 优先检查 | 按需阅读 |
| --- | --- | --- |
| 状态变了，文字/颜色不变 | `attr {}` 是否直接读取当前响应式值 | [属性、结构与列表](references/reactive-rendering.md) |
| 列表增删有效，同 ID 内容不变 | 列表成员变化与 item 字段变化是否分开处理 | [属性、结构与列表](references/reactive-rendering.md) |
| 分支或节点数量不变 | 普通 `if/forEach` 是否只在构建时运行 | [属性、结构与列表](references/reactive-rendering.md) |
| 点击没命中或触发两次 | 遮罩、父子事件、触摸与 click 的提交点 | [输入、手势与异步竞态](references/interaction-races.md) |
| 输入后旧值反灌、光标跳动 | 用户输入与程序回填是否形成循环 | [输入、手势与异步竞态](references/interaction-races.md) |
| 横滑闪过中间项、斜滑卡住 | 预览态与最终态、父级纵滚事件归属 | [输入、手势与异步竞态](references/interaction-races.md) |
| 先正确后回旧值，返回后重复请求 | 旧回包/动画回调与页面恢复是否重复提交 | [输入、手势与异步竞态](references/interaction-races.md) |

## 两个决定性区分

- **属性变化还是节点结构变化？** 文字、颜色等已有节点属性先检查 `attr {}` 的响应式读取；条件、数量和顺序改变再评估 `vif`、`vfor`、`vbind` 等指令。不要为了改一行字清空整个列表。`vbind` 可能重建子节点，输入焦点、滚动位置和事件引用需额外验收。
- **临时预览还是已提交状态？** 滑动/动画中的候选目标不是最终选中项；异步返回也不是天然最新结果。给手势或请求建立代次，迟到结果只能在仍属于当前页面/筛选条件时提交。

优先做局部修复：把响应式值的读取放回被跟踪的闭包、把结构变化改成合适指令，或限制一次结果提交；只有证据显示数据模型不支持局部更新时，才扩大为组件/状态重构。平台手势差异属于平台适配，需分别验证，不把实验性事件规则写成三端共性。

需要对照最小复现时，读[三个交互案例](references/casebook.md)：同 ID 列表旧内容、非相邻分页闪过中间项、A→B→A 迟到回包。案例只展示故障条件和局部修复，不要求项目采用相同的数据模型或动画方案。

## 修复后怎么验收

至少重现原故障，并测相邻边界：快速反向切换、同 ID 内容变化、输入后立即校验、斜向滑动取消、子页返回、A→B→A 请求乱序、离页后的迟到回包。每项分别看事件次数、最终状态和屏幕可见内容；只确认代码可编译不能写成交互通过。没有运行条件时，明确标注哪些平台尚未实测。

输出说明复现步骤、第一处不一致、最小修复、影响范围以及逐平台验证状态。不要为修一个 Tab 改动无关的 Bridge、宿主或打包逻辑。

框架核对入口：[响应式更新](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/reactive-update.md)、[指令系统](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/directive.md)、[基础事件](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/API/components/basic-attr-event.md)、[Pager 生命周期](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/pager-lifecycle.md)。参考文件中的故障片段是独立总结，不是官方文档的复制。
