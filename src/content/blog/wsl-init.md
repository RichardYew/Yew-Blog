---
title: "WSL 初始化"
date: "2026-10-02T00:00:00+08:00"
description: "初始化 WSL 开发环境"
categories:
  - "开发环境"
tags:
  - "wsl"
  - "ubuntu"
  - "开发环境"
---

# WSL2 Ubuntu 24.04 全栈开发环境从零配置教程
> 本文档基于实际部署操作记录整理，覆盖 **C/C++、Java、Python、Go、Node.js** 五大主流开发语言，以及 Git、Oh My Zsh、OpenSSH、Redis 等常用工具，包含所有踩坑点与对应解决方案，全程优化国内网络访问速度。
>
> **适用环境**：WSL2 + Ubuntu 24.04 LTS (noble)
> **用户场景**：面向全栈开发，兼顾后端、前端、系统级开发需求
---
## 第一章 系统基础初始化
### 1.1 启用 Systemd 支持
WSL2 默认未完整启用 systemd，会导致 Docker、服务管理等功能异常，必须先开启。
```bash
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true
EOF
```
> ⚠️ **注意**：不建议配置 `generateResolvConf = false`，除非你明确需要自定义 DNS，否则会引发 DNS 解析异常。

**配置生效方式**：切换到 Windows 终端 / PowerShell，执行：
```powershell
wsl --shutdown
```
等待 3 秒后重新打开 Ubuntu 终端，验证 systemd 是否启用：
```bash
ps -p 1 -o comm=
```
输出 `systemd` 即为成功。
---
### 1.2 配置国内软件源（以清华 TUNA 镜像为例）
Ubuntu 默认官方源在国内网络环境下访问速度较慢，切换为清华 TUNA 镜像源可大幅提升软件安装与更新效率。

> ⚠️ **Ubuntu 24.04 重大变化**：24.04 起默认源**不再是** `/etc/apt/sources.list`，而是 deb822 格式的 `/etc/apt/sources.list.d/ubuntu.sources`（指向 `archive.ubuntu.com` / `security.ubuntu.com`）。新装系统下的 `sources.list` 通常只是一段"源已迁移"的注释，**备份它没有任何回滚价值**。
> 如果只往 `sources.list` 写清华源而不动 `ubuntu.sources`，结果是**两套源并存**：`apt update` 仍会请求官方源（慢、易超时），且同一包会重复列出条目。

```bash
# 1. 备份真正的主源文件（出错可快速回滚）
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak
```
```bash
# 2. 把 URIs 原地替换为清华 TUNA（suites 不用动，TUNA 提供 noble / noble-security / noble-updates / noble-backports）
sudo sed -i \
  -e 's|http://archive.ubuntu.com/ubuntu/|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|g' \
  -e 's|http://security.ubuntu.com/ubuntu/|https://mirrors.tuna.tsinghua.edu.cn/ubuntu/|g' \
  /etc/apt/sources.list.d/ubuntu.sources
```
```bash
# 3. 更新软件索引并完成系统全量升级
sudo apt update && sudo apt full-upgrade -y
```

> 💡 **如果你曾按旧版教程写过 `/etc/apt/sources.list`**（文件里是 `deb https://mirrors.tuna.tsinghua.edu.cn/...` 这种行），请二选一，避免双源并存：
> - 方案 A（推荐）：删掉旧追加，只保留 1.2 的 deb822 写法 —— `sudo mv /etc/apt/sources.list /etc/apt/sources.list.old`
> - 方案 B：保留 `sources.list`，但删掉官方 `ubuntu.sources` —— `sudo rm /etc/apt/sources.list.d/ubuntu.sources`
>
> **验证是否只用清华源**（输出里应只有 `mirrors.tuna.tsinghua.edu.cn`，不应出现 `archive.ubuntu.com` / `security.ubuntu.com`）：
> ```bash
> apt-get indextargets --format '$(SITE)' | sort -u
> ```

> 💡 补充说明
> - 本配置仅适配 **Ubuntu 24.04 LTS（版本代号 noble）**，其他版本需替换为对应代号（如 22.04 为 jammy；22.04 及更早版本仍用 `sources.list`）。
> - `deb` 开头的是二进制软件包源，日常开发装依赖、更新系统都走这部分；`deb-src` 是源码源，普通应用开发无需开启，保持注释即可。
> - `apt full-upgrade` 会完整升级所有已安装包，同时智能处理依赖变动 —— 自动安装新增依赖、移除冲突包。相比保守的 `apt upgrade`，升级更彻底，适合换源后、大版本更新时使用；执行前可留意终端提示，确认无重要软件包被标记移除。
> - WSL 环境无需额外调整架构或网络配置，上述命令可直接复制执行。
---
### 1.3 安装基础开发工具合集
按模块安装编译、调试、效率工具等通用基础套件，覆盖绝大多数开发场景依赖。
```bash
# ===== 1. 编译构建核心工具链（C/C++ 开发必装） =====
sudo apt install -y build-essential cmake ninja-build meson pkg-config ccache

# ===== 2. 版本控制与网络工具 =====
sudo apt install -y git curl wget

# ===== 3. 编辑器与系统监控工具 =====
sudo apt install -y vim neovim htop tmux tree

# ===== 4. 文本处理与终端效率工具 =====
sudo apt install -y jq fzf ripgrep fd-find bat

# ===== 5. 调试与代码质量检查工具 =====
sudo apt install -y gdb valgrind clang clang-format clang-tidy cppcheck lldb

# ===== 6. 压缩与网络诊断工具 =====
sudo apt install -y unzip zip p7zip-full xz-utils dnsutils net-tools iputils-ping

# ===== 7. 通用开发依赖库 =====
sudo apt install -y libssl-dev libffi-dev zlib1g-dev libreadline-dev libsqlite3-dev \
  libbz2-dev liblzma-dev libxml2-dev libxslt1-dev

# ===== 8. 可选效率增强工具 =====
sudo apt install -y autojump
```
---
## 第二章 Zsh + Oh My Zsh 终端美化
> **安装逻辑**：先安装 Zsh 本体，再安装 Oh My Zsh 框架，最后安装插件与配置，顺序不可颠倒。
### 2.1 安装 Zsh 并设为默认 Shell
```bash
sudo apt install -y zsh
chsh -s $(which zsh)
```
> 💡 执行 `chsh` 后需输入当前用户密码，重启终端后生效。

### 2.2 安装 Oh My Zsh（国内镜像）
```bash
# 从 Gitee 下载安装脚本
sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
```
> 💡 **交互提示**（两处提示都是 **`[Y/n]`** 格式，即"默认 Yes"）：
> - 询问是否覆盖 `.zshrc`：直接回车或输入 `y`/`Y` 均可（脚本 `case` 同时接受 `[Yy]*` 和空回车）；只有输入 `n`/`N` 才会跳过
> - 询问是否切换默认 shell：同上
>
> **关于"Gitee 镜像"的准确边界**：Gitee 只加速了**安装脚本本身的下载**，脚本内 `REMOTE` 默认仍是 `https://github.com/ohmyzsh/ohmyzsh.git`，**克隆仓库仍走 GitHub**。若 GitHub 克隆也超时，用 `REMOTE` 指向 Gitee 镜像仓库（已实测该镜像可 `git ls-remote`）：
> ```bash
> REMOTE=https://gitee.com/mirrors/ohmyzsh.git \
>   sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
> ```
> 克隆完成后可用 `git -C ~/.oh-my-zsh remote -v` 确认 origin 指向哪里。
>
> 无人值守免交互版本（跳过所有确认）：
> ```bash
> sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)" -- --unattended
> ```

### 2.3 安装常用插件
```bash
# 命令自动补全
git clone https://gitee.com/zsh-users/zsh-autosuggestions \
  ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions

# 语法高亮
git clone https://gitee.com/zsh-users/zsh-syntax-highlighting \
  ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting
```

### 2.4 完整配置 .zshrc
覆盖模板并追加所有开发环境配置：
```bash
# 复制官方模板
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc

# 追加完整配置
cat >> ~/.zshrc << 'EOF'
# ========== 开发环境配置 ==========

# 用户级可执行文件（ensurepip 生成的 pip3.13、pip install --user 安装的 CLI）
# 注意：Ubuntu 24.04 默认 PATH 里没有 ~/.local/bin，不加这一行 pip 装的命令会 command not found
export PATH=$HOME/.local/bin:$PATH

# Java 21
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

# Go
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
export GOPROXY=https://goproxy.cn,direct
export GOMODCACHE=$HOME/go/pkg/mod

# Python 别名（仅对交互式 zsh 生效，脚本/非交互场景请显式用 python3.13）
alias python="python3.13"
alias python3="python3.13"
alias pip="python3.13 -m pip"
alias pip3="python3.13 -m pip"

# 工具别名
alias fd=fdfind
alias bat=batcat

# NVM Node 版本管理
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# Zsh 行为优化
setopt nonomatch                # 通配符不匹配时不报错
setopt interactive_comments     # 交互模式下 # 开头视为注释
typeset -U path                 # PATH 去重：反复 source ~/.zshrc 不会重复追加
EOF
```

> 💡 **别名的作用域**：`python`/`pip` 是 zsh **交互式别名**，只在你手动敲命令的终端里生效；`zsh -c 'python --version'`、`bash -c 'python --version'`、cron/脚本里都会 `command not found`。脚本中请统一写 `python3.13`。另外 `.bashrc` 不会自动继承这些别名（本教程默认 shell 是 zsh）。

### 2.5 启用插件与设置主题
```bash
# 启用插件（注意：不要加入不存在的 java 插件）
sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting autojump golang python docker)/' ~/.zshrc

# 设置简洁兼容主题 ys（无需额外字体）
sed -i 's/ZSH_THEME="robbyrussell"/ZSH_THEME="ys"/' ~/.zshrc

# 生效配置
source ~/.zshrc
```
> ⚠️ **踩坑提示**：
> 1. Oh My Zsh 默认插件库**没有单独的 `java` 插件**，不要加入插件列表，否则会报 `plugin 'java' not found`。
> 2. 插件列表里的 **`docker` 插件在未安装 Docker 时无害但无意义**；若你暂不打算装 Docker，把 `docker` 从列表中去掉即可（后续装好 Docker 再加回来）。
> 3. `source ~/.zshrc` 反复执行会重复追加 PATH 段，上面模板中的 `typeset -U path` 会自动去重，可放心执行。
---
## 第三章 各语言开发环境配置
### 3.1 C/C++ 工具链（GCC 14）
系统默认自带 GCC 13，升级到最新稳定版 GCC 14，并设为默认。
```bash
# 安装 GCC 14 与 G++ 14
sudo apt install -y gcc-14 g++-14

# 设为系统默认版本（优先级100）
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-14 100

# 验证版本
gcc --version
g++ --version
```
> 💡 说明：若后续需要切回默认 GCC 13，可执行 `sudo update-alternatives --config gcc` 手动选择版本。
---
### 3.2 Java JDK 21（OpenJDK）
采用系统包安装方式，稳定可靠。
```bash
# 安装 OpenJDK 21 开发版
sudo apt install -y openjdk-21-jdk

# 验证安装
java -version
javac -version
```
💡 说明
- Ubuntu 系统安装的 OpenJDK 会自动配置 `java` 命令的全局软链，`JAVA_HOME` 已统一写入 `.zshrc` 配置。
- 环境变量已在第二章终端配置中统一添加，重启终端或 `source ~/.zshrc` 生效。
---
### 3.3 Python 最新稳定版（3.13）
通过 deadsnakes PPA 安装最新版 Python，比系统自带版本更新。
```bash
# 1. 添加 deadsnakes PPA 软件源
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# 2. 安装 Python 3.13 完整开发套件
sudo apt install -y python3.13 python3.13-venv python3.13-dev

# 3. 初始化 Python 3.13 内置 pip 模块
python3.13 -m ensurepip --upgrade

# 4. 给 Python 3.13 的 pip 配置清华镜像源
python3.13 -m pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 5. 验证安装
python3.13 --version
python3.13 -m pip --version
# ensurepip 会在用户目录生成独立的 pip3.13（依赖 2.4 节把 ~/.local/bin 加入 PATH）
pip3.13 --version
```
> ⚠️ 踩坑提示
> 1. **关于 `pip3.13` 的准确说法**：deadsnakes 不提供**系统级** `/usr/bin/pip3.13`，`update-alternatives --query pip3` 会返回 `no alternatives for pip3`，强行 `--install` 会报 `alternative path doesn't exist`——所以**不要用 `update-alternatives` 配置 pip3**，统一用 `python3.13 -m pip`。
>    但 `python3.13 -m ensurepip --upgrade` 之后，**用户级** `~/.local/bin/pip3.13` 是存在的（可用 `ls ~/.local/bin/pip*` 确认），前提是 2.4 节已把 `$HOME/.local/bin` 加进 PATH；否则会 `command not found`。
> 2. 不推荐强制修改系统全局 `python3` 指向（`update-alternatives` 方式），会导致 Ubuntu 系统工具、`command-not-found` 等组件因依赖报错（详见 8.1 节）；用户级别名方案足够开发使用，且不影响系统稳定性。
> 3. 若后续编译 Python C 扩展报错，可安装基础编译依赖：`sudo apt install -y build-essential libssl-dev libffi-dev`。
> 4. 清华 PyPI 镜像地址 `https://pypi.tuna.tsinghua.edu.cn/simple` 为纯索引接口，浏览器访问解析异常属于正常现象，不影响 pip 正常使用。
> 5. Python 别名已统一写入 `.zshrc`，重启交互终端后 `python`/`pip` 命令默认指向 3.13 版本；**脚本与非交互场景不生效**，请显式使用 `python3.13`。
---
### 3.4 Go 最新稳定版（1.27.1）
从官方中文站下载固定版本安装，配置国内代理加速模块下载。
```bash
# 1. 下载 Go 1.27.1 安装包
wget https://golang.google.cn/dl/go1.27.1.linux-amd64.tar.gz

# 2. 安装到系统目录
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.27.1.linux-amd64.tar.gz

# 3. 清理安装包
rm go1.27.1.linux-amd64.tar.gz

# 4. 验证安装
go version
```
> ⚠️ **踩坑提示**：
> - `golang.google.cn` 直接 `curl -I` 看到的是 **302 跳转**（跳到 `dl.google.com`），浏览器/wget 会自动跟随，属正常现象，不是下载失败。
> - Go 环境变量已统一写入 `.zshrc`，重启终端或 `source ~/.zshrc` 后全局生效。
> - 若 `golang.google.cn` 下载失败，换阿里云镜像：`wget https://mirrors.aliyun.com/golang/go1.27.1.linux-amd64.tar.gz`
> - 升级版本时只需替换 `go1.27.1` 为目标版本号即可，无需改动其他步骤。
> - PATH 写法必须是 `$PATH:/usr/local/go/bin:$HOME/go/bin`，不要写成 `$(PATH:/usr/local/go/bin:)` 这种错误语法，否则 go 命令找不到。
---

### 3.5 Node.js 最新 LTS 版（NVM 管理）
使用 NVM 管理多版本 Node.js，国内网络下采用 Git 克隆方式安装更稳定。
#### 步骤 1：Git 克隆安装 NVM
```bash
# 清理旧残留
rm -rf ~/.nvm
# Gitee 镜像克隆稳定版
git clone https://gitee.com/mirrors/nvm.git ~/.nvm -b v0.40.1
```
#### 步骤 2：加载 NVM 配置
```bash
# NVM 加载逻辑已统一写入 .zshrc，直接生效即可
source ~/.zshrc
```
#### 步骤 3：安装 Node.js LTS
```bash
# 安装最新 LTS 并设为全局默认
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'
```
#### 步骤 4：配置国内镜像 + 安装 pnpm
```bash
# npm 国内镜像
npm config set registry https://registry.npmmirror.com
# 允许 pnpm 安装脚本（消除警告）
npm config set allow-scripts=pnpm --location=user
# 安装 pnpm
npm install -g pnpm

# 验证
node -v
npm -v
pnpm -v
```
> ⚠️ **踩坑提示**：
> 1. 官方在线安装脚本在国内常因网络超时失败；Gitee Git 克隆方式是国内最稳定的安装方案。
> 2. Zsh 中通配符 `lts/*` 会触发 `no matches found` 报错，用单引号包裹参数 `'lts/*'`；`.zshrc` 中已配置 `setopt nonomatch` 也可避免该问题。
> 3. 若之前通过 Windows 端 npm 安装过 pnpm，WSL 中可能调用到 `/mnt/c/Users/.../npm/pnpm` 而报 `node: not found`，确保 NVM 加载后再执行 `npm install -g pnpm`。
> 4. `pnpm -v` 随 npm 源更新会漂移（本文实测环境为 12.8.1），**版本号变化不代表安装失败**。
---
## 第四章 Git 基础配置
### 4.1 用户信息配置
```bash
# 替换为你的信息
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
# 设置默认分支为 main
git config --global init.defaultBranch main
```
### 4.2 生成 SSH 密钥
```bash
# 生成 ED25519 密钥（更安全、更短）
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
# 查看公钥（复制到代码托管平台即可）
cat ~/.ssh/id_ed25519.pub
```
### 4.3 HTTPS 推送认证失败：切换 SSH 方式
**现象**：`git push` 时提示 `Invalid username or token. Password authentication is not supported for Git operations.`——GitHub 已不支持 HTTPS 账号密码推送。
**解决**：
```bash
# 1. 验证 SSH 密钥已生效（首次连接提示指纹时输入 yes）
ssh -T git@github.com
# 输出 "Hi 用户名! You've successfully authenticated..." 即为成功

# 2. 将远程地址从 HTTPS 切换为 SSH
git remote set-url origin git@github.com:用户名/仓库名.git

# 3. 确认修改成功
git remote -v

# 4. 提交后再推送
git add .
git commit -m "提交信息"
git push
```
> ⚠️ **踩坑提示**：若切换后 `git push` 返回 `Everything up-to-date`，说明改动只是 `git add` 暂存了但**尚未 commit**，先执行 `git commit` 再推送。
---
## 第五章 OpenSSH 服务配置（WSL 专属坑点全解）
### 5.1 问题说明
WSL2 环境下 OpenSSH 有三大经典坑：
1. 安装后 dpkg 配置脚本调用 systemctl 失败（`Could not execute systemctl`）
2. `ssh.socket` 套接字激活模式与 WSL 网络栈不兼容，导致 `Dependency failed`
3. 22 端口常被系统残留进程占用，报 `Address already in use`

### 5.2 完整修复与配置
```bash
# 1. 安装 openssh-server
sudo apt install -y openssh-server

# 2. 强制清理所有 SSH 进程与服务
sudo systemctl stop ssh.service ssh.socket 2>/dev/null
sudo killall -9 sshd 2>/dev/null

# 3. 彻底屏蔽 socket 激活（永久避免冲突）
sudo systemctl disable ssh.socket 2>/dev/null
sudo systemctl mask ssh.socket 2>/dev/null
sudo systemctl daemon-reload

# 4. 修复 dpkg 安装残留错误
sudo dpkg --configure -a 2>/dev/null
sudo dpkg-reconfigure -f noninteractive openssh-server 2>/dev/null || true

# 5. 修改端口为 2222，彻底避开 22 端口冲突
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# 6. 启动服务并设置开机自启
sudo systemctl enable --now ssh.service

# 7. 验证运行状态
systemctl status ssh.service
```
看到输出中 `Active: active (running)` 且日志显示 `Server listening on 0.0.0.0 port 2222` 即为成功。

> ⚠️ **踩坑提示**：
> - 不要用 `sudo service ssh start` + 写入 `.bashrc` 的方式替代 systemd，会导致每次开终端都弹报错。
> - 关键是 **`mask ssh.socket`**（而不仅是 disable），否则 socket 仍会被依赖触发。
> - 端口是 **2222**，不是 22。
> - 验证命令备查：
> ```bash
> systemctl is-enabled ssh.socket   # 期望：masked
> ss -tlnp | grep 2222              # 期望：LISTEN 0.0.0.0:2222
> ```

### 5.3 Windows 端连接方式
```bash
ssh WSL用户名@localhost -p 2222
```
> 💡 不知道 WSL 用户名的话，在 WSL 中执行 `whoami` 查看。
---
## 第六章 Redis 安装与配置（WSL2 Ubuntu 24.04）
> **前置条件**：必须先完成 1.1 节 systemd 启用，否则 `systemctl` 命令无法使用，Redis 服务无法正常管理。
> 开发环境推荐用官方仓库安装最新稳定版。

### 6.1 方式一：系统源安装（简单稳定，Redis 7.0）
```bash
# 安装 Redis 服务端与客户端
sudo apt install -y redis-server redis-tools

# 启动并设置开机自启
sudo systemctl enable --now redis-server

# 验证（此时还没设密码，可直接 ping）
redis-cli ping
```
返回 `PONG` 即为成功。

### 6.2 方式二：官方仓库安装最新稳定版（推荐，Redis 8.x）
```bash
# 1. 安装依赖
sudo apt install -y curl gpg

# 2. 添加 Redis 官方 GPG 密钥
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

# 3. 添加官方 APT 仓库
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# 4. 安装最新版 Redis
sudo apt update
sudo apt install -y redis

# 5. 启动并设置开机自启
sudo systemctl enable --now redis-server

# 6. 验证版本
redis-server --version
redis-cli ping
```

### 6.3 基础配置（开发环境常用）
> ⚠️ **本节一旦执行，Redis 就带密码了**：之后所有 `redis-cli` 裸连接都会返回 `NOAUTH Authentication required`，包括第七章的验证命令（第七章已按带密码写法给出）。跳过本节的话，第七章的无密码写法同样可用。
```bash
# 备份原配置
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.bak

# 1. 设置访问密码（将 your_password 替换为你的密码）
sudo sed -i 's/# requirepass .*/requirepass your_password/' /etc/redis/redis.conf

# 2. 允许远程访问（开发环境可选，生产环境不建议）
sudo sed -i 's/^bind 127.0.0.1 -::1/bind 0.0.0.0/' /etc/redis/redis.conf

# 3. 以守护进程方式运行（WSL + systemd 环境保持默认 no 即可，由 systemd 管理）
# 如需手动后台运行可改为 yes：sudo sed -i 's/^daemonize no/daemonize yes/' /etc/redis/redis.conf

# 重启生效
sudo systemctl restart redis-server

# 确认密码已生效（必须带 -a，否则 NOAUTH）
redis-cli -a your_password ping
```

### 6.4 常用操作
```bash
# 启动 / 停止 / 重启 / 状态
sudo systemctl start redis-server
sudo systemctl stop redis-server
sudo systemctl restart redis-server
sudo systemctl status redis-server

# 连接 Redis（设置了 6.3 密码后必须带认证，否则报 NOAUTH）
redis-cli -a your_password        # 方式 1：-a 参数
# 方式 2：环境变量（无 warning，适合脚本）
# REDISCLI_AUTH=your_password redis-cli
# 方式 3：先裸连接，再手动认证
# redis-cli
# AUTH your_password

# 测试读写
SET mykey "hello"
GET mykey

# 查看所有键
KEYS *

# 查看服务信息
INFO server
```
> 💡 若**未执行 6.3**（无密码），直接 `redis-cli` 连接即可，`SET/GET` 等命令同样可用。

### 6.5 修改密码不生效的踩坑与修复
**现象**：执行 `sed` 改密码后，用新密码连接报 `AUTH failed: WRONGPASS invalid username-password pair or user is disabled.`，旧密码仍能用。
**原因**：第一次执行 `sed 's/# requirepass .*/requirepass your_password/'` 时，已把注释行 `# requirepass ...` 改成不带 `#` 的 `requirepass your_password`；第二次再用**同样的模式 `# requirepass .*`** 去匹配时，找不到带 `#` 的行，替换静默失败，Redis 运行时仍是旧密码。
**修复**：
```bash
# 1. 先查看当前配置文件里的密码行
grep -n requirepass /etc/redis/redis.conf

# 2. 用能同时匹配带/不带 # 的 sed 模式替换
sudo sed -i 's/^#\?requirepass .*/requirepass 新密码/' /etc/redis/redis.conf

# 3. 确认修改成功
grep -n requirepass /etc/redis/redis.conf

# 4. 重启 Redis 让新密码生效
sudo systemctl restart redis-server

# 5. 用新密码验证
redis-cli -a 新密码 ping
```
返回 `PONG` 即为成功。

> ⚠️ **踩坑提示**：
> 1. **WSL 必须先启用 systemd**（见文档 1.1 节），否则 `systemctl` 命令无法使用，Redis 服务无法正常管理。
> 2. **设置密码后**，`redis-cli` 直接连接执行命令会报 `NOAUTH Authentication required`，需用 `-a` 参数、`REDISCLI_AUTH` 环境变量或连接后执行 `AUTH`——**第七章的验证命令也遵守这条**。
> 3. **开启远程访问后**，Windows 端可通过 `localhost:6379` 连接 WSL 里的 Redis，但务必设置密码，避免暴露无密码实例。
> 4. **官方仓库安装的包名是 `redis`**（不是 `redis-server`），会同时安装服务端和客户端；系统源安装则需要分别装 `redis-server` 和 `redis-tools`。
> 5. 配置文件修改后必须执行 `sudo systemctl restart redis-server` 才会生效；以后改密码记住两点：① sed 模式要能匹配到当前实际的行（带不带 `#`）；② 改完必须重启。
---
## 第七章 全环境验证
> ⚠️ 请在**登录后的 zsh 交互终端**中执行本脚本：`python`/`pip` 是 zsh 别名，写进脚本或在 bash 里跑会 `command not found`。
> 若执行过 6.3 设置了 Redis 密码，把下面的 `your_password` 换成真实密码（或先 `export REDISCLI_AUTH=你的密码`）。
```bash
echo "=== 开发环境全景 ==="
echo -e "\nC/C++:"
gcc --version | head -n1
g++ --version | head -n1
echo -e "\nJava:"
java -version 2>&1 | head -n1
javac -version
echo -e "\nPython:"
python --version
pip --version
echo -e "\nGo:"
go version
echo -e "\nNode.js:"
node -v
npm -v
pnpm -v
echo -e "\nGit:"
git --version
echo -e "\nSSH 服务:"
systemctl status ssh.service | grep Active
echo -e "\nRedis:"
redis-server --version
REDISCLI_AUTH=your_password redis-cli ping
echo -e "\nZsh:"
zsh --version
```

**预期输出参考**（版本号会随源更新轻微漂移，`git`/`zsh`/`gcc` 等系统包以实际为准）：
```
C/C++:
gcc (Ubuntu 14.2.0-4ubuntu2~24.04.1) 14.2.0
g++ (Ubuntu 14.2.0-4ubuntu2~24.04.1) 14.2.0
Java:
openjdk version "21.0.12.1" 2026-08-18
javac 21.0.12.1
Python:
Python 3.13.16
pip 26.2.1 from /home/yewwsl/.local/lib/python3.13/site-packages/pip (python 3.13)
Go:
go version go1.27.1 linux/amd64
Node.js:
v24.21.0
11.19.0
12.8.1
Git:
git version 2.43.0
SSH 服务:
     Active: active (running) since ... CST; ... ago
Redis:
Redis server v=8.10.2 sha=00000000:1 malloc=jemalloc-5.3.0 bits=64 build=677c1e5d953828c4
PONG
Zsh:
zsh 5.9 (x86_64-ubuntu-linux-gnu)
```

> 💡 两个容易误判的点：
> - **Redis 一行若显示 `NOAUTH Authentication required.`**：说明 6.3 的密码生效了而本脚本没带认证——按上面的写法带上 `REDISCLI_AUTH` 即可，不是安装失败。
> - **`pnpm -v` 与 12.8.1 不同**：pnpm 从 npm 源安装，版本持续更新，属正常漂移。
---
## 第八章 常见问题排错
### 8.1 命令不存在时弹出 Python 报错
> ⚠️ **先确认你是否真的会遇到**：本教程（3.3 节）**不修改系统 `python3`**，只在 `.zshrc` 加用户级别名。只要 `python3 --version` 仍是 `3.12.x`、`/usr/lib/command-not-found` 首行仍是 `#!/usr/bin/python3`，**就不会触发本问题，也无需执行本节修复**。只有你曾用 `update-alternatives` 把系统 `python3` 切到 3.13 时才需要修。

**现象**：输入错误命令时，抛出 `ModuleNotFoundError: No module named 'apt_pkg'`
**原因**：系统 `command-not-found` 工具依赖 Python 3.12 的 `apt_pkg` 模块（`apt_pkg` 只装在 3.12 的 `dist-packages` 下），将默认 `python3` 切换到 3.13 后不兼容。
**自查**：
```bash
python3 --version                 # 若已是 3.13.x → 说明系统 python3 被切过
head -1 /usr/lib/command-not-found   # 期望 #!/usr/bin/python3
python3.12 -c "import apt_pkg" && echo ok
```
**解决**（把 shebang 固定到 3.12，绕开被改过的 `python3`）：
```bash
sudo sed -i '1s|.*|#!/usr/bin/python3.12|' /usr/lib/command-not-found
```
> 💡 更彻底的做法是把系统 `python3` 切回 3.12（`update-alternatives` 若无可选项则说明从未切换过，无需处理）。
---
### 8.2 Zsh 中 `#` 开头的注释被当作命令报错
**现象**：在 Zsh 交互终端输入或粘贴 `# 注释` 行时，报 `zsh: command not found: #`
**真正原因**：Zsh 默认**关闭**了 `interactive_comments` 选项，交互模式下 `#` 不会被识别为注释（与 Bash 不同）。并非"复制粘贴换行异常"。
**解决**：在 `.zshrc` 中加入：
```bash
setopt interactive_comments
```
本教程第二章的完整配置已包含此行。
---
### 8.3 NVM 命令找不到
**原因**：NVM 是 shell 函数，必须在配置文件中 source 加载后才可用。
**解决**：确认 `.zshrc` 中有 NVM 加载代码（见第二章 2.4 节），执行 `source ~/.zshrc` 生效。若 `~/.nvm` 目录不存在或为空，重新用 Git 克隆方式安装。
---
### 8.4 SSH 启动失败：Address already in use
**原因**：22 端口被残留进程占用，或 `ssh.socket` 激活模式冲突。
**解决**：按第五章步骤完整执行——`mask ssh.socket` + 更换端口为 2222 + `systemctl enable --now ssh.service`。
---
### 8.5 Go 命令找不到
**原因**：PATH 中未包含 `/usr/local/go/bin`，或 PATH 写法错误。
**解决**：确认 `.zshrc` 中是 `export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin`，然后 `source ~/.zshrc`。
---
### 8.6 pip 安装的命令 command not found
**原因**：`python3.13 -m pip install xxx` 的可执行文件装到 `~/.local/bin`，Ubuntu 24.04 默认 PATH 不含该目录。
**解决**：确认 `.zshrc` 中有 `export PATH=$HOME/.local/bin:$PATH`（见 2.4 节），然后 `source ~/.zshrc`；验证 `pip3.13 --version` 可直接执行。
---
## 第九章 WSL2 性能优化建议
### 9.1 限制内存与 CPU
在 Windows 用户目录（`C:\Users\你的用户名\`）创建 `.wslconfig` 文件：
```ini
[wsl2]
memory=8GB
swap=4GB
processors=4
```
Windows 终端执行 `wsl --shutdown` 后重启 WSL 生效。

### 9.2 文件系统优化
项目代码放在 WSL 原生目录（`~/` 下），不要放在 `/mnt/c/`，IO 性能差距可达 10 倍以上。

### 9.3 磁盘空间回收
分两层，**Linux 侧清理并不能缩小 WSL 虚拟磁盘（ext4.vhdx）占用的 Windows 磁盘**：
```bash
# 1) Linux 侧：清理 apt 缓存与无用依赖（只释放 WSL 内部可用空间）
sudo apt autoremove -y && sudo apt clean
```
```powershell
# 2) Windows 侧：回收 vhdx 实际占用（在 PowerShell 中执行，需先 wsl --shutdown）
wsl --shutdown

# 方式 A：启用稀疏磁盘（Windows 11 / 新版 WSL 支持，自动回收）
wsl --manage Ubuntu-24.04 --set-sparse true

# 方式 B：手动 compact（任意 WSL 版本可用，路径按发行版名调整）
# wsl --shutdown 后，管理员 PowerShell：
#   wsl --export Ubuntu-24.04 D:\wsl-backup\ubuntu.tar   # 或直接 diskpart compact vhd
# diskpart:
#   select vdisk file="C:\Users\你的用户名\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu24.04LTS_79rhkp1fndgsc\LocalState\ext4.vhdx"
#   compact vdisk
```
> 💡 `wsl --manage <发行版名> --set-sparse true` 是最省事的做法；老版本 WSL 用 diskpart 的 `compact vdisk`。执行前先在 WSL 内 `sudo apt clean && sudo apt autoremove`，回收效果更好。
---
## 附录：完整执行顺序速查
```
1.1 启用 systemd → wsl --shutdown → 重启验证
1.2 换清华源（改 ubuntu.sources，勿只写 sources.list）→ apt update && full-upgrade
1.3 安装基础工具合集
2.1 安装 Zsh
2.2 安装 Oh My Zsh（[Y/n] 直接回车即可；克隆仍走 GitHub，超时用 REMOTE=gitee）
2.3 安装插件
2.4 写入完整 .zshrc（含 ~/.local/bin 入 PATH）
2.5 启用插件 + 设置主题 + source（未装 Docker 可去掉 docker 插件）
3.1 GCC 14
3.2 Java 21
3.3 Python 3.13（含 pip 清华源；勿用 update-alternatives 配 pip）
3.4 Go（含国内代理）
3.5 NVM + Node.js + pnpm
4.1 Git 用户信息
4.2 SSH 密钥
4.3 HTTPS 推送切换 SSH
5.2 OpenSSH 完整修复
6.2 Redis 官方仓库安装 + 6.3 基础配置（设置密码后验证需带 -a / REDISCLI_AUTH）
8.1 修复 command-not-found（仅当你曾把系统 python3 切到 3.13 才需要）
第七章 全环境验证（在登录后的 zsh 交互终端执行，Redis 带密码 ping）
9.3 磁盘回收（可选：apt clean + wsl --set-sparse / compact vhdx）
```
---
至此，WSL2 Ubuntu 24.04 全栈开发环境配置完成，涵盖后端、前端、系统开发所需的全部基础工具链，所有国内网络优化与踩坑点均已处理。