# 一条宿主能力的端到端适配示例

本例从跨端工程的联调经验抽象而来：旧 Web 页面和 Kuikly 页面都需要读取宿主设置，页面只关心结果，通道名、封包和原生处理器由目标应用决定。以下 `getPreferredLanguage`、字段和 JSON 是**示意协议**，不是 Kuikly 内置标准，也不是要求项目照抄的公共 Bridge。

## 1. 先写出旧链路与预期契约

| 边界 | 要核对的样本（本例） | 容易漏掉的点 |
| --- | --- | --- |
| 旧 Web 调用 → 宿主 | 方法 `getPreferredLanguage`，无业务参数 | 旧页面是否通过另一层通道/封包传递方法名 |
| Kuikly 页面 → 适配层 | `loadLanguage(callback)` | 页面不拼宿主 JSON，也不判断旧 selector |
| 适配层 → Kuikly Module | 方法 `getPreferredLanguage`，请求 `{}` | `Module` 名及原生注册名是否一致 |
| 宿主 → Kuikly 回调 | `{"result":"{\"language\":\"zh-CN\"}"}` | 外层 `result` 是对象还是 JSON 字符串，不能凭感觉二次解析 |
| 适配层 → 页面 | `Result.success("zh-CN")` 或 `Result.failure(...)` | 空值、解析失败、宿主失败不能静默变成功 |

这张表中的原始回包只是一个**可复现样本**。目标宿主若直接返回 `{ "language": "zh-CN" }`，就按那个契约解析，不额外包装 `result`。旧调用若没有回调，也先确认它是合法的立即动作，还是宿主漏回结果。

## 2. Kuikly 侧：职责型接口和单点解码

下面仅展示调用层片段，省略页面基类、Module 注册和 import。示例使用 Kuikly `Module.asyncToNativeMethod` 的 `JSONObject` 路径；目标版本的真实签名仍以源码为准。

```kotlin
class HostSettingsModule : Module() {
    override fun moduleName() = "HostSettings"

    fun loadLanguage(onResult: (Result<String>) -> Unit) {
        asyncToNativeMethod("getPreferredLanguage", JSONObject()) { raw ->
            val result = runCatching {
                val payload = JSONObject(raw?.optString("result", "").orEmpty())
                check(!payload.has("error")) { "host returned an error" }
                payload.optString("language", "").takeIf(String::isNotBlank)
                    ?: error("missing language")
            }
            onResult(result)
        }
    }
}
```

页面只调用 `loadLanguage`，并在更新 UI 前检查页面是否仍有效；生产代码还应按目标协议区分“宿主失败、格式错误、空语言”，不能只把三者合并为示例中的通用异常。旧 Web selector 只在适配层或宿主边界出现。如果宿主本就有可复用处理器，Kuikly Module 应转发到同一处理器，不复制一份设置读取逻辑。

## 3. 宿主侧：分发、回包和注册

Android 宿主的形状如下；`readPreferredLanguage()` 代表目标应用已实现的能力，并非 Kuikly API。`KuiklyRenderBaseModule`、`KuiklyRenderCallback` 的具体包名和注册函数按目标 Kuikly 版本确认。

```kotlin
override fun call(method: String, params: String?, callback: KuiklyRenderCallback?): Any? {
    if (method != "getPreferredLanguage") {
        callback?.invoke(mapOf("result" to "{\"error\":\"unknown_method\"}"))
        return null
    }
    val result = runCatching {
        JSONObject().put("language", readPreferredLanguage()).toString()
    }.getOrElse {
        "{\"error\":\"read_failed\"}"
    }
    callback?.invoke(mapOf("result" to result))
    return null
}
```

这是契约示意，不是可直接复制的宿主类：实际实现必须把错误结果解析成明确失败，且不能把异常信息、Token 或个人数据直接回传/记录。iOS、HarmonyOS 不照搬 Android 类；分别验证同一逻辑 Module 名、方法名、入参类型、回包层级及成功/失败终态。Demo/mock 也要按同一契约返回，但不能代替真实宿主验收。

## 4. 最小联调记录

| 用例 | 期望证据 |
| --- | --- |
| 正常读取 | 旧 Web 与 Kuikly 命中同一宿主能力；最终可见语言一致 |
| 未知方法/宿主异常 | 一次失败终态，页面不永久 loading、不把空字符串当成功 |
| 两次请求反序返回 | 每个回包回到发起调用；不按“先发先回”配对 |
| 页面退出后回包 | 不再写入已失活页面 |
| 三端逐项验证 | 每端分别记录注册、编译、真实宿主回调；未运行写“未验证” |

**导航能力要单独判断目标栈。** 项目实践中，旧 Web 的“关闭页面”曾命中 Web 容器关闭，而 Kuikly 子页需要弹出 Kuikly 内部页栈；两者不能仅因名称相似就共用同一动作。先画出宿主容器栈和 Kuikly 内部栈，再决定复用旧处理器还是新增宿主适配。不能把本例的读取设置回包格式套用到导航动作。
