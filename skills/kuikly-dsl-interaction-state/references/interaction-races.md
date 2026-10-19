# 输入、手势与异步竞态：按操作顺序取证

只在事件命中异常、旧值回写、滑动闪帧或页面恢复后出现旧结果时读本页。以下操作 ID 和日志字段是诊断示例，不要求项目新增统一日志框架。

## 点击无效或触发两次

先记录 `op`、命中节点、父/子事件来源、状态提交次数。点击无日志优先看透明遮罩、图片/文本覆盖和实际命中区域；有两次提交优先看父子都绑定 click 或触摸按下与 click 都写状态。一次手势通常只应有一个业务提交点，不能靠固定毫秒阈值盲目去重。自动化点击还要确认截图像素、设备触控坐标与 Kuikly 布局坐标的缩放关系。

## 输入框旧值反灌

若 `Input.attr { text(ctx.query) }` 与 `textDidChange` 同时写 `query`，把每次写入标为 `user`、`validation`、`remote`、`restore`。目标版本若支持同步编辑事件，`Pager.body()` 的局部写法可参考下面的片段（假定 `val ctx = this`、`query` 是响应式字段）：

```kotlin
Input {
    attr { text(ctx.query) }
    event {
        textDidChange(true) { ctx.query = it.text }
    }
}
```

这里的 `true` 是同步编辑选项，只处理编辑事件与程序回填的时序问题，不替代来源与代次检查。例如：

```text
op=21 user     text="12"
op=21 remote   text="1"   ← 旧回包覆盖
op=21 visible  text="1"   ← 不是键盘漏字
```

再检查程序设置 text 是否又触发原生文本变化事件、输入法组合期间是否提前格式化。若问题涉及光标、选区或输入法组合态，核对目标版本的 `textInputState` 与 `textInputStateChange`，不要只保存文字。上述同步编辑选项需核对目标版本和平台行为，再比较焦点、光标、删除键、程序清空、失焦回填。不要通过反复重建输入框或延时清空掩盖循环。

## 滑动从第 1 项到第 3 项却闪第 2 项

临时预览和最终选中分开记录：

```text
gesture=8 start=1 target=3 preview=3 commit=3 end=3
```

若轨迹是 `preview=2 → preview=3`，查是否按索引逐格替换了真实内容；若 `commit=3` 后又 `end=2`，查上一次动画完成回调是否还在写状态。新手势开始时清理旧预览，取消手势时复位，提交时只消费本次目标。

横向卡片嵌在纵向滚动容器时，分别试横滑、竖滑、小幅斜滑、手指离开区域和中途取消。先确认事件由谁认领、何时父容器接管，再处理拦截；过早拦截会影响纵滚，过晚接管会截断横滑。官方组件文档中的 `capture` 是鸿蒙实验性能力，不能直接写成 Android/iOS 共通方案；按目标平台版本实测。

## 旧异步结果覆盖新状态

“当前 Tab 仍是 A”不足以判定一个 A 请求仍有效：用户可能经历 A→B→A。为页面、筛选条件和操作增加可比较的代次，在回包或动画结束时只允许**当前代次**提交。示例：

```text
request=41 tab=A generation=1  发出
request=42 tab=B generation=2  发出
request=43 tab=A generation=3  发出
request=43                 返回并提交
request=41                 迟到，丢弃或仅缓存；不得覆盖 generation=3
```

先用可控乱序回包复现，再在“取消旧任务”“忽略迟到结果”“合并结果”中选符合业务的方案；不要无证据加全局锁。代次只解决**结果提交顺序**，不会自动取消网络副作用或防止重复下单；对有副作用的操作，仍需按业务协议确认幂等与取消语义。页面离开后回包还要区分允许缓存数据与允许更新可见 UI。状态更新遵循所用 Kuikly 版本的线程约束。

## 返回后重复刷新

子页返回、Tab 切回、App 回前台、容器重建是不同事件。给一次恢复记录 `appearCount/requestCount/applyCount`；如果一个返回动作导致两个等价请求，先找两个触发入口，不要靠最终数据相同掩盖重复。一次性子页结果应消费后清除或标记；在 `pageWillDestroy` 中清理监听和计时任务，但不要调用 Native Module（目标版本仍需核对生命周期约束）。

验收按“事件次数、最终状态、可见内容”三列记录。设备不可用时只能写源码或编译检查，不能写真机交互通过。

官方核对：[基础事件](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/API/components/basic-attr-event.md)、[Input 事件](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/API/components/input.md)、[Pager 生命周期](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/pager-lifecycle.md)。
