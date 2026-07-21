---
tags: [entity, 工具, 版本控制, 开发]
created: 2026-06-20
updated: 2026-06-20
type: entity
---

# Git 常用命令手册

## 一、基础配置

### 1.1 用户信息配置

# 设置全局用户名和邮箱  
git config --global user.name "Your Name"  
git config --global user.email "your.email@example.com"  
​  
# 查看配置信息  
git config --list  
git config user.name  
git config user.email

### 1.2 编辑器配置

# 设置默认编辑器为 VS Code  
git config --global core.editor "code --wait"  
​  
# 设置默认编辑器为 Vim  
git config --global core.editor vim

### 1.3 代理配置

# 设置 HTTP 代理  
git config --global http.proxy http://127.0.0.1:7890  
git config --global https.proxy http://127.0.0.1:7890  
​  
# 取消代理  
git config --global --unset http.proxy  
git config --global --unset https.proxy

---

## 二、仓库操作

### 2.1 初始化仓库

# 在当前目录初始化 Git 仓库  
git init  
​  
# 初始化并指定初始分支名  
git init -b main

### 2.2 克隆仓库

# 克隆远程仓库  
git clone https://github.com/user/repo.git  
​  
# 克隆到指定目录  
git clone https://github.com/user/repo.git my-folder  
​  
# 克隆指定分支  
git clone -b develop https://github.com/user/repo.git  
​  
# 浅克隆（只克隆最近一次提交）  
git clone --depth 1 https://github.com/user/repo.git

---

## 三、分支管理

### 3.1 分支查看

# 查看本地分支  
git branch  
​  
# 查看所有分支（本地 + 远程）  
git branch -a  
​  
# 查看远程分支  
git branch -r  
​  
# 查看分支最后提交信息  
git branch -v

### 3.2 分支创建与切换

# 创建新分支  
git branch feature/new-feature  
​  
# 切换分支  
git checkout feature/new-feature  
​  
# 创建并切换到新分支  
git checkout -b feature/new-feature  
​  
# 切换到上一个分支  
git checkout -  
​  
# 使用 switch 命令切换分支（Git 2.23+）  
git switch main  
git switch -c feature/new-feature  # 创建并切换

### 3.3 分支合并

# 合并指定分支到当前分支  
git merge feature/new-feature  
​  
# 合并并创建合并提交（禁止快进合并）  
git merge --no-ff feature/new-feature  
​  
# 变基合并  
git rebase main

### 3.4 分支删除

# 删除本地分支  
git branch -d feature/old-feature  
  
# 强制删除本地分支  
git branch -D feature/old-feature  
  
# 删除远程分支  
git push origin --delete feature/old-feature

### 3.5 分支重命名

# 重命名当前分支  
git branch -m new-name  
  
# 重命名指定分支  
git branch -m old-name new-name

---

## 四、代码提交

### 4.1 添加文件到暂存区

# 添加指定文件  
git add file1.txt file2.txt  
  
# 添加所有修改  
git add .  
  
# 添加所有修改（包括删除）  
git add -A  
  
# 交互式添加  
git add -p

### 4.2 提交代码

# 提交暂存区内容  
git commit -m "commit message"  
  
# 添加并提交所有修改  
git commit -am "commit message"  
  
# 修改上一次提交  
git commit --amend  
  
# 修改上一次提交信息  
git commit --amend -m "new message"

### 4.3 提交规范

# 常用提交类型  
# feat:     新功能  
# fix:      修复 bug  
# docs:     文档更新  
# style:    代码格式（不影响代码运行的变动）  
# refactor: 重构（既不是新增功能，也不是修改bug的代码变动）  
# perf:     性能优化  
# test:     增加测试  
# chore:    构建过程或辅助工具的变动  
  
# 示例  
git commit -m "feat: 添加用户登录功能"  
git commit -m "fix: 修复登录验证失败的问题"

---

## 五、远程操作

### 5.1 远程仓库管理

# 查看远程仓库  
git remote -v  
  
# 添加远程仓库  
git remote add origin https://github.com/user/repo.git  
  
# 修改远程仓库 URL  
git remote set-url origin https://github.com/user/new-repo.git  
  
# 删除远程仓库  
git remote remove origin

### 5.2 拉取代码

# 拉取远程代码并合并  
git pull origin main  
  
# 拉取远程代码并变基  
git pull --rebase origin main  
  
# 拉取所有远程分支更新  
git fetch --all

### 5.3 推送代码

# 推送到远程仓库  
git push origin main  
  
# 推送并设置上游分支  
git push -u origin feature/new-feature  
  
# 强制推送（危险操作）  
git push --force origin main  
  
# 推送所有标签  
git push --tags

---

## 六、标签管理

### 6.1 查看标签

# 查看所有标签  
git tag  
  
# 查看指定标签详情  
git show v1.0.0  
  
# 查看匹配的标签  
git tag -l "v1.*"

### 6.2 创建标签

# 创建轻量标签  
git tag v1.0.0  
  
# 创建附注标签  
git tag -a v1.0.0 -m "Release version 1.0.0"  
  
# 为指定提交创建标签  
git tag -a v1.0.0 abc1234 -m "Release version 1.0.0"

### 6.3 推送标签

# 推送指定标签  
git push origin v1.0.0  
  
# 推送所有标签  
git push origin --tags

### 6.4 删除标签

# 删除本地标签  
git tag -d v1.0.0  
  
# 删除远程标签  
git push origin --delete v1.0.0

---

## 七、查看状态与日志

### 7.1 查看状态

# 查看工作区状态  
git status  
  
# 简洁模式  
git status -s

### 7.2 查看差异

# 查看工作区与暂存区的差异  
git diff  
  
# 查看暂存区与上次提交的差异  
git diff --staged  
git diff --cached  
  
# 查看两次提交之间的差异  
git diff commit1 commit2  
  
# 查看指定文件的差异  
git diff file.txt

### 7.3 查看提交历史

# 查看完整日志  
git log  
  
# 单行显示  
git log --oneline  
  
# 图形化显示分支  
git log --graph --oneline --all  
  
# 查看最近 n 次提交  
git log -5  
  
# 查看指定文件的修改历史  
git log file.txt  
  
# 查看某人的提交  
git log --author="name"  
  
# 搜索提交信息  
git log --grep="keyword"

### 7.4 查看提交详情

# 查看指定提交详情  
git show abc1234  
  
# 查看指定提交的文件列表  
git show --stat abc1234

---

## 八、撤销与回退

### 8.1 撤销工作区修改

# 撤销指定文件的修改  
git checkout -- file.txt  
  
# 撤销所有修改  
git checkout -- .

### 8.2 撤销暂存

# 将文件从暂存区移回工作区  
git reset HEAD file.txt  
  
# 撤销所有暂存  
git reset HEAD

### 8.3 版本回退

# 回退到上一个版本（保留修改）  
git reset --soft HEAD~1  
  
# 回退到上一个版本（保留修改在工作区）  
git reset --mixed HEAD~1  
git reset HEAD~1  
  
# 回退到上一个版本（丢弃所有修改）  
git reset --hard HEAD~1  
  
# 回退到指定提交  
git reset --hard abc1234

### 8.4 反转提交

# 创建一个新提交来撤销指定提交  
git revert abc1234  
  
# 反转并直接提交  
git revert --no-edit abc1234

### 8.5 暂存工作区

# 暂存当前修改  
git stash  
  
# 暂存并添加描述  
git stash save "work in progress"  
  
# 查看暂存列表  
git stash list  
  
# 恢复最近一次暂存  
git stash pop  
  
# 恢复但不删除暂存  
git stash apply  
  
# 删除指定暂存  
git stash drop stash@{0}  
  
# 清空所有暂存  
git stash clear

---

## 九、文件操作

### 9.1 删除文件

# 从工作区和暂存区删除  
git rm file.txt  
  
# 只从暂存区删除（保留工作区）  
git rm --cached file.txt  
  
# 递归删除目录  
git rm -r folder/

### 9.2 移动/重命名文件

# 移动或重命名文件  
git mv old-name.txt new-name.txt  
  
# 移动到指定目录  
git mv file.txt folder/file.txt

### 9.3 忽略文件

# .gitignore 文件示例  
# 忽略所有 .log 文件  
*.log  
  
# 忽略 node_modules 目录  
node_modules/  
  
# 忽略编译输出  
build/  
dist/  
  
# 忽略 IDE 配置  
.idea/  
.vscode/  
*.swp  
  
# 忽略系统文件  
.DS_Store  
Thumbs.db

---

## 十、高级操作

### 10.1 Cherry-pick

# 将指定提交应用到当前分支  
git cherry-pick abc1234  
  
# cherry-pick 多个提交  
git cherry-pick abc1234 def5678  
  
# 只应用不提交  
git cherry-pick --no-commit abc1234

### 10.2 Rebase 变基

# 将当前分支变基到目标分支  
git rebase main  
  
# 交互式变基（修改最近 3 次提交）  
git rebase -i HEAD~3  
  
# 中止变基  
git rebase --abort  
  
# 继续变基（解决冲突后）  
git rebase --continue

### 10.3 Reflog 查看引用日志

# 查看所有操作记录  
git reflog  
  
# 恢复误删的提交  
git reflog  
git reset --hard HEAD@{2}

### 10.4 清理

# 查看将要删除的未跟踪文件  
git clean -n  
  
# 删除未跟踪的文件  
git clean -f  
  
# 删除未跟踪的文件和目录  
git clean -fd

---

## 十一、常见问题解决

### 11.1 解决合并冲突

# 1. 查看冲突文件  
git status  
  
# 2. 手动编辑冲突文件，解决冲突标记  
# <<<<<<< HEAD  
# 当前分支的内容  
# =======  
# 合并分支的内容  
# >>>>>>> branch-name  
  
# 3. 标记冲突已解决  
git add resolved-file.txt  
  
# 4. 完成合并  
git commit

### 11.2 撤销合并

# 合并过程中撤销  
git merge --abort  
  
# 合并完成后撤销  
git reset --hard HEAD~1

### 11.3 撤销 rebase

# rebase 过程中撤销  
git rebase --abort  
  
# rebase 完成后撤销  
git reflog  
git reset --hard HEAD@{n}  # n 为 rebase 前的位置

### 11.4 修改远程仓库历史

# 交互式 rebase 修改历史  
git rebase -i HEAD~3  
  
# 强制推送（谨慎使用）  
git push --force origin main

---

## 十二、Git 工作流示例

### 12.1 Feature Branch 工作流

# 1. 创建功能分支  
git checkout -b feature/user-login main  
  
# 2. 开发并提交  
git add .  
git commit -m "feat: implement user login"  
  
# 3. 推送到远程  
git push -u origin feature/user-login  
  
# 4. 创建 Pull Request (在 GitHub/GitLab 上操作)  
  
# 5. 合并后删除分支  
git checkout main  
git pull origin main  
git branch -d feature/user-login

### 12.2 Git Flow 工作流

# 主要分支  
# main:     生产环境代码  
# develop:  开发环境代码  
  
# 辅助分支  
# feature/*: 功能分支  
# release/*: 发布分支  
# hotfix/*:  热修复分支  
  
# 创建功能分支  
git checkout -b feature/new-feature develop  
  
# 完成功能，合并到 develop  
git checkout develop  
git merge --no-ff feature/new-feature  
git branch -d feature/new-feature  
  
# 创建发布分支  
git checkout -b release/1.0.0 develop  
  
# 发布完成后合并到 main 和 develop  
git checkout main  
git merge --no-ff release/1.0.0  
git tag -a v1.0.0 -m "Release 1.0.0"  
  
git checkout develop  
git merge --no-ff release/1.0.0  
git branch -d release/1.0.0

---

## 十三、实用别名配置

# 设置常用别名  
git config --global alias.st status  
git config --global alias.co checkout  
git config --global alias.br branch  
git config --global alias.ci commit  
git config --global alias.unstage 'reset HEAD --'  
git config --global alias.last 'log -1 HEAD'  
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"  
  
# 使用别名  
git st          # git status  
git co main     # git checkout main  
git br          # git branch  
git ci -m "msg" # git commit -m "msg"

---

_最后更新：2026年5月_。

## 相关实体
- [[tools/Obsidian]] — Git 管理的 vault 即为 Obsidian 的知识库
- [[tools/Obsidian_Web_Clipper]] — 剪裁的文章可通过 Git 追踪变更

## 相关概念
- [[meta/LLM_Wiki_方法论]] — 原文在"Tips and tricks"中提到
- [[meta/知识库三层架构]] — Git 是整个三层架构的版本控制底层

## 来源
- [[meta/LLM_Wiki_方法论]] — 原文提到 wiki 本质上是 git 仓库
