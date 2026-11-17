# HideForJianxing

一个基于 [libxposed API 102](https://github.com/libxposed/api) 的 LSPosed 模块。

> 本项目仅供个人学习与本地研究使用。

---

## 功能概览

| 目标 | 实现方式 |
| 广告屏蔽 | 拦截 `AdsBootstrap#initialize` 返回 false，使广告 SDK 根本不启动；`getFeedAds` / `loadAdvertising` 返回空表 |
| 抓包取证 | 请求/响应落盘到 `runlog/captures.jsonl`，落盘前脱敏；支持 root 直读、SAF 导入、hook 推送三种取数方式 |


> 在 1.0.17 原生架构里，L2–L9 的目标类虽然仍存在且 hook 装得上，但宿主代码已不再引用它们——
> 绿灯只代表 hook 已安装，不代表被调用。真正生效的是 L1 / L10 / L11 / L12 / L13 与抓包层。

---

## 安装

1. 安装 APK
2. LSPosed → 模块 → 启用本模块，勾选作用域 `yx.c520.xyz`
3. 若同时启用了 `me.simpleHook` 等通用 hook 工具，请取消其对同一 App 的作用域（会互相覆盖）
4. **强停目标 App 后重新打开**——本模块不支持热重载

---

## 验证

`SHOW_HEARTBEAT` 默认为 `false`（避免 Toast 明文暴露模块），判活改看日志：

```
[HideForJX] INSTALL RESULT: 13/13 ALL OK
[HideForJX] ALIVE 13/13 ALL OK
```

- `13/13` = 全部层安装成功
- `N/13` = 有失败，日志会点名失败层
- 完全没有日志 = 模块未加载，检查启用状态、作用域、是否强停过 App

抓包文件（在宿主 App 外部私有目录，无需 root 即可用文件管理器导出）：

```
/sdcard/Android/data/yx.c520.xyz/files/runlog/
├── captures.jsonl   # 一行一条请求/响应，落盘前已脱敏
├── index.tsv        # 时间/类型/URL 摘要
└── device_seed.txt  # 设备种子，敏感，用完请删除
```

拉取（scoped storage 下 `adb pull` 可能失败，用 `exec-out cat` 更稳）：

```powershell
adb exec-out cat /sdcard/Android/data/yx.c520.xyz/files/runlog/captures.jsonl > captures.jsonl
```

---

## 构建

需要 JDK 21、Android SDK（compileSdk 37）、Android Studio 新版（支持 AGP 9.2）。

```powershell
$env:JAVA_HOME = 'E:\JDK\dragonwell-21.0.11.0.11+10-GA'
.\gradlew.bat assembleDebug     # 调试包
.\gradlew.bat assembleRelease   # 混淆后的 release 包
```

工程目录含中文时需 `android.overridePathCheck=true`（已在 `gradle.properties` 中配置）。

---



## 免责声明

- 本项目仅供个人技术学习与本地研究使用，不构成任何操作建议。
- 使用者应自行确认对目标设备与账号拥有合法权利，并遵守所在地法律法规及服务条款。
- 因使用、修改或分发本项目导致的账号异常、数据丢失、设备故障或其他后果，由使用者自行承担。
- 本项目与文中提及的第三方公司、组织及其产品无任何关联，未获其授权或背书。
- 作者不对本项目作任何明示或暗示担保，亦不承担任何直接或间接责任。

---
## 许可证

[MIT](LICENSE) © 2026 ybtou
