# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 [smallprogram/OpenWrtAction](https://github.com/smallprogram/OpenWrtAction) 的衍生分支，专为 **R66S (Rockchip RK3566)** 设备定制 OpenWrt/ImmortalWrt 固件。核心逻辑是通过 GitHub Actions 自动化拉取上游源码、自定义配置、编译并发布固件。

## 源码平台

当前仅编译 **immortalwrt** 分支 (`openwrt-25.12`)。平台配置在 `compile_script/platforms.sh` 中定义：

- `source_code_platforms=(immortalwrt)` — 仅编译 immortalwrt
- `immortalwrt_platforms=(R66S)` — 仅编译 R66S 设备
- 可通过修改 `platforms.sh` 中的数组变量启用 openwrt / lede 源码平台

## 目录结构

```
.
├── .github/workflows/          # GitHub Actions 工作流
│   ├── Build_Multi_Platform(V6).yml    # 主构建工作流
│   ├── Cache_Multi_Platform(V6).yml    # 缓存构建工作流
│   ├── Update_Checker.yml              # 源码更新检查器（每周一触发）
│   ├── defconfig.yml                   # 手动生成 .config 种子
│   └── ...                             # 辅助工作流
├── compile_script/             # 编译流程脚本（step01~step07）
│   ├── platforms.sh            # 平台和源码配置定义
│   ├── main_and_feeds_url.sh   # 动态解析 feeds/packages URL
│   ├── step01_make_defconfig.sh
│   ├── step02_generate_release_tag.sh
│   ├── step03_generate_git_log.sh
│   ├── step04_copy_backgroundfiles.sh
│   ├── step06_update_git_log.sh
│   ├── step07_organize_tag.sh
│   ├── update_checker.sh       # 上游源码 hash 检查
│   ├── check_lifecycle.sh      # 生命周期检查（决定是否强制重编译）
│   └── matrix_job_status.sh    # 构建矩阵状态管理
├── diy_script/                 # 自定义脚本（三阶段）
│   ├── custom_feeds_and_packages.sh    # feeds 扩展 + 自定义包列表
│   ├── immortalwrt_diy/
│   │   ├── diy-part1.sh        # 阶段1：更新 feeds 前（添加自定义 feeds 和包）
│   │   ├── diy-part2.sh        # 阶段2：更新 feeds 后（配置修改）
│   │   └── diy-part3.sh        # 阶段3：编译前（最终调整）
│   ├── openwrt_diy/            # 同上，针对 openwrt 源码
│   └── lean_diy/               # 同上，针对 lede 源码
├── config/                     # 设备 .config 文件
│   ├── immortalwrt_config/     # R66S.config 等
│   ├── leanlede_config/
│   ├── openwrt_config/
│   └── seed/                   # 种子配置（defconfig 生成用）
├── bash_script/                # 辅助脚本（cache测试、本地构建测试等）
├── git_log/                    # 上游源码 commit 日志缓存
│   ├── immortalwrt/            # 各源码平台的 commit 日志
│   ├── feeds/                  # feeds 仓库的 commit 日志
│   └── custompackages/         # 自定义包的 commit 日志
├── patches/                    # 补丁文件
├── source/                     # 上游源码引用（logo、壁纸等资源）
├── tmp/                        # 临时禁用文件（disable_* 子目录）
├── platform_function.sh        # 通用函数库（日志、Git 设置、构建函数）
├── platform_immortalwrt.sh     # immortalwrt 平台特定函数
├── platform_openwrt.sh         # openwrt 平台特定函数
├── platform_lean.sh            # lede 平台特定函数
├── wsl2op.sh                   # WSL2 本地编译入口脚本
└── library/                    # 预编译库文件（如 dns2socks）
```

## 构建流程

### GitHub Actions 工作流

1. **Update_Checker**（每周一 UTC 16:00 触发）
   - 检查上游源码是否有新 commit
   - 清理旧 releases 和 workflow runs
   - 有更新时触发 `repository_dispatch` 事件，启动主构建

2. **Build_Multi_Platform(V6)**（被 Update_Checker 触发或手动）
   - `job_init`：生成 release tag，读取平台矩阵
   - `job_source_init`：克隆上游源码，执行 diy-part1.sh
   - `job_build`：并行编译各平台固件，执行 diy-part2.sh / diy-part3.sh
   - 编译完成后上传固件到 Release / Artifact

3. **Cache_Multi_Platform(V6)**：类似但侧重缓存优化
4. **defconfig**：手动触发，用于生成/更新种子配置

### 本地编译

```bash
# 在 WSL2 环境中
git clone https://github.com/your-repo/OpenWrtAction-R66sONLY
cd OpenWrtAction-R66sONLY
bash wsl2op.sh
```

## 自定义开发

### 添加自定义包

编辑 `diy_script/custom_feeds_and_packages.sh`：

1. **feeds 扩展**：在 `repos` 数组中添加 `src-git` 条目
2. **自定义包**：在 `clone_custom_packages()` 函数中 `git clone` 需要的仓库

### 三阶段 DIY 脚本

| 阶段 | 文件 | 执行时机 | 用途 |
|------|------|----------|------|
| Part1 | `diy-part1.sh` | feeds update 之前 | 修改 feeds.conf.default，克隆自定义包 |
| Part2 | `diy-part2.sh` | feeds update 之后 | 配置修改、源码替换、补丁应用 |
| Part3 | `diy-part3.sh` | make 编译之前 | 最终调整 |

### 添加新设备

1. 在 `config/immortalwrt_config/` 下创建 `{设备名}.config`
2. 在 `compile_script/platforms.sh` 的数组中添加设备名
3. 可选：在 `diy_script/immortalwrt_diy/` 中添加设备特定配置

## 关键配置项

- **默认地址**：`10.10.0.253`
- **默认账号**：`root`
- **默认密码**：无（首次登录设置）
- **SSH**：Dropbear (port 2222) + OpenSSH (port 22)
- **默认主题**：Argon

## 预置插件

- 网络代理：PassWall / PassWall2 / OpenClash / SSR-Plus / Nikki / HomeProxy
- 网络工具：AdGuardHome / MosDNS / SmartDNS / DDNS-Go / Zerotier / Cloudflared
- 系统管理：TTYD / DiskMan / NetSpeedTest / 文件管理
- 流控：Bandix / EQoS / VnStat2 / OpenAppFilter
- 主题：Argon / Kucat / Aurora / Material3 / Alpha