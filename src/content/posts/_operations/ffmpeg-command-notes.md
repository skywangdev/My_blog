---
title: "FFmpeg 常用命令笔记：合并、压缩、裁剪与转码"
author: "Sky Wang"
pubDatetime: 2024-09-17T12:00:00+08:00
featured: false
draft: false
tags:
  - FFmpeg
  - 视频
  - 工具
description: "整理 FFmpeg 常用命令，包括批量合并视频、H.264/H.265 压缩、硬件编码、裁剪、音视频合并、图片转换和音频循环。"
---

<figure>
  <img src="https://data.skywangdev.com/blog/S-9.jpeg" alt="FFmpeg 常用命令封面图" />
  <figcaption class="text-center">FFmpeg 是处理音视频的瑞士军刀，日常最常用的是合并、压缩、裁剪和转码。</figcaption>
</figure>

这篇是旧 FFmpeg 笔记的整理版。原文里有一些 HTML 表格代码块，这里改成标准 Markdown 代码块，方便复制和阅读。

## Table of contents

## 合并一个文件夹内的所有视频

如果多个 MP4 编码一致，可以用 concat 方式无损合并。

```bash
find *.mp4 | sed 's:\ :\\\ :g' | sed 's/^/file /' > fl.txt
ffmpeg -f concat -i fl.txt -c copy output.mp4
rm fl.txt
```

如果路径或文件名比较复杂，可以加 `-safe 0`：

```bash
find *.mp4 | sed 's:\ :\\\ :g' | sed 's/^/file /' > fl.txt
ffmpeg -safe 0 -f concat -i fl.txt -c copy output.mp4
rm fl.txt
```

参考：

[How can I merge all the videos in a folder?](https://stackoverflow.com/questions/28922352/how-can-i-merge-all-the-videos-in-a-folder-to-make-a-single-video-file-using-ffm)

## 视频压缩：H.264 与 H.265

H.264 兼容性最好：

```bash
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -c:a aac output.mp4
```

H.265 体积更小，但兼容性略差：

```bash
ffmpeg -i input.mp4 -c:v libx265 -crf 28 -c:a aac output.mp4
```

如果希望在 Apple 设备上更好识别 H.265，可以加 `-vtag hvc1`：

```bash
ffmpeg -i input.mp4 -c:v libx265 -crf 28 -vtag hvc1 -c:a copy output_hevc.mp4
```

`CRF` 可以粗略理解为质量参数：

| CRF | 效果 |
| --- | --- |
| 数值越小 | 质量越高，文件越大 |
| 数值越大 | 文件越小，画质越差 |

常用范围：

- H.264：`18-28`
- H.265：`24-32`
- VP9：`28-36`

参考：

[FFmpeg CRF Guide](https://slhck.info/video/2017/02/24/crf-guide.html)

## 使用 NVIDIA 硬件编码

软件编码质量好，但速度慢。如果机器有 NVIDIA 显卡，可以用 NVENC。

H.264 硬件编码：

```bash
ffmpeg -i input.mp4 -c:v h264_nvenc -cq 23 -c:a aac output.mp4
```

H.265 硬件编码：

```bash
ffmpeg -i input.mp4 -c:v hevc_nvenc -cq 28 -c:a aac output.mp4
```

旧笔记里有一个实际结果：将视频从 H.264 转码到 H.265，体积从 3.8GB 降到 430MB，耗时约 55 分钟。

```bash
ffmpeg -i 1.mp4 -c:v libx265 -vtag hvc1 -c:a copy 1_hevc.mp4
```

## 按时间裁剪视频

使用原编码快速裁剪：

```bash
ffmpeg -ss 00:05 -to 08:53.500 -i input.mp4 -c copy output.mp4
```

另一个例子：

```bash
ffmpeg -ss 07:18 -to 13:45 -i aaa.mkv -c copy bbb.mkv
```

参数说明：

- `-ss`：开始时间
- `-to`：结束时间
- `-i`：输入文件
- `-c copy`：不重新编码，速度快

如果裁剪点不准，可以把 `-ss` 放到 `-i` 后面，精度更高，但速度可能更慢：

```bash
ffmpeg -i input.mp4 -ss 00:05 -to 08:53.500 -c copy output.mp4
```

## 合并视频和音频

视频保留原始编码，音频转为 AAC：

```bash
ffmpeg -i 1.mp4 -i 1.opus -c:v copy -c:a aac output.mp4
```

如果音频也能直接兼容，可以尝试：

```bash
ffmpeg -i 1.mp4 -i 1.m4a -c copy output.mp4
```

## 图片格式转换和缩放

PNG 转 JPG：

```bash
ffmpeg -i image.png -preset ultrafast image.jpg
```

修改图片尺寸：

```bash
ffmpeg -i image.jpeg -vf scale=413:626 2寸.jpeg
ffmpeg -i image.jpeg -vf scale=390:567 1寸.jpeg
```

如果只指定宽度，让高度等比例：

```bash
ffmpeg -i image.jpeg -vf scale=1200:-1 output.jpeg
```

## 音频重复

将一个音频重复 10 次：

```bash
ffmpeg -stream_loop 10 -i input.m4a -c copy output.m4a
```

注意 `-stream_loop 10` 表示额外循环 10 次，不是总共播放 10 次。

## Windows 上安装 FFmpeg

旧笔记里使用 scoop：

```powershell
scoop install ffmpeg
```

更新 scoop 安装的所有程序：

```powershell
scoop list | foreach { scoop update $_.Name }
```

## 小结

我最常用的 FFmpeg 命令可以归为四类：

- 合并：`concat + -c copy`
- 压缩：`libx264/libx265 + crf`
- 裁剪：`-ss + -to + -c copy`
- 硬件加速：`h264_nvenc/hevc_nvenc`

如果目标是“快”，优先 `-c copy` 或 NVENC。如果目标是“体积小且画质可控”，优先软件编码加 CRF。
