#### 安装

```
brew install ffmpeg
```



#### 基础用法

```bash
ffmpeg -i 源文件         // 查看源文件信息
ffmpeg -i 源文件 目标文件 // 格式转换
```



#### 常用参数

```bash
-fs 15MB // file size 指定输出文件大小，不好使
-preset  // 压缩预设参数，可选 ultrafast / fast / medium / slow 等
```



#### 视频压缩

```bash
ffmpeg -i xxx.mp4 -c:v libx265 -preset ultrafast xxx2.mp4
```



#### 批量从视频中提取音频

```bash
for s in *.mp4;do ffmpeg -i $s ${s}.mp3;done
```

