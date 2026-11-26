# 自动暂停（Auto Pause）2.1.0 — 分支说明

本分支基于上游 `v1.0.0` 之后，做了中文化并加了一组「让被限制的应用真的起不来」的能力。
包名保持 `io.github.burakgon.stop` 不变，可与上游版本互相覆盖安装（但签名不同，正式替换前需卸载）。

## 中文化

- `app/src/main/res/values-zh-rCN/` 全量中文（应用名 `自动暂停`，默认语言为 `Auto Pause`）。
- 系统会把 locale 透传给应用，没有硬编码 `Locale.US`，所以跟随系统语言即可。
- 少数脱壳字符串在 `values/strings.xml` 与 smali 之外的部分全部走资源，无遗漏。

## 工作模式（设置 → 工作模式）

| 模式 | 行为 |
| --- | --- |
| `idle` 空闲 | 只做额外限制（电池/待机），不杀进程 |
| `stop` 后台停止 | 离开宽限期后 `forceStop` |
| `suspend_stop` 强停挂起 | 先 `pm suspend` 再 `forceStop`，应用无法自行在后台起来 |
| `freeze` 冻结停用 | 直接 `pm disable-user`（必要时退到 hide），图标变灰/消失，记录实际生效方式 |

「冻结停用」与「强停挂起」的区别在于记录的集合：
`restrictions.auto`（跟规则走，取消限制即释放）与 `restrictions.frozen` + `manual`（改配置不会偷偷解冻）。

## 挂起调用（强停挂起 / 首页「挂起」）

`SystemApis.setSuspended()` 不再写死参数个数，而是按 `IPackageManager.setPackagesSuspendedAsUser`
在本机的真实签名逐位构造参数。Android 15 上它是 9 参且返回 `String[]`：

```
setPackagesSuspendedAsUser(String[], boolean, PersistableBundle, PersistableBundle,
                           SuspendDialogInfo, int flags, String callingPackage,
                           int userId, int callingUid)
```

旧实现用 `parameterTypes.size in 7..8` 过滤，在 9 参机型上匹配不到方法、直接返回 `false`，
界面于是弹「系统拒绝挂起这个包」。现在按名字匹配并逐位填充，`SuspendDialogInfo` 走反射构造，
并带 `BUTTON_ACTION_UNSUSPEND` 中性按钮（系统自带的「解除挂起」）。

挂起失败时回退 `pm suspend`（shell 身份，仅用于用户显式动作，后台拦截不 fork）。
冻结停用不走挂起：`freeze()` 直接 `setEnabled(false)`，失败才退到 `pm disable-user` 与 `setHidden(true)`。

## 点图标即打开

被挂起的应用被点开时，系统会起 `SuspendedAppActivity` 而不是应用本身，任务栈里也查不到原包名。
守护进程改为监听 `logcat -s ActivityTaskManager:I` 的 `START u0 {… cmp=<pkg>/<activity>}`，
命中挂起集合就解挂、记录「冻结期间被点开」事件并拉起原 Activity；同时 5 秒看门狗负责重启监听。

## 其他

- 预装清理：`assets/bloatware/tiers.csv`（269 条，A/B/C 三档）+ `ui/bloatware/` 批量冻结页。
- 桌面快捷方式：`ui/shortcut/` 一键冻结/解冻（`requestPinShortcut`）。
- 事件记录：`tapped` / `freeze` / `thaw` 三种新 reason，活动页加了对应筛选。
- 诊断：`/data/local/tmp/<pkg>/debug.log`（守护进程跑在 Shizuku user service 里，日志进不了 logcat）。
- UI 状态：回到界面与每次冻结类事件后都会重读包状态，避免「已冻结但其实在跑」。

## 构建

上游依赖 AGP 9 / compileSdk 37 / Kotlin 2.4。本分支为了能在 AGP 8.11.1 + compileSdk 36 的
环境里复现，把 `gradle/libs.versions.toml` 与 `compileSdk` 降了一档，并关闭了 `aidl` 特性
（缺可执行的 `aidl` 二进制），`daemon/IStopDaemon.java` 与 `IDaemonListener.java` 是手写的
AIDL 生成物，接口与 `*.aidl` 保持一致。

## 每个应用单独选限制方式（2.0.6 起）

除了全局工作模式，单个应用还可以有自己的限制方式，存在 DataStore 的 `app_actions`
（`package=code;package=code`），随 `DaemonConfig.Rule.action` 下发给守护进程；
守护进程里 `block()` 先看单应用、没有再回落到全局。

- 应用列表每行右侧是「限制方式」按钮，对话框五项：允许后台运行 / 跟随全局 / 后台停止 /
  强停挂起 / 冻结停用。
- 行内状态行显示**实际会生效**的方式（`limitMode = action ?: globalMode`），不再显示「今日已停止 N 次」。
- 应用详情页同样四个按钮（运行 / 停止 / 挂起 / 停用）。
- 新装应用的通知与首页卡片也是同一组选择，选完立刻写入并执行一次。

## 宽限期（2.0.7 / 2.0.8 起）

设置 → 宽限期：预设 立即 / 30 秒 / 1 分钟 / 5 分钟 / 15 分钟 / 1 小时，另有「自定义…」，
自定义可分别填小时与分钟（存 `小时 × 3600 + 分钟 × 60`）。

## 设置页（2.1.0 起）

保护开关换成和首页一致的「暂停 / 恢复」按钮；文案带后端名称：
开启「Shizuku · ADB 功能 · 离开时，以及它们试图自行运行时停止」，
暂停「Shizuku · ADB 功能 · 离开应用时，不会限制应用后台运行。」。
