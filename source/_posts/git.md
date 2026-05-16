# Git与GitHub操作笔记

## 目录

1. Git基础概念
2. Git安装与配置
3. Git核心操作
4. 分支管理
5. GitHub操作
6. 高级操作
7. 常见问题与解决方案
8. 最佳实践

------

## 1.Git基础概念

### 版本控制系统

- **分布式**：每个开发者都有完整的仓库副本
- **快照式**：记录文件状态而非差异

### 核心区域

1. **工作区 (Working Directory)**：本地文件系统
2. **暂存区 (Staging Area)**：准备提交的变更
3. **本地仓库 (Local Repository)**：完整的项目历史
4. **远程仓库 (Remote Repository)**：共享的中央仓库（如GitHub）

### 文件状态

- **未跟踪 (Untracked)**：新文件，未被Git管理
- **已修改 (Modified)**：文件内容已更改
- **已暂存 (Staged)**：变更已添加到暂存区
- **已提交 (Committed)**：变更已永久存储到本地仓库

------

## 2.Git安装与配置

### 安装

- Windows：下载 Git for Windows
- macOS：`brew install git`
- Linux：`sudo apt install git`

### 基础配置

bash

复制

```bash
# 设置用户名
git config --global user.name "Aster"

# 设置邮箱
git config --global user.email "1563544148@qq.com"

# 设置默认编辑器
git config --global core.editor "code --wait"  # VS Code

# 查看配置
git config --list
```

### 常用配置项

bash

复制

```bash
# 启用彩色输出
git config --global color.ui auto

# 设置默认分支名（新仓库）
git config --global init.defaultBranch main

# 设置换行符处理（Windows）
git config --global core.autocrlf true

# 设置换行符处理（Linux/macOS）
git config --global core.autocrlf input
```

------

## 3.Git核心操作

### 创建仓库

bash

复制

```bash
# 初始化新仓库
git init

# 克隆现有仓库
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git  # SSH方式
```

### 添加与提交

bash

复制

```bash
# 添加单个文件
git add filename.txt

# 添加所有变更（包括新文件）
git add -A
# 或
git add .

# 添加已跟踪文件的变更（不包括新文件）
git add -u

# 交互式添加（选择部分变更）
git add -p

# 提交变更
git commit -m "描述性提交信息"

# 修改最后一次提交
git commit --amend
```

### 查看状态

bash

复制

```bash
# 查看当前状态
git status

# 简洁状态
git status -s

# 查看提交历史
git log

# 图形化查看历史
git log --graph --oneline --all
```

### 撤销操作

bash

复制

```bash
# 撤销工作区修改（危险！不可恢复）
git checkout -- filename

# 取消暂存
git reset HEAD filename

# 撤销最近一次提交（保留修改）
git reset --soft HEAD~1

# 完全撤销提交（丢弃修改）
git reset --hard HEAD~1
```

------

## 4.分支管理

### 基础操作

bash

复制

```bash
# 创建分支
git branch new-feature

# 切换分支
git checkout new-feature
# 或（创建并切换）
git checkout -b new-feature

# 查看所有分支
git branch -a

# 删除分支
git branch -d feature-old
```

### 合并与冲突解决

bash

复制

```bash
# 合并分支（快速前进）
git merge feature-branch

# 合并分支（创建合并提交）
git merge --no-ff feature-branch

# 冲突解决步骤：
# 1. 编辑冲突文件（标记为 <<<<<<<, =======, >>>>>>>）
# 2. 添加解决后的文件
git add resolved-file.txt
# 3. 完成合并
git commit
```

### 变基操作

bash

复制

```bash
# 变基（重写提交历史）
git checkout feature
git rebase main

# 交互式变基（修改历史提交）
git rebase -i HEAD~3
```

------

## 5.GitHub操作

### 远程仓库管理

bash

复制

```bash
# 添加远程仓库
git remote add origin https://github.com/user/repo.git

# 查看远程仓库
git remote -v

# 重命名远程仓库
git remote rename origin upstream

# 删除远程仓库
git remote remove origin
```

### 推送与拉取

bash

复制

```bash
# 首次推送（设置上游分支）
git push -u origin main

# 后续推送
git push

# 拉取远程变更
git pull

# 强制推送（谨慎使用）
git push -f
```

### Fork与协作

1. Fork目标仓库到自己的GitHub账户
2. 克隆自己的仓库副本

bash

复制

```bash
git clone git@github.com:yourname/repo.git
```

1. 添加上游仓库

bash

复制

```bash
git remote add upstream git@github.com:original/repo.git
```

1. 同步上游变更

bash

复制

```bash
git fetch upstream
git merge upstream/main
```

### Pull Request流程

1. 创建新分支开发功能
2. 推送分支到自己的仓库
3. 在GitHub创建Pull Request
4. 审查讨论代码
5. 合并到目标分支

------

## 高级操作

### 储藏变更

bash

复制

```bash
# 储藏当前工作
git stash

# 查看储藏列表
git stash list

# 应用最新储藏
git stash apply

# 应用特定储藏
git stash apply stash@{1}

# 删除储藏
git stash drop stash@{0}
```

### 标签管理

bash

复制

```bash
# 创建轻量标签
git tag v1.0

# 创建附注标签
git tag -a v1.1 -m "Release version 1.1"

# 推送标签到远程
git push origin --tags

# 删除标签
git tag -d v1.0
git push origin --delete v1.0
```

### 子模块

bash

复制

```bash
# 添加子模块
git submodule add https://github.com/user/submodule.git

# 克隆包含子模块的仓库
git clone --recurse-submodules https://github.com/user/repo.git

# 更新子模块
git submodule update --init --recursive
```

### 二分查找

bash

复制

```bash
# 启动二分查找
git bisect start

# 标记错误提交
git bisect bad

# 标记正确提交
git bisect good <commit-hash>

# 结束二分查找
git bisect reset
```

------

## 常见问题与解决方案

### 1. 推送失败：远程包含本地没有的更新

bash

复制

```bash
# 拉取远程变更并合并
git pull origin main

# 解决冲突后重新推送
git push
```

### 2. 错误提交到错误分支

bash

复制

```bash
# 创建新分支保存当前状态
git branch temp-branch

# 重置原分支到正确状态
git checkout main
git reset --hard HEAD~1

# 将提交移动到正确分支
git checkout feature-branch
git cherry-pick <commit-hash>
```

### 3. 忘记添加文件到提交

bash

复制

```bash
# 添加遗漏文件
git add missed-file.txt

# 修改最后一次提交
git commit --amend
```

### 4. 撤销已推送的提交

bash

复制

```bash
# 本地撤销提交
git reset --hard HEAD~1

# 强制推送（通知团队成员）
git push -f
```

### 5. 大文件无法推送

bash

复制

```bash
# 使用Git LFS管理大文件
git lfs install
git lfs track "*.psd"

# 重新添加并提交
git add .gitattributes
git add file.psd
git commit -m "Add PSD file with LFS"
```

### 6. 认证失败

- **HTTPS**：使用个人访问令牌代替密码
- **SSH**：检查SSH密钥是否添加到GitHub账户

------

## 最佳实践

### 提交规范

- 使用语义化提交信息

- 遵循Conventional Commits规范

- 示例：

  复制

  ```markdown
  feat: add user authentication
  fix: resolve login page crash
  docs: update API documentation
  ```

### 分支策略

- **主分支**：`main`/`master`（稳定版本）
- **开发分支**：`develop`（集成环境）
- **功能分支**：`feature/*`（新功能开发）
- **修复分支**：`hotfix/*`（紧急修复）

### .gitignore 配置

gitignore

复制

```gitignore
# 忽略操作系统文件
.DS_Store
Thumbs.db

# 忽略编辑器文件
.idea/
.vscode/
*.suo

# 忽略依赖目录
node_modules/
vendor/

# 忽略编译文件
*.class
*.exe
*.dll
```

### 工作流程

1. 开发前拉取最新代码：`git pull`
2. 创建新分支：`git checkout -b feature/new-feature`
3. 小步提交：频繁提交小变更
4. 推送分支：`git push -u origin feature/new-feature`
5. 创建Pull Request进行代码审查
6. 合并后删除本地分支：`git branch -d feature/new-feature`

### 安全实践

- 使用SSH密钥代替HTTPS密码
- 定期轮换个人访问令牌
- 敏感信息使用环境变量或专用配置管理
- 使用`git-secrets`防止提交敏感信息

------

> 本笔记持续更新，建议定期查看Git官方文档获取最新信息：https://git-scm.com/doc