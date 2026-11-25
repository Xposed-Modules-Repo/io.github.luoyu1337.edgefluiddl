# EdgeFluidDL
Edge 下载显示到流体云

在 Edge 进程里改写下载通知，让它变成 Android 16 的 ProgressStyle 通知。
ColorOS 16 / 17 的流体云会把它显示成胶囊或卡片，其它 Android 16+ 系统显示为普通进度通知。

- 下载中：显示文件名、进度、剩余时间，保留 Edge 的暂停和取消
- 下载完成：显示「打开」和「关闭」两个按钮；点「打开」会立即撤掉通知并弹出系统选择器，通知最长保留 1 分钟

## 要求

| 项目 | 要求 |
| --- | --- |
| 系统 | Android 16（API 36）及以上 |
| 框架 | LSPosed（API 93+，在 2.2.1 / API 102 上测过） |
| 目标应用 | Microsoft Edge（`com.microsoft.emmx`） |

## 兼容性

| 环境 | 表现 |
| --- | --- |
| ColorOS 16 / 17 | 胶囊、进度条、打开/关闭按钮 |
| 其它 Android 16+ 系统 | 普通进度通知，不会报错 |
| Android 15 及以下 | 不做任何修改 |

用到的系统 API 都会先探测（`ProgressStyle`、`setProgressTrackerIcon`、
`setRequestPromotedOngoing`、`setShortCriticalText`）。缺任何一个就放弃改写，
原通知照常显示。

## 安装

1. 安装 `Edgefluiddl.apk`。
2. 在 LSPosed 里启用模块，作用域勾选 Microsoft Edge。manifest 里用 `xposedscope`
   写好了推荐作用域。
3. 强制停止 Edge，让模块重新加载：

   ```bash
   am force-stop com.microsoft.emmx
   ```

每次更新模块后都要重新强停 Edge，否则运行的仍是旧代码。

## 工作原理

Edge 的下载通知由 Edge 进程自己发出，不走系统 DownloadManager。通道是 `downloads`，
下载完成后是 `completed_downloads`，通知 extras 里带 `android.progress`。

模块在 Edge 进程内 hook `NotificationManager.notify(...)`，用 `recoverBuilder()`
重建通知后放回参数：

| 状态 | 改写方式 |
| --- | --- |
| 下载中 | `ProgressStyle` + `setProgress` + `setRequestPromotedOngoing(true)` + `setOngoing(true)` |
| 完成 | `ProgressStyle(100)` + promoted ongoing + 两个 `Notification.Action` |

几个容易踩的点：

- 完成通知的 extras 要和 style 一致，否则流体云按 0/0 排版，按钮会被挤出卡片。
- `setProgressTrackerIcon` 传 `null`，去掉进度条上的默认圆形图标。
- 按钮不要设图标，否则 ColorOS 会预留图标位，标题整体右偏。
- 完成通知设置了 60 秒超时。「打开」时系统先直接启动 chooser，再由模块里的 `OpenProxyActivity` 发广播让 Edge 撤掉通知；收尾页起不来也不影响打开。

## 构建

仓库用离线脚本构建，不需要 Gradle：

```bash
bash scripts/build.sh      # Linux：javac + d8
sh scripts/link.sh         # 设备：aapt2 link
bash scripts/package.sh    # Linux：塞 dex + 签名
# 产物：build/Edgefluiddl.apk
```

脚本里的 SDK 路径默认是 `/data/local/tmp/fluiddl/sdk`，aapt2 在
`/data/local/tmp/tprefix/.../usr/bin`（Termux 解包版）。换机器时改这两个变量。

## 仓库结构

```
app/src/main/
├── AndroidManifest.xml        xposedmodule / xposeddescription / xposedminversion / xposedscope
├── assets/xposed_init         入口类（io.github.luoyu1337.edgefluiddl.Module）
├── res/                       启动图标（自适应图标 + 各密度 PNG）
└── java/io/github/luoyu1337/edgefluiddl/
    ├── Module.java             IXposedHookLoadPackage，只作用于 Edge
    ├── NotifyHook.java         hook NotificationManager.notify
    ├── NotificationRewriter.java  改写通知、按钮、MediaStore、广播接收器
    └── OpenProxyActivity.java    「打开」收尾页：撤通知，不参与打开流程
AGENTS.md                     构建、坑和验证方法
SUMMARY                       LSPosed 模块仓库首页简介
version.env                   版本号（link.sh 读取）
compat/stubs/                 Xposed API 编译占位（只编译，不打包）
tools/DumpApi.java            真机反射探测小工具（查 ProgressStyle 方法签名）
scripts/                      build.sh / link.sh / package.sh / 调试签名
```

## 已知限制

- 只处理 Edge 的 `downloads` / `completed_downloads` 通道，Edge 改通道名后需要同步修改。
- 「关闭」和「打开」后的自动清除都依赖 Edge 进程内的广播接收器，Edge 进程被杀后这两步无效；
  完成通知最多 1 分钟后由系统自动清理。
- 「打开」的文件 URI 来自 MediaStore（按通知标题匹配最近的下载记录），
  匹配不到时退化成不带 URI 的 `ACTION_VIEW`。
- 流体云由 SystemUI 插件实现，系统大版本升级后 API 可能变化，
  用 `tools/DumpApi.java` 重新探测方法签名。
- 不要 kill 或手动重启 lspd，会断开 zygote / system_server 的连接，
  只能重启手机恢复。

## 许可

[MIT](https://github.com/Luoyu1337/edge-fluiddl/blob/main/LICENSE)

源码与问题反馈：[Luoyu1337/edge-fluiddl](https://github.com/Luoyu1337/edge-fluiddl)

## 图标

`mahiro_edge.svg` 由 mahiroDesu 提供 · [B站主页](https://space.bilibili.com/511020312/)


---

当前版本 1.0（versionCode 1）。改代码、构建、验证的具体步骤见 [AGENTS.md](https://github.com/Luoyu1337/edge-fluiddl/blob/main/AGENTS.md)。
