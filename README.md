# 考研工作台（KaoYan Workbench）

一个纯本地运行的考研备考工作台 Android 应用：把**倒计时、任务计划、刷题、错题本、专注计时、学习统计**收在一个 App 里，接 DeepSeek 做 AI 助手与学习分析。

数据全部保存在设备本地（`localStorage`），不经过任何自建服务器。

---

## 功能特性

### 首页
- 距离初试倒计时、目标院校与目标分数展示
- 今日任务清单、各科目进度、今日 / 本周学习时长统计
- 玻璃拟态卡片样式，支持长按卡片单独更换背景图

### 计划
- 任务增删改查，支持优先级（高 / 中 / 低）、截止日期、科目归属、标签
- 截止日智能提示：`逾期 N 天` / `今天截止` / `明天截止` / `N 天后截止`

### 刷题
- 题库管理，支持**选择题**与**填空题**两种题型
- 抽题设置：按科目筛选、随机 / 顺序、题量可选
- 答题后给出答案与解析，记录每次作答（`quizLogs`），统计正确率
- 真题记录、错题本、模考记录三套独立台账

### 记录
- 专注计时器：环形进度 + SVG 渐变，默认 45 分钟专注 / 10 分钟短休 / 30 分钟长休 / 4 个循环，参数可改
- 专注期间调用原生接口保持屏幕常亮，阶段结束触发震动提醒
- 学习时长饼图与明细列表，支持编辑、补录历史记录

### 更多
- **AI 助手**：对接 DeepSeek，支持 `deepseek-chat` 与 `deepseek-reasoner` 两个模型
- **学习情况分析**：把任务、错题、刷题记录汇总后交给 AI 生成学习建议
- 科目管理（自定义名称与配色）、标签管理、外观设置（背景图 / 玻璃透明度 / 虚化 / 暗化）
- 资料库、目标设置（考试日期 / 院校 / 分数）
- 数据备份：导出 JSON 到手机、从 JSON 还原、清空全部数据

---

## 技术架构

前端没有使用任何框架，是**一个 HTML 文件承载全部界面与业务逻辑**（`app/src/main/assets/index.html`，约 5700 行 / 208 KB，内含 CSS、原生 JS、SVG 图表与模板字符串渲染）。Android 侧只做一层 WebView 壳，负责承载页面、权限与原生能力桥接。

```
┌─────────────────────────────────────────┐
│  MainActivity (Java)                     │
│  ├─ WebView 加载 file:///android_asset/index.html
│  ├─ WebSettings：JS / DOM Storage / Database / 文件访问
│  ├─ WebChromeClient.onShowFileChooser → 系统文件选择器（支持多选）
│  ├─ addJavascriptInterface(WebAppInterface, "AndroidNative")
│  └─ onKeyDown：返回键回退 WebView 历史
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│  index.html（前端单页）                    │
│  ├─ 5 个 Tab：首页 / 计划 / 刷题 / 记录 / 更多 │
│  ├─ state → localStorage('kaoyan_wb_v5') │
│  └─ fetch → https://api.deepseek.com     │
└─────────────────────────────────────────┘
```

### 原生桥接接口

页面通过 `window.AndroidNative` 调用：

| 方法 | 作用 |
|---|---|
| `keepScreenOn()` | 进入专注时保持屏幕常亮 |
| `clearScreenOn()` | 退出专注时解除常亮 |

### 关键实现细节

- `allowUniversalAccessFromFileURLs` / `allowFileAccessFromFileURLs` 已开启，用于让 `file://` 页面直接请求 `https://api.deepseek.com`（属于为本地页面调用外部 API 做的安全性权衡，仅适用于本项目的纯本地场景）。
- 状态持久化统一走 `save()`，写入 `localStorage` 键 `kaoyan_wb_v5`；`migrate()` 负责旧版本数据结构升级。
- 文件选择器通过 `startActivityForResult` + `onActivityResult` 回传，兼容单选与 `ClipData` 多选。
- 已声明权限：`INTERNET`、`VIBRATE`。

---

## 目录结构

```
kaoyangongzuotai/
├── app/
│   ├── build.gradle                    # 模块构建配置
│   ├── proguard-rules.pro
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml      # 应用名「工作台」，无 ActionBar 主题
│       │   ├── assets/
│       │   │   └── index.html           # ★ 全部界面与业务逻辑
│       │   ├── java/com/example/kaoyanworkbench/
│       │   │   └── MainActivity.java    # WebView 壳 + 原生桥接
│       │   └── res/                     # 布局、图标、颜色、字符串
│       ├── test/                        # 单元测试模板
│       └── androidTest/                 # 仪器测试模板
├── build.gradle                         # 顶层构建，含阿里云 Maven 镜像
├── settings.gradle                      # rootProject.name = kaoyanWorkbench
├── gradle.properties                    # AndroidX + Jetifier
└── gradle/wrapper/                      # Gradle 8.7（腾讯云镜像）
```

---

## 构建与运行

**环境要求**

| 项 | 版本 / 说明 |
|---|---|
| JDK | **17**（AGP 8.6.0 强制要求） |
| Gradle | 8.7（wrapper 已内置，自动下载） |
| Android Gradle Plugin | 8.6.0 |
| compileSdk / targetSdk | 30（Android 11） |
| minSdk | 21（Android 5.0） |
| buildTools | 37.0.0（需在本机 SDK 中已安装） |

**步骤**

1. 用 Android Studio 打开工程根目录，等待 Gradle 同步。
2. 工程未提交 `local.properties`（含本机 SDK 路径）。首次打开时由 Android Studio 自动生成；若命令行构建，请手动创建：

   ```properties
   sdk.dir=C\:\\Users\\<你的用户名>\\AppData\\Local\\Android\\Sdk
   ```

3. 命令行构建：

   ```bash
   # Windows
   gradlew.bat assembleDebug

   # macOS / Linux
   ./gradlew assembleDebug
   ```

4. 产物位于 `app/build/outputs/apk/debug/app-debug.apk`，可直接安装到 Android 5.0 及以上设备。

> 依赖走阿里云镜像（`maven.aliyun.com`），Gradle 发行包走腾讯云镜像，国内网络环境下无需额外代理。

---

## AI 功能配置

1. 在 [platform.deepseek.com](https://platform.deepseek.com/api_keys) 申请 API Key（形如 `sk-...`）。
2. App 内进入 **更多 → AI 助手 → 右上角设置**，填入 Key 并选择模型。
3. Key 仅保存在本机 `localStorage`，随备份文件一并导出，**请自行妥善保管备份文件**。

AI 调用产生的费用由 DeepSeek 账户自行承担。

---

## 数据与备份

- 全部数据（任务、题库、刷题记录、专注记录、外观配置、API Key）存在本机 `localStorage`，键名 `kaoyan_wb_v5`。
- 卸载 App 或清除应用数据会丢失全部内容，**换机前请先导出**。
- 导出的备份文件名为 `考研工作台_备份_YYYYMMDD_HHmm.json`；恢复时会覆盖当前数据，操作前建议先另存一份。

---

## 已知限制与注意事项

- **compileSdk / targetSdk 为 30**：可在 Android 11 及以下正常构建。若要上架 Google Play 或适配 Android 14+，需升级到 34 及以上，并同步调整 AGP 与 `namespace`。
- **`namespace` 与 `applicationId` 不一致**：`namespace 'com.example.kaoyan'`，而 `applicationId "com.example.kaoyanworkbench"`，两者均可编译，但包名统一会更规范。
- **`activity_main.xml` 为模板残留**：`MainActivity` 实际通过代码 `new WebView(this)` 构建界面，该布局文件未被引用，可删除。
- **`vibrate()` 未暴露给前端**：该方法虽带 `@JavascriptInterface`，但定义在 `MainActivity` 上，而注入到 WebView 的是内部类 `WebAppInterface` 实例，因此页面无法调用；实际可用的原生接口只有 `keepScreenOn` / `clearScreenOn`。若需要答题震动反馈，应把 `vibrate()` 移入 `WebAppInterface`。
- **单文件前端体量大**：`index.html` 约 208 KB、5700 行以上，修改时建议按区块（样式 / 图标 / 状态 / 渲染 / 交互）定位，注意模板字符串嵌套与转义。
- **未开启代码混淆**：release 构建 `minifyEnabled false`，未配置签名，仅适合本地安装使用。

---

## 许可

仓库暂未附 License 文件。如需开源发布，建议补充 MIT 或 Apache-2.0 并同步声明第三方服务（DeepSeek API）使用条款。

本项目为个人备考自用工具，与任何院校、机构无关联。
