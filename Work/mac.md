##### 启动台无法搜索

```bash
#reset launchpad
killall Dock
```



##### 三指拖移功能

可用三指调整表格宽度

系统偏好设置 - 辅助功能 - 指针控制 - 触控板选项 - 启用拖移 - 三指拖移



##### 禁用两指后退功能

经常造成我用浏览器想左右滑动查看表格，结果后退到上一个页面

系统偏好设置 - 触控板 - 更多手势 - 取消在页面之间轻扫



##### python环境安装

到 https://www.python.org/ 下载包并安装即可



##### oh-my-zsh 安装

```shell
#下载并安装iTerm2
xcode-select --install

#HomeBrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

# oh-my-zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```



##### zsh 安装 autojump 插件

```shell
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



##### zsh配置别名

```shell
alias gs="git status"
alias gcam="git commit -a -m"
alias gm="git merge --no-ff"
alias ggpull="git pull"
alias ggpush="git push"

alias idea="open -a Intellij\ IDEA"
alias goland="open -a GoLand"
alias typora="open -a typora"
```

