# HideForJianxing

一个基于 [libxposed API 102](https://github.com/libxposed/api) 的 LSPosed 模块。
在宿主进程内按账号派生设备标识、中和反作弊上报、屏蔽广告，使同一台设备上的多个账号
在服务端看起来像多台设备。

- 目标作用域固定为 `yx.c520.xyz`（健行校园）
- 当前版本：**v2.2**（`versionCode=10`）
- 分层：**13 层 hook**，逐层成败在状态页与日志中可见
- 兼容：1.0.17+ 原生 Compose 版为主目标，1.0.14 uni-app 旧壳分支保留

> 本项目仅供个人学习与本地研究使用。

---

## 功能概览

| 目标 | 实现方式 |
|---|---|
| 多账号设备隔离 | 按「设备种子 + 学号」确定性派生 `cid` / `device_id` / `oaid` / `installation_id`，统一走 `deriveId()` |
| 反作弊中和 | 覆盖 `DeviceSecurityScan#scan/nativeScan/assess`、`RunningEvidence#securityReport`、`movementSample` 等出口；只清除检测类字段，保留 App 自产的非检测字段 |
| 广告屏蔽 | 拦截 `AdsBootstrap#initialize` 返回 false，使广告 SDK 根本不启动；`getFeedAds` / `loadAdvertising` 返回空表 |
| 抓包取证 | 请求/响应落盘到 `runlog/captures.jsonl`，落盘前脱敏；支持 root 直读、SAF 导入、hook 推送三种取数方式 |

### 13 层

`L1 报告出口 · L2 Scanner · L3 Detector ×8 · L4 MockLocationProbe · L5 NativeEngine ·
L6 NativeSignalMerger · L7 AppListManager · L8 LocationRiskAnalyzer · L9 RiskScorer ·
L10 isFromMockProvider · L11 安装实例 · L12 设备标识 · L13 广告屏蔽`

> 在 1.0.17 原生架构里，L2–L9 的目标类虽然仍存在且 hook 装得上，但宿主代码已不再引用它们——
> 绿灯只代表 hook 已安装，不代表被调用。真正生效的是 L1 / L10 / L11 / L12 / L13 与抓包层。

### 设备标识派生

```
id = SHA-256(seed + "|HideForJianxing|" + role + "|" + 学号) 前 16 字节 hex

seed = Settings.Secure.ANDROID_ID（App 视角）
role = cid | device | oaid | install | logincid | applog
```

三个 role 使用不同盐，模拟真机上这些值互不相等；同账号跨设备得到不同 id，
避免「一个 cid 挂两套硬件特征」。账号尚未出现时回退到进程内随机值，账号到达后
由 `respinRandomIds()` 重算，确保同会话内不出现两个设备身份。

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

### 混淆设计

源码侧用构建期生成的 `Enc.java` 密文表替换 hook 侧字符串字面量
（`MainHook` / `RootRunlog` / `RunlogParser` / `RunRecord`），
解密采用单字节 XOR + UTF-8 循环解码，避免被 R8 常量折叠还原。
`_transaction/Enc.golden.txt` 是改写前的明文基准，用于回归比对。

release 开启 R8（`isMinifyEnabled` + `isShrinkResources`），
`proguard-rules.pro` 中的 `-adaptresourcefilecontents` 会把混淆后的入口类名
自动同步写进 `java_init.list`——**这些规则不能删**，否则 LSPosed 找不到入口类。

> `app/build/outputs/mapping/release/mapping.txt` 必须随发布包归档，否则混淆堆栈无法还原。

---

## 工程结构

```
.
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/yg/hideforjianxing/
│       │   ├── MainHook.java          # hook 主体、派生、抓包落盘
│       │   ├── Enc.java               # 构建期生成的字符串解密表
│       │   ├── MainActivity.kt        # MIUIX 状态页
│       │   ├── RunLogPage.kt          # 跑步记录取数外壳
│       │   ├── RunLogScreen.kt        # 跑步记录 UI
│       │   ├── RunRecord.kt           # 记录模型与序列化
│       │   ├── RunlogParser.kt        # captures.jsonl 解析
│       │   ├── RunlogProvider.kt      # hook → UI 跨进程数据桥
│       │   └── RootRunlog.kt          # 一次 su + cat 流式读取
│       └── resources/META-INF/xposed/ # java_init.list / module.prop / scope.list
├── _transaction/                       # baseline 快照、diff、回滚脚本、验证记录
├── FEATURES.md                         # 分层细节与历史实测数据
└── VERIFICATION_v2.0_1.0.17.txt        # 原生架构适配验证记录
```

---

## 已知限制

- **服务端关联维度未验证**：证据只证明「发出了什么」，不能证明服务端如何判定
- 硬件字段（`brand/model/manufacturer/system/os_version`）为真实值，未伪造
- `location_risk_events[]` 未清洗，`accuracy` 保留真实值
- 出口 IP 不受模块控制
- 微信 openid 未处理
- 不支持热重载；与 `me.simpleHook` 等模块同作用域会互相覆盖
- `LtmbContentProvider#onCreate` 仍会每进程上报一次 `/sdk/user/report`，绕过广告初始化拦截

详见 `FEATURES.md` 与交接文档中的缺口清单。

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
