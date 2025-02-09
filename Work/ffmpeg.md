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
-c:v h264 // 指定视频编码格式（默认h264）

-preset  // 压缩预设参数，可选 ultrafast / fast / medium / slow 等
-b:v 4M  // 码率 4Mbps，v表示只设置视频，a表示设置音频
-vf yadif=1 // 反交错
-vf setdar=dar=16/9 // 设置播放像素比例，多个vf时用逗号分隔
-s "1920*1080" // 分辨率，有时需要加引号
-c copy      // 不进行编解码，不能和 -b 一起使用
-r 29.97     // 指定帧率
-vol 256     // 音量，256=原音量，512=两倍音量

# 截取相关。如果放在 -i 之后，则会精确剪切，但可能会以黑屏开头；放在之后则会寻找前面最近一个关键帧
-ss 1:23.456 // 从这个时间开始裁剪
-to 1:23:456 // 裁剪至这个时间

# 音频
-acodec copy // 不对音频编码
-c:a copy    // 同上
```



#### 硬件加速

```bash
ffmpeg -codecs | grep "h264" #查看支持的编解码器
ffmpeg -codecs | findstr "h264" (win)
-c:v h264_qsv # intel 硬件加速编码
-c:v h264_amf # amd 硬件加速
-c:v libx265 -vtag hvc1 # 兼容 quicktime
```





```bash
ffmpeg \
  -hwaccel videotoolbox \
  -hwaccel_output_format videotoolbox_vld \
  -i input.mov \
  -c:v hevc_videotoolbox \ # h265 压缩率更高，更耗CPU
  -c:v h264_videotoolbox \ # h264 更大
  -profile:v main \
  -b:v 4M \
  -tag:v hvc1 \ # 兼容 apple quicktime
  /tmp/test.mp4
```

- -hwaccel videotoolbox 提供videotoolbox硬件上下文，主要功能是硬件内存buffer分配；它的另一个功能是封装了CPU和GPU之间的数据传输拷贝，但在这个例子中用不到
- -hwaccel_output_format videotoolbox_vld 指定了硬件输出图像格式；
  - 如不指定，一般默认会选中NV12
  - 指定了videotoolbox_vld，FFmpeg不会自动插入CPU filter，图像在GPU内存，如果需要的话，可以用hwdownload filter下载到CPU
  - 用软件解码器的时候，也可以用硬件filter，只需要先hwupload到GPU
- FFmpeg videotoolbox视频解码不是单独的一个解码器，是在h264、h265解码器上的加速。有几个硬件视频解码加速都是这么实现的，一个公共的管理调度 + 不同的硬件加速。最新的是av1解码，FFmpeg内置的av1解码器，没有软件实现，全靠硬件加速完成解码功能。指定了-hwaccel videotoolbox之后，FFmpeg自动启用videotoolbox解码
- -c:v hevc_videotoolbox指定编码器
- -tag:v hvc1与mp4封装有关，Apple平台要求hvc1，不指定的话默认是hev1，Apple系统播放器不支持



#### 视频压缩

```bash
ffmpeg -i xxx.mp4 -c:v libx265 -preset ultrafast xxx2.mp4
```



#### 从视频中提取音频

```bash
ffmpeg in.mp4 -ab 码率k out.mp3

# 批量
for s in *.mp4;do ffmpeg -i $s -ab 256k ${s}.mp3;done
```



#### 常用码率

1. 360p或480p的视频：这类视频的比特率最好大于等于0.8M。

2. 720p的视频：这类视频的比特率应该大于等于1.5M，其中以2M比特率最佳。

3. 1080p的视频：≥2.5M，最优比特率为4M-8M。

4. 4k视频：4k作为作为目前画质最高的视频类型，其比特率一般为12M。



#### 批量按章节切割

```bash
ffprobe -i xxx.mp4 2>&1 | grep Chapter | sed -E "s/ *Chapter #([0-9]+):([0-9]+): start ([0-9]+\.[0-9]+), end ([0-9]+\.[0-9]+)/ -i xxx.mp4 -ss \3 -to \4 -c copy xxx-\2.mp4/" | xargs -n 11 ffmpeg
```

