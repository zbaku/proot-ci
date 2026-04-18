# proot Android Patching Plan

## 概述

将 Termux/proot 的 Android 补丁精准移植到 proot-me v5.4.0，在 ubuntu-24.04-arm 原生环境构建静态 ARM64 二进制。

## 目标

- **源码主体**: proot-me/proot 最新版 (commit: 5f780cba)
- **补丁源**: termux/proot 最新版 (commit: ab2e346)
- **编译环境**: ubuntu-24.04-arm 原生 ARM64
- **输出**: 静态链接 ARM64 glibc 二进制

## 移植清单

### 1. 新增扩展目录

| 目录 | 功能 | 优先级 |
|------|------|--------|
| `extension/ashmem_memfd/` | Android ashmem/memfd 模拟 | P0 |
| `extension/sysvipc/` | System V IPC 共享内存 | P0 |
| `extension/mountinfo/` | /proc/PID/mountinfo 覆盖 | P1 |
| `extension/port_switch/` | 端口切换 | P2 |
| `extension/hidden_files/` | 隐藏文件支持 | P2 |
| `extension/fix_symlink_size/` | 符号链接大小修复 | P2 |

### 2. 修改现有文件

| 文件 | 修改内容 |
|------|----------|
| `src/GNUmakefile` | 添加新扩展到 OBJECTS |
| `src/extension/extension.c` | 注册新扩展 |
| `src/arch.h` | 可能需要 ARM64 相关定义 |

### 3. 编译环境配置

- 安装 Android ashmem stub header
- 修复 --rosegment linker 问题
- 禁用 32-bit loader 构建

## 实施步骤

### Step 1: 准备环境
```bash
# 安装编译依赖
apt install libtalloc-dev libarchive-dev libseccomp-dev make git bc

# 安装 Android ashmem header
cat > /usr/include/linux/ashmem.h << 'EOF'
#ifndef _LINUX_ASHMEM_H
#define _LINUX_ASHMEM_H
#include <sys/ioctl.h>
#define ASHMEM_GET_SIZE _IO('a', 3)
#define ASHMEM_SET_SIZE _IO('a', 5)
#endif
EOF
```

### Step 2: 克隆源码
```bash
git clone --depth=1 https://github.com/proot-me/proot.git
cd proot
```

### Step 3: 应用 Termux 补丁

#### 3.1 复制 ashhmem_memfd 扩展
```bash
# 从 Termux 复制
cp -r /tmp/proot-termux/src/extension/ashmem_memfd src/extension/
```

#### 3.2 复制 sysvipc 扩展
```bash
cp -r /tmp/proot-termux/src/extension/sysvipc src/extension/
```

#### 3.3 复制 mountinfo 扩展
```bash
cp -r /tmp/proot-termux/src/extension/mountinfo src/extension/
```

#### 3.4 复制其他扩展
```bash
cp -r /tmp/proot-termux/src/extension/port_switch src/extension/
cp -r /tmp/proot-termux/src/extension/hidden_files src/extension/
cp -r /tmp/proot-termux/src/extension/fix_symlink_size src/extension/
```

### Step 4: 修改 Makefile

在 `src/GNUmakefile` 的 OBJECTS 列表中添加：
```makefile
extension/ashmem_memfd/ashmem_memfd.o \
extension/sysvipc/sysvipc.o \
extension/sysvipc/sysvipc_msg.o \
extension/sysvipc/sysvipc_sem.o \
extension/sysvipc/sysvipc_shm.o \
extension/mountinfo/mountinfo.o \
extension/port_switch/port_switch.o \
extension/hidden_files/hidden_files.o \
extension/fix_symlink_size/fix_symlink_size.o \
```

### Step 5: 修复编译问题

#### 5.1 修复 --rosegment linker 问题
```bash
sed -i 's/,--rosegment,/,/' src/GNUmakefile
```

#### 5.2 禁用 32-bit loader
```bash
sed -i '/define_from_arch.h,,HAS_LOADER_32BIT/a HAS_LOADER_32BIT :=' src/GNUmakefile
```

### Step 6: 构建
```bash
make -C src proot GIT=false PYTHON_CONFIG=/bin/false HAS_PYTHON_CONFIG=""
```

### Step 7: 验证
```bash
file src/proot
readelf -h src/proot | grep Machine
ldd src/proot  # 确认静态或动态链接
```

### Step 8: 打包发布
```bash
VERSION=$(git describe --tags --abbrev=0)-$(git rev-parse --short HEAD)
mkdir -p dist
cp src/proot dist/proot-$VERSION-arm64-linux-glibc
tar -czf dist/proot-$VERSION-arm64-linux-glibc.tar.gz -C dist proot-$VERSION-arm64-linux-glibc
```

## 文件清单

```
proot-android/
├── src/
│   ├── extension/
│   │   ├── ashmem_memfd/
│   │   │   └── ashmem_memfd.c      # 238 lines
│   │   ├── sysvipc/
│   │   │   ├── sysvipc.c           # 主文件
│   │   │   ├── sysvipc_msg.c      # 消息队列
│   │   │   ├── sysvipc_sem.c      # 信号量
│   │   │   └── sysvipc_shm.c      # 共享内存 749 lines
│   │   ├── mountinfo/
│   │   │   └── mountinfo.c         # 143 lines
│   │   ├── port_switch/
│   │   │   └── port_switch.c
│   │   ├── hidden_files/
│   │   │   └── hidden_files.c
│   │   └── fix_symlink_size/
│   │       └── fix_symlink_size.c
│   └── GNUmakefile                 # 修改添加新扩展
└── doc/
    └── android-patches.md          # 补丁说明
```

## 验证检查项

| 检查项 | 命令 | 预期结果 |
|--------|------|----------|
| ELF 架构 | `readelf -h proot \| grep Machine` | `Machine: AArch64` |
| 文件类型 | `file proot` | `ELF 64-bit LSB executable` |
| 链接方式 | `ldd proot` | 静态: "statically linked"，动态: 显示依赖库 |
| 二进制大小 | `ls -lh proot` | > 1MB |

## 风险与注意事项

1. **文件冲突**: Termux 和 proot-me 的 fake_id0 目录结构不同，需单独处理
2. **API 变更**: proot-me 新版本 API 可能有变化，需要调整补丁代码
3. **依赖问题**: 静态链接时 libtalloc 可能需要从源码编译

## 时间估算

- 环境配置: 5 分钟
- 文件移植: 10 分钟
- Makefile 修改: 5 分钟
- 编译调试: 15-30 分钟
- 验证打包: 10 分钟
- **总计**: 约 45-60 分钟
