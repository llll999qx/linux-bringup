
**1.初始化 Git 仓库**
```bash
git init -b main
```
让 Git 开始管理这个文件夹，分支名为 `main`

**2. 把代码、笔记和图片加入暂存区**
```bash
git add README.md kernel device-tree rootfs docs
```
选中这些文件，准备保存。`docs` 里的笔记和图片会一起加入

查看准备提交的内容：
```bash
git diff --cached --stat
```
这里会列出文件及变化数量。确认有你的 Markdown、图片和代码文件。

**3. 保存第一个版本**
```bash
git commit -m "记录 i.MX6ULL Linux 启动实验、内核修复和学习笔记"
```
把暂存区的内容保存成一个可追溯的版本，`-m` 后面是这次修改的说明。

如果提示：
```
Author identity unknown
```
需要设置提交者信息。**把下面的名字和邮箱替换成自己的，再执行：**
```bash
git config user.name "你的名字"
git config user.email "你的邮箱"
```
这不是输入登录密码，只是记录“这个版本是谁提交的”。设置后重新执行上面的 `git commit`。

**4. 检查状态**
```bash
git status
```
如果出现：
```
nothing to commit, working tree clean
```
就说明当前文件已经保存到本地 Git 仓库。（`commit` 的意思是“提交”，也就是把暂存区里的内容保存成 Git 的一个版本记录）

**5.上传**
`git commit`，文件保存在 Ubuntu 本地，还没有上传 GitHub
**`gh` 是 GitHub 官方提供的命令行工具，全名 GitHub CLI。** 它让你在终端中登录 GitHub、管理仓库等。
```bash
sudo apt install gh
```
**让这个工具登录 GitHub**
```bash
gh auth login --hostname github.com --git-protocol https --web
```

**6. 关联 GitHub 仓库**
先在 GitHub 上创建空的 `linux-bringup` 仓库，然后把它的地址关联到本地：
```bash
git remote add origin GitHub仓库地址
```

`origin` 是远程仓库的名字。如果已经关联过了，以后不用重复执行。可以用下面的命令查看关联地址：
```bash
git remote -v
```
xq999@xq999-virtual-machine:git remote -v  
origin [https://github.com/llll999qx/linux-bringup.git](https://github.com/llll999qx/linux-bringup.git) (fetch)  （从 GitHub 获取内容）
origin [https://github.com/llll999qx/linux-bringup.git](https://github.com/llll999qx/linux-bringup.git) (push)（把本地提交上传到 GitHub）



**7. 配置网络并登录 GitHub**
因为 Ubuntu 直接连接 GitHub 会超时，这次先让当前终端使用代理：
```bash
export http_proxy="http://192.168.77.1:12450"
export https_proxy="http://192.168.77.1:12450"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
```
这个地址是电脑代理软件提供的地址；这些设置只对当前终端及其启动的进程生效。
然后使用 GitHub CLI 登录：
```bash
gh auth login --hostname github.com --web
```

按照终端提示，在浏览器输入一次性验证码并授权。登录成功后执行：
```bash
gh auth setup-git
```
它让 Git 推送时使用 `gh` 的登录凭据。

**8. 上传提交**
```bash
git push -u origin main
```
- `push`：上传本地提交。
- `origin`：上传到关联的 GitHub 仓库。
- `main`：上传本地的 `main` 分支。
- `-u`：建立本地分支与远程分支的关联。


### 一个新项目，上传到一个新仓库

**1. 在 GitHub 网页上创建仓库**
点击右上角 **`+` → New repository**，填写仓库名，例如 `my-notes`
**2.在 Ubuntu 进入你的项目目录**
```bash
cd ~/imx6ull/my-notes
```
需要先创建文件夹并把笔记放进去
```bash
git init
git add .
git commit -m "第一次提交"
```
`git add .` 会把当前目录下符合条件的新增和修改加入暂存区，因此执行前可以用 `git status` 看一下有哪些文件。

| 命令 | 作用 | 文件在哪里 |
| ---- | ---- | ---- |
| git add ssh-test.md | 选中这个文件的改动，放入暂存区 | 还在本地 Ubuntu |
| git commit -m "..." | 把暂存区里的内容保存为一条本地版本记录 | 还在本地 Ubuntu |
| git push | 把本地提交上传到关联的远程仓库 | 上传到 GitHub |



### 以后修改文件后怎么上传

之后在同一个项目中，一般只需要：
```bash
cd ~/imx6ull/linux-bringup
git status
git add .
git commit -m "填写这次修改的说明"
git push
```
之后修改了学习笔记：
```bash
git commit -m "更新根文件系统学习笔记"
```


### ssh密钥

Ubuntu 保管私钥 id_rsa
          ↓ 连接时证明自己持有私钥
GitHub 根据你添加的公钥验证身份
          ↓
验证通过后，允许你操作有权限的仓库

**方法1**
HTTPS 地址 + gh 登录授权 → git push 成功

**方法2**
SSH 地址 + SSH 密钥认证 → git push
**1.**
```bash
ssh-keygen -t rsa -b 4096 -C "你的邮箱"
```

| 内容 | 作用 |
| ---- | ---- |
| ssh-keygen | 生成 SSH 密钥 |
| -t rsa | 选择 RSA 密钥类型 |
| -b 4096 | RSA 密钥长度 4096 bit |
| -C "你的邮箱" | 添加备注，方便识别这把密钥 |
**2.生成后有两个文件：**
```
~/.ssh/id_rsa       私钥：放在 Ubuntu
~/.ssh/id_rsa.pub   公钥：复制到 GitHub
```
查看公钥：
```bash
cat ~/.ssh/id_rsa.pub
```
随后测试：
```bash
ssh -T git@github.com
```
如果是 `Hi llll999qx! You've successfully authenticated`，说明**现有的密钥已经可以使用**。如果重新生成时提示覆盖 `id_rsa`，不要确认覆盖。

**（可以把http改成git ssh可以连接的）**
```bash
cd ~/imx6ull/linux-bringup
```
`git remote set-url`：**修改远程仓库的地址**， 
`origin`：远程仓库的默认别名 
`git@github.com:llll999qx/linux-bringup.git`：**SSH 协议的远程地址**
```bash
git remote set-url origin git@github.com:llll999qx/linux-bringup.git
```
xq999@xq999-virtual-machine:~/imx6ull/linux-bringup$ git remote -v
origin	git@github.com:llll999qx/linux-bringup.git (fetch)
origin	git@github.com:llll999qx/linux-bringup.git (push)

```bash
git add "docs/git和ssh密钥.md"
git commit -m "添加 Git 和 SSH 密钥学习笔记"
git push
```
- **`add`**：选中这份笔记，准备提交。
- **`commit`**：在本地创建一条包含它的版本记录。
- **`push`**：上传到当前关联的 GitHub 仓库。
```bash
git log --oneline
```
它会列出**当前分支的提交历史，最新的在最上面**
