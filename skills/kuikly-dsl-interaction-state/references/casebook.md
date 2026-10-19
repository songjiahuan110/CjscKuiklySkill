# 三个交互案例：复现、修复与验收

以下案例来自跨端业务页面联调中反复遇到的状态问题，已去掉业务名称、资源和接口协议。代码是 `Pager` 内的局部示意，不是独立可编译文件；先核对项目锁定的 Kuikly 版本，再使用对应指令、组件和线程规则。

## 案例一：同 ID 行的数据变了，屏幕仍是旧标题

**复现。** 列表已有 `Row(id="B", title="旧")`，请求返回 `Row(id="B", title="新")`；增加新 ID 能显示，但 B 仍显示“旧”。如果使用 `diffUpdate`，先记录比较函数和实际 item 更新事件，不能只看请求数据。

```kotlin
// 易错：比较器只认为 ID 是否相同，不能据此推断标题会被刷新。
rows.diffUpdate(newRows) { old, new -> old.id == new.id }
```

**局部修复。** 对必须更新的行，按当前数据模型选择响应式字段或显式更新对应 `ObservableList` 元素；不要为了一个标题清空整个列表。例如已定位到 B 所在位置时：

```kotlin
rows[indexOfB] = rows[indexOfB].copy(title = "新")
```

**验收。** 依次测试同 ID 改标题、插入首行、重排和删除；记录可见标题、节点/行数及选中业务 ID。若只改状态对象而可见文字不变，再看 `attr {}` 是否读取了可观察字段。不要把“新对象已传给 `diffUpdate`”当作屏幕已更新的证据。

## 案例二：点第 1 项切到第 3 项，短暂闪过第 2 项

**复现。** 水平 `PageList` 的三项分别有明显不同的内容；从索引 0 点击索引 2，录屏逐帧观察。若滚动动画逐页经过索引 1，而页变更事件又提交了“当前选中项”，中间内容会短暂成为可见状态。这不是一次请求返回错了，而是“动画预览”被当作“最终提交”。

```kotlin
// 易错：非相邻点击仍要求逐页滚动，且中途页变更可能被写入最终状态。
selectedIndex = targetIndex
pagerRef.view.scrollToPageIndex(targetIndex, animated = true)
```

**局部修复。** 若交互要求直接跳到目标，非相邻点击不播放逐页滚动动画，并让页变更事件区分程序跳转与用户手势。相邻切换是否保留动画由产品交互决定。

```kotlin
selectedIndex = targetIndex
pagerRef.view.scrollToPageIndex(targetIndex, animated = false)
```

**验收。** 0→2、2→0 连续操作不出现索引 1 内容；手指逐页横滑仍能正常选中；中途取消不会提交目标页。若问题来自旧动画完成回调，仍需给回调加本次操作标识，仅关闭动画并不能解决旧回调覆盖。

## 案例三：A→B→A 后，第一次 A 的慢请求覆盖第三次 A

**复现。** 让第一次 A 请求慢于第三次 A，请求返回顺序为 `A(3) → A(1) → B(2)`。只判断 `selectedTab == "A"` 时，第一次 A 的旧数据仍有机会覆盖第三次 A。

```kotlin
// 易错：相同 Tab 不代表相同请求上下文。
load(tab) { result ->
    if (selectedTab == tab) visibleData = result
}
```

**局部修复。** 在筛选条件或 Tab 发起新请求时递增代次，回包时同时核对代次和页面存活状态。这里的 `load`、`isPageActive` 是业务侧占位接口，不是 Kuikly 内置方法。

```kotlin
val generationAtStart = ++requestGeneration
load(tab) { result ->
    if (generationAtStart != requestGeneration || !isPageActive()) return@load
    visibleData = result
}
```

**验收。** A(3) 最终保持可见；A(1)、B(2) 不提交到当前 UI；离页后回包不更新已失活节点。若请求有下单、支付等副作用，代次只保护 UI 提交，仍需另行核对服务端幂等与取消语义。

官方核对：[响应式更新](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/reactive-update.md)、[指令系统](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/directive.md)。
