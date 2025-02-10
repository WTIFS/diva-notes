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

// 或者执行
git config --global url."git@gitlab.xxx.com:".insteadOf http://gitlab.xxx.com/
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

然后 git push origin 分支名 --force
```



##### 保存密码

```bash
git config --global credential.helper 'cache –timeout=3600' #保存一小时
```



##### centOS升级git版本

```go
1. 卸载旧版git
yum remove git

2.安装必要的依赖
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker libcurl-devel

3.切换到路径
mkdir -p /data/lib
cd /data/lib

4.下载源码
wget https://www.kernel.org/pub/software/scm/git/git-2.17.1.tar.xz

5.解包
tar -vxf git-2.17.1.tar.xz

6.切换路径
mv git-2.17.1 git2.17.1
cd git2.17.1

7.编译安装
./configure --with-openssl=/usr/local/openssl

8.编译 (如果这一步出错可能是openssl版本太低，1.0版本)这时候需要先升级到1.1.x版本
make

9.安装
make install

10.检查(有输出证明好了)
./git --version

11.打开环境变量配置文件，修改环境变量
vim /etc/profile.d/sh.local

#在底部加上git相关配置
export PATH=/data/lib/git2.17.1:$PATH
```



##### tag 命名规范
参考：https://acenet-arc.github.io/git-collaboration/08-Tags-Releases/index.html

- `X.Y.Z-dev`: A development snapshot leading up to version `X.Y.Z`
- `X.Y.Z-alpha`: An “alpha” release leading up to version `X.Y.Z` intended for internal testing.
- `X.Y.Z-beta2`: The second “beta” release leading up to version `X.Y.Z` intended for external testing.
- `X.Y.Z-rc1`: The first “Release Candidate” of upcoming version `X.Y.Z`. If no significant errors are found, this commit could also receive the `vX.Y.Z` tag, otherwise a `-rc2` release might follow.
