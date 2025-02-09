# gitlab 代码仓库瘦身



# 安装 git-filter-repo

`yum` 安装：

```
yum install git-filter-repo
```

或者直接从 github 上下载：

```go
https://github.com/newren/git-filter-repo/blob/main/git-filter-repo
```



# 从存储库历史记录中清除文件

1. gitlab - Settings - General - Advanced - Export 导出并下载项目

2. 使用 `tar` 解压缩备份：

   ```go
   mkdir tmp
   mv project-backup.tar.gz tmp/
   cd tmp
   tar xzf project-backup.tar.gz
   cd project.git
   ```

3. 使用 `--bare` 和 `--mirror` 选项从包中克隆存储库的新副本：

   ```shell
   git clone --bare --mirror project.bundle
   ```

4. 清除所有大于10M的文件

   ```go
   git-filter-repo --strip-blobs-bigger-than 10M
   ```

5. 由于从捆绑文件克隆会将 `origin` 远程设置为本地捆绑文件，因此请删除此 `origin` 远程，并将其设置为存储库的 URL：

   ```go
   git remote remove origin
   git remote add origin https://gitlab.example.com/<namespace>/<project_name>.git
   ```

6. 强制推送更改以覆盖 GitLab 上的所有分支：

   ```shell
   git push origin --force 'refs/heads/*'
   git push origin --force 'refs/tags/*'
   git push origin --force 'refs/replace/*'
   ```

7. 运行存储库清理：前面执行 `git-filter-repo` 命令后，会在 `filter-repo` 文件夹下生成一个 `commit-map` 文件，用于标识可清理的文件，将该文件上传至 gitlab-Settings-Repository-Repository cleanup 里，然后 gitlab 就会释放空间了