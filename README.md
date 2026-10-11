# 亦冬测权

安卓权限探测与信息可见性分析工具。对每一项敏感权限，同时展示「未授权时可获取什么」与「授权后可获取什么」，并区分「始终允许 / 仅在使用中 / 仅此一次」与「未请求 / 可再询问 / 永久拒绝」。

- 网页端仓库：[亦冬权限站](https://github.com/ldy2330785100/Lidawn-Permission)
- 包名：`eydawnlink.lidawn.permission.apk`
- SDK范围：Min 26→Target 36

<div>
  <a href="https://ldy2330785100.github.io/Lidawn-Permission/">
    <img src="https://img.shields.io/badge/website-live-brightgreen" alt="Website">
  </a>
  <a href="https://github.com/ldy2330785100/Lidawn-PermApp">
    <img src="https://img.shields.io/badge/Maintenance-Active-brightgreen" alt="Maintenance">
  </a>
</div>

## 📦 下载与安装

- 👉 [前往下载](https://github.com/ldy2330785100/Lidawn-PermApp/releases/)

- <div>
  <a href="https://github.com/ldy2330785100/Lidawn-PermApp/releases">
    <img src="https://img.shields.io/github/v/release/ldy2330785100/Lidawn-PermApp" alt="GitHub release">
  </a>
  <br>
  <a href="https://github.com/ldy2330785100/Lidawn-PermApp/releases">
    <img src="https://img.shields.io/github/release-date/ldy2330785100/Lidawn-PermApp" alt="Release date">
  </a>
  </div>

## 📝 更新日志

- 👉[ChangeLog](https://ldy2330785100.github.io/Lidawn-PermApp/changelog.html)

## 技术栈

| 项 | 版本 |
| --- | --- |
| Gradle | 8.13 |
| Android Gradle Plugin | 8.11.1 |
| Kotlin | 2.1.21（KSP 2.1.21-2.0.2） |
| Compose BOM | 2025.06.01（Material 3） |
| Hilt | 2.56.2 |
| DataStore | 1.1.7 |
| Navigation Compose | 2.9.0 |
| Biometric | 1.1.0（Fragment 1.8.6） |
| Shizuku | 13.1.5 |
| 测试 | JUnit5 5.13.2 + MockK 1.14.2 + Turbine 1.2.1 + Compose UI Test |

## 构建

Android Studio 打开根目录直接构建，或：

```bash
./gradlew assembleRelease
./gradlew testDebugUnitTest
```

`local.properties` 中的 `sdk.dir` 需指向本机 Android SDK。签名配置从 `keystore.properties` 读取（该文件与 `*.jks` 均已加入 `.gitignore`，不入库）。

## ARM64 Linux 上构建的补充说明

本工程的首次构建在 ARM64 Linux 环境完成。Google 未提供 linux-aarch64 的 aapt2，若在该类环境构建，需要：

```bash
apt-get install -y qemu-user-static libc6:amd64 libgcc-s1:amd64
# 提供一个以 aapt2 结尾的包装脚本（AGP 会校验文件名后缀）
./gradlew assembleRelease -Pandroid.aapt2FromMavenOverride=/opt/aapt2-qemu/aapt2
```

x86_64 机器、Windows、macOS 无需以上步骤。

## 包结构

```
com.ldy.wtpermission.ad
├── WtPermissionApp / MainActivity
├── di/                     Hilt 绑定
├── domain/
│   ├── model/              PermissionState、PermissionSpec、InfoKey、InfoValue、RiskLevel、
│   │                       PermissionKind（六类）、SpecialAccess、VisualizationKind
│   ├── port/               PermissionChecker、AppOpsProbe、ActivityProvider、TimeProvider、ApiLevelProvider
│   ├── repository/         PermissionRepository、HistoryStore（接口）
│   └── usecase/            探测、状态刷新、请求历史、Shizuku、报告用例
├── data/
│   ├── platform/           Context 相关实现（AppOps、权限检查、当前 Activity、API 等级）
│   ├── history/            DataStore 请求历史
│   ├── prefs/              DataStore 应用偏好（主题、动态取色、生物识别、脱敏开关）
│   ├── security/           生物识别认证封装
│   ├── shizuku/            Shizuku 桥接与两个空实现服务（无障碍 / 通知监听，用于检测开启状态）
│   ├── live/               实时数据流（定位、音频电平、传感器三轴）
│   ├── state/              StateDetector 状态推断引擎
│   ├── probe/              ProbeCatalog（62 项核心）＋ ProbeCatalogExtended（185 项扩展）、
│   │                       InfoCollector 采集、Sanitizer 脱敏
│   └── repository/         PermissionRepositoryImpl、ProbeCache
└── ui/
    ├── theme/              Color/Type/Shape/Theme/Motion（动态取色、MD3、弹簧动效）
    ├── components/         StateBadge、CapsuleNavBar、InfoCompareCard、PermissionCard、
    │                       Visualizations、CameraLivePreview 等
    ├── home/               权限总览（底部悬浮胶囊 + 左右滑动切换敏感度）
    ├── detail/             权限详情（状态依据、双卡片对比、实时可视化、平台行为说明）
    ├── settings/           设置（外观、安全、权限申请、Shizuku、数据、关于）
    ├── security/           生物识别锁定页
    ├── report/             报告预览 / Markdown / 文本与图片导出
    └── navigation/         NavHost 与路由
```

## 权限状态推断

系统不暴露细粒度状态，`StateDetector` 组合以下信号推断：

1. `checkSelfPermission`（特殊权限走 `Settings`、`AppOpsManager` 等专门查询）
2. `AppOpsManager.unsafeCheckOpNoThrow`（API 29+）
3. `shouldShowRequestPermissionRationale`（需要 Activity 上下文，取不到时为 unknown）
4. 本地 DataStore 请求历史（请求次数、上次请求/授权/拒绝时间、是否曾被拒绝）
5. 权限在清单中的声明情况与平台 API 区间

| 条件 | 结果 |
| --- | --- |
| 未在清单声明（工具条与能力项除外） | 策略限制（未声明） |
| 当前 API 低于权限最低版本或高于其上限 | 策略限制（版本不支持） |
| AppOps 返回 MODE_ERRORED | 策略限制（设备策略） |
| 特殊权限且未授予 | 策略限制（需设置页开启） |
| 特权权限且未授予 | 策略限制（系统特权，需 root / Shizuku） |
| 工具集成条目且未安装 | 策略限制（本机未安装该工具） |
| 能力项且不可用 | 策略限制（需 Shizuku 或特权工具） |
| 未授予 + 从未请求 | 从未请求 |
| 未授予 + 曾请求 + shouldShow = true | 已拒绝，可再询问 |
| 未授予 + 曾请求 + shouldShow = false | 永久拒绝 |
| 已授予 + 一次性权限 + 曾被拒绝 + 授权在 30 秒内 | 仅此一次 |
| 已授予 + AppOps 返回 MODE_FOREGROUND | 仅在使用中 |
| 已授予 + 其他情况 | 始终允许 |

说明：「仅此一次」优先于「仅在使用中」判定；一次性授权同样会把 AppOps 置为 MODE_FOREGROUND，若按后者先判会误标，因此把时间窗 + 历史拒绝这一更具体的规则前置（已在单元测试中固定）。

## 权限清单

`ProbeCatalog` 共收录 **247 项**，按极高 / 高 / 中 / 低 / 极低五级敏感度分组，每项标注未授权可获取项与授权后可获取项，并附平台行为说明（一次性授权、后台定位、Android 10 起 IMEI 不可读、13 起媒体权限拆分、12 起 WiFi 名称需位置权限等）。

按类型分为六类：

| 类型 | 数量 | 说明 |
| --- | --- | --- |
| 运行时权限 | 运行时申请 | 需要用户在授权对话框中手动允许 |
| 特殊权限 | 需设置页开启 | 悬浮窗、使用情况访问、无障碍、通知读取、修改系统设置、安装未知应用、精确闹钟等 |
| 普通权限 | 安装即授予 | 网络状态、WiFi 状态、振动、前台服务、NFC 等 |
| 特权权限 | 需 root / Shizuku | 系统签名级权限，普通应用即使声明也无法获得，列为只读探测项 |
| 工具集成 | 检测第三方应用 | Dhizuku、黑阈、小黑屋、Shevery、Termux、Shizuku 的安装与运行状态 |
| 能力项 | 特权能力 | AppOps 设置、冻结解冻、Shell 调用、受限私有目录访问等能力可用性 |

## 采样与脱敏

所有采集值统一经 `Sanitizer` 处理。默认**完整显示**采集值（`Sanitizer.reveal = true`）；可在「设置 → 安全」中关闭，关闭后按以下规则脱敏：坐标保留两位小数、IMEI/序列号/ANDROID_ID/IMSI 只留首尾、联系人只留首字、通话号码只留前 3 后 2 位、MAC 只留后两段、文件名保留扩展名、环境音量输出分贝。

## 实时可视化

详情页对以下权限提供实时采样（仅在页面可见期间运行）：

| 权限 | 可视化 |
| --- | --- |
| 精确定位 / 粗略定位 / 后台定位 | 十字准星 + 精度圈 + 采样轨迹 |
| 麦克风 | 实时电平条（超阈值转红） |
| 活动识别 | 加速度计三轴曲线（附陀螺仪） |
| 相机 | 实时预览画面（TextureView + camera2），含预览分辨率与相机状态 |

## 安全

- 生物识别登录：开启后每次打开应用或切回前台均需验证（`androidx.biometric`，MainActivity 继承 `FragmentActivity`）。
- Shizuku 集成：授权后可直接执行 `pm grant` / `pm revoke` / `appops set`，实现免弹窗授予与撤销。
- 无障碍与通知监听：仅提供空实现服务，用于检测该权限是否开启，**不读取任何屏幕内容或通知内容**。

## 已验证

- `:app:assembleRelease` 构建通过，产物约 13.1 MB。
- `:app:testDebugUnitTest` 通过：47 个单元测试，0 失败（StateDetector 全分支 18 项、ProbeCatalog 7 项、Sanitizer 8 项、InfoRowResolver 5 项、HomeViewModel 9 项）。
- 未执行：`connectedAndroidTest`（需要真机/模拟器），Compose UI 测试源码已提供。

## 声明

1. **项目性质**：本项目为个人技术练习与作品展示，非商业用途，不代表任何官方立场。
2. **版权与许可**：本项目代码采用 **[CC-BY-NC-SA-4.0 License](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh)** 开源。

- <div>
  <a href="https://github.com/ldy2330785100/Lidawn-PermApp/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/license-CC--BY--NC--SA--4.0-brightgreen" alt="License: CC-BY-NC-SA-4.0">
  </a>
   </div>

3. **责任限制**：作者不对因使用本项目代码或安装包导致的任何损失承担责任。
4. **隐私说明**：所有探测均在本机完成，不收集也不上传任何用户数据；请求历史等状态仅保存在本机 DataStore。
5. **权限用途**：应用声明了大量高敏感权限用于探测演示，其中多数仅用于读取真实授权状态，不会主动滥用。

## 🙏 特别鸣谢

- [GitHub](https://github.com) 提供仓库与 Pages 托管服务

- <div>
    <a href="https://github.com/ldy2330785100/Lidawn-PermApp">
    <img src="https://img.shields.io/github/repo-size/ldy2330785100/Lidawn-PermApp" alt="Repo size">
  </a>
  </div>

- [DeepSeek](https://platform.deepseek.com) 提供技术建议与代码优化
- [RikkaHub](https://rikka-ai.com)提供高效的开发环境


## 💬 讨论群聊

- [Telegram](https://t.me/WT_Permission)

---

<div align="center">
  &copy; 2026 亦冬链迹 版权所有
</div>
