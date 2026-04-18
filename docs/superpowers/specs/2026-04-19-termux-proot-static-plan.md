# Termux proot 静态构建计划

## 概述

基于 termux/proot 源码，在 ubuntu-24.04-arm 原生环境构建静态 ARM64 glibc 二进制。

## 目标

| 项目 | 值 |
|------|-----|
| 源码 | termux/proot (https://github.com/termux/proot.git) |
| 最新 Commit | ab2e346 (2026-02-21) |
| 架构 | ARM64 (aarch64) |
| C 库 | glibc |
| 链接 | 静态 |
| 环境 | ubuntu-24.04-arm 原生 |

## 已知问题

1. **ashmem.h 缺失** - Android 专用头文件，Linux 系统没有
2. **--rosegment linker 问题** - ARM64 原生 ld 不支持
3. **-m32 32位构建** - ARM64 原生不支持，需要禁用
4. **静态链接缺少 libtalloc** - 可能需要从源码编译

## 实施步骤

### Step 1: 准备环境

```bash
# 1.1 更新 apt
sudo apt-get update

# 1.2 安装编译依赖
sudo apt-get install -y --no-install-recommends \
    libarchive-dev \
    libseccomp-dev \
    make \
    git \
    bc \
    autoconf \
    automake \
    libtool

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

# 1.4 验证
cat /usr/include/linux/ashmem.h
```

### Step 2: 克隆 termux/proot 源码

```bash
# 2.1 克隆源码
git clone --depth=1 https://github.com/termux/proot.git
cd proot

# 2.2 验证版本
echo "Commit: $(git rev-parse --short HEAD)"
echo "Date: $(git log -1 --format='%ci')"
git log --oneline -3
```

### Step 3: 修复 Makefile

```bash
cd proot

# 3.1 备份
cp src/GNUmakefile src/GNUmakefile.bak

# 3.2 修复 --rosegment（ARM64 原生 ld 不支持）
sed -i 's/,--rosegment,/,/' src/GNUmakefile

# 3.3 禁用 32-bit loader（ARM64 原生不支持 -m32）
sed -i '/define_from_arch.h,,HAS_LOADER_32BIT/a HAS_LOADER_32BIT :=' src/GNUmakefile

# 3.4 验证修改
grep "rosegment" src/GNUmakefile && echo "需修复" || echo "--rosegment OK"
grep "HAS_LOADER_32BIT :=" src/GNUmakefile && echo "32-bit 已禁用"
```

### Step 4: 构建静态 libtalloc（如需要）

```bash
cd /tmp

# 4.1 下载 talloc 源码
curl -sL https://github.com/talloc/talloc/archive/refs/tags/v2.4.0.tar.gz -o talloc.tar.gz
tar -xzf talloc.tar.gz
cd talloc-2.4.0

# 4.2 配置和编译静态库
./autogen.sh
./configure --enable-static --disable-shared
make -j$(nproc)

# 4.3 安装
sudo make install
sudo ldconfig

# 4.4 验证
ls -la /usr/local/lib/libtalloc.*
```

### Step 5: 首次编译尝试（动态链接）

```bash
cd proot

# 5.1 先尝试动态链接构建（确认基础能编译）
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG=""

# 5.2 检查产物
file src/proot
ls -lh src/proot
```

### Step 6: 静态编译尝试

```bash
cd proot

# 6.1 清理
make -C src clean

# 6.2 静态编译
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG="" \
    LDFLAGS="-static"

# 6.3 检查结果
file src/proot
ldd src/proot 2>&1 | head -5
```

### Step 7: 如果静态编译失败，尝试混合方案

```bash
cd proot

# 清理
make -C src clean

# 混合方案：主要静态，talloc 动态
make -C src proot GIT=false \
    PYTHON_CONFIG=/bin/false \
    HAS_PYTHON_CONFIG="" \
    LDFLAGS="-Wl,-z,noexecstack -static -ltalloc"

# 检查
file src/proot
ldd src/proot
```

### Step 8: 验证产物

```bash
cd proot

# 8.1 基本检查
file src/proot
ls -lh src/proot

# 8.2 架构检查
readelf -h src/proot | grep -E "Machine:|Class:|Type:"

# 8.3 链接方式
if ldd src/proot 2>&1 | grep -q "not a dynamic"; then
    echo "✓ 静态链接"
else
    echo "○ 动态链接"
    ldd src/proot | head -5
fi

# 8.4 运行测试
./src/proot --help | head -10
```

### Step 9: 打包

```bash
cd proot

# 9.1 版本信息
VERSION=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
COMMIT=$(git rev-parse --short HEAD)
FULL_VERSION="${VERSION}-${COMMIT}"
echo "版本: $FULL_VERSION"

# 9.2 打包
mkdir -p dist
cp src/proot dist/proot-${FULL_VERSION}-arm64-linux-glibc
tar -czf dist/proot-${FULL_VERSION}-arm64-linux-glibc.tar.gz \
    -C dist proot-${FULL_VERSION}-arm64-linux-glibc

# 9.3 验证
ls -lh dist/
file dist/proot-*
```

## GitHub Actions Workflow

```yaml
name: Build proot ARM64 Static

on:
  schedule:
    - cron: '0 0 */10 * *'
  workflow_dispatch:

env:
  PROOT_REPO: "https://github.com/termux/proot.git"

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-24.04-arm

    steps:
      - uses: actions/checkout@v4

      - name: Clone proot
        run: |
          git clone --depth=1 "$PROOT_REPO" proot-src
          cd proot-src
          echo "sha=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          echo "date=$(git log -1 --format='%ci' | cut -d' ' -f1)" >> $GITHUB_OUTPUT
          LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
          echo "version=$LATEST_TAG-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Check release
        run: |
          TAG="${{ steps.clone.outputs.version }}"
          RESP=$(curl -s -o /dev/null -w "%{http_code}" \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            "https://api.github.com/repos/${{ github.repository }}/releases/tags/${TAG}")
          [ "$RESP" = "200" ] && echo "exists=true" || echo "exists=false"
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - if: steps.check.outputs.exists == 'true'
        run: exit 78

      - name: Install deps
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            libarchive-dev libseccomp-dev make git bc autoconf automake libtool
          # Android ashmem header
          sudo tee /usr/include/linux/ashmem.h > /dev/null << 'EOF'
          #ifndef _LINUX_ASHMEM_H
          #define _LINUX_ASHMEM_H
          #include <sys/ioctl.h>
          #define ASHMEM_GET_SIZE _IO('a', 3)
          #define ASHMEM_SET_SIZE _IO('a', 5)
          #endif
          EOF

      - name: Build talloc static
        run: |
          cd /tmp
          curl -sL https://github.com/talloc/talloc/archive/refs/tags/v2.4.0.tar.gz -o talloc.tar.gz
          tar -xzf talloc.tar.gz
          cd talloc-2.4.0
          ./autogen.sh
          ./configure --enable-static --disable-shared
          make -j$(nproc)
          sudo make install
          sudo ldconfig

      - name: Build proot
        run: |
          cd proot-src
          sed -i 's/,--rosegment,/,/' src/GNUmakefile
          sed -i '/define_from_arch.h,,HAS_LOADER_32BIT/a HAS_LOADER_32BIT :=' src/GNUmakefile
          make -C src proot GIT=false \
              PYTHON_CONFIG=/bin/false HAS_PYTHON_CONFIG="" \
              LDFLAGS="-static"

      - name: Package
        run: |
          mkdir -p dist
          cp proot-src/src/proot dist/proot-${{ steps.clone.outputs.version }}-arm64-linux-glibc
          tar -czf dist/proot-${{ steps.clone.outputs.version }}-arm64-linux-glibc.tar.gz \
            -C dist proot-${{ steps.clone.outputs.version }}-arm64-linux-glibc

      - name: Verify
        run: |
          file dist/proot-*
          readelf -h dist/proot-* | grep Machine

      - name: Release
        if: steps.check.outputs.exists == 'false'
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.check.outputs.tag }}
          body: "proot ARM64 build"
          files: dist/proot-*.tar.gz
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 验证检查项

| 检查 | 命令 | 预期 |
|------|------|------|
| 架构 | `readelf -h proot \| grep Machine` | `AArch64` |
| 文件类型 | `file proot` | `ELF 64-bit LSB executable` |
| 链接 | `ldd proot` | "not a dynamic executable" 或显示依赖 |
| 大小 | `ls -lh proot` | > 1MB |

## 风险清单

1. **静态 libtalloc** - 可能编译失败，需要调试
2. **proot 静态链接** - termux 默认动态，可能有未发现的依赖问题
3. **编译时间** - ARM64 原生编译比交叉编译慢

## 预计时间

- 环境准备: 5 min
- 源码克隆: 2 min
- libtalloc 编译: 10 min
- proot 编译调试: 20-30 min
- 打包验证: 5 min
- **总计: 约 40-50 min**
