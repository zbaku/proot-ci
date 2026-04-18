# proot CI/CD 设计文档

## 概述

自动化 GitHub Actions 工作流，每15天检测 proot-me/proot 上游仓库是否有新版本，并构建发布 ARM64 glibc 静态二进制。

## 架构

```
上游 proot-me/proot
       │
       │ (每15天检测)
       ▼
GitHub Actions Runner (ubuntu-latest)
       │
       ├── git clone proot 最新源码
       ├── 获取 commit SHA 作为版本号
       ├── 检查是否已发布相同版本
       ├── 安装 aarch64-linux-gnu 交叉编译工具链
       ├── meson + ninja 交叉编译 (ARM64 + static + glibc)
       ├── 打包 + strip 优化
       └── softprops/action-gh-release 发布
                 │
                 ▼
          GitHub Releases
```

## 构建配置

| 项目 | 值 |
|------|-----|
| 目标架构 | ARM64 (aarch64) |
| C 库 | glibc (非 musl) |
| 链接方式 | 静态链接 |
| 构建方式 | 交叉编译 (x64 → ARM64) |
| 构建系统 | Meson + Ninja |
| Runner | ubuntu-latest |

## CI 配置

| 项目 | 值 |
|------|-----|
| 触发方式 | schedule (每15天) + workflow_dispatch (手动) |
| Cron | `0 0 */15 * *` |
| 版本号格式 | `v0.0.0-{sha}` (如 `v0.0.0-a1b2c3d`) |
| 去重机制 | 构建前检查 Release 标签是否存在 |

## 目录结构

```
proot-ci/
├── .github/
│   └── workflows/
│       └── build.yml
├── meson/
│   └── cross-aarch64.txt
└── README.md
```

## 依赖包

- gcc-aarch64-linux-gnu
- g++-aarch64-linux-gnu
- libc6-dev-arm64-cross
- pkg-config-aarch64-linux-gnu

## 版本管理

- 版本格式: `v0.0.0-{short_sha}`
- 检查机制: API 查询 `GET /repos/{owner}/{repo}/releases/tags/{tag}`
- 保留策略: GitHub 默认保留所有 Release
