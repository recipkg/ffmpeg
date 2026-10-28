# 视频裁剪、缩放和帧率：上游样本

参考 [FFmpeg 过滤器手册](https://ffmpeg.org/ffmpeg-filters.html)及本机 `ffmpeg` / `ffprobe` n9.0.2。版本号与配方固定来源一致，但二进制的源码构建来源未核验。以下为特定输入、选项和构建上的观察，不是所有滤镜、编解码器、像素格式或时间戳的完整合同。

## 独立输入

在隔离目录自行生成 4 帧原始 RGB24 数据：每帧 4×4，每个像素 3 字节，按行排列，第 `n` 帧位置 `(x,y)` 的 RGB 值为 `(n*40,x*50,y*50)`，`n`、`x`、`y` 分别从 0 到 3。文件一共 192 字节；不依赖 FFmpeg 的图像源生成预期数据。作为输入的选项按顺序放在 `-i` 前：

```sh
ffmpeg -v error -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb ...
```

因此此样本为每秒 4 帧，总共 1 秒。其他文件格式的输入、压缩解码和时间基准不能从此推断。

## 裁剪和缩放

在上述输入之后设置 `-vf 'crop=2:2:1:1,scale=4:4:flags=neighbor' -fps_mode passthrough -c:v rawvideo -pix_fmt rgb24 -f rawvideo cropped.rgb`。样本输出为 4 帧 RGB24、每帧 4×4；第 `n` 帧 `(x,y)` 的 RGB 为 `(n*40,(1+x//2)*50,(1+y//2)*50)`，其中 `//` 为整数除法。即先截取原图左上角为 `(1,1)` 的 2×2 区域，再用邻近采样放大；顺序和每个像素均需核对。

对相同输入，`-vf 'crop=5:5:0:0'` 和 `-vf 'crop=0:2:0:0'` 均失败并产生 stderr；Linux 本机构建均退出 234。不要将该数值推广到其他平台，也不要假定错误时输出文件不存在或完整。其他缩放算法、色彩转换、色度抽样、旋转、宽高表达式和动态尺寸尚未验证。

## 帧率与无损转码

保持输入和原始 RGB 输出设置，仅用 `-vf fps=8 -fps_mode passthrough` 时，输出 8 帧，按输入帧下标排列为 `[0,0,1,1,2,2,3,3]`；用 `fps=2` 时输出两帧 `[0,2]`。按完整帧字节核对，不只检查输出大小或声明帧率。这里采用默认 fps 滤镜选项和规则时间戳；不同起始时间、末帧、舍入和可变帧率仍需验证。

从同一输入执行 `-vf fps=2 -fps_mode passthrough -c:v ffv1 out.mkv`；本机 `ffprobe` 报视频流 codec `ffv1`、宽高 4×4，两个解码帧的 `best_effort_timestamp_time` 为 `0.0` 和 `0.5` 秒。再将该文件解码为 RGB24 原始帧，字节应与原始第 0、2 帧完全相同。该样本需要 rawvideo、RGB24、`crop`、`scale`、`fps`、FFV1 和 Matroska 的相应编解码与封装能力；不能假设所有构建均提供。

运行 [standard-library-verification](https://github.com/recipkg/standard-library-verification) 的 `FFmpegSamples` 视频用例时，同时用 `RECIPKG_TEST_FFMPEG` 和 `RECIPKG_TEST_FFPROBE` 指定目标程序。测试程序在临时目录生成输入和独立预期字节，结束后清理。上游对照不能代替重建实现验收；硬件加速、性能、跨平台、不同构建及其他媒体格式均未验证。
