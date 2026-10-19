# 属性、结构与列表：找到为什么没刷新

只在状态已提交但可见内容不对时读本页。以下是 `Pager.body()` 中的对照片段，假定页面有相应的 `observable` 字段、`val ctx = this`，代码放在返回的 `ViewBuilder` 内；不是独立可编译文件。类的基类、import 和指令签名以目标项目锁定版本核对。

## 1. 已有节点的文字不变

假设页面的 `title` 是 `observable`。构建时先把值取出来，`attr` 只看到普通局部值，后续依赖可能断开：

```kotlin
override fun body(): ViewBuilder {
    val ctx = this
    val titleSnapshot = ctx.title // 易错：构建时取出的普通快照
    return {
        Text { attr { text(titleSnapshot) } }
    }
}
```

把读取留在 `attr {}` 中，改值后观察是否触发属性更新：

```kotlin
override fun body(): ViewBuilder {
    val ctx = this
    return {
        Text { attr { text(ctx.title) } }
    }
}
// 之后更新 title = "新标题"
```

排查证据应有 `state.title=新标题` 与 `visible.title=新标题`。只有前者，查属性绑定；两者都有但屏幕仍旧，查是否观察了错误节点、遮罩或宿主仍在运行旧产物。不要先加只用于强刷的计数器。

## 2. 条件分支与节点数量不变

`body()` 中普通 Kotlin `if` 不会因为状态更新自动重跑整页。需要结构变化时，使用当前版本支持的指令：

```kotlin
View {
    vif({ ctx.loading }) {
        Text { attr { text("加载中") } }
    }
    velse {
        Text { attr { text("内容") } }
    }
}
```

只有一块子树必须随状态整体重建时可评估 `vbind`；不要把输入框、滚动容器和稳定点击节点无故包进去。重建会改变节点身份，可能清掉焦点、光标、滚动位置和旧事件引用。若已有手动 `removeChildren()` 封装，先确认框架指令是否足够，再决定是否保留原封装。`vbind` 的 `null` 值、相邻条件分支限制以当前指令文档核对。

## 3. 列表新增有效，同一行内容还是旧的

响应式列表容器知道“成员增加/删除”，不必然知道某个普通对象的内部 `title` 被赋了新值。用三份输入复现：追加 ID=3、同 ID=2 改标题、ID=1 与 ID=2 重排。记录列表顺序、节点数和可见文字，不能只看集合内容。

纯字符串列表的基本结构示意：

```kotlin
List {
    attr { flex(1f) }
    vfor({ ctx.labels }) { label ->
        Text { attr { text(label) } }
    }
}
```

对象列表若只按 ID 判断差异，必须另行保证同 ID 内容更新：可让行内字段独立响应式、显式更新对应列表元素，或由行组件读取已绑定状态。尤其使用 `diffUpdate(newList) { old, new -> old.id == new.id }` 时，不能认为“传入了新对象”就一定会更新同 ID 行的文字；按目标版本验证比较与节点更新行为。选哪种取决于现有数据模型；不要为了一个字段更新执行 `clear() + addAll()` 让整个列表重建。官方指令文档要求 `vfor` 的 item 闭包恰好创建一个根节点；多字段行应包进一个 `View`。`vforIndex` 的 index/count 不是所有存量行变化时都会自动刷新，若 UI 依赖位置，重排用例必须单独测。`vforLazy` 仅用于官方支持的 `List`，不适用于 `PageList`、`WaterfallList` 等子类，也不是 `vfor` 的无条件替换。

| 操作 | 期望观察 | 若不符优先追查 |
| --- | --- | --- |
| 同 ID 改标题 | 仅对应行文字更新 | 对象字段是否被观察、替换是否触发列表更新 |
| 插入首行 | 行数和顺序更新 | 是否误用构建时 `forEach`、item 根节点是否符合指令要求 |
| 删除/重排行 | 选中项仍指向正确业务 ID | 业务状态是否绑在瞬时 index 上；`vforIndex` 的位置依赖 |
| 改页外状态 | 输入焦点与滚动位置保持 | 是否用 `vbind`/清空列表无谓重建稳定节点 |

## 修复验收

逐项核对“状态值 → 依赖读取 → 节点变化 → 屏幕呈现”。连续快速切换后仍正确、同 ID 更新可见、列表重排不丢选中、输入焦点和滚动位置不被无关状态变化重置，才算这段链路修好。

官方核对：[响应式更新](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/reactive-update.md)、[指令系统](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/directive.md)。
