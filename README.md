# proot CI/CD

自动构建 proot 静态二进制 ARM64 glibc 版本。

## 功能

- 每 15 天自动检测上游 [proot-me/proot](https://github.com/proot-me/proot) 是否有新版本
- 交叉编译生成 ARM64 (aarch64) 静态二进制
- 自动发布到 GitHub Releases
- 支持手动触发构建

## 下载预编译版本

前往 [Releases](https://github.com/proot-me/proot/releases) 页面下载最新版本。

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
| 构建系统 | Meson + Ninja |
| CI 平台 | GitHub Actions |

## 使用方法

```bash
# 下载解压
wget https://github.com/proot-me/proot/releases/download/v0.0.0-xxxxxxx/proot-v0.0.0-xxxxxxx-arm64-linux-glibc.tar.gz
tar -xzf proot-*.tar.gz

# 运行
./proot -h
```

## 本地构建

```bash
# 安装依赖
sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
  libc6-dev-arm64-cross pkg-config-aarch64-linux-gnu ninja-build meson

# 克隆源码
git clone https://github.com/proot-me/proot.git

# 配置编译
meson setup proot-build proot --cross-file meson/cross-aarch64.txt -Dstatic=true

# 编译
meson compile -C proot-build

# 安装
meson install -C proot-build
```

## CI/CD 工作流

```
每15天自动检测上游
       │
       ▼
  有新版本？ ──否──→ 结束
       │
      是
       │
       ▼
  检查是否已发布
       │
       ▼
  交叉编译 ARM64 静态版
       │
       ▼
  发布到 GitHub Releases
```

## License

继承 proot 项目 LICENSE
