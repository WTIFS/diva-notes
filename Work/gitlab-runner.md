# 安装

指引：https://docs.gitlab.com/runner/install/linux-repository.html



1. 添加源（CentOS）

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
```

2. 安装

```bash
sudo yum install gitlab-runner
# or sudo apt-get install gitlab-runner
```

3. 注册

```bash
sudo gitlab-runner register
```

之后输入 gitlab 地址和 token 等，从 gitlab 仓库 - Settings - Runners 里找



# 切换用户

默认使用 gitlab-runner 用户启动，可能会碰到缺少读写权限问题，可以设置使用别的用户。以 gitlab-runner 用户为例

```bash
sudo su
gitlab-runner stop
gitlab-runner uninstall
gitlab-runner install --working-directory /home/gitlab-runner  --user gitlab-runner
gitlab-runner start
ps aux | grep gitlab-runner
```



# 配置示例

```yaml
# .gitlab-ci.yml
before_script:
  - export GOPRIVATE=gitlab.xxx.com

stages:
  - build
  - test

build-job:
  stage: build
  tags:
    - xxx # 使用的 gitlab-runner 的 tag 名称
  only:
    - merge_requests
  script:
    - make build

test-job:
  stage: test
  tags:
    - xxx
  only:
    - merge_requests
  script:
    - make test
```

