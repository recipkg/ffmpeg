# FFmpeg 时间定位与截取观察

本文只记录固定上游提交和本机 n9.0.2 的一组最小样本，不把局部观察扩展成所有格式的通用保证。

## 固定来源

- 上游版本：FFmpeg n9.0.2，提交 946fcce07b6dcd0331c8cc609192aeff5e1924f8。
- 行为说明：[固定版本 doc/ffmpeg.texi](https://github.com/FFmpeg/FFmpeg/blob/946fcce07b6dcd0331c8cc609192aeff5e1924f8/doc/ffmpeg.texi)，另见 [FFmpeg 命令行手册](https://www.ffmpeg.org/ffmpeg.html)。
- 实测目标：本机 ffmpeg/ffprobe n9.0.2。二进制来源与独立重建状态未确认。

## 可复现样本

测试使用四帧 4×4 RGB24 rawvideo，帧率 4 fps。每帧填入不同且固定的像素值，因此可通过输出字节确认源帧序号。命令和样本生成逻辑位于 standard-library-verification/tests/test_recipes.py 的 FFmpegSamples 类；该类 11 个测试方法在记录时全部通过。

在源时间戳从零开始的前提下，观察结果如下：

| 命令选项位置 | 结果帧 |
| --- | --- |
| 输入前 `-ss 0.5` | 2、3 |
| 输出前 `-ss 0.5` | 2、3 |
| 输入前 `-t 0.5` | 0、1 |
| 输出前 `-t 0.5` | 0、1 |
| 输入前 `-ss 0.5 -t 0.5` | 2、3 |
| 输出前 `-ss 0.5 -t 0.5` | 2、3 |
| 输出前 `-to 0.75` | 0、1、2 |
| 输出前 `-t 0.25 -to 0.75` | 0 |

这些样例核对了输入与输出选项的简单位置差异、seek 窗口、时长、结束位置，以及同时指定 -t 和 -to 时由 -t 优先的结果。它们是 rawvideo 的零起始样本，不能代表长 GOP 文件上的 seek 读取成本或关键帧边界。

## 单 GOP 输入 seek 对照

为区分快速 rawvideo seek 与真正的关键帧 seek，再生成 8 帧 16×16 RGB24，帧率 4 fps，第 i 帧像素 (x,y) 为 (4×i, 5×x, 5×y)。使用 MPEG-2 编码，最大 GOP 长度 8、关闭 B 帧、将 scene-change 阈值设高。`ffprobe` 确认关键帧标志为 `[1,0,0,0,0,0,0,0]`，因此 0.5 秒目标落在第 0 帧关键帧之后、下一关键帧之前。

先将完整压缩样本解码为 RGB24 基线；随后从输入位置 `-ss 0.5` 分别以默认 accurate seek 转码、以 `-noaccurate_seek` 转码、以及 streamcopy。再次解码后逐字节比较：默认准确转码仅保留基线帧 2–7；禁用 accurate seek 的转码和输入 streamcopy 都保留全部 8 帧，包括 seek point 到目标之间的前两帧。这些结果复核了本机 FFmpeg n9.0.2 的输入 seek 行为。它们不说明输出 seek 的 streamcopy 行为，也未测量 seek 性能。

## 上游规则与本样本的边界

- 选项位置决定参数属于下一个输入还是输出文件；该规则也适用于这里测试的 -ss、-t 和 -to。
- 输入 -ss 会定位到目标前最近的 seek point。accurate seek 默认开启；转码时会解码并丢弃 seek point 与目标之间的数据。streamcopy 或禁用 accurate seek 时，这段内容可能保留在输出中。当前 rawvideo 样本不能覆盖此差异。
- 输出 -ss 会先解码并丢弃时间戳早于目标的数据；当前样本确认输出帧从目标时间开始，但没有测量复杂输入的性能。
- -t 表示持续时长；-to 表示停止位置。输入选项限制读取，输出选项限制输出。手册规定二者同时出现时 -t 优先；样本只复核了输出选项的优先级。
- -seek_timestamp 会影响非零起始时间戳上的绝对/相对 seek 解释；未在本轮生成此类输入。

## 暂不承诺

本观察未验证含关键帧间隔的 H.264/H.265 文件、streamcopy 下的首尾包、非零起始时间戳、seek_timestamp、accurate_seek 的关闭、实时或不可 seek 输入、网络协议、封装格式差异以及跨平台差异。若这些成为目标功能，必须先为每种行为建立独立的可复现样本和帧/时间戳断言。
