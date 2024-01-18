#### 安装

```
brew install ffmpeg
```



#### 基础用法

```bash
ffmpeg -i 源文件         // 查看源文件信息
ffmpeg -i 源文件 目标文件 // 格式转换
ffprobe xxx.mp4         // 查看视频信息
```



#### 常用参数

ffmpeg会默认以输入视频原来的编码格式进行编码

```bash
-fs 15MB // file size 指定输出文件大小，不好使
-c:v h264 // 指定视频编码格式（默认h264）
-preset  // 压缩预设参数，可选 ultrafast / fast / medium / slow 等
-b:v 4M  // 码率 4Mbps，v表示只设置视频，a表示设置音频
-vf yadif=1 // 反交错
-vf setdar=dar=16/9 // 设置播放像素比例，多个vf时用逗号分隔
-s 1920*1080 // 分辨率
-c copy      // 不进行编解码，不能和 -b 一起使用
-r 29.97     // 指定帧率
-vol 256     // 音量，256=原音量，512=两倍音量

-ss 1:23.456 // 从这个时间开始裁剪
-to 1:23:456 // 裁剪至这个时间，建议放在 -i 之前
```



#### 视频压缩

```bash
ffmpeg -i xxx.mp4 -c:v libx265 -preset ultrafast xxx2.mp4
```



#### 批量从视频中提取音频

```bash
for s in *.mp4;do ffmpeg -i $s ${s}.mp3;done
```

