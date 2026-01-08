# GitHub CLI (gh) 实战教程

这份教程是为您当前的 `openwrt-docker` 项目定制的实战指南。`gh` 将 GitHub 的功能带到了终端，让您无需离开命令行即可完成代码管理、Issues 处理和 Releases 发布。

## 0. 什么是 gh (GitHub CLI)？

**`gh` 是 GitHub 官方开发的命令行工具**。

它的主要作用是**将 GitHub 网站的功能带到终端**，让您无需频繁切换浏览器，即可直接在命令行中管理代码仓库、Issues、Pull Requests 和 Releases。

### `gh` vs `git`：有什么区别？

*   **`git`**：通用的**版本控制工具**。
    *   **核心功能**：代码提交 (commit)、分支管理 (branch)、版本回退 (reset)、推送 (push)。
    *   **适用范围**：任何 Git 仓库 (GitHub, GitLab, Bitbucket, 本地仓库)。
    *   **关注点**：**代码本身**的历史和状态。

*   **`gh`**：GitHub 专用的**平台交互工具**。
    *   **核心功能**：创建仓库 (repo create)、拉取请求 (pr)、问题追踪 (issue)、发布版本 (release)。
    *   **适用范围**：仅限 GitHub 平台。
    *   **关注点**：**项目协作**与**平台流程**。

**总结**：您用 `git` 写代码、存代码；用 `gh` 在 GitHub 上分享代码、协作开发。

## 1. 初始配置 (一次性)

在开始之前，确保您已登录。

```bash
gh auth login
```
*   **Account**: GitHub.com
*   **Protocol**: HTTPS (推荐) 或 SSH
*   **Authenticate**: Y (Login with a web browser)

---

## 2. 实战：将当前 OpenWrt 项目上传

鉴于您在 `/Users/libo/Berry/openwrt-docker` 目录下，且其中包含大量编译生成的临时文件（如 `dl/`, `build_dir/`），**配置 `.gitignore` 至关重要**。

### 步骤 A：初始化与清理
```bash
# 1. 初始化 Git 仓库
git init

# 2. 创建 OpenWrt 专用的 .gitignore
# 防止上传数 GB 的编译中间文件
cat <<EOF > .gitignore
/bin
/build_dir
/staging_dir
/tmp
/dl
/logs
/feeds
/.config.old
*.o
*.a
.DS_Store
EOF
```

### 步骤 B：提交与上传
```bash
# 3. 提交本地文件
git add .
git commit -m "Initial commit: OpenWrt Docker build environment"

# 4. 创建远程仓库并推送
# 这条命令会自动：
# - 在 GitHub 创建 rtoser/openwrt-docker
# - 添加远程 origin
# - 推送代码
gh repo create rtoser/openwrt-docker --public --source=. --remote=origin --push
```

---

## 3. 日常开发命令

### 仓库操作
| 动作 | 命令 | 说明 |
| :--- | :--- | :--- |
| **打开网页** | `gh browse` | 在浏览器中直接打开当前仓库页面 |
| **克隆** | `gh repo clone <user>/<repo>` | 自动处理鉴权，比 git clone 方便 |
| **查看信息** | `gh repo view` | 在终端查看仓库 README 和元数据 |

### Issue (问题追踪)
| 动作 | 命令 | 说明 |
| :--- | :--- | :--- |
| **列表** | `gh issue list` | 查看当前打开的 Issues |
| **创建** | `gh issue create` | 交互式创建一个新 Issue |
| **查看** | `gh issue view 12` | 查看编号为 12 的 Issue 详情 |

### Pull Requests (代码合并)
这是 `gh` 最强大的功能之一。

| 动作 | 命令 | 说明 |
| :--- | :--- | :--- |
| **列表** | `gh pr list` | 查看所有 PR |
| **检出(神技)** | `gh pr checkout 5` | **将 PR #5 的代码拉取到本地并切换分支进行测试** |
| **创建** | `gh pr create` | 将当前分支提交为 PR |
| **合并** | `gh pr merge 5` | 合并 PR #5 |

### Releases (发布固件)
当您的 OpenWrt 编译完成后，可以直接发布固件。

```bash
# 创建 Release v1.0.0 并上传固件
gh release create v1.0.0 \
    ./bin/targets/x86/64/openwrt-x86-64-generic-squashfs-combined.img.gz \
    --title "v1.0.0 稳定版" \
    --notes "包含 Docker 支持的 OpenWrt 固件"
```

---

## 4. 高级技巧

### 别名 (Alias)
简化长命令。例如，用 `gh co` 代替 `gh pr checkout`：
```bash
gh alias set co 'pr checkout'
# 使用方法: gh co 12
```

### 搜索
查找 GitHub 上的相关项目：
```bash
gh search repos "openwrt docker" --sort stars
```
