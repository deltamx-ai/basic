# mise (mise-en-place) 安装与使用指南

`mise`（读作 *MEEZ ahn plahs*）是一个用 Rust 编写的现代开发环境管理工具，集成了以下三大核心功能：

1. **多语言/工具版本管理**（替代 `nvm`、`pyenv`、`rbenv`、`asdf` 等）
2. **目录级环境变量管理**（替代 `direnv`，支持 `.env` 与项目专属变量）
3. **任务运行器**（替代 `Makefile`、`npm scripts`）

---

## 一、安装 mise

### 1. Linux / WSL / macOS（官方推荐脚本）

适用于所有主流 Linux 发行版（包括 WSL Fedora/Ubuntu）及 macOS，安装位置默认为 `~/.local/bin/mise`：

```bash
curl -fsSL https://mise.run | sh
```

验证安装：

```bash
~/.local/bin/mise --version
```

### 2. Fedora / RHEL 系（通过 DNF COPR 仓库）

如果你使用的是 Fedora（如 WSL Fedora 42/43/44+），也可以直接用系统包管理器安装并随系统更新：

```bash
sudo dnf copr enable -y jdxcode/mise
sudo dnf install -y mise
```

### 3. 其他常见系统

* **macOS (Homebrew)**：

  ```bash
  brew install mise
  ```

* **Ubuntu / Debian**：

  ```bash
  sudo apt update -y && sudo apt install -y extrepo
  sudo extrepo enable mise
  sudo apt update -y && sudo apt install -y mise
  ```

* **Windows (PowerShell)**：

  ```powershell
  winget install jdx.mise
  # 或使用官方脚本：irm https://mise.run/powershell | iex
  ```

---

## 二、配置 Shell 激活（必做）

安装完成后，必须将 `mise` 挂载到你的 Shell 中，才能在切换目录时自动切换工具版本和环境变量。

> **注意**：如果是通过官方 `curl` 脚本安装的，请使用 `~/.local/bin/mise`；如果是通过 `dnf` / `brew` / `apt` 安装到系统路径的，可直接写 `mise`。

### Bash（Linux / WSL 默认）

```bash
# 若为 curl 安装：
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
# 若为 dnf/apt 安装：
# echo 'eval "$(mise activate bash)"' >> ~/.bashrc

source ~/.bashrc
```

### Zsh

```bash
echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc
```

### Fish

```bash
echo '~/.local/bin/mise activate fish | source' >> ~/.config/fish/config.fish
```

### 补充：配置 Shims（供 IDE / 图形化工具识别）

VS Code、JetBrains 等编辑器有时不会加载交互式 Shell 配置，将 shims 目录加入系统 `PATH` 可确保编辑器正常找到当前版本的工具（如 Python、Node）：

```bash
# 添加到 ~/.bash_profile 或 ~/.zprofile
echo 'export PATH="$HOME/.local/share/mise/shims:$PATH"' >> ~/.bash_profile
```

---

## 三、核心使用教程

### 1. 查找支持的工具与版本

```bash
# 搜索官方内置支持的所有工具（如 node、python、go、redis 等）
mise registry | grep node

# 查看某个工具所有可安装的版本
mise ls-remote node
mise ls-remote python@3.13
mise ls-remote redis
```

### 2. 安装并启用版本

* **为当前项目指定版本**（会在当前目录生成或更新 `mise.toml`）：

  ```bash
  cd /path/to/your-project
  mise use node@24 python@3.13
  ```

* **设置全局默认版本**（写入 `~/.config/mise/config.toml`）：

  ```bash
  mise use -g node@lts
  mise use -g go@latest
  ```

* **仅安装不写入配置文件**：

  ```bash
  mise install node@22
  ```

### 3. 临时使用某个版本（不改动任何配置）

```bash
mise exec node@20 -- node -v
```

### 4. 查看、升级与卸载

```bash
# 查看已安装且当前生效的所有工具版本
mise ls

# 查看当前目录激活的工具状态
mise current

# 升级所有工具到配置允许的最新版本
mise upgrade

# 卸载指定版本
mise uninstall node@20.11.0

# 清理当前未被任何配置引用的冗余版本
mise prune
```

### 5. 接手包含 `mise.toml` 的现有项目

当从 Git 克隆一个已配置好 `mise.toml` 的项目时：

```bash
git clone <repo-url> && cd <repo-dir>

# 1. 信任该项目的配置文件（首次进入新项目出于安全考虑需要确认）
mise trust

# 2. 一键安装项目所需的所有工具和版本
mise install
```

---

## 四、`mise.toml` 完整配置示例

在项目根目录创建 `mise.toml`，即可用**一个文件**统一管理运行环境、环境变量和常用命令：

```toml
# 1. 指定项目所需的工具与版本
[tools]
node = "24"
python = "3.13"
redis = "7.4"
# 也可以直接指定 GitHub Release、npm 或 pipx 等后端工具
"npm:prettier" = "latest"

# 2. 项目级环境变量（进入目录自动生效，离开自动清除）
[env]
NODE_ENV = "development"
API_BASE_URL = "http://127.0.0.1:8000"
# 自动加载本地的 .env 文件
_.file = ".env"
# 自动激活或创建 Python 虚拟环境
_.python.venv = { path = ".venv", create = true }

# 3. 任务运行器（使用 `mise run <任务名>` 执行）
[tasks.dev]
description = "启动开发服务器"
run = "npm run dev"

[tasks.lint]
description = "代码格式检查"
run = [
  "npx prettier --check .",
  "python -m flake8"
]

[tasks.redis]
description = "启动项目对应版本的 Redis"
run = "redis-server --port 6379"
```

运行任务：

```bash
# 列出当前项目的所有任务
mise tasks ls

# 运行任务
mise run dev
mise run lint
```

---

## 五、常用排查与维护命令

```bash
# 诊断环境配置是否正确（检查 PATH、Shell 激活状态、依赖缺失等）
mise doctor

# 更新 mise 本体到最新版（仅限非包管理器安装方式）
mise self-update

# 刷新工具注册表缓存
mise cache clear
```
