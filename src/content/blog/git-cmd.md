---
title: "Git 最全面命令大全"
date: "2026-10-05T00:00:00+08:00"
description: "Git 开发全流程命令速查与常见问题解决方案"
categories:
  - "开发工具"
tags:
  - "git"
  - "版本控制"
  - "命令行"
---

# Git 最全面命令大全（开发流程 + 常见问题解决方案）

> 本文整合日常开发、团队协作、远程仓库管理、分支操作、撤销回退、文件追踪、`.gitignore` 管理等全部常用场景，并附带高频报错的解决方案。
>
> **约定**：默认远程名为 `origin`，默认主分支为 `main`（如果你的仓库是 `master`，自行替换即可）。
>
> **一句话记忆工作流**：`pull → branch → add -A → commit → push → PR`，出问题先 `git status` 和 `git log --oneline --graph` 看现场。

---

## 一、基础配置（首次使用必做）

```bash
# 设置全局身份（所有仓库生效，提交记录里的作者信息）
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 查看所有配置 / 查看单个配置
git config --list
git config user.name
git config --local --list        # 只看当前仓库的配置

# 默认分支名设为 main（git init 时生效）
git config --global init.defaultBranch main

# 命令自动着色（新版本 Git 默认开启）
git config --global color.ui true

# Windows 换行符处理（可选，团队协作跨平台时建议配置）
git config --global core.autocrlf true

# 设置命令别名（提高效率）
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit

# 取消 Git 的 HTTP 代理（代理出问题时用）
git config --global --unset http.proxy

# pull 默认使用 rebase（配合第七节，可选但推荐）
git config --global pull.rebase true
```

> 💡 `--global` 作用于当前用户的所有仓库；不加则是单个仓库级别（`--local`），仓库级配置优先级更高。

---

## 二、创建 / 克隆仓库

```bash
git init                              # 当前目录初始化为 Git 仓库
git clone <远程地址>                   # 克隆远程仓库
git clone -b 分支名 <地址>             # 克隆指定分支
git clone <地址> 本地目录名            # 克隆并指定本地目录名
git clone --depth 1 <地址>            # 浅克隆（只要最近一次提交，速度快）
```

---

## 三、远程仓库管理

```bash
git remote -v                          # 查看远程仓库（fetch/push 地址）
git remote show origin                 # 查看远程详细信息（含分支跟踪关系）
git remote add origin <URL>            # 添加远程仓库
git remote set-url origin <新URL>      # 修改远程地址（报 already exists 用这个）
git remote set-url --push origin <URL> # 只改 push 地址
git remote rename origin upstream      # 重命名远程
git remote remove origin               # 删除远程
```

两种协议示例：

```bash
# HTTPS（需 Personal Access Token，见常见问题 7）
git remote add origin https://github.com/用户名/仓库名.git

# SSH（推荐，免密）
git remote add origin git@github.com:用户名/仓库名.git
```

首次推送与上游绑定：

```bash
git branch -M main                              # 本地分支改名为 main
git push -u origin main                         # 首次推送并绑定上游（之后直接 git push）
git branch --set-upstream-to=origin/main main   # 已有分支补绑上游
git branch -vv                                  # 查看本地分支与远程的跟踪关系
```

---

## 四、日常提交（工作区 → 暂存区 → 版本库）

```bash
git status                    # 查看状态（最常用）
git status --short            # 简短输出

git add 文件名                # 暂存单个文件
git add .                     # 暂存当前目录的变更
git add -A                    # 暂存所有变更（含删除/重命名/新文件）⭐ 推荐
git add -u                    # 只暂存已跟踪文件的修改和删除

git commit -m "说明"          # 提交
git commit -am "说明"         # add + commit 一步（仅限已跟踪文件）
git commit --amend -m "新说明" # 修改上一次提交（未推送时用）
git commit --amend --no-edit  # 把漏掉的文件补进上一次提交

git rm 文件名                 # 删除文件并暂存删除
git rm -r 目录名              # 递归删除目录
git rm --cached 文件名        # 只从版本库移除，保留本地文件
```

> ⚠️ **重点**：改名/移动文件后不要用 `git commit -am`——它不会包含未跟踪的新路径文件，会导致只提交了删除、丢新文件。统一用 `git add -A`。

差异对比：

```bash
git diff                          # 工作区 vs 暂存区
git diff --cached                 # 暂存区 vs 最新提交（--staged 是等价别名）
git diff HEAD                     # 工作区+暂存区 vs 最新提交
git diff --stat                   # 只看改了哪些文件（不显示内容）
git diff HEAD^                    # 与上一个版本比较
git diff HEAD -- ./lib            # 与 HEAD 比较 lib 目录
git diff origin/main..main        # 本地比远程多出的内容
git diff 分支1..分支2              # 两个分支间的差异
```

---

## 五、文件 / 文件夹改名与移动

```bash
# 推荐：git 自动暂存重命名
git mv old.txt new.txt
git mv old_dir new_dir
git commit -m "重命名"
git push

# 已经用系统命令 mv 改完了
git add -A
git status                        # 通常显示 renamed: old -> new
git commit -m "重命名"
git push

git log --follow -- 新路径/文件    # 追踪改名前的历史
```

> 💡 要点：
> - Git 跟踪的是**文件**而不是文件夹，空文件夹不会被跟踪（可放一个 `.gitkeep` 占位）。
> - Windows / macOS 文件系统默认不区分大小写，只改大小写需要两步：
>   ```bash
>   git mv readme.md readme.tmp
>   git mv readme.tmp README.md
>   # 或一步强制
>   git mv -f readme.md README.md
>   ```

---

## 六、分支管理

```bash
git branch                        # 查看本地分支
git branch -a                     # 查看所有分支（含远程）
git branch -r                     # 只看远程分支
git branch -vv                    # 查看分支及跟踪关系、领先/落后情况
git branch --merged               # 已合并到当前分支的分支
git branch --no-merged            # 未合并到当前分支的分支
git branch --contains 提交ID      # 包含某次提交的分支

git switch -c 分支名              # 创建并切换（推荐，Git 2.23+）
git switch 分支名                 # 切换分支
git switch -                      # 切回上一个分支（类似 cd -）⭐
git checkout -b 分支名            # 创建并切换（旧写法）
git checkout 分支名               # 切换分支（旧写法）
git checkout -b devel origin/develop   # 基于远程分支建本地分支并切换
git checkout --track origin/分支名     # 检出远程分支并建立跟踪

git branch -m old new             # 本地分支改名
git branch -M main                # 强制改名为 main

git merge 分支名                  # 合并指定分支到当前分支
git merge --no-ff 分支名          # 禁用快进合并，保留分支拓扑

git rebase 分支名                 # 变基到指定分支
git rebase origin/main            # 变基到远程 main
git rebase -i HEAD~3              # 交互式整理最近 3 次提交

git cherry-pick 提交ID            # 把某次提交单独摘到当前分支

git branch -d 分支名              # 删除已合并分支（安全）
git branch -D 分支名              # 强制删除（未合并也删）
git push origin --delete 分支名   # 删除远程分支
git push origin :分支名           # 删除远程分支（旧语法）
```

---

## 七、远程同步：fetch / pull / push

```bash
git fetch origin                  # 只拉取不合并（最安全，先看再合）
git fetch --all --prune           # 拉取全部远程并清理已删除的远程分支引用

git pull                          # 拉取并合并（= fetch + merge）
git pull origin main              # 拉取指定远程分支并合并
git pull --rebase origin main     # 拉取并变基，历史更干净 ⭐ 推荐
git pull --rebase --autostash     # 有未提交修改时自动 stash 再恢复（神器）

git push                          # 推送（已绑定上游时）
git push origin main              # 推送到指定远程分支
git push -u origin main           # 首次推送并绑定上游
git push --tags                   # 推送所有标签
git push --force-with-lease       # 安全强推（远程有他人提交会拒绝）⭐
git push --force                  # 强推（危险，可能覆盖他人工作）

git log origin/main..main         # 本地领先远程的提交
git log main..origin/main         # 远程领先本地的提交
```

---

## 八、暂存工作进度（stash）

场景：改到一半需要切分支处理别的事，又不想提交半成品。

```bash
git stash                         # 暂存当前修改，工作区回到 HEAD
git stash push -m "说明"          # 带说明暂存（推荐）
git stash list                    # 查看暂存列表
git stash show -p 'stash@{0}'     # 查看某次暂存的具体改动
git stash pop                     # 恢复最近一次并删除记录
git stash apply 'stash@{0}'       # 恢复指定暂存但保留记录
git stash drop 'stash@{0}'        # 删除指定暂存记录
git stash clear                   # 清空所有暂存
```

**提交错分支的补救**：

```bash
git stash
git switch 正确分支
git stash pop
```

---

## 九、查看历史与排查

```bash
git log                                    # 完整提交历史
git log --oneline                          # 一行一条
git log --oneline --graph --decorate --all # 图形化全部分支历史 ⭐
git log -n 5                               # 只看最近 5 条
git log --stat                             # 每次提交改了哪些文件
git log -p                                 # 每次提交的具体改动内容
git log --follow -- 文件名                 # 追踪文件改名前的历史
git log --pretty=format:'%h %s' --graph    # 自定义格式 + 图形化
git log 'main@{yesterday}'                 # 查看分支昨天的状态（按时间）

git show 提交ID                            # 查看某次提交详情（ID 可只写前几位）
git show HEAD                              # 最近一次提交
git show HEAD^                             # 上一次提交（^^ 上两次，~5 上五次）
git show v1.0.0                            # 查看某个标签对应的提交

git blame 文件名                           # 逐行查看是谁改的
git grep "关键字"                          # 在版本库文件中搜索文本

git reflog                                 # 所有 HEAD 操作记录（救命命令）⭐
git show 'HEAD@{5}'                        # 查看 reflog 中第 5 步的状态

git show-branch --all                      # 图示所有分支历史
git log --raw --no-merges                  # 提交历史对应的文件变更（whatchanged 已废弃，官方等价替代）
```

---

## 十、撤销与回退

### 按场景选择

| 场景 | 命令 | 影响 |
|---|---|---|
| 丢弃工作区修改（未 add） | `git restore 文件` / `git checkout -- 文件` | 只丢工作区 |
| 撤出暂存区（已 add 未 commit） | `git restore --staged 文件` / `git reset HEAD 文件` | 改动回到工作区 |
| 撤销最近一次提交 | `git reset --soft HEAD~1` | 改动留在暂存区 |
| 撤销提交并回到工作区 | `git reset --mixed HEAD~1` | 改动留在工作区 |
| 彻底回退（丢代码） | `git reset --hard HEAD~1` | ⚠️ 改动全丢 |
| 撤销**已推送**的提交 | `git revert 提交ID` | 生成新提交抵消，安全 ⭐ |
| 放弃进行中的合并 | `git merge --abort` | 干净地回到合并前 |
| 放弃合并失败的现场 | `git reset --hard HEAD` | 回到合并前（`--abort` 无效时用） |

### 补充命令

```bash
git revert 提交ID                # 反向生成新提交（已推送的代码用这个，不要 reset）
git clean -n                     # 预演：看看会删哪些未跟踪文件
git clean -fd                    # 删除未跟踪的文件和目录 ⚠️ 慎用
git push --force-with-lease      # 改完历史后安全强推
```

> ⚠️ 公共分支（多人协作）上**只准用 `revert`**，`reset` + 强推会破坏他人的仓库。

### 找回误删的提交 / 误 reset 的代码

```bash
git reflog                       # 找到丢失提交的 ID
git reset --hard <提交ID>        # 直接回到那个状态
# 或
git branch 救援分支 <提交ID>     # 给它建个分支慢慢看
```

---

## 十一、标签管理

```bash
git tag                           # 查看所有标签
git tag v1.0.0                    # 轻量标签
git tag -a v1.0.0 -m "版本说明"   # 附注标签（推荐，含说明和作者）
git show v1.0.0                   # 查看标签详情
git push origin v1.0.0            # 推送单个标签
git push origin --tags            # 推送全部标签
git tag -d v1.0.0                 # 删除本地标签
git push origin --delete v1.0.0   # 删除远程标签
```

---

## 十二、`.gitignore` 与文件追踪（含 `.idea` 实战）

### 查看追踪状态

```bash
git ls-files                              # 查看被 Git 追踪的所有文件 ⭐
git ls-files -s                           # 详细模式
git ls-files 文件名                       # 有输出 = 被追踪
git ls-files --others --exclude-standard  # 查看未跟踪且未被忽略的文件
git ls-tree -r HEAD                       # 查看版本库里的文件树
git check-ignore -v 文件名                # 查看文件被哪条 ignore 规则命中
```

### 取消追踪但保留本地文件

```bash
git rm --cached 文件名
git rm -r --cached 目录名
```

> ⚠️ **新加 `.gitignore` 不会自动忽略已被追踪的文件**，必须先 `git rm -r --cached` 再提交。

### 实战：取消追踪 `.idea`（Windows PowerShell）

`.idea` 是 JetBrains IDE 的项目配置目录，不应进版本库。若已被误提交，按以下步骤处理（**只删 Git 追踪记录，本地 IDE 配置原样保留**）：

```powershell
# 1. 取消追踪（-r 递归，--cached 只移除索引）
git rm -r --cached .idea

# 2. 在 .gitignore 中加入 .idea/（见下方模板）

# 3. 提交并推送
git add -A
git commit -m "取消追踪 .idea 文件夹"
git push origin main

# 4. 验证：输出中不再有 .idea/ 开头的文件
git ls-files
```

**防止 IDEA 再次自动加入追踪**：`File → Settings → Version Control → Confirmation → When files are created` 选择 **Do not add**。

### 标准 `.gitignore` 模板（Java / IDEA 项目）

```gitignore
# IDE
.idea/
*.iml
.vscode/

# 构建产物
target/
out/
build/
*.class

# 日志与临时文件
*.log
*.tmp

# 系统文件
.DS_Store
Thumbs.db

# 依赖目录（前端）
node_modules/
```

> 💡 如果删除了 `.gitignore`，Git 不会再自动忽略编译产物、日志、临时文件，后续提交需手动筛选，避免把无关文件推送到远程。

---

## 十三、标准团队协作开发流程

```bash
# 1. 从主分支拉最新代码
git switch main
git pull --rebase origin main

# 2. 开功能分支
git switch -c feature/login

# 3. 开发，小步提交
git status
git add -A
git commit -m "feat: 登录接口"

# 4. 推送前同步主分支最新代码（减少冲突）
git fetch origin
git rebase origin/main

# 5. 推送功能分支
git push -u origin feature/login

# 6. 在 GitHub/GitLab 上发 Pull Request → Code Review → 合并

# 7. 合并后清理
git switch main
git pull --rebase origin main
git branch -d feature/login
git push origin --delete feature/login

# 8. 需要发布时打标签
git tag -a v1.0.0 -m "v1.0.0 发布"
git push origin v1.0.0
```

提交信息建议遵循 Conventional Commits 规范，详见下一章。

---

## 十四、提交信息规范（Conventional Commits）

规范提交信息的好处：历史可读、可按 type 检索、能自动生成 CHANGELOG、配合 SemVer 自动定版本号。推荐业界事实标准 [Conventional Commits](https://www.conventionalcommits.org/zh-hans/)（Angular 提交规范的通用化）。

### 格式

```
<type>(<可选 scope>): <简短描述>
                             ← 空行
<可选正文：说明为什么改、怎么改>
                             ← 空行
<可选脚注：BREAKING CHANGE / 关联 issue>
```

### 常用 type

| type | 用途 | 示例 |
|---|---|---|
| `feat` | 新功能（SemVer minor） | `feat(user): 新增微信登录` |
| `fix` | 修复 bug（SemVer patch） | `fix: 修复分页越界` |
| `docs` | 文档变更 | `docs: 更新 README 安装说明` |
| `style` | 代码格式（空格、分号，不影响逻辑） | `style: 统一两空格缩进` |
| `refactor` | 重构（非新功能、非修 bug） | `refactor: 抽取公共校验逻辑` |
| `perf` | 性能优化 | `perf: 列表改为虚拟滚动` |
| `test` | 测试相关 | `test: 补充登录接口单测` |
| `build` | 构建系统 / 依赖变更 | `build: 升级 vite 到 7.x` |
| `ci` | CI 配置变更 | `ci: 新增自动部署 workflow` |
| `chore` | 其他杂项 | `chore: 清理无用依赖` |
| `revert` | 回退提交 | `revert: 回滚 feat: 新增微信登录` |

### 示例

```bash
# 单行（最常用）
git commit -m "feat(login): 支持手机号验证码登录"

# 多行：第一个 -m 是标题，后续 -m 依次是正文段落
git commit -m "feat: 支持深色模式" -m "跟随系统主题自动切换，设置页可手动覆盖。

Close #123"

# 不兼容变更：type 后加 !，或脚注写 BREAKING CHANGE（触发 SemVer major）
git commit -m "feat!: 下线 v1 旧接口" -m "BREAKING CHANGE: /api/v1/* 全部移除，请迁移到 /api/v2"
```

### 书写规则

- 标题 ≤ 50 字，用祈使句（"新增"而不是"新增了"），结尾不加句号。
- 一个提交只做一件事（小步提交）；正文解释"为什么"而非"是什么"，每行 ≤ 72 字。
- 关联 issue 用 `Close #123` / `Fix #123`，合并后自动关闭对应 issue。

> 💡 想在团队里强制执行：用 **commitlint** + **husky** 在提交时自动校验（不规范直接拒绝），用 **Commitizen**（`git cz`）交互式生成规范信息。

---

## 十五、常见问题解决方案速查

### 1. `! [rejected] main -> main (fetch first)` ⭐

远程有你本地没有的提交（常见原因：建仓库时勾选了自动生成 README）。

```bash
git pull origin main --rebase --allow-unrelated-histories
git push -u origin main
```

### 2. `refusing to merge unrelated histories`

两条历史没有共同祖先（本地 init + 远程建仓库各有一条初始提交）。

```bash
git pull origin main --allow-unrelated-histories
```

### 3. `untracked working tree files would be overwritten by merge`

本地未跟踪文件与远程文件冲突（如 `.idea/`）。先把本地文件移走或删除再 pull：

```powershell
git rebase --abort                      # 先中止中断的 rebase
Move-Item .idea .idea_backup            # 备份（确认无用可 Remove-Item -Recurse -Force .idea）
git pull origin main --rebase --allow-unrelated-histories
git push -u origin main
```

### 4. rebase 中途冲突 / 想放弃

```bash
# 解决冲突文件后
git add -A
git rebase --continue

# 跳过当前提交 / 彻底放弃回到 rebase 前
git rebase --skip
git rebase --abort
```

### 5. merge / pull 冲突

```bash
git pull
# 打开冲突文件，处理 <<<<<<< ======= >>>>>>> 标记
git add -A
git commit -m "解决冲突"      # rebase 流程则用 git rebase --continue
```

### 6. `remote origin already exists`

```bash
git remote set-url origin 新地址     # 推荐：直接改地址
# 或
git remote remove origin
git remote add origin 新地址
```

### 7. HTTPS 推送提示要密码（认证失败）

GitHub 已不支持账号密码认证，两种方案：

- **Personal Access Token**：GitHub → `Settings → Developer settings → Personal access tokens` 生成，推送时密码栏填 Token。
- **改用 SSH（推荐）**：

```bash
ssh-keygen -t ed25519 -C "你的邮箱"     # 一路回车生成密钥
# 把 ~/.ssh/id_ed25519.pub 的内容添加到 GitHub → Settings → SSH and GPG keys
git remote set-url origin git@github.com:用户名/仓库名.git
ssh -T git@github.com                   # 验证连接
```

### 8. 本地分支没有关联远程 / 推送提示 no upstream

```bash
git push -u origin main
# 或已存在的分支补绑
git branch --set-upstream-to=origin/main main
```

### 9. 拉取远程新建的分支到本地

```bash
git fetch origin
git switch -c 本地分支名 origin/远程分支名
```

### 10. 修改最后一次提交（已推送过）

```bash
git commit --amend -m "新说明"
git push --force-with-lease      # 已推送则必须强推，用安全版
```

### 11. 忽略文件不生效

文件已被追踪时 `.gitignore` 对其无效，先取消追踪：

```bash
git rm -r --cached 文件或目录
git add -A
git commit -m "取消追踪"
git push
```

### 12. 推送被拒想强制覆盖（⚠️ 危险）

```bash
git push --force-with-lease      # 远程若有他人新提交会拒绝，比 --force 安全
```

---

## 十六、维护与底层命令（了解即可）

```bash
git gc                       # 垃圾回收：压缩对象数据库，清理冗余
git fsck                     # 检查仓库完整性，找出损坏/悬空对象
git rev-parse HEAD           # 查看某个引用对应的完整 SHA1
git rev-parse v1.0.0         # 查看标签对应的 SHA1
git ls-tree -r HEAD          # 显示某次提交的文件树
git count-objects -v         # 查看仓库对象统计
```

---

## 十七、最常用命令速查表

| 场景 | 命令 |
|---|---|
| 查看状态 | `git status` |
| 添加所有变更 | `git add -A` |
| 提交 | `git commit -m "说明"` |
| 查看远程 | `git remote -v` |
| 添加远程 | `git remote add origin URL` |
| 修改远程地址 | `git remote set-url origin URL` |
| 首次推送 | `git push -u origin main` |
| 拉取（变基） | `git pull --rebase origin main` |
| 只拉取不合并 | `git fetch origin` |
| 查看分支 | `git branch -a` |
| 创建并切换分支 | `git switch -c 分支名` |
| 合并分支 | `git merge 分支名` |
| 变基 | `git rebase origin/main` |
| 摘取单个提交 | `git cherry-pick 提交ID` |
| 暂存工作 | `git stash` |
| 恢复暂存 | `git stash pop` |
| 查看历史 | `git log --oneline --graph --all` |
| 查看操作记录 | `git reflog` |
| 查看追踪文件 | `git ls-files` |
| 取消追踪 | `git rm -r --cached 目录` |
| 重命名文件 | `git mv old new` |
| 撤销已推送提交 | `git revert 提交ID` |
| 回退到某提交 | `git reset --hard 提交ID` |
| 丢弃工作区修改 | `git restore 文件` |
| 查看改名历史 | `git log --follow -- 文件` |
| 打标签 | `git tag -a v1.0.0 -m "说明"` |
| 安全强推 | `git push --force-with-lease` |
| 删除远程分支 | `git push origin --delete 分支名` |

---

> 实际使用时，把 `origin`、`main`、`分支名`、`文件路径` 替换成你自己的即可。遇到问题记住三件套：先 `git status` 看现场，再 `git log --oneline --graph` 看历史，rebase 卡住就 `git rebase --abort` 回到起点。
