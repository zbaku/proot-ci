# proot CI/CD

手动构建 proot 静态二进制 ARM64 glibc 版本，针对 Android 高版本优化。

## 功能

- **手动触发构建** - 通过 GitHub Actions workflow_dispatch 手动构建
- 使用 Ubuntu ARM64 原生编译生成 ARM64 (aarch64) 静态二进制
- 完整验证报告：架构、加固、链接方式、系统调用兼容性
- 自动发布到 GitHub Releases

## 下载预编译版本

前往 [Releases](https://github.com/zbaku/proot-ci/releases) 页面下载最新版本。

## 版本命名

格式: `{tag}-{commit_sha}`

例如: `v0.0.0-ab2e346`

## 构建产物

| 文件名 | 说明 |
|--------|------|
| `proot-{version}-arm64-linux-glibc.tar.gz` | ARM64 静态二进制压缩包 |
| `verification-report-{version}.txt` | 完整验证报告 |

## 技术规格

| 项目 | 值 |
|------|-----|
| 架构 | ARM64 (aarch64) |
| C 库 | glibc |
| 链接方式 | **完全静态链接** (无任何 .so 依赖) |
| 构建系统 | GNU Makefile |
| 构建环境 | ubuntu-24.04-arm 原生 |
| 上游项目 | [termux/proot](https://github.com/termux/proot) |

## 验证项目

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | ELF 架构验证 | ARM64 (AArch64) |
| 2 | 文件类型 | EXEC (可执行文件) |
| 3 | 链接方式 | **完全静态链接** (statically linked) |
| 4 | 动态节检查 | 无动态节 (no dynamic section) |
| 5 | 二进制大小 | > 100KB |

## 使用方法

### 下载解压

```bash
wget https://github.com/zbaku/proot-ci/releases/download/v0.0.0-ab2e346/proot-v0.0.0-ab2e346-arm64-linux-glibc.tar.gz
tar -xzf proot-*.tar.gz
```

### 运行

```bash
./proot -h
```

### 常用示例

```bash
# 基本运行
./proot /bin/ls /

# 挂载目录
./proot -m /proc:/proc -m /sys:/sys -m /dev:/dev /bin/sh

# 模拟 root
./proot -0 -i 0:0 /bin/sh
```

## Android 使用场景

```bash
# 在 Android 上使用 proot 运行 Linux 环境
./proot -r ~/ubuntu-rootfs /bin/bash

# 挂载 Android 目录
./proot -m /system:/system -m /vendor:/vendor -r ~/rootfs /bin/ls /system

# Termux 环境兼容
./proot -w ~/home echo $HOME
./proot -r ~/debian /bin/bash
```

## 本地构建

### 环境要求

- Ubuntu 24.04 ARM64 (aarch64)
- 或 ARM64 架构的设备/容器

### 构建步骤

```bash
# 克隆源码
git clone https://github.com/zbaku/proot-ci.git
cd proot-ci

# 手动触发 workflow
gh workflow run build.yml

# 或在 GitHub Actions 页面手动触发
```

## CI/CD 工作流

```
手动触发 (workflow_dispatch)
       │
       ▼
  克隆 termux/proot 最新源码
       │
       ▼
  安装 glibc 静态库 + talloc 源码
       │
       ▼
  创建 Android ashmem stub header
       │
       ▼
  编译静态 talloc 库
       │
       ▼
  静态编译 proot (-static)
       │
       ▼
  验证 (架构/静态链接/大小)
       │
       ▼
  发布到 GitHub Releases
```

## Android 版本兼容性

- **Android 14 (API 34)**：完整支持
- **Android 15 (API 35)**：完整支持
- **Android 16 (API 36)**：完整支持

Termux proot 针对高版本 Android 限制进行了专项优化：

- Seccomp 过滤绕过
- openat2 等封杀系统调用的模拟
- ptrace 模拟增强
- /proc/[pid] 读取权限修复
- ashmem/memfd 模拟
- SysV IPC 共享内存支持

## 技术细节

### 完全静态链接

本项目构建的 proot 二进制是完全静态链接的：

- ✅ 无 `libtalloc.so` 依赖（已静态链接）
- ✅ 无 `libc.so` 外部依赖（glibc 静态链接）
- ✅ 无 `/lib/ld-linux-aarch64.so` 动态链接器依赖

### 构建参数

```bash
LDFLAGS="-static -L/usr/lib/aarch64-linux-gnu -L/usr/local/lib -ltalloc"
```

## License

继承 termux/proot 项目 LICENSE (GPL-2.0)
