这份内容从零基础到企业级实战全覆盖，兼顾理论、实操、避坑，新手可直接按路线循序渐进学习，老手可直接查阅对应模块快速查漏补缺。

---

## 一、Git 完整学习路线（从入门到企业落地）

### 阶段 1：零基础入门（1-2 天）

**学习目标**：理解 Git 核心设计，完成环境配置，能独立完成本地代码基础版本管理

**核心学习内容**：

1. Git 本质：分布式 vs 集中式版本控制（与 SVN 的核心区别）
2. 环境安装（Windows/Mac/Linux）、初始全局配置
3. 核心四大区域：工作区、暂存区、本地版本库、远程版本库（新手必懂，所有命令的底层逻辑）
4. 基础核心命令：`init`、`status`、`add`、`commit`、`log`
5. `.gitignore`忽略文件的配置规则
    
    **实操任务**：
6. 完成 Git 安装与全局`user.name`/`user.email`配置
7. 本地初始化 Git 仓库，创建 3 个文件，完成 2 次规范提交
8. 配置`.gitignore`，忽略日志、临时文件、`node_modules`等常见无需跟踪的内容
9. 用`git log`查看提交历史，理解 commit 哈希值的核心作用

### 阶段 2：本地版本进阶控制（2-3 天）

**学习目标**：熟练掌控本地版本的回溯、修改、对比、暂存，能处理所有本地版本相关问题

**核心学习内容**：

1. 版本对比：`diff`命令（工作区 vs 暂存区、暂存区 vs 版本库、版本间对比）
2. 版本回滚与重置：`reset`命令的三个核心参数`--soft`/`--mixed`/`--hard`，区别与适用场景
3. 撤销操作：`restore`/`checkout` 撤销工作区、暂存区的修改
4. 临时存储：`stash`命令（暂存未提交的修改，无缝切换分支）
5. 提交优化：`commit --amend` 修改最近一次提交
6. 日志高级用法：`log --oneline`/`--graph`/`--all`等参数
    
    **实操任务**：
7. 对已提交代码做修改，用`diff`查看差异，完成撤销修改的全流程操作
8. 模拟错误提交，用`reset`分别完成软重置、混合重置、硬重置，吃透三者区别
9. 模拟开发中途需切换分支的场景，用`stash`暂存代码，切换后完成恢复
10. 修改最近一次的提交信息，用`amend`追加遗漏文件到上一次提交

### 阶段 3：团队协作核心（3-4 天）【企业工作核心】

**学习目标**：熟练使用远程仓库，掌握分支管理规范，能独立完成团队协作中的代码推拉、合并、冲突解决

**核心学习内容**：

1. 远程仓库：GitHub/Gitee 注册、SSH 密钥配置、远程仓库创建与关联
2. 远程核心命令：`remote`、`clone`、`push`、`pull`、`fetch`
3. 分支核心操作：`branch`、`switch`/`checkout` 创建、切换、删除、重命名分支
4. 代码合并：`merge`命令，快进合并、三方合并的区别
5. 冲突解决：合并冲突的产生原因、手动解决冲突的标准流程
6. 企业级分支模型：Git Flow、GitHub Flow 的区别与适用场景
7. 变基操作：`rebase`基础用法，和`merge`的核心区别、使用禁忌
    
    **实操任务**：
8. 在 Gitee/GitHub 创建远程仓库，将本地仓库关联并推送代码
9. 克隆公开仓库，练习`fetch`和`pull`的核心区别
10. 创建 dev、feature 分支，在不同分支修改代码，完成合并，模拟冲突并按标准流程解决
11. 用`rebase`完成提交历史线性化，理解和`merge`的差异
12. 按照 GitHub Flow 规范，完成一次 feature 开发到合并主分支的全流程

### 阶段 4：高阶实战与问题排查（2-3 天）

**学习目标**：能处理 Git 复杂场景，恢复误操作代码，掌握高效排查问题的技巧

**核心学习内容**：

1. 标签管理：`tag`命令，轻量标签、附注标签，版本发布打标规范
2. 误操作恢复神器：`reflog` 命令，恢复 hard 重置的代码、误删的分支
3. 二分法查 bug：`bisect` 命令，快速定位引入 bug 的提交
4. 挑选提交：`cherry-pick` 命令，将指定分支的单个 / 多个提交合并到当前分支
5. 子模块管理：`submodule` 管理项目依赖的子仓库
6. Git 钩子：`hooks` 自定义提交前、推送前的校验（eslint、commit 规范校验）
    
    **实操任务**：
7. 为稳定版本打附注标签，推送标签到远程仓库，完成本地与远程标签的删除
8. 模拟误执行`git reset --hard`，用`reflog`恢复丢失的提交
9. 模拟 bug 引入，用`bisect`快速定位出问题的 commit
10. 用`cherry-pick`将 dev 分支的 bug 修复提交，合并到 master 分支
11. 配置 pre-commit 钩子，实现提交前自动格式化代码

### 阶段 5：规范与工具落地（1-2 天）

**学习目标**：掌握企业级 Git 使用规范，熟练使用提效工具，对接 CI/CD 流程

**核心学习内容**：

1. Git 提交规范：Conventional Commits 规范（feat/fix/docs/style/refactor 等）
2. Code Review 流程：基于 PR/MR 的 CR 规范与评审要点
3. GUI 工具：SourceTree、VS Code Git 插件的使用
4. 大文件处理：Git LFS 管理大文件（图片、视频、二进制文件）
5. CI/CD 集成：基于 Git 提交触发自动化构建、测试、部署
    
    **实操任务**：
6. 配置 commitlint+husky，强制校验提交信息符合规范
7. 用 VS Code 完成 Git 可视化操作、差异对比、冲突解决
8. 基于 GitHub PR 完成一次完整的 CR 流程与代码合并
9. 配置 Git LFS，管理项目中的大图片 / 视频文件

---

## 二、Git 核心学习笔记（按场景分类，可直接复制使用）

### （一）核心基础概念（新手必懂，拒绝死记命令）

1. **Git 本质**：分布式版本控制系统，每个开发者本地都有完整的版本库，离线可提交，远程仓库仅用于团队代码同步。
2. **四大核心区域**

| 区域    | 全称                | 核心作用                                                     |
| ----- | ----------------- | -------------------------------------------------------- |
| 工作区   | Working Directory | 电脑上可直接编辑的项目文件夹，日常写代码的位置                                  |
| 暂存区   | Stage/Index       | 临时存放即将提交的修改，`git add`将修改从工作区同步到这里                        |
| 本地版本库 | Local Repository  | 存放所有提交的版本历史，项目根目录的`.git`文件夹就是版本库，`git commit`将暂存区内容提交到这里 |
| 远程版本库 | Remote Repository | 远程服务器上的仓库（GitHub/Gitee 等），`git push`/`pull`实现与本地的同步      |

3. **代码生命周期**：代码修改 → 工作区 → `git add` → 暂存区 → `git commit` → 本地版本库 → `git push` → 远程版本库

### （二）基础配置命令

```bash
# 1. 全局配置（安装后必做，所有仓库生效）
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
# 查看配置
git config --list
git config user.name

# 2. 单个仓库局部配置（仅当前仓库生效，优先级高于全局）
git config user.name "仓库专属用户名"
git config user.email "仓库专属邮箱"

# 3. 配置命令别名（提效神器）
git config --global alias.st status # git st 代替 git status
git config --global alias.co checkout # git co 代替 git checkout
git config --global alias.cm "commit -m" # git cm "提交信息" 简化提交
git config --global alias.lg "log --oneline --graph --all" # 美化分支日志
```

### （三）本地仓库基础操作

```bash
# 1. 初始化本地仓库（在项目根目录执行）
git init

# 2. 查看仓库状态（高频使用，每次操作前建议执行）
git status
git status -s # 精简输出

# 3. 将工作区修改添加到暂存区
git add 文件名 # 添加指定文件
git add 文件夹名/ # 添加指定文件夹
git add . # 添加当前目录所有修改（新增、修改、删除，不含忽略文件）
git add -u # 添加所有已跟踪文件的修改（不含新增文件）
git add -A # 等价于 git add . + git add -u，添加所有修改

# 4. 将暂存区内容提交到本地版本库
git commit -m "提交说明：清晰描述本次修改内容" # 标准提交
git commit -am "提交说明" # 跳过git add，直接提交已跟踪文件的修改
git commit --amend # 修改最近一次提交信息/追加遗漏文件

# 5. 查看提交历史
git log # 完整日志，显示哈希值、作者、时间、提交信息
git log --oneline # 精简日志，一行显示短哈希+提交信息
git log --oneline --graph # 图形化显示分支合并历史
git log --oneline --all # 显示所有分支的日志
git log -n 5 # 显示最近5条提交
git log --author="用户名" # 查看指定作者的提交
```

### （四）撤销与回滚操作（新手高频使用）

```bash
# 1. 撤销工作区的修改（未执行git add）
git restore 文件名 # 撤销指定文件的修改，恢复到最近一次commit状态
git restore . # 撤销当前目录所有工作区修改
# 旧版本兼容方案
git checkout -- 文件名

# 2. 撤销暂存区的修改（已执行git add，未commit）
git restore --staged 文件名 # 把文件从暂存区撤回工作区，工作区修改保留
git restore --staged . # 撤销所有暂存的文件
# 旧版本兼容方案
git reset HEAD 文件名

# 3. 回滚已提交的版本（已commit，核心命令reset）
# 3.1 --soft 软重置：只重置HEAD，暂存区、工作区内容不变
# 适用场景：合并多个连续提交，重新整理提交信息
git reset --soft 提交哈希值
git reset --soft HEAD~1 # 回滚到上一个提交

# 3.2 --mixed 混合重置（默认）：重置HEAD+暂存区，工作区内容不变
# 适用场景：撤销错误commit，重新修改后再提交
git reset --mixed 提交哈希值
git reset 提交哈希值 # 等价于上面

# 3.3 --hard 硬重置：重置HEAD+暂存区+工作区，所有内容恢复到指定提交
# 适用场景：彻底放弃当前所有修改，回滚到历史版本 ⚠️ 慎用！会删除未提交修改
git reset --hard 提交哈希值
git reset --hard HEAD~n # 回滚到上n个提交

# 4. 临时暂存修改（开发中途需切分支，不想提交半成品代码）
git stash # 暂存当前工作区、暂存区的修改，恢复干净工作区
git stash save "暂存说明" # 带说明的暂存，方便识别
git stash list # 查看所有暂存记录
git stash pop # 恢复最近一次暂存，并删除该暂存记录
git stash apply # 恢复最近一次暂存，保留暂存记录
git stash drop stash@{n} # 删除指定序号的暂存
git stash clear # 清空所有暂存记录
```

### （五）远程仓库操作

```bash
# 1. 关联远程仓库（别名默认用origin）
git remote add 远程仓库别名 远程仓库地址
# 示例：git remote add origin git@gitee.com:xxx/xxx.git

# 2. 查看远程仓库信息
git remote -v # 查看远程仓库的别名和地址
git remote show origin # 查看origin远程仓库详细信息

# 3. 修改/删除远程仓库
git remote set-url origin 新地址 # 修改远程仓库地址
git remote remove origin # 删除远程仓库关联

# 4. 克隆远程仓库到本地
git clone 远程仓库地址 # 克隆完整仓库
git clone 远程仓库地址 自定义文件夹名 # 克隆到指定文件夹
git clone -b 分支名 远程仓库地址 # 克隆指定分支

# 5. 拉取远程代码
# 5.1 fetch：只拉取远程最新内容到本地，不自动合并，安全可控
git fetch origin # 拉取origin所有分支的更新
git fetch origin 分支名 # 拉取指定分支的更新

# 5.2 pull：拉取远程代码并自动合并到当前分支，等价于 git fetch + git merge
git pull origin 分支名 # 拉取远程指定分支，合并到当前本地分支
git pull --rebase origin 分支名 # 用rebase方式合并，避免产生冗余merge提交

# 6. 推送本地代码到远程仓库
git push origin 分支名 # 推送本地指定分支到远程同名分支
git push -u origin 分支名 # 首次推送，建立本地与远程分支的追踪关系，后续可直接git push
git push # 推送当前分支到已关联的远程分支
git push origin --all # 推送所有本地分支到远程
git push origin --force-with-lease # 安全强制推送，仅远程无新提交时生效，团队协作推荐
# ⚠️ 禁忌：git push --force 绝对不要在团队公共分支使用，会覆盖他人代码
```

### （六）分支管理核心操作

```bash
# 1. 查看分支
git branch # 查看本地所有分支，带*的是当前分支
git branch -r # 查看远程所有分支
git branch -a # 查看本地+远程所有分支
git branch -v # 查看分支的最后一次提交信息

# 2. 创建分支
git branch 分支名 # 基于当前分支创建新分支，不切换
git branch 分支名 提交哈希值 # 基于指定提交创建分支
git branch 分支名 origin/远程分支名 # 基于远程分支创建本地分支

# 3. 切换分支（Git 2.23+ 推荐用switch，语义更清晰）
git switch 分支名 # 切换到已存在的分支
git switch -c 分支名 # 创建并切换到新分支
git switch - # 切换到上一个分支
# 旧版本兼容方案
git checkout 分支名
git checkout -b 分支名 # 创建并切换到新分支

# 4. 分支重命名
git branch -m 旧分支名 新分支名 # 重命名本地分支
# 重命名远程分支：先改本地，删除旧远程分支，推送新分支
git push origin --delete 旧分支名
git push -u origin 新分支名

# 5. 删除分支
git branch -d 分支名 # 安全删除，仅分支已合并到主分支时生效
git branch -D 分支名 # 强制删除，无论是否合并 ⚠️ 慎用
# 删除远程分支
git push origin --delete 分支名

# 6. 合并分支（把目标分支的代码合并到当前分支）
# 示例：把dev合并到master，先切换到master，再执行merge
git merge 目标分支名
git merge --no-ff 目标分支名 # 禁用快进合并，强制生成merge提交，方便追溯历史

# 7. 变基操作 rebase（让提交历史变成一条直线，避免杂乱merge）
# 示例：把dev分支的修改基于master最新提交变基，先切换到dev，再执行rebase
git rebase 目标分支名
# ⚠️ 绝对禁忌：不要对已经推送到远程的公共分支执行rebase，会导致团队提交历史混乱

# 8. 挑选单个提交合并 cherry-pick
git cherry-pick 提交哈希值 # 把指定提交合并到当前分支
git cherry-pick 哈希1 哈希2 # 合并多个指定提交
git cherry-pick 哈希1^..哈希2 # 合并两个哈希之间的所有提交（包含哈希1）
```

### （七）标签管理（版本发布）

```bash
# 1. 创建标签
git tag 标签名 # 轻量标签，仅指向提交哈希
git tag -a 标签名 -m "标签说明" # 附注标签，带作者、时间、说明，版本发布推荐
git tag -a 标签名 提交哈希值 # 给历史提交打标签

# 2. 查看标签
git tag # 查看所有标签
git tag -l "v1.0.*" # 模糊搜索标签
git show 标签名 # 查看标签对应的提交详情

# 3. 推送标签到远程
git push origin 标签名 # 推送指定标签
git push origin --tags # 推送所有本地标签

# 4. 删除标签
git tag -d 标签名 # 删除本地标签
git push origin --delete 标签名 # 删除远程标签
```

### （八）.gitignore 忽略文件配置

- 作用：指定 Git 无需跟踪的文件 / 文件夹，不会被 add、commit、push
- 核心配置规则：

```bash
# 注释用#
# 1. 忽略指定文件
test.txt
build.log

# 2. 忽略指定文件夹
node_modules/
dist/
temp/

# 3. 通配符匹配
*.log # 忽略所有.log后缀的文件
*.zip # 忽略所有压缩包
*.tmp # 忽略所有临时文件

# 4. 取反！不忽略指定文件
!important.log # 即使上面忽略了*.log，该文件仍会被跟踪

# 5. 根目录匹配
/README.md # 只忽略根目录的README.md，子目录的不忽略

# 6. 递归匹配
**/logs # 忽略所有目录下的logs文件夹
**/*.txt # 忽略所有目录下的.txt文件
```

- 补充：可直接使用 GitHub 官方 gitignore 模板仓库，对应开发语言直接复制即可。

---

## 三、Git 高频常见错误与解决方案

### （一）本地操作常见错误

#### 错误 1：git commit 报错 Author identity unknown

- 完整报错：

plaintext

```
*** Please tell me who you are.
Run
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
to set your account's default identity.
```

- 原因：Git 安装后未配置全局用户名和邮箱，无法识别提交者身份
- 解决方案：

```bash
# 替换为自己的用户名和邮箱，与远程仓库账号一致
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
# 仅当前仓库生效，去掉--global即可
```

#### 错误 2：git add 警告 LF will be replaced by CRLF in xxx

- 原因：Windows 与 Mac/Linux 的换行符格式不一致（Windows 是 CRLF，Mac/Linux 是 LF），Git 自动转换的提示
- 解决方案：

```bash
# Windows系统执行，关闭自动转换
git config --global core.autocrlf false
# Mac/Linux系统执行
git config --global core.autocrlf input
```

#### 错误 3：git reset --hard 后丢失未提交代码 / 误删提交

- 原因：硬重置会清空工作区和暂存区的修改，新手易误执行
- 解决方案：用`reflog`恢复（Git 会记录所有 HEAD 的变更历史，即使是 hard 重置）

```bash
# 1. 查看所有HEAD变更记录，找到要恢复的提交哈希值
git reflog
# 2. 执行硬重置恢复到指定提交
git reset --hard 目标提交哈希值
# 3. 误删分支恢复方案
git checkout -b 恢复的分支名 目标提交哈希值
```

#### 错误 4：git commit --amend 修改了已推送到远程的提交

- 原因：amend 会修改提交的哈希值，本地与远程提交哈希不一致，后续推送会失败
- 解决方案：

```bash
# 1. 个人仓库、无他人协作：安全强制推送
git push --force-with-lease origin 分支名

# 2. 团队公共分支：绝对不要强制推送！
# 正确做法：放弃amend修改，重新拉取远程代码，再修改提交
git reset --hard origin/分支名
git pull origin 分支名
# 重新修改代码，正常commit、push
```

### （二）远程协作常见错误

#### 错误 1：git push 报错 failed to push some refs to xxx

- 完整报错：

```bash
error: failed to push some refs to 'git@gitee.com:xxx/xxx.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
```

- 原因：远程分支有新的提交，本地分支不是最新版本，Git 拒绝推送
- 解决方案：

```bash
# 1. 先拉取远程最新代码，合并到本地
git pull origin 分支名
# 2. 若拉取出现冲突，解决完冲突后，执行git add + git commit
# 3. 重新推送
git push origin 分支名

# 进阶：保持提交历史线性，用rebase方式拉取
git pull --rebase origin 分支名
# 解决冲突后，执行 git rebase --continue，再push
```

#### 错误 2：git clone 报错 Permission denied (publickey)

- 完整报错：

```bash
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.
```

- 原因：SSH 密钥配置错误，远程仓库无法验证身份，或无仓库访问权限
- 解决方案：

1. 检查是否生成 SSH 密钥
    
    - Windows：`C:\Users\用户名\.ssh` 查看是否有`id_rsa`和`id_rsa.pub`
    - Mac/Linux：`~/.ssh` 查看对应文件
    
2. 无密钥则执行生成命令，一路回车默认即可
    
    bash
    
    运行
    
    ```
    ssh-keygen -t ed25519 -C "你的邮箱"
    ```
    
3. 复制公钥（`id_rsa.pub`文件全部内容），添加到 GitHub/Gitee 的「SSH 公钥」设置中
4. 测试连接
    
    bash
    
    运行
    
    ```
    ssh -T git@github.com # 测试GitHub
    ssh -T git@gitee.com # 测试Gitee
    ```
    
5. 额外检查：仓库地址是否正确，账号是否有该仓库的读写权限

#### 错误 3：git pull 报错 refusing to merge unrelated histories

- 完整报错：

plaintext

```
fatal: refusing to merge unrelated histories
```

- 原因：两个仓库无共同的提交历史（比如本地 init 的仓库，关联了已有提交的远程仓库），Git 拒绝合并
- 解决方案：添加参数允许合并不相关历史

bash

运行

```
git pull origin 分支名 --allow-unrelated-histories
# 合并完成后，正常push即可
```

### （三）合并冲突类错误

#### 错误 1：merge/rebase 报错 Automatic merge failed; fix conflicts and then commit the result.

- 原因：两个分支修改了同一个文件的同一行代码，Git 无法自动合并，需手动解决
- merge 冲突标准解决步骤：

1. 执行`git status`，查看标有`both modified`的冲突文件
2. 打开冲突文件，找到冲突标记块：
    
    plaintext
    
    ```
    <<<<<<< HEAD
    当前分支的代码
    =======
    合并过来的分支的代码
    >>>>>>> 目标分支名
    ```
    
3. 手动修改代码，保留正确内容，删除`<<<<<<<`、`=======`、`>>>>>>>`所有冲突标记
4. 所有冲突文件修改完成后，执行`git add 冲突文件名`，将修改加入暂存区
5. 执行`git commit`完成合并提交，后续正常 push 即可

- rebase 冲突标准解决步骤：

1. 修改冲突文件，保留正确代码，删除冲突标记
2. 执行`git add 冲突文件名`
3. 执行`git rebase --continue`继续 rebase 流程
4. 若仍有冲突，重复上述步骤，直到 rebase 完成
5. 放弃 rebase：执行`git rebase --abort`，恢复到 rebase 前的状态

#### 错误 2：合并后发现代码错误，想撤销合并

- 解决方案：

bash

运行

```
# 1. 合并后未push到远程：直接reset回滚
git reset --hard HEAD~1

# 2. 已push到远程公共分支：用revert撤销（不修改历史，安全）
# 先通过git log找到合并提交的哈希值
git revert -m 1 合并提交哈希值
# -m 1 表示保留当前分支的修改，撤销合并过来的分支的修改
# 执行后push到远程即可
```

### （四）其他高频错误

#### 错误 1：.gitignore 配置后不生效，忽略的文件仍被跟踪

- 原因：该文件之前已经被 Git 跟踪，`.gitignore`仅对未被跟踪的文件生效
- 解决方案：清除文件的 Git 跟踪缓存，重新提交

bash

运行

```
# 1. 清除所有文件的跟踪缓存（不会删除本地文件）
git rm -r --cached .
# 2. 重新添加所有文件，此时.gitignore会生效
git add .
# 3. 提交修改
git commit -m "fix: 修复.gitignore不生效问题"
```

#### 错误 2：git stash pop 恢复时出现冲突，stash 记录丢失

- 原因：恢复的 stash 内容与当前工作区代码有冲突，Git 会保留冲突内容，但 stash 记录会被删除
- 解决方案：

1. 按 merge 冲突的标准流程，修改冲突文件，保留正确代码，删除冲突标记
2. 执行`git add 冲突文件`，完成恢复
3. 放弃恢复：执行`git reset --hard HEAD`，恢复到干净状态

---

## 四、学习资源与学习建议

### 优质学习资源

1. 官方文档：[Git 官方中文文档](https://git-scm.com/book/zh/v2)（最权威、最全面）
2. 交互式练习：[Learn Git Branching](https://learngitbranching.js.org/?locale=zh_CN)（可视化理解分支，新手神器）
3. 速查表：Git 官方速查表，随时查阅高频命令

### 核心学习建议

1. **先懂概念，再记命令**：不要死记硬背命令，先吃透四大区域、分支、合并的核心逻辑，命令自然会用。
2. **多实操，多踩坑**：Git 是工具，光看没用，一定要自建仓库，多做提交、合并、回滚操作，踩坑后记忆更深刻。
3. **新手慎用高危参数**：`--hard`、`--force`是新手最容易出问题的参数，执行前一定要确认，最好先备份代码。
4. **养成良好习惯**：提交前先`git status`看状态，提交信息写清晰，每天下班前 push 代码，避免代码丢失。
5. **团队协作前先练熟**：不要在团队公共分支上做测试，先在个人仓库把分支、合并、冲突解决练熟，再用于工作中。