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

## 详细实施步骤

### Step 1: 准备环境和克隆源码

```bash
# 1.1 更新 apt
sudo apt-get update

# 1.2 安装编译依赖
sudo apt-get install -y --no-install-recommends \
    libtalloc-dev \
    libarchive-dev \
    libseccomp-dev \
    make \
    git \
    bc

# 1.3 创建 Android ashmem stub header
sudo tee /usr/include/linux/ashmem.h > /dev/null << 'EOF'
#ifndef _LINUX_ASHMEM_H
#define _LINUX_ASHMEM_H
#include <sys/ioctl.h>
#define ASHMEM_GET_SIZE _IO('a', 3)
#define ASHMEM_SET_SIZE _IO('a', 5)
#define ASHMEM_SET_NAME _IO('a', 1)
#define ASHMEM_GET_NAME _IO('a', 2)
#define ASHMEM_PIN _IO('a', 7)
#define ASHMEM_UNPIN _IO('a', 8)
#define ASHMEM_MAP _IO('a', 9)
#define ASHMEM_UNMAP _IO('a', 10)
#define ASHMEM_VERSION _IO('a', 0)
#define ASHMEM_NAME_LEN 256
struct ashmem_pin { unsigned int offset; unsigned int len; };
struct ashmem_info { unsigned int size; unsigned int prot; char name[ASHMEM_NAME_LEN]; };
#endif
EOF

# 1.4 克隆 proot-me 源码
git clone --depth=1 https://github.com/proot-me/proot.git
cd proot

# 1.5 克隆 Termux 源码（用于提取补丁）
cd ..
git clone --depth=1 https://github.com/termux/proot.git proot-termux
```

### Step 2: 提取 Termux 扩展文件

```bash
cd proot

# 2.1 创建扩展目录
mkdir -p src/extension/ashmem_memfd
mkdir -p src/extension/sysvipc
mkdir -p src/extension/mountinfo
mkdir -p src/extension/port_switch
mkdir -p src/extension/hidden_files
mkdir -p src/extension/fix_symlink_size

# 2.2 复制 ashmem_memfd
cp ../proot-termux/src/extension/ashmem_memfd/*.c src/extension/ashmem_memfd/
cp ../proot-termux/src/extension/ashmem_memfd/*.h src/extension/ashmem_memfd/ 2>/dev/null || true

# 2.3 复制 sysvipc
cp ../proot-termux/src/extension/sysvipc/*.c src/extension/sysvipc/
cp ../proot-termux/src/extension/sysvipc/*.h src/extension/sysvipc/ 2>/dev/null || true

# 2.4 复制 mountinfo
cp ../proot-termux/src/extension/mountinfo/*.c src/extension/mountinfo/
cp ../proot-termux/src/extension/mountinfo/*.h src/extension/mountinfo/ 2>/dev/null || true

# 2.5 复制 port_switch
cp ../proot-termux/src/extension/port_switch/*.c src/extension/port_switch/
cp ../proot-termux/src/extension/port_switch/*.h src/extension/port_switch/ 2>/dev/null || true

# 2.6 复制 hidden_files
cp ../proot-termux/src/extension/hidden_files/*.c src/extension/hidden_files/
cp ../proot-termux/src/extension/hidden_files/*.h src/extension/hidden_files/ 2>/dev/null || true

# 2.7 复制 fix_symlink_size
cp ../proot-termux/src/extension/fix_symlink_size/*.c src/extension/fix_symlink_size/
cp ../proot-termux/src/extension/fix_symlink_size/*.h src/extension/fix_symlink_size/ 2>/dev/null || true

# 2.8 验证文件已复制
ls -la src/extension/
```

### Step 3: 修改 GNUmakefile

```bash
cd proot

# 3.1 查看当前 Makefile 中的 extension 对象列表
grep -n "extension/" src/GNUmakefile | head -20

# 3.2 在 extension/extension.o 后添加新扩展
# 找到 "extension/extension.o \" 所在行

# 3.3 备份 Makefile
cp src/GNUmakefile src/GNUmakefile.bak

# 3.4 使用 sed 插入新扩展（在 extension.o 之后）
sed -i '/extension\/extension.o \\\\/a\
\	extension/ashmem_memfd/ashmem_memfd.o \\\n\
\	extension/sysvipc/sysvipc.o \\\n\
\	extension/sysvipc/sysvipc_msg.o \\\n\
\	extension/sysvipc/sysvipc_sem.o \\\n\
\	extension/sysvipc/sysvipc_shm.o \\\n\
\	extension/mountinfo/mountinfo.o \\\n\
\	extension/port_switch/port_switch.o \\\n\
\	extension/hidden_files/hidden_files.o \\\n\
\	extension/fix_symlink_size/fix_symlink_size.o \\' src/GNUmakefile

# 3.5 验证修改
grep -A 15 "extension/extension.o" src/GNUmakefile | head -20
```

### Step 4: 修复编译问题

```bash
cd proot

# 4.1 修复 --rosegment linker 问题（ARM64 原生 ld 不支持）
sed -i 's/,--rosegment,/,/' src/GNUmakefile

# 4.2 禁用 32-bit loader（ARM64 原生不支持 -m32）
sed -i '/define_from_arch.h,,HAS_LOADER_32BIT/a HAS_LOADER_32BIT :=' src/GNUmakefile

# 4.3 验证修改
grep "rosegment" src/GNUmakefile && echo "需要修复 --rosegment" || echo "--rosegment 已修复"
grep "HAS_LOADER_32BIT :=" src/GNUmakefile && echo "32-bit loader 已禁用" || echo "未禁用"
```

### Step 5: 首次编译尝试

```bash
cd proot

# 5.1 首次编译
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG="" 2>&1 | tee build.log

# 5.2 如果编译失败，查看错误
if [ $? -ne 0 ]; then
    echo "=== 编译失败，查看错误 ==="
    tail -100 build.log
fi
```

### Step 6: 解决编译错误

根据错误信息，可能需要：

```bash
# 6.1 缺少头文件 - 复制头文件
cp ../proot-termux/src/extension/*/*.h src/extension/ 2>/dev/null || true

# 6.2 函数未定义 - 检查 extension.h 注册
grep -n "ashmem_memfd\|sysvipc\|mountinfo" src/extension/extension.c

# 6.3 重新编译
make -C src clean
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG="" 2>&1 | tee build2.log
```

### Step 7: 验证构建产物

```bash
cd proot

# 7.1 检查二进制
file src/proot
ls -lh src/proot

# 7.2 检查架构
readelf -h src/proot | grep -E "Machine:|Class:|Type:"

# 7.3 检查链接方式
ldd src/proot 2>&1 | head -5

# 7.4 如果是动态链接，检查依赖
ldd src/proot

# 7.5 如果需要静态链接，添加 LDFLAGS
make -C src clean
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG="" \
    LDFLAGS="-static" 2>&1 | tee build_static.log
```

### Step 8: 打包发布

```bash
cd proot

# 8.1 获取版本信息
VERSION=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
COMMIT=$(git rev-parse --short HEAD)
FULL_VERSION="${VERSION}-${COMMIT}"
echo "版本: $FULL_VERSION"

# 8.2 创建分发包
mkdir -p dist
cp src/proot dist/proot-${FULL_VERSION}-arm64-linux-glibc
tar -czf dist/proot-${FULL_VERSION}-arm64-linux-glibc.tar.gz \
    -C dist proot-${FULL_VERSION}-arm64-linux-glibc

# 8.3 验证包
ls -lh dist/
file dist/proot-*

# 8.4 生成验证报告
cat > dist/verification-report-${FULL_VERSION}.txt << 'REPORT'
proot ARM64 Verification Report
================================
Version: ${FULL_VERSION}
Date: $(date +%Y-%m-%d)
Source: proot-me/proot + Termux Android patches

ELF Architecture:
$(readelf -h dist/proot-${FULL_VERSION}-arm64-linux-glibc | grep -E "Machine:|Class:|Type:")

Linking:
$(ldd dist/proot-${FULL_VERSION}-arm64-linux-glibc 2>&1 | head -5 || echo "Static binary")

Binary Size:
$(ls -lh dist/proot-${FULL_VERSION}-arm64-linux-glibc | awk '{print $5}')
REPORT

echo "=== 完成 ==="
echo "包位置: dist/proot-${FULL_VERSION}-arm64-linux-glibc.tar.gz"
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
