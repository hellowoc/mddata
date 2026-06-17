# Git 与 GitHub 使用指南

---

## 一、安装 Git

```bash
sudo apt-get update
sudo apt-get install -y git
```

装完验证：

```bash
git --version    # 看到版本号就是装好了
```

---

## 二、环境配置（做一次就行）

```bash
# 设置你的名字和邮箱（会记录在每次提交里）
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"

# 设置默认分支名为 main
git config --global init.defaultBranch main

# 查看配置
git config --global --list
```

---

## 三、基本工作流程

### 场景1：从零开始建一个项目

```bash
cd ~/Desktop/my-project          # 进到项目目录
git init                          # 初始化，创建一个隐藏的 .git 文件夹
git add .                         # 把所有文件加入暂存区
git commit -m "第一次提交"          # 提交，生成一个版本快照
```

### 场景2：从 GitHub 克隆别人的项目

```bash
git clone https://github.com/xxx/yyy.git
cd yyy
```

### 场景3：日常修改代码

```bash
# 改了代码之后...
git add 文件名            # 把修改加入暂存区（或用 git add . 加全部）
git commit -m "修了个bug"  # 提交
```

---

## 四、修改/撤销

| 命令 | 作用 |
|------|------|
| `git checkout 文件名` | 撤销还没 add 的修改（回到上次 commit 的状态） |
| `git reset HEAD 文件名` | 把 add 过的文件撤出暂存区 |
| `git reset --soft HEAD~1` | 撤销最近一次 commit（修改保留在工作区） |
| `git reset --hard HEAD~1` | 撤销最近一次 commit（修改也丢掉） |
| `git revert HEAD` | 安全撤销：生成一个新 commit 来抵消上次改动 |

---

## 五、暂存与切换（临时放下手头工作）

```bash
git stash                    # 把当前未提交的修改暂存起来
git stash pop                # 恢复刚才暂存的修改
git stash list               # 查看所有暂存
```

---

## 六、查看信息

| 命令 | 作用 |
|------|------|
| `git status` | 看哪些文件改过、哪些已暂存 |
| `git diff` | 看具体改了啥（未暂存的） |
| `git diff --staged` | 看已暂存的修改 |
| `git log --oneline` | 看提交历史（简洁版） |
| `git log --oneline --graph --all` | 图形化看所有分支历史 |
| `git blame 文件名` | 看每一行代码是谁、哪次提交改的 |

---

## 七、分支管理

```bash
git branch              # 查看所有分支
git branch 分支名        # 新建分支
git checkout 分支名      # 切换到那个分支
git checkout -b 分支名   # 新建并切换（合一操二）
git merge 分支名         # 把指定分支合并到当前分支
git branch -d 分支名     # 删除分支
```

### 合并冲突怎么办？

1. 手动编辑冲突文件，删掉 `<<<<<<<` `=======` `>>>>>>>` 标记
2. `git add 冲突文件`
3. `git commit` 完成合并

---

## 八、连接 GitHub（同步到云端）

```bash
# 1. 先在 GitHub 网页上建一个空仓库，拿到地址

# 2. 关联远程仓库
git remote add origin https://github.com/你的用户名/仓库名.git

# 3. 推送代码上去
git push -u origin main
```

### SSH 方式（不用每次输密码）

```bash
# 生成 SSH 密钥（一路回车）
ssh-keygen -t ed25519 -C "你的邮箱@example.com"

# 查看公钥
cat ~/.ssh/id_ed25519.pub

# 把公钥内容复制到 GitHub → Settings → SSH and GPG keys → New SSH key

# 之后用 SSH 地址克隆
git clone git@github.com:用户名/仓库名.git
```

---

## 九、日常同步流程

```bash
# 开始工作前拉取最新代码
git pull

# 写代码...

# 提交
git add .
git commit -m "描述你的改动"

# 推到云端
git push
```

---

## 十、.gitignore 文件

在项目根目录创建 `.gitignore`，让 Git 忽略不需要管理的文件：

```
node_modules/
*.log
.env
.DS_Store
build/
dist/
```

---

## 十一、常见问题

### Q: commit 写错了怎么办？
```bash
git commit --amend -m "新的提交信息"
```

### Q: 不小心提交了不该提交的文件？
```bash
git rm --cached 文件名    # 从 Git 中移除，但保留本地文件
```

### Q: 想回到某个旧的版本看看？
```bash
git checkout 提交的hash值
```

### Q: 本地和远程都有改动，pull 报错？
```bash
git pull --rebase    # 把你的改动放在远程最新代码之后
```
