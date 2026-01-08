# macOS 使用 Docker 编译 OpenWrt（Step by Step）

本指南以当前目录结构为例：

- `openwrt-docker/`：项目根目录
- `openwrt-docker/openwrt/`：OpenWrt 源码目录

## 1. 准备镜像（Dockerfile 构建）

在项目根目录执行：

```bash
docker build -t openwrt-build .
```

> 如已有镜像可跳过。

## 2. 确认源码与 feeds

进入源码目录并更新 feeds（若已完成可跳过）：

```bash
cd /Users/libo/Berry/openwrt-docker/openwrt
./scripts/feeds update -a
./scripts/feeds install -a
```

## 3. 配置目标

```bash
make menuconfig
```

保存后会生成 `.config`。

## 4. 使用 Docker 编译（推荐命令模板）

在项目根目录执行：

```bash
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -v /Users/libo/Berry/openwrt-docker/openwrt:/work \
  -w /work \
  openwrt-build \
  sh -lc 'make -j$(nproc)'
```

说明：

- `-u $(id -u):$(id -g)` 避免以 root 身份构建导致部分工具（如 tar）报错
- `-v ...:/work` 将源码挂载到容器
- `-w /work` 设定工作目录
- `make -j$(nproc)` 使用容器 CPU 线程数并行编译

## 5. 常见问题处理

如果构建失败，先单线程获取详细日志：

```bash
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -v /Users/libo/Berry/openwrt-docker/openwrt:/work \
  -w /work \
  openwrt-build \
  sh -lc 'make -j1 V=s'
```

## 6. 输出位置

编译成功后固件一般在：

```
openwrt/bin/targets/<target>/<subtarget>/
```

常见文件：

- `*-factory.bin`：首次刷机用
- `*-sysupgrade.bin`：OpenWrt 内升级用

## 7. 生成与官方发行版一致的固件与包集合

要复现官方发行版，核心是**版本/配置/feeds 提交**一致。做法如下：

1) 使用与官方一致的源码版本（tag 或分支）  
例如发行版 `24.10.5` 对应 `v24.10.5`：

```bash
git checkout v24.10.5
```

2) 获取官方构建配置与 feeds 版本  
到官方下载页面的对应目标目录，下载：

- `config.buildinfo`（官方构建 .config）
- `feeds.buildinfo`（官方 feeds 提交）
- `packages.manifest`（官方包清单，用于核对）

下载路径示例（把 `<target>/<subtarget>` 换成你的目标）：

- `https://downloads.openwrt.org/releases/24.10.5/targets/<target>/<subtarget>/config.buildinfo`
- `https://downloads.openwrt.org/releases/24.10.5/targets/<target>/<subtarget>/feeds.buildinfo`
- `https://downloads.openwrt.org/releases/24.10.5/targets/<target>/<subtarget>/packages.manifest`

3) 应用官方配置与 feeds

```bash
cp config.buildinfo .config
./scripts/feeds update -a
./scripts/feeds install -a
make defconfig
```

如果需要严格对齐 feeds 版本，按 `feeds.buildinfo` 中列出的 commit 进行 checkout。

4) 构建（同上 Docker 命令）

```bash
docker run --rm \
  -u "$(id -u):$(id -g)" \
  -v /Users/libo/Berry/openwrt-docker/openwrt:/work \
  -w /work \
  openwrt-build \
  sh -lc 'make -j$(nproc)'
```

5) 校验  
对比你生成的 `packages.manifest` 与官方发布的 `packages.manifest`，确保包集合一致。
