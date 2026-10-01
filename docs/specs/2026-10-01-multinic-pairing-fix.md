# 多网卡直连配对故障与正式版迁移记录

## 状态

2026-10-01：用户确认 `1.0.0-alpha.17` 自定义修复包在原现场测试通过。
本记录保留故障背景和验证依据；正式版 `v1.0.1` 迁移及两端打包已完成。
新版双机实测尚未执行，不能将旧包的结果视作新版已经通过。

## 故障场景

- 一台 Windows 与一台 macOS 通过网线直连，直连子网为 `10.172.6.0/24`。
- 每台设备同时保留各自的上网连接，因此两端均为多网卡环境。
- 邀请码配对失败，交换邀请发起方也不能解决。
- 2026-09-24 提供过 Engine、daemon、GUI 调试日志用于诊断；原始日志不纳入公开仓库。

## 根因和修复

问题不在直连子网编号。配对 mDNS 将本机 IP 填入公告，却未显式配置全部 IPv4
组播接口；系统可能只从默认上网接口发送公告或查询，直连接口因此无法完成发现。

修复属于 Infra 的接口配置缺陷，发布端与解析端共享一个接口快照模块，统一筛选
合格地址，并通过 `with_multicast_interfaces_v4` 明确指定 IPv4 收发接口。
旧的发布端独立枚举实现被替换，不新增另一套配对流程。
协议、认证与持久化格式保持原设计。长期技术说明由
[Engine 接口选择文档](https://github.com/lolinyaanyaamoe/Engine/blob/fix/v1.0.1-multinic-mdns/docs/design-docs/pairing-mdns-interfaces.md)
维护。

## 已验证的历史版本

| 项目 | 来源或结果 |
| --- | --- |
| 原 Engine 修复 | `89236be6ff05e884ae571cbaba009799aef0e30b` |
| 原桌面构建提交 | `9d438404`，分支 `build/multinic-mdns-fix` |
| 交付版本 | `1.0.0-alpha.17`，macOS Apple Silicon DMG、Windows x64 安装版和便携版 |
| Windows 构建 | [Actions 36221075585](https://github.com/lolinyaanyaamoe/UniClipboard/actions/runs/36221075585)，成功 |
| 双机实测 | 2026-10-01 用户反馈“测试通过了，这个问题确实修复了” |

该反馈确认原报告场景已解决；未记录具体测试轮数、每种内容类型、长期断线重连、
角色交换或 IPv6 专项结果，不将这些项目补记为通过。

## 正式版迁移

上游已于 2026-09-28 发布 [v1.0.0](https://github.com/UniClipboard/UniClipboard/releases/tag/v1.0.0)，
随后于 2026-09-29 发布 [v1.0.1](https://github.com/UniClipboard/UniClipboard/releases/tag/v1.0.1)。
本次采用 2026-10-01 查询到的最新正式版 `v1.0.1`。

- 桌面基线：上游 `v1.0.1` 标签。
- Engine 基线：该标签锁定的 `3f3eef7450e06013c3178b716eee9d5a3c419349`。
- Engine 修复分支：`fix/v1.0.1-multinic-mdns`。
- Engine 修复提交：`5b7d59c0073f45413bd608f1c08f71a72c4968da`。
- 桌面构建分支：`build/v1.0.1-multinic-fix`。
- 两端编译源码提交：`e216a1e2374e634b9992086c02db2fd1bb7e2d7c`。
- Engine 依赖锁定自有 fork 的完整提交 SHA，`Cargo.lock` 同步更新。
- 保留正式版已有错误链及隐私改进；接口枚举失败仅记录固定错误类别。
- 使用 release 优化配置；自定义包不含官方签名，关闭 updater 产物生成。
- macOS 重新封装时使用 `signingIdentity: "-"`，对应用和 daemon 作 ad-hoc 签名，未作 Apple 公证。
- 用户明确要求不使用本机开发者账号。ad-hoc 签名不使用 Apple 账号、证书或私钥，
  不登录 Apple、不提交公证、不修改钥匙串或其他应用的签名设置；后续打包继续遵守此约束。
- 安装包沿用 `1.0.1` 版本号和应用标识，交付文件名加 `multinic-fix` 以区分。
- 自动更新仍指向官方渠道；升级到尚未包含该修复的上游版本可能重新出现故障。

所有修改和构建位于自有 fork，不代表上游已经接受修复或发布了这些自定义包。

## 新版验证与复测

已通过 4 项 mDNS 回归测试、Engine 全工作区/全 target 编译检查、格式/架构/隐私检查，
以及桌面依赖锁定和消费方边界检查。消费方检查脚本的固定来源同步为自有 fork，
仍要求全部核心包来自同一完整提交，不接受本地路径、标签漂移或 GUI 链接 Engine。
原脚本硬编码上游地址，因此直接用于 fork 时曾报来源检查失败；调整来源后正反例均通过。

macOS 已完成 release 构建、前端类型及兼容性检查、DMG 校验、镜像内应用深度签名校验。
包内主程序及 daemon 均为 ARM64，应用版本 `1.0.1`、最低系统版本 `12.5`。
初次未签名封装不能通过应用签名完整性检查，交付文件已替换为重新 ad-hoc 签名的版本。

Windows x64 构建记录：[Actions 36815170063](https://github.com/lolinyaanyaamoe/UniClipboard/actions/runs/36815170063)。
构建成功，用时 31 分 33 秒；安装版为 NSIS 包，便携 ZIP 的 CRC 检查通过，包含主程序、
daemon、便携标记和说明文件，两个可执行文件均为 x86-64。未执行 Windows 现场安装和新版双机配对。
当前新版实机矩阵为：

| 验证项 | 状态 |
| --- | --- |
| macOS Apple Silicon 与 Windows x64 邀请码配对 | 待用户复测 |
| 交换邀请发起方后配对 | 待用户复测 |
| 双向文本复制同步 | 待用户复测 |
| 保留两端各自上网连接时重启应用后同步 | 待用户复测 |

复测时使用两端新版程序，完整退出旧 GUI 与后台 daemon 后再启动。
已有配对先检查同步；若需要验证首次配对路径，使用可丢弃的测试配置或设备进行，
不要为测试删除唯一的正式历史资料。防火墙应允许应用访问直连网络。

## 交付文件与校验

构建源码为 `e216a1e2374e634b9992086c02db2fd1bb7e2d7c`；后续提交只补充本文及
fork 来源检查脚本，不改变已编译的产品源码、依赖或资源。

| 文件 | 字节数 | SHA-256 |
| --- | --- | --- |
| `UniClipboard_1.0.1_multinic-fix_aarch64.dmg` | 26391084 | `81e3aba75936602d5c3961e73c5a75aa085308766360add7c297a6590d4a24d8` |
| `UniClipboard_1.0.1_multinic-fix_x64-setup.exe` | 20556782 | `718881a4633894a1be7fcb08ae950676b130a360f64738b6b9d698fa89a5223a` |
| `UniClipboard_1.0.1_multinic-fix_x64-portable.zip` | 26037932 | `70d065e6323778ec6dcfad17bf44917f9406651c76184b207b9cdb24a9f5f729` |

以上是正式版源码加自定义修复的构建，并非上游签名发布包。

## 复现打包

在本构建分支根目录，使用 `rust-toolchain.toml` 指定的 Rust 和锁定的前端依赖。
Node.js 应满足当前 Vite 的要求（20.19+ 或 22.12+；CI 使用 LTS）：

```bash
bun install --frozen-lockfile
export APP_ENV=production VITE_APP_ENV=production VITE_APP_VERSION=1.0.1
node scripts/prepare-daemon-sidecar.mjs --timings
bun run tauri build --bundles app,dmg --config '{"bundle":{"macOS":{"signingIdentity":"-"}}}' -- --locked --timings
```

macOS 本地环境额外指定已安装的 `DEVELOPER_DIR`、`SDKROOT` 与
`MACOSX_DEPLOYMENT_TARGET=12.5`；使用有效 Command Line Tools SDK，无需改变系统全局设置。
若 Rust 编译已完成而只需修复封装，可以运行：

```bash
bun run tauri bundle --bundles app,dmg --config '{"bundle":{"macOS":{"signingIdentity":"-"}}}' --ci
```

Windows 使用 fork 的 `build.yml`，输入 `platform=windows-x86_64`、`channel=stable`、
`build_mode=release`、`save_cache=false`、`package_cli=false`。本次没有更改 release 优化级别。
