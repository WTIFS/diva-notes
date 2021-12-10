##### 安装 zsh

```bash
git clone git@github.com:ohmyzsh/ohmyzsh.git

./ohmyzsh/tools/install.sh
```



##### 安装 autojump 插件

```bash
# 安装 brew，如果已有则跳过
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"

brew install autojump

vim ~/.zshrc

# 把以下代码加到尾部
[[ -s $(brew --prefix)/etc/profile.d/autojump.sh ]] && . $(brew --prefix)/etc/profile.d/autojump.sh

# 修改 plugins 字段，增加 autojump，sublime 也是加在这里的
plugins=(
  git
  autojump
  sublime
)
```





##### 配置快捷键

```bash
vim ~/.zshrc

# 增加以下行:
alias gs="git status"
```



