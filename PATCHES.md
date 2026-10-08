# IOC-RustDesk 技术债台账

本项目 = rustdesk/rustdesk 1.5.0 的一次性定制二开（政务网专用版）。**不是持续迭代的敏捷项目**，不存在"迭代期→清偿期"的循环；本文件的作用是登记全部改动，方便日后上游升级时对照检查，以及回滚时定位。

基线：`rustdesk/rustdesk` tag `1.5.0`（commit `fada664df7a294d1d1a9ca3e7cd3637069122f17`）
前一基线：`1.4.9`（commit `6c578292e8ebbbec708b76986ba8c4bc7c509747`）——2026-10-08 完成 1.5.0 重放，见 §9。
分支：`ioc-gov-build`（主仓库）；`libs/hbb_common` = 子模块，分支 `ioc-gov`。
构建：`.github/workflows/ioc-build.yml`（Windows x64 exe + Linux x86_64 deb）。

> 位置列以「文件 + 符号名」为准。行号随上游版本漂移，仅少量给出了当前实测值；定位请按符号名搜索。

---

## 1. 服务器与公钥（需求 2）

| ID | 位置 | 改动 | 备注 |
|---|---|---|---|
| IOC-001 | `libs/hbb_common/src/config.rs` `RENDEZVOUS_SERVERS` | `["rs-ny.rustdesk.com"]` → `["10.211.0.10"]` | hbbs ID 注册 / 中继 / 在线查询的默认落点。标准端口 21116/21117 无需改动 |
| IOC-002 | `libs/hbb_common/src/config.rs` `RS_PUB_KEY` | 官方 ed25519 公钥 → 自建 hbbs 公钥 | **key 不匹配会导致客户端一直"未就绪"**，是本项目最敏感的常量 |
| IOC-003 | `src/common.rs` `get_api_server_()` 兜底 | `https://admin.rustdesk.com` → `http://10.211.0.10:21114` | 登录 / 地址簿 / 审计 / 心跳的 API 落点 |

**上游升级影响**：若未来目标版本把 `RENDEZVOUS_SERVERS` 改为运行时注入而非编译期常量，需改为构建期注入。

## 2. 数据上报（需求 1）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-004 | `libs/hbb_common/src/lib.rs` `version_check_request()` | 版本检查 URL 常量置空 `""`。唯一一处硬编码的官方版本检查端点；空 URL 使所有调用方立即失败，不产生任何外发 |
| IOC-005 | `src/common.rs` `test_ipv6()` | STUN 全停用：`test_ipv6()` 改为直接 `return None`（函数首行；后方代码用 `#[allow(unreachable_code)]` 保留以备回滚）。**1.5.0 变更**：上游删除了整套 IPv4 STUN 探测（`test_nat_ipv4`、`stun_ipv4_test`、`STUNS_V4/V6` 已不存在），原先对 `test_nat_ipv4` 的 `bail!` 改动随之退出，无需保留 |
| IOC-006 | `flutter/lib/main.dart` | 无需改动 | 

IOC-006 详情：官方自己已把 Firebase Analytics 全部注释掉，本项目确认无实际上报。**核实结论，非改动**。

**重要机制说明（不是补丁，是上游既有行为）**：
`src/common.rs` `is_public()` 判定 URL 是否 `rustdesk.com` 系，命中则**主动关闭**心跳上报和审计上报。把 API 指向内网后 `is_public()` 返回 false，这两个上报**反而会被激活**——这是自建服务器的功能（设备在线状态、审计日志），政务网需要，故保留。需求 1 的"去除上报"因此不等于"关掉 API"，两者方向相反。

## 3. 第三方服务与官网入口（用户追加决策：纯内网，第三方全部拆掉）

本节的扩展背景：第一轮只做"去除数据上报"，用户随后明确要求"所有访问第三方外部服务的功能全部拆掉"，范围扩大到运行时零外部依赖。

### 3.1 nip.io —— 最重要的一处，第一轮遗漏

| ID | 位置 | 改动 |
|---|---|---|
| IOC-022 | `libs/hbb_common/src/socket_client.rs` `ipv4_to_ipv6()` | 原本在 `!ipv4 && 是 IPv4 字面量` 时把地址改写成 `<ip>.nip.io`。**改为原样返回**，强制走纯 IPv4 中继 |
| IOC-023 | `libs/hbb_common/src/socket_client.rs` `query_nip_io()` | 原本 `lookup_host("<ip>.nip.io:<port>")`。**改为 `bail!()`**；同时给 `anyhow::{bail, Context}` 补 import |

**为什么第一轮漏掉**：这两个函数不属于"数据上报"，是 NAT 穿透的实现手段。调用链为 `create_relay` → `ipv4_to_ipv6(...)` → 改写地址 → `connect_tcp`，即**每一次中继连接**都会去公共 DNS 查一次 nip.io。第一轮只扫了 URL 字面量，而 nip.io 是**拼接出来的**（`format!("{ip}.nip.io")`），字面量扫描查不到，必须顺着 `to_socket_addrs` / `lookup_host` 的调用链反查才找得到。

### 3.2 STUN（第一轮已做，见 IOC-005；1.5.0 新增 WebRTC 面，见 §8）

### 3.3 全部厂商 URL 置空

| ID | 位置 | 改动 |
|---|---|---|
| IOC-007 | `flutter/lib/common.dart` `loadPowered()` | 首行 `return SizedBox.shrink()`，移除 "powered by" 徽章（原逻辑保留于 `// ignore: dead_code` 之后） |
| IOC-008 | `flutter/lib/desktop/pages/desktop_setting_page.dart`（关于页） | 删除"隐私声明"和"网站"两个 InkWell 行 |
| IOC-024 | `flutter/lib/desktop/pages/connection_page.dart` `onUsePublicServerGuide()` | 置空（原本点击跳 `rustdesk.com/pricing`） |
| IOC-025 | `flutter/lib/desktop/pages/desktop_home_page.dart` | 三张警告卡（SELinux / Wayland / Wayland 登录屏）的 `link` 置空（`doc_mac_permission` 等 translate 结果为空串） |
| IOC-026 | `flutter/lib/desktop/pages/desktop_home_page.dart` Help 行渲染条件 | 从 `help != null` 改为 `help != null && link != null && link.isNotEmpty`（**IOC-025 的必须配套**：不加此条件会渲染出点击后执行 `launchUrl(Uri.parse(""))` 的 Help 行） |
| IOC-027 | `src/client.rs` `SCRAP_X11_REF_URL` | 置空（X11 截屏错误弹窗的文档链接） |
| IOC-028 | `src/client.rs` `LOGIN_ERROR_MAP` | Wayland 登录错误的 `link` 置空 |
| IOC-029 | `src/lang/en.rs` | `doc_mac_permission`、`doc_fix_wayland` 置空 |
| IOC-030 | `libs/hbb_common/src/config.rs` | `LINK_DOCS_HOME`、`LINK_DOCS_X11_REQUIRED` 置空。**1.5.0 变更**：`LINK_HEADLESS_LINUX_SUPPORT`（原 github.com wiki 链接）由上游删除（常量与引用一并移除），本补丁不再涉及该常量 |
| IOC-037 | `flutter/lib/desktop/pages/install_page.dart`（安装/欢迎页） | 删除 "End-user license agreement" 行（原链接 `rustdesk.com/privacy.html`）。1.4.9 两轮遗漏此页面，1.5.0 升级复查时补上 |

### 3.4 核实为安全、未改动的项

| 位置 | 结论 |
|---|---|
| `src/plugin/manager.rs` | `raw.githubusercontent.com` 在**被注释掉的** `vec![]` 内，插件源列表实际为空，不发请求 |
| `src/common.rs` 测试区（`test_is_public` 等） | 不编入 release |
| `libs/hbb_common/src/websocket.rs` 测试区 | 不编入 release |
| `flutter/lib/main.dart` | Firebase Analytics 官方自己已全部注释掉 |

### 3.5 关于 `.gitmodules` 中的 GitHub URL（用户曾质疑）

`https://github.com/xiaochen301/hbb_common.git` **不是运行时行为**。`.gitmodules` 只在
`git submodule update --init` 时被读取，即 GitHub Actions **编译阶段**拉源码用；编译产物 deb/exe 内不含此 URL，装机后运行时不会访问 GitHub。它是构建依赖，必须指向某个 git 托管地址——改成本地路径只会让云端编译失败。

## 4. 去除检查更新（需求 6）

| ID | 位置 | 改动 | 层次 |
|---|---|---|---|
| IOC-009 | `libs/hbb_common/src/lib.rs` `version_check_request()` | 同 IOC-004，URL 置空 | 端点层 |
| IOC-010 | `src/common.rs` `check_software_update()` | 首行 `return`（原本靠 `is_custom_client()` 短路，现显式短路） | 调度层 |
| IOC-011 | `src/updater.rs` `manually_check_update()` | 改为直接 `Ok(())`，不再往 `TX_MSG` 投 `CheckUpdate` 消息（1.5.0 中自动带上 `#[allow(dead_code)]`） | 调度层 |
| IOC-012 | `flutter/lib/desktop/pages/desktop_home_page.dart` `buildHelpCards()` | 首个分支条件改为 `if (true)`，整个"新版本可用"卡片不出现（连带 `rustdesk.com/download` 和 GitHub release 链接；原逻辑保留于不可达分支） | UI 层 |
| IOC-013 | `flutter/lib/desktop/pages/desktop_setting_page.dart` | **无需改动**：`!isCustomClient()` 包裹，APP_NAME 改为 IOC-RustDesk 后自动消失。**核实结论** | |

**设计意图**：APP_NAME 改名后 `is_custom_client()` 已为 true，官方本就会短路更新检查。IOC-010/011/012 是**第二道显式防线**——即使日后有人改回 APP_NAME="RustDesk"，更新检查仍然关闭。删除整个 updater 线程会牵连自动更新重试逻辑，属于过度改动，故按"三重防护 + 保留原逻辑"处理。

## 5. 改名 IOC-RustDesk（需求 3）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-014 | `libs/hbb_common/src/config.rs` `APP_NAME` | `"RustDesk"` → `"IOC-RustDesk"`（**核心**，官方原生注入点） |
| IOC-015 | `build.py` `generate_control_file()` | deb `Package: rustdesk` → `ioc-rustdesk`；`Maintainer`/`Description` 改写 |
| IOC-016 | `build.py` deb 构建函数（两处） | dpkg-deb 中间产物名与最终文件名 → `ioc-rustdesk-{version}.deb`（1.5.0 上游新增 DRM 变体改名逻辑，已融合：源路径同步改为 ioc 名） |
| IOC-017 | `res/rustdesk.desktop` | `Name=RustDesk`→`IOC-RustDesk`，`Comment`→`IOC-政务网专用版` |
| IOC-018 | `res/rustdesk-link.desktop` | `Name` → `IOC-RustDesk` |
| IOC-019 | `flutter/windows/runner/Runner.rc` | `CompanyName`→`IOC`，`FileDescription`→`IOC-RustDesk Remote Desktop`，`LegalCopyright`→`Government network edition.`，`ProductName`→`IOC-RustDesk` |

**IOC-014 的连锁效果**（官方设计，无需额外代码）：
- `is_custom_client()` 变 true → 检查更新短路、设置页相关项消失
- 界面所有 "RustDesk" 文案自动替换为 `IOC-RustDesk`
- 配置目录 → `~/.config/IOC-RustDesk`，与官方客户端完全隔离，不串号
- URL 协议 → `ioc-rustdesk://`
- Linux IPC socket → `/tmp/IOC-RustDesk/`
- 托盘图标 tooltip、窗口标题 → `IOC-RustDesk`

**已决策不改的项：可执行文件名保持 `rustdesk` / `rustdesk.exe`。**
理由：改二进制名会踩三处上游硬编码，风险高收益低——
1. `src/core_main.rs` `pkill -f "{app_name().to_lowercase()} --tray"`：改了就与实际进程名不符，托盘无法互杀，会残留多进程
2. `libs/portable/src/bin_reader.rs` 自解压包的 `"rustdesk"` 魔数双向匹配；`libs/portable/generate.py` 写入魔数
3. `src/privacy_mode/win_topmost_window.rs` 的 `WIN_TOPMOST_INJECTED_PROCESS_EXE`，以及 `libs/portable/src/main.rs` 的 taskkill `/IM`

最终形态：包名 `ioc-rustdesk`、界面/托盘/标题 `IOC-RustDesk`、二进制 `rustdesk`。
**如需连二进制一并改名，是独立的一次改动，需同步上述 5 处，请单独提需求。**

## 6. 标语（需求 5）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-020 | `flutter/lib/consts.dart` | 新增 `const String kGovEditionSlogan = 'IOC-政务网专用版'` |
| IOC-021 | `flutter/lib/desktop/widgets/tabbar_widget.dart` | `DesktopTab` 新增 `showSlogan` 字段（默认 `true`）；`_buildBar()` 的返回值从 `Row` 改为局部变量 `bar`，再包一层 `Stack(alignment: center)` + `Positioned.fill` + `IgnorePointer` + `Center` 承载标语 |

**设计要点**：
- 用 `IgnorePointer` 覆盖而非插入 `Row` 布局，保证不吞掉 `bar` 内部的 `GestureDetector` 拖拽移动窗口手势
- `maxLines:1` + `overflow: clip` + `softWrap:false`，标语在窄窗口下裁切而非换行撑破标题栏
- 字体 13 / w600，取 `MyTheme.tabbar(context).selectedTextColor` 跟随明暗主题
- 垂直方向 `Stack` + `Center` 天然与右侧四键（设置/最小化/最大化/关闭）同高齐平

**未决策项**：标语对所有 `DesktopTab` 生效（主界面、远程控制页、文件传输页等多处调用 `DesktopTab(...)`）。用户需求只指定主界面。如需限定为仅主窗口，在 `desktop_tab_page.dart` 主界面那处传 `showSlogan: true`，其余传 `false`。

---

## 7. 三项默认配置（第三轮追加）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-031 | `libs/hbb_common/src/config.rs` `option2bool()` | 空值分支：`enable-lan-discovery` 未设置时按「拒绝」（false）——设备出厂默认不响应局域网发现广播 |
| IOC-032 | 同上 | 空值分支：`direct-server` 未设置时按「允许」（true）——出厂默认开启 IP 直接访问 |
| IOC-033 | 同上 | 空值分支：`enable-check-update` 未设置时按「关闭」（false）——与第 4 节三层关闭配套，读取层默认也为关 |
| IOC-034 | `flutter/lib/common.dart` `option2bool()` | Dart 镜像实现同步加入同款空值分支（上游注释要求 rust/dart/sciter 行为一致；sciter 不在交付物内，未改） |
| IOC-035 | `libs/hbb_common/src/config.rs` 测试模块 | 新增单测 `test_ioc_option2bool_defaults`，锁定「空值默认」与「显式值不改」两组行为 |

**1.5.0 适配（重要）**：上游把 hbb_common 的 `keys` 模块精简为"仅 hbb_common 自身引用的键"，`OPTION_ENABLE_LAN_DISCOVERY` / `OPTION_ENABLE_CHECK_UPDATE` 移出该模块（其余键迁往主仓库 `libs/base/src/config/keys.rs`）。本补丁引用的这两个常量已在本子模块重新落位（提交 `ded1e27`），字符串值 `enable-lan-discovery` / `enable-check-update` 与 libs/base 保持一致；`OPTION_DIRECT_SERVER` 本就在模块内，无变化。

**设计要点**：三项改动全部收敛在「选项从未被设置过」（空值）这一分支——不写配置文件、不覆盖任何显式设定过的值。新装机器按新默认，已改过个人设置的机器不受打扰。所有布尔读取路径（Rust `Config::get_bool_option`、Dart `mainGetBoolOption*` / `mainGetLocalBoolOption*`）都经过 `option2bool`，UI 显示与实际行为天然一致。

**LAN 发现语义**：对应上游设置项 `Deny LAN discovery`（`reverse: true` 显示），默认勾选 = 不响应发现广播（隐身）。主动扫描仅在用户手动打开发现页时触发，属显式操作，不干预。

**验证**（1.5.0 轮）：`cargo test --lib` 全量 101 通过 0 失败，含 `test_ioc_option2bool_defaults`。

---

## 8. WebRTC ICE / STUN 加固（第四轮 · 1.5.0 升级新增）

| ID | 位置 | 改动 |
|---|---|---|
| IOC-036 | `libs/hbb_common/src/webrtc.rs` `DEFAULT_ICE_SERVERS` + `parse_ice_servers()` | 默认 ICE/STUN 列表**清空**（空数组）；prepend 分支加空判卫（`!has_stun && !DEFAULT_ICE_SERVERS.is_empty()`）。STUN/TURN 需经 `ice-servers` 选项显式配置 |

**为什么必须做**：1.5.0 把 WebRTC 接入连接路径。应答端（`rendezvous_mediator` 的 `spawn_webrtc_answerer`）对**每一个收到的 offer** 都建立 ICE，且上游明确注释**不检查 enable-webrtc**——该选项属于 UI 进程的 LocalConfig，服务进程读不到（原文注释在案）。ICE 建立调用 `get_ice_servers()`，在 `ice-servers` 未配置时会 **prepend 默认公共 STUN**（cloudflare / google / antisip / nextcloud）→ 即**无需任何本端设置，被控端就会联系公共服务器**，违反"纯内网、第三方全拆"要求。

**改后语义**：ICE 回退到 host candidates（内网直连）；跨 NAT 的 WebRTC 需要 STUN/TURN 时，通过 `ice-servers` 选项配置自建服务。`default_stun_servers()` 随空列表自然返回空；测试 `test_webrtc_ice_server_list` 已同步改造为"空默认"行为。

**回滚**：恢复上游列表即可（列表原值记录在该常量注释中，源为 hbb_common `229b9045`）。

---

## 欠账与风险

| # | 项 | 说明 |
|---|---|---|
| R-1 | **连通性未验证** | 本机不在政务网段，无法 ping 通 `10.211.0.10`。编译通过后仍需在目标网段实测 hbbs 握手 |
| R-2 | **安装包双链路待验** | 1.5.0 轮：`cargo test --lib` 已在 101 通过；Windows exe / Linux deb 需 GitHub Actions 实跑确认（ioc-build.yml，本机无法本地构建） |
| R-3 | 链接残留（明示） | `flutter/lib/mobile/**`（settings/connection 页）与 `src/ui/*.tis`（sciter 旧界面）中仍有 rustdesk.com 链接：前者不在交付物（不构建移动版），后者上游已弃用不构建。与 1.4.9 口径一致，未处理 |
| R-4 | 标语作用域 | 见 §6 IOC-021 下方"未决策项" |
| R-5 | 上游升级 | 基线已从 1.4.9 升至 1.5.0（重放流程与提交映射见 §9）。今后升级沿用同一流程：双仓库（主仓库 + hbb_common）逐提交重放 → 全量 IOC 落点复查 → **上游新增外联面扫描**（本次新增 WebRTC/STUN 面即为实例）→ CI 双链路构建 |

## 9. 1.5.0 升级记录（2026-10-08）

**主仓库** `ioc-gov-build`：9 个提交重放到 `1.5.0`（`fada664d`，较 1.4.9 共 255 个上游提交）：

| 补丁 | 1.4.9 链 | 1.5.0 链 |
|---|---|---|
| IOC 主补丁（服务器/改名/标语） | `4794b27c2` | `5f831d199` |
| .gitmodules 指向 IOC fork | `789181062` | `22d2014ad` |
| 去第三方 / nip.io 配合 | `8d18ef201` | `cbf64cc20` |
| fix(ci) pwsh | `7f3a1c125` | `c450908e7` |
| fix(ci) flutter PATH | `4b48672bf` | `b01659862` |
| fix Dart 类型 | `b7d707f77` | `691389ac3` |
| fix(ci) generate.py 顺序 | `67ce35762` | `07f525b0b` |
| fix(ci) brotli | `0fe66c78e` | `e5b29ea87` |
| defaults（defaults + keys 适配） | `c02856843` | `f65331f98` |
| （新）CI 配方同步 | — | `84474df00` |
| （新）IOC-037 install 页 | — | `a6b27238f` |
| （新）submodule 加固指针 | — | `0cf5df74a` |

**hbb_common** `ioc-gov`：3 个补丁重放到 `229b9045`（较旧基共 120 个上游提交），并新增 2 个提交：

| 补丁 | 1.4.9 链 | 1.5.0 链 |
|---|---|---|
| self-hosted / 改名 | `22704b2` | `045091c` |
| nip.io / 厂商 URL | `d124e3a` | `8597ebb` |
| defaults（含 keys 适配 amend） | `38a0d89` | `ded1e27` |
| （新）WebRTC STUN 清空 | — | `fc01759` |
| （新）nat64 测试适配 | — | `d420e19` |

**升级中的关键适配**（详见各节）：
1. §2 IOC-005：上游删除 IPv4 STUN 全套，补丁收敛；IPv6 STUN 保持禁用。
2. §3 IOC-030：`LINK_HEADLESS_LINUX_SUPPORT` 随上游删除。
3. §7：keys 模块精简 → 两个常量在子模块重新落位（`ded1e27`）。
4. §8：新增 WebRTC ICE 公共 STUN 默认列表的清除（`fc01759`）。
5. CI 配方（`84474df00`）：`VERSION` → 1.5.0；`VCPKG_COMMIT_ID` 已是 1.5.0 值（`9e593bb1`，与 vcpkg.json baseline 一致）；回补 `usbmmidd_v2` / 打印机驱动段与 `dpiAware` sed（1.4.9 初次裁剪 CI 时遗漏）；删除 `libpam0g-dev`（上游 1.5.0 已移除）。
6. `flutter/pubspec.lock`：取上游 1.5.0 版本（1.4.9 轮有一处 flutter 工具自动更新的侧带改动随升级弃用）。
7. `build.py`：融合上游新增的 DRM 变体改名逻辑（改名点贯穿为 `ioc-rustdesk`）。
8. `src/socket_client.rs` 测试 `test_nat64`：适配 nip.io 禁用后的行为（`d420e19`）。
