##### 设置全局用户名和邮箱

```bash
git config --global user.name
git config --global user.email

git config --global --get user.name
git config --global --get user.email
```



##### git如何为内外网配置不同的username？

1. 修改 `~/.gitconfig` 文件：

```bash
# see https://git-scm.com/docs/git-config#_conditional_includes
[includeIf "gitdir:github.com/"]
    path = ./.github

[includeIf "gitdir:gitlab.xx.com/"]
    path = ./.gitlab
```



2. 新增 `~/.github` 和 `~/.gitlab` 文件：

```bash
# .github
[user]
    email = a
    name = a
    
# .gitlab
[user]
    email = b
    name = b
```



##### 将 http 方式替换为 ssh

修改 `~/.gitconfig` 文件：

```text
[url "git@gitlab.xxx.com:"]
    insteadOf = https://gitlab.xxx.com/
[url "git@gitlab.xxx.com:"]
    insteadOf = http://gitlab.xxx.com
```



##### 设置默认推送同名分支

```bash
git config --global --add push.default current
```



##### CentOS 升级 git 版本

```
yum -y install curl-devel

wget https://www.kernel.org/pub/software/scm/git/git-2.32.0.tar.gz

tar -zxvf git-2.9.5.tar.gz

cd git-2.32.0
./configure –prefix=/usr/local/git

make && make install
```



##### 添加多个远端

```bash
# [alias]是你给新远程仓库起的名字（例如origin，upstream，或其他），[url]是新远程仓库的URL
git remote add [alias] [url]
# 拉取远端分支
get fetch [alias] 远端分支名1:本地分支名1

git checkout main
git branch 新分支2
git checkout 新分支2
git merge --no-ff 分支1（合并远端更新到分支2上）（--allow-unrelated-histories）
```



##### rebase 合并多次 commit

```bash
git log 找到要合并的最早的提交的前一次提交的 hash
git rebase -i hash

# 此时会弹出 vim
# 保留第一行 pick ，其余的全改成 s
# 然后再编辑一次 commit log
```



##### 保存密码

```bash
git config --global credential.helper 'cache –timeout=3600' #保存一小时
```

