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



