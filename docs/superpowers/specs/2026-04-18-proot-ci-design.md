# proot CI/CD 设计文档

## 概述

自动化 GitHub Actions 工作流，每月检测 termux/proot 上游仓库是否有新版本，并构建发布 ARM64 glibc 静态二进制，针对 Android 高版本优化。

## 架构

```
上游 termux/proot
       │
       │ (每月1号检测)
       ▼
GitHub Actions Runner (ubuntu-22.04)
       │
       ├── git clone proot 最新源码
       ├── 获取 commit SHA 作为版本号
       ├── 检查是否已发布相同版本
       ├── 安装 aarch64-linux-gnu 交叉编译工具链 (Ubuntu ports)
       ├── GNU Make 交叉编译 (ARM64 + static + glibc)
       ├── 18 项验证测试
       ├── 生成验证报告
       └── softprops/action-gh-release 发布
                 │
                 ▼
          GitHub Releases (含验证报告)
```

## 构建配置

| 项目 | 值 |
|------|-----|
| 目标架构 | ARM64 (aarch64) |
| C 库 | glibc (非 musl) |
| 链接方式 | 静态链接 |
| 构建方式 | 交叉编译 (x86_64 → ARM64) |
| 构建系统 | GNU Make |
| Runner | ubuntu-22.04 |
| 上游项目 | termux/proot (Android 优化版) |

## CI 配置

| 项目 | 值 |
|------|-----|
| 触发方式 | schedule (每月1号) + workflow_dispatch (手动) |
| Cron | `0 0 1 * *` |
| 版本号格式 | `v2.X.X-{sha}` (如 `v2.3.0-a1b2c3d`) |
| 去重机制 | 构建前检查 Release 标签是否存在 |

## 验证清单

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | ELF 架构验证 | Machine: AArch64, Class: ELF64 |
| 2 | 编译加固检查 | PIE、RELRO、NX |
| 3 | 静态链接检查 | ldd 输出 "statically linked" |
| 4 | ptrace 系统调用 | PTRACE_TRACEME 支持 |
| 5 | 文件权限 | 可执行权限 |
| 6 | 二进制大小 | > 1MB |
| 7 | chroot 模拟 | -r 参数 |
| 8 | 目录挂载模拟 | -m /proc:/proc 等 |
| 9 | 用户权限模拟 | -0 -i 0:0 |
| 10 | Android 读权限 | /proc、/system、/vendor |
| 11 | Seccomp 过滤 | Android 14-16 兼容性 |
| 12 | Termux/Linux 仿真 | HOME 路径、/proc/self |
| 13 | 系统调用 blocklist | openat2 等 |
| 14 | Bionic 链接器 | Android 动态链接兼容 |
| 15 | PTY 终端支持 | 交互式 shell |
| 16 | Overlay/Bind 挂载 | 存储层虚拟化 |
| 17 | 运行参数完整性 | --help |
| 18 | 特性检测摘要 | 编译特性 |

## Android 版本兼容性

- Android 14 (API 34)：完整支持
- Android 15 (API 35)：完整支持
- Android 16 (API 36)：完整支持

Termux proot 针对高版本 Android 限制进行了专项优化。

## 目录结构

```
proot-ci/
├── .github/
│   └── workflows/
│       └── build.yml
└── README.md
```

## 依赖包

- gcc-aarch64-linux-gnu
- g++-aarch64-linux-gnu
- libc6-dev-arm64-cross
- libtalloc-dev:arm64
- libarchive-dev:arm64
- libseccomp-dev:arm64

## 版本管理

- 版本格式: `v2.X.X-{short_sha}`
- 检查机制: API 查询 `GET /repos/{owner}/{repo}/releases/tags/{tag}`
- 保留策略: GitHub 默认保留所有 Release
