##### 安装

```bash
git clone git@github.com:ohmyzsh/ohmyzsh.git

./ohmyzsh/tools/install.sh
```



##### 安装 autojump 插件

```bash
brew install autojump

vim ~/.zshrc

# 把以下代码加到尾部
[[ -s $(brew --prefix)/etc/profile.d/autojump.sh ]] && . $(brew --prefix)/etc/profile.d/autojump.sh
```





##### 配置快捷键

```bash
vim ~/.zshrc

# 增加以下行:
alias gs="git status"
```



