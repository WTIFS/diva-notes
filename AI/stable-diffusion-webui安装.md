stable-diffusion-webui 是一款可以在服务器上运行 stable-diffusion 的 WebUI 界面。通过安装它，你可以启动一个 AI作图后台程序并通过 WebUI 访问它。

项目的 github repo: https://github.com/AUTOMATIC1111/stable-diffusion-webui



# 安装

环境：CentOS

先安装一些依赖：

```bash
# torch 的 liblzma-dev 依赖
sudo yum install -y xz-devel

sudo yum install xdg-utils
```



安装 Python

1. 下载：https://www.python.org/ftp/python/，选择 3.10.6 版本（WebUI 使用的是这个版本）
2. 解压文件：`tar -zxvf Python-3.10.6.tgz，cd Python-3.10.6`
3. 配置编译保存的路径：`./configure --prefix=/usr/local/python3.10.6; sudo mkdir /usr/local/python3.10.6`
4. 编译：`make、sudo make altinstall`
5. 创建 python3.10 软连接：`sudo ln -s /usr/local/python3.10.6/bin/python3.10 /usr/local/bin/python3.10`
6. 如果后面启动 WebUI 时缺少 `fastapi` 的错，安装这个依赖：`pip3.10 install fastapi`



安装SSL

```bash
sudo yum install -y epel
sudo yum install -y openssl11-devel

cd Python-3.10.6
sed -i 's/PKG_CONFIG openssl /PKG_CONFIG openssl11 /g' configure
./configure --enable-optimizations
sudo make altinstall
```



下载 stable-diffusion-webui 仓库代码

```bash
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
```



修改 `webui-user.sh` 文件，修改里面的默认目录、 `python` 命令安装位置

```bash
vim webui-user.sh
install_dir="stable-diffusion-webui所在的目录"
python_cmd="python3.10"

# 添加如下参数：单个渲染速度下降，但多个上升
export COMMANDLINE_ARGS="--medvram"

# 添加如下参数：解决CUDA有时报错内存不够的问题，通过缩小单位内存大小修复：
export PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:32
```



# 启动

```bash
# 参数说明：
# --listen 开启本机外监听
# --port 80 可以指定为80端口
# --enable-insecure-extension-access 允许安装外部扩展
./webui.sh --listen --enable-insecure-extension-access
```



#### 可选：做成系统服务，崩溃自动重启

```bash
sudo vim /etc/systemd/system/sd.service

# 文件内容如下：
[Unit]
Description=stable diffusion web
After=network.target

[Service]
Type=simple
Environment=PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:32
ExecStart=/安装位置/stable-diffusion-webui/webui.sh --listen --enable-insecure-extension-access
User=你的用户名
Restart=1
RestartSec=5s

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl start sd
```

但是有时配置的 `PYTORCH_CUDA_ALLOC_CONF` 变量不对程序生效，仍然会报内存错误。不知道为什么



#### 查看显存

```
nvidia-smi
```



# 添加模型

```bash
wget "模型下载链接"
```

将下载的模型（`.ckpt文件` 或 `.safetensors文件`）放置于 `models/Stable-diffution/` 文件夹下，然后点击网页底部的 `Reload UI` 按钮重新加载 WebUI

Lora模型放到 `models/Lora` 文件夹下

> 在 WebUI 界面的 `Extentions` tab里，安装这个扩展可以方便的直接从 `civitai` 网站检索和下载模型：https://github.com/camenduru/sd-civitai-browser
>
> 安装后 `Reload UI`，可以看到多了一个 `CivitAi` tab，这个 tab 下勾选 `Search by term` 并输入关键词就可以检索了



# 扩展

可以安装一些扩展模块

浏览历史生成图片：https://github.com/yfszzx/stable-diffusion-webui-images-browser

生成相同构图、动作：https://github.com/Mikubill/sd-webui-controlnet

- 这个也需要下载模型，安装指引：https://www.bilibili.com/read/cv21829826?spm_id_from=333.999.0.0
- 模型下载地址：https://huggingface.co/lllyasviel/ControlNet/tree/main/models

Lora模型插件（现在 WebUI 已经内置 Lora功能了，一般不用）：https://github.com/kohya-ss/sd-webui-additional-networks




# 绘制

## settings 模块参数说明

拷贝别人图片里的参数时，可能有些需要到 `settings` tab里修改：

- Sampler parameters 下方的 eta(noise multiplier) for ancestral samplers：Novel AI 为 0.68
- Sampler parameters 下方的 Eta noise seed delta：Novel AI 为 31337
- Stable Diffusion 下的 Clip skip：很多人设为 2
- 设置后点击 Apply settings 和 Reload UI

## 绘制参数说明

- Prompt: 正向特征点：想要风格、图像的描述
- Negative Prompt: 负向特征点，不想要的风格或者图像信息
- Sampling Steps: 生成图片的迭代步数，也就是让AI推演多少步，这个数不会改变画面内容，只会让内容更精细，推荐值20~30
- Sampling method: 扩散去噪算法的采样模式，可以理解为AI推演的算法，推荐Euler a/Euler/DDIM等
- CFG Scale: 分类器自由引导尺度，这个值决定了按照设定生成的程度，开到越大，越接近给的提示词；开的越低，AI越放飞自我
- Batch count & Batch size: 每次出图的数量和次数
- Seed: 随机种子。同一台机器上，相同的参数+种子可以保证每次固定出同一张图片。
  - Extra: 以种子为基础，变化生成一些别的种子

- Hires. fix: 由于高清图片生成的效果比较一般，因此可以先生成较低分辨率的图片后，再用这个功能基于低分辨率图片做修复，效果会更好
  - Upscaler: 是用的算法
  - Denoising strength: 放飞自我强度


## img2img 图生图参数说明

Resize mode：比如根据一张1:1的图生成1:2的图，那么对于多出的空间有这些选项：

- Just resize：直接拉伸
- Crop and resize：裁剪多出的部分
- Resize and fill：填充多出的部分

Denoising strength：与原图一致性的程度。一般大于0.7出来的都是新效果，小于0.3基本就会原图缝缝补补

## 参数复制

在 civitai.com 里可以浏览别人的作品并复制参数，将参数拷贝至 `prompts` 一栏并点击 `Generate` 下的小箭头，便可自动解析和填充参数

## 图片保存

点击图片下方的 `Save` 按钮可生成图片下载链接，点击链接下载图片



# PNG Info 模块

可用 PNG Info 模块从 PNG图片中提取提示词



# 脚本功能

生成页面下方的 Script 里可以选择脚本。

把 https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Features 里的都看一遍，对各功能有详细介绍，非常有用

- Prompt matrix 脚本：使用 | 可以触发提示词矩阵，会分别遍历提示词生成图像

- Prompts from file or textbox：可以按行写提示词，会对每行都生成图

- X/Y/Z plot：设置三种可变参数，看微调对最后结果的影响。格式：

  - Simple ranges:
    - `1-5` = 1, 2, 3, 4, 5

  - Ranges with increment in bracket:
    - `1-5 (+2)` = 1, 3, 5
    - `10-5 (-3)` = 10, 7
    - `1-3 (+0.5)` = 1, 1.5, 2, 2.5, 3

  - Ranges with the count in square brackets:
    - `1-10 [5]` = 1, 3, 5, 7, 10
    - `0.0-1.0 [6]` = 0.0, 0.2, 0.4, 0.6, 0.8, 1.0



# 相关链接

入门教程：[元素同典：确实不完全科学的魔导书](https://docs.qq.com/doc/DWFdSTHJtQWRzYk9k)

奇怪咒语：

- https://wwc.lanzouv.com/i2Mp20dw7tre 密码:bq1q

- NSFW: https://wwl.lanzouy.com/ivusH0f9cdkf

艺术家列表：https://rentry.org/artists_sd-v1-4

词语列表：http://www.prompttool.com/NovelAI

Lora角色滤镜扩展教程：https://sparkly-day-9db.notion.site/AI-1962de6fa0b44378b2fed3b79df5252b





# 一些模型链接

在 https://civitai.com 模型页面的下载按钮上复制链接后，可以使用 `wget` 命令直接下载；也可以去 webUI 里用 civitai 插件下载

- 动漫风格 Anything V3: https://civitai.com/models/66/anything-v3
- 真人风格 ChilloutMix: https://civitai.com/models/6424/chilloutmix
- korean Doll Likeness(Lora): https://huggingface.co/dadadadadatou/KbrDollLikeness/tree/main
- St. Louis (Luxurious Wheels) (Azur Lane) (Lora): https://civitai.com/models/6669/st-louis-luxurious-wheels-azur-lane



