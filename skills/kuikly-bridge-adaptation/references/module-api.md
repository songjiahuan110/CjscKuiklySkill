# Kuikly Module API：核对真实签名后再写适配层

本页以 KuiklyUI 源码 commit [`77bd2bba`](https://github.com/Tencent-TDS/KuiklyUI/blob/77bd2bba597dc12c2abdd73c8541bc6018b12dae/core/src/commonMain/kotlin/com/tencent/kuikly/core/module/Module.kt) 的 `Module` 为核对基线；项目实际依赖可能不同，先看锁定版本源码。示例只展示调用层，不是可直接粘贴的完整页面或宿主实现。

## 一次性异步调用

`Module` 提供 `asyncToNativeMethod(methodName: String, data: JSONObject?, callbackFn: CallbackFn?)`。下面假设 `hostModule` 是当前 Pager 从已注册模块取得的实例，`request` 是按**目标宿主契约**组装的 `JSONObject`：

```kotlin
hostModule.asyncToNativeMethod("actualHostMethod", request) { result ->
    // result 的业务字段仍需按目标宿主返回样本检查。
    // 回调可能在页面离开后到达：提交 UI 前检查页面/操作代次。
}
```

在上述源码中，`data` 经 `toString()` 进入 Native 调用；Native 返回的 `String`/`JSONObject` 会转换为 `JSONObject` 交给回调。**转换并不等于业务成功**，且非法 JSON 字符串可能在进入业务适配前失败。对照宿主是否返回 JSON 对象、JSON 字符串或内层字符串，不在页面里猜层级。`asyncToNativeMethod` 还有 `Array<Any>` 参数及基础类型/`ByteArray` 回调路径；只有宿主协议确实需要这些数据类型时才选它，并按目标版本核对允许的元素类型。

## 回调生命周期与同步能力

`toNative` 有 `keepCallbackAlive` 与 `syncCall` 参数；常规异步单次调用不要默认打开常驻回调。持续订阅需要明确解除路径，源码提供 `removeCallback(callbackRef)`；订阅 API 的引用保存和解绑方式仍要对照目标版本。同步能力存在不代表适合等待网络或阻塞 UI，超时与取消应是业务协议的一部分。

## 宿主注册

[官方扩展文档](https://github.com/Tencent-TDS/KuiklyUI/blob/main/docs/DevGuide/expand-native-api.md)分别给出 Android、iOS、HarmonyOS 的外部 Module 注册方式。跨端核对的是同一逻辑 Module 名、调用方法和输入输出契约；平台基类、注册函数和线程细节以目标版本及宿主代码为准。不要把某一端的注册样例原样移到另一端，也不要仅凭 Kuikly 侧有类就认为宿主已接入。

最小联调顺序：注册可发现 → 空/最小参数命中方法 → 一个成功样本 → 一个失败样本 → 并发反序样本 → 页面退出后的迟到回包。每一步记录 Kuikly 调用标识与宿主接收/完成标识，不记录密钥或完整个人数据。
