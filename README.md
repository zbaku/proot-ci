# proot CI/CD

自动构建 proot 静态二进制 ARM64 glibc 版本，针对 Android 高版本优化。

## 功能

- 每月 1 号自动检测上游 [termux/proot](https://github.com/termux/proot) 是否有新版本
- 使用 Ubuntu ports 交叉编译生成 ARM64 (aarch64) 静态二进制
- 完整验证报告：架构、加固、链接方式、系统调用兼容性、Android 兼容性
- 自动发布到 GitHub Releases
- 支持手动触发构建

## 下载预编译版本

前往 [Releases](https://github.com/zbaku/proot-ci/releases) 页面下载最新版本。

## 版本命名

格式: `v2.X.X-{commit_sha}`

例如: `v2.3.0-a1b2c3d`

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
| 链接方式 | 静态链接 |
| 构建系统 | GNU Makefile |
| 交叉编译 | gcc-aarch64-linux-gnu (Ubuntu ports) |
| 上游项目 | [termux/proot](https://github.com/termux/proot) |

## 验证项目

| # | 检查项 | 说明 |
|---|--------|------|
| 1 | ELF 架构验证 | ARM64 (AArch64) |
| 2 | 编译加固检查 | PIE、RELRO、NX |
| 3 | 静态链接检查 | 无动态库依赖 |
| 4 | ptrace 系统调用 | PTRACE_TRACEME 支持 |
| 5 | chroot 模拟 | 根目录仿真 |
| 6 | 目录挂载模拟 | /proc、/sys、/dev、/system |
| 7 | 用户权限模拟 | UID/GID 仿真 |
| 8 | Android 读权限 | /proc、/system、/vendor、/data |
| 9 | Seccomp 过滤 | Android 14-16 兼容性 |
| 10 | Termux 仿真 | HOME 路径、/proc/self |
| 11 | 系统调用 blocklist | openat2 等受限调用处理 |
| 12 | Bionic 链接器 | Android 动态链接兼容 |
| 13 | PTY 终端支持 | 交互式 shell |
| 14 | Overlay/Bind 挂载 | 存储层虚拟化 |
| 15 | 运行参数完整性 | 支持的启动参数 |
| 16 | 特性检测摘要 | 编译特性汇总 |

## 使用方法

```bash
# 下载解压
wget https://github.com/zbaku/proot-ci/releases/download/v2.3.0-xxxxxxx/proot-v2.3.0-xxxxxxx-arm64-linux-glibc.tar.gz
tar -xzf proot-*.tar.gz

# 运行
./proot -h

# 常用示例
./proot -r /path/to/rootfs /bin/ls /

# 挂载目录
./proot -m /proc:/proc -m /sys:/sys -m /dev:/dev -r /path/to/rootfs /bin/sh

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

## 本地构建（交叉编译 ARM64）

### 环境要求

- Ubuntu 22.04+ x86_64
- 安装 ARM64 交叉编译工具链

### 构建步骤

```bash
# 安装交叉编译工具链
sudo dpkg --add-architecture arm64
sudo apt-get update
sudo apt-get install -y \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    libc6-dev-arm64-cross \
    libtalloc-dev:arm64 \
    libarchive-dev:arm64 \
    libseccomp-dev:arm64

# 克隆源码
git clone https://github.com/termux/proot.git
cd proot

# 交叉编译
make -C src proot GIT=false \
    CROSS_COMPILE=aarch64-linux-gnu- \
    CC=aarch64-linux-gnu-gcc \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG=""

# 输出: src/proot
```

## CI/CD 工作流

```
每月1号自动检测上游
       │
       ▼
  有新版本？ ──否──→ 结束
       │
      是
       │
       ▼
  克隆 termux/proot 最新源码
       │
       ▼
  Ubuntu ports 交叉编译 ARM64
       │
       ▼
  18 项验证测试
       │
       ▼
  发布到 GitHub Releases（含验证报告）
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

## License

继承 termux/proot 项目 LICENSE (GPL-2.0)
