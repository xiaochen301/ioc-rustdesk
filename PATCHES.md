# IOC-RustDesk 技术债台账

本项目 = rustdesk/rustdesk 1.4.9 的一次性定制二开（政务网专用版）。**不是持续迭代的敏捷项目**，不存在"迭代期→清偿期"的循环；本文件的作用是登记全部改动，方便日后上游升级时对照检查，以及回滚时定位。

基线：`rustdesk/rustdesk` tag `1.4.9`（commit `6c578292e8ebbbec708b76986ba8c4bc7c509747`）
分支：`ioc-gov-build`
构建：`.github/workflows/ioc-build.yml`（Windows x64 exe + Linux x86_64 deb）

---

## 1. 服务器与公钥（需求 2）

| ID | 位置 | 改动 | 备注 |
|---|---|---|---|
| IOC-001 | `libs/hbb_common/src/config.rs:120` | `RENDEZVOUS_SERVERS`：`["rs-ny.rustdesk.com"]` → `["10.211.0.10"]` | hbbs ID 注册 / 中继 / 在线查询的默认落点。标准端口 21116/21117 无需改动 |
| IOC-002 | `libs/hbb_common/src/config.rs:121` | `RS_PUB_KEY`：官方 ed25519 公钥 → 自建 hbbs 公钥 | **key 不匹配会导致客户端一直"未就绪"**，是本项目最敏感的常量 |
| IOC-003 | `src/common.rs:1083` | `get_api_server_()` 兜底：`https://admin.rustdesk.com` → `http://10.211.0.10:21114` | 登录 / 地址簿 / 审计 / 心跳的 API 落点 |

**上游升级影响**：若目标 RustDesk 版本把 `RENDEZVOUS_SERVERS` 改为运行时注入而非编译期常量，需改为构建期注入。

## 2. 数据上报（需求 1）

| ID | 位置 | 改动 | 备注 |
|---|---|---|---|
| IOC-004 | `libs/hbb_common/src/lib.rs:496` | `version_check_request()` 的 URL 常量置空 `""` | 唯一一处硬编码的官方版本检查端点。空 URL 使所有调用方立即失败，不产生任何外发 |
| IOC-005 | `src/common.rs:2369-2377` | `STUNS_V4` / `STUNS_V6` 三个公共 STUN 全部停用：`test_nat_ipv4()` 改为直接 `bail!()`，`test_ipv6()` 改为直接 `return None` | Google / Cloudflare / Nextcloud。**不删常量、只断调用**，便于回滚。`test_ipv6()` 返回 `None` 是安全契约（调用方 `get_ipv6_socket()` 随之返回 `None`），无 panic 风险 |
| IOC-006 | `flutter/lib/main.dart:380,487,521` | 无需改动 | 官方自己已把 Firebase Analytics 全部注释掉，本项目确认无实际上报。**核实结论，非改动** |

**重要机制说明（不是补丁，是上游既有行为）**：
`src/common.rs:1087` `is_public()` 判定 URL 是否 `rustdesk.com` 系，命中则**主动关闭**心跳上报（`src/hbbs_http/sync.rs:286`）和审计上报（`src/common.rs:1122`）。把 API 指向内网后 `is_public()` 返回 false，这两个上报**反而会被激活**——这是自建服务器的功能（设备在线状态、审计日志），政务网需要，故保留。需求 1 的"去除上报"因此不等于"关掉 API"，两者方向相反。

## 3. 去除官网入口（用户补充决策：全部去掉）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-007 | `flutter/lib/common.dart:3739` | `loadPowered()` 首行 `return SizedBox.shrink()`，移除 "powered by" 徽章（点击原本跳转 rustdesk.com） |
| IOC-008 | `flutter/lib/desktop/pages/desktop_setting_page.dart:2449` | 删除"隐私声明"和"网站"两个 InkWell 行（`privacy.html` / 官网首页） |

**未处理项（如实记录）**：`desktop_home_page.dart:537-555` 的 SELinux / Wayland 警告卡片仍带 `link:` 指向 rustdesk.com 文档站。这些是 Linux 平台问题提示卡，只有在 SELinux enforcing 或 Wayland 登录场景下才出现，非政务网常规路径。**如需彻底清除请告知**，改动方式同 IOC-008。

## 4. 去除检查更新（需求 6）

| ID | 位置 | 改动 | 层次 |
|---|---|---|---|
| IOC-009 | `libs/hbb_common/src/lib.rs:496` | 同 IOC-004，URL 置空 | 端点层 |
| IOC-010 | `src/common.rs:941` | `check_software_update()` 首行 `return`（原本靠 `is_custom_client()` 短路，现显式短路） | 调度层 |
| IOC-011 | `src/updater.rs:37` | `manually_check_update()` 改为直接 `Ok(())`，不再往 `TX_MSG` 投 `CheckUpdate` 消息 | 调度层 |
| IOC-012 | `flutter/lib/desktop/pages/desktop_home_page.dart:432` | `buildHelpCards()` 首个分支条件改为 `if (true)`，整个"新版本可用"卡片不出现（连带 `rustdesk.com/download` 和 GitHub release 链接） | UI 层 |
| IOC-013 | `flutter/lib/desktop/pages/desktop_setting_page.dart:551` | **无需改动**：`!isCustomClient()` 包裹，APP_NAME 改为 IOC-RustDesk 后自动消失。**核实结论** |

**设计意图**：APP_NAME 改名后 `is_custom_client()` 已为 true，官方本就会短路更新检查（`src/common.rs:942`、updater 里的 msi 逻辑等 8 处）。IOC-010/011/012 是**第二道显式防线**——即使日后有人改回 APP_NAME="RustDesk"，更新检查仍然关闭。删除整个 updater 线程会牵连自动更新重试逻辑，属于过度改动，故按"三重防护 + 保留原逻辑"处理。

## 5. 改名 IOC-RustDesk（需求 3）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-014 | `libs/hbb_common/src/config.rs:72` | `APP_NAME`：`"RustDesk"` → `"IOC-RustDesk"`（**核心**，官方原生注入点） |
| IOC-015 | `build.py:296` | deb `Package: rustdesk` → `ioc-rustdesk`；`Maintainer`/`Description` 改写 |
| IOC-016 | `build.py:360,364,397,401` | deb 中间产物名与最终文件名 → `ioc-rustdesk-{version}.deb` |
| IOC-017 | `res/rustdesk.desktop` | `Name=RustDesk`→`IOC-RustDesk`，`Comment`→`IOC-政务网专用版` |
| IOC-018 | `res/rustdesk-link.desktop` | `Name` → `IOC-RustDesk` |
| IOC-019 | `flutter/windows/runner/Runner.rc:92,93,96,98` | `CompanyName`→`IOC`，`FileDescription`→`IOC-RustDesk Remote Desktop`，`LegalCopyright`→`Government network edition.`，`ProductName`→`IOC-RustDesk` |

**IOC-014 的连锁效果**（官方设计，无需额外代码）：
- `is_custom_client()` 变 true → 检查更新短路、设置页相关项消失
- `src/lang.rs:225` 界面所有 "RustDesk" 文案自动替换为 `IOC-RustDesk`
- 配置目录 → `~/.config/IOC-RustDesk`，与官方客户端完全隔离，不串号
- URL 协议 → `ioc-rustdesk://`
- Linux IPC socket → `/tmp/IOC-RustDesk/`（`libs/hbb_common/src/config.rs:870`）
- 托盘图标 tooltip、窗口标题 → `IOC-RustDesk`

**已决策不改的项：可执行文件名保持 `rustdesk` / `rustdesk.exe`。**
理由：改二进制名会踩三处上游硬编码，风险高收益低——
1. `src/core_main.rs:399` `pkill -f "{app_name().to_lowercase()} --tray"`：改了就与实际进程名不符，托盘无法互杀，会残留多进程
2. `libs/portable/src/bin_reader.rs:76,82` 自解压包的 `"rustdesk"` 魔数双向匹配；`libs/portable/generate.py:42,57` 写入魔数
3. `src/privacy_mode/win_topmost_window.rs:33` `WIN_TOPMOST_INJECTED_PROCESS_EXE = "RuntimeBroker_rustdesk.exe"`，以及 `libs/portable/src/main.rs:235` 的 taskkill `/IM`

最终形态：包名 `ioc-rustdesk`、界面/托盘/标题 `IOC-RustDesk`、二进制 `rustdesk`。
**如需连二进制一并改名，是独立的一次改动，需同步上述 5 处，请单独提需求。**

## 6. 标语（需求 5）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-020 | `flutter/lib/consts.dart:43` | 新增 `const String kGovEditionSlogan = 'IOC-政务网专用版'` |
| IOC-021 | `flutter/lib/desktop/widgets/tabbar_widget.dart` | `DesktopTab` 新增 `showSlogan` 字段（默认 `true`）；`_buildBar()` 的返回值从 `Row` 改为局部变量 `bar`，再包一层 `Stack(alignment: center)` + `Positioned.fill` + `IgnorePointer` + `Center` 承载标语 |

**设计要点**：
- 用 `IgnorePointer` 覆盖而非插入 `Row` 布局，保证不吞掉 `bar` 内部的 `GestureDetector` 拖拽移动窗口手势
- `maxLines:1` + `overflow: clip` + `softWrap:false`，标语在窄窗口下裁切而非换行撑破标题栏
- 字体 13 / w600，取 `MyTheme.tabbar(context).selectedTextColor` 跟随明暗主题
- 垂直方向 `Stack` + `Center` 天然与右侧四键（设置/最小化/最大化/关闭）同高齐平

**未决策项**：标语对所有 `DesktopTab` 生效（主界面、远程控制页、文件传输页等 8 处调用 `DesktopTab(...)`）。用户需求只指定主界面。如需限定为仅主窗口，在 `desktop_tab_page.dart:97` 那处传 `showSlogan: true`，其余 7 处传 `false`。

---

## 欠账与风险

| # | 项 | 说明 |
|---|---|---|
| R-1 | **连通性未验证** | 本机在 `192.168.33.102`，ping `10.211.0.10` 100% 丢包，不在政务网段。编译通过后仍需在目标网段实测 hbbs 握手 |
| R-2 | **编译未验证** | 见 PR 交付时的验证记录。Windows / Linux 两条链路均需 GitHub Actions 实跑确认 |
| R-3 | 文档站链接残留 | 见 IOC-007 下方"未处理项" |
| R-4 | 标语作用域 | 见 IOC-021 下方"未决策项" |
| R-5 | 上游升级 | 长期停在 1.4.9。`cargo` 依赖 `libs/hbb_common` submodule，升级时需同步子模块并复查 IOC-001~006 的常量位置 |

## 编译期未使用的代码说明

`src/common.rs` 中 `STUNS_V4`/`STUNS_V6` 常量、`stun_ipv4_test`/`stun_ipv6_test` 函数、`test_bind_ipv6` 在停用后成为死代码。**保留不删**：删除会产生大面积 diff、增加与上游合并的冲突面，且一旦日后需要恢复 STUN 只要改回两行调用。已用 `#[allow(unreachable_code)]` 抑制警告（仅 `test_ipv6` 因是 `if` 链需要；`test_nat_ipv4` 整体替换为 `bail!`，无残留）。
