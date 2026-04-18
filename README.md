# proot CI/CD

自动构建 proot 静态二进制 ARM64 glibc 版本。

## 功能

- 每月 1 号自动检测上游 [proot-me/proot](https://github.com/proot-me/proot) 是否有新版本
- 使用 dockcross 交叉编译生成 ARM64 (aarch64) 静态二进制
- 自动发布到 GitHub Releases
- 支持手动触发构建

## 下载预编译版本

前往 [Releases](https://github.com/zbaku/proot-ci/releases) 页面下载最新版本。

## 版本命名

格式: `v0.0.0-{commit_sha}`

例如: `v0.0.0-a1b2c3d`

## 构建产物

| 文件名 | 说明 |
|--------|------|
| `proot-{sha}-arm64-linux-glibc.tar.gz` | ARM64 静态二进制压缩包 |

## 技术规格

| 项目 | 值 |
|------|-----|
| 架构 | ARM64 (aarch64) |
| C 库 | glibc |
| 链接方式 | 静态链接 |
| 构建系统 | GNU Makefile |
| 交叉编译 | dockcross/linux-arm64 |

## 使用方法

```bash
# 下载解压
wget https://github.com/zbaku/proot-ci/releases/download/v0.0.0-xxxxxxx/proot-v0.0.0-xxxxxxx-arm64-linux-glibc.tar.gz
tar -xzf proot-*.tar.gz

# 运行
./proot -h
```

## 本地构建（交叉编译 ARM64）

### 使用 dockcross

```bash
# 克隆源码
git clone https://github.com/proot-me/proot.git
cd proot

# 使用 dockcross 交叉编译
docker run --rm -v "$PWD:/workdir" -w /workdir \
  dockcross/linux-arm64 \
  make -C src proot GIT=false \
    CROSS_COMPILE=aarch64-linux-gnu- \
    LDFLAGS="-static" \
    CC=aarch64-linux-gnu-gcc
```

### 原生构建（x86_64 Linux）

```bash
# 安装依赖
sudo apt-get install build-essential libtalloc-dev libarchive-dev

# 克隆源码
git clone https://github.com/proot-me/proot.git
cd proot

# 编译
make -C src proot GIT=false LDFLAGS="-static"
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
  使用 dockcross 交叉编译 ARM64 静态版
       │
       ▼
  检查是否已发布（避免重复）
       │
       ▼
  发布到 GitHub Releases
```

## 交叉编译说明

使用 [dockcross/linux-arm64](https://github.com/dockcross/dockcross) Docker 镜像进行交叉编译，该镜像包含：
- ARM64 (aarch64-linux-gnu) 交叉编译工具链
- 所有必要的 ARM64 静态库（libc.a, libtalloc.a 等）
- 完整的开发环境

## License

继承 proot 项目 LICENSE
