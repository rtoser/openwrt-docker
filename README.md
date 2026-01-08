# OpenWrt Docker 构建环境

本项目提供了基于 Docker 的 OpenWrt 固件和 SDK 构建方案，旨在简化 macOS 和 Linux 下的交叉编译流程。

## 项目结构

### 1. 固件构建 (`firmware/`)
用于从源码构建完整 OpenWrt 固件的工具和指南。
- [macOS Docker 构建指南](firmware/DOCKER_OPENWRT_BUILD_MACOS.md)：在 macOS 上使用 Docker 构建固件的分步指南。
- [VMware 构建模板](firmware/VMWARE_OPENWRT_BUILD_TEMPLATE.md)：基于 VMware 的构建模板。

### 2. SDK 构建 (`sdk/`)
用于构建预配置 OpenWrt SDK Docker 镜像（已优化集成 `libubus`, `libgpiod` 等）的脚本，以加速下游开发。
- [SDK Docker 构建指南](sdk/docker/README.md)：构建 SDK 环境的文档。

## 使用说明
请参考上述链接中的具体文档以获取详细的操作说明。
