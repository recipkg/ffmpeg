# FFmpeg 验收条件（草案）

验收工具须是根据配方生成的实现；如果还使用系统 FFmpeg，结果仅是上游观察，不是重建验收。所有用例应在隔离临时目录运行，记录二进制版本、构建配置、命令、输出和退出状态。

1. `ffmpeg -version` 输出可识别的版本；`-h` 和能力列表能让用户了解当前构建支持的项目。
2. 用 1 秒、1000 Hz 的 lavfi sine 输入生成 PCM 16-bit WAV；用 `ffprobe -v error -show_entries stream=codec_name,sample_rate,channels:format=duration -of json` 检查 codec 为 `pcm_s16le`、采样率 44100、单声道、时长约 1 秒。该用例要求构建支持所需输入过滤器、WAV 与 PCM。
3. 对已存在文件运行 `-n`，比较前后文件字节或哈希，确认没有覆盖；另测 `-y` 允许覆盖。记录实际退出状态和 stderr。
4. 对不存在的输入执行转换，要求不会产生可误认为完整结果的输出，并记录错误与退出状态。
5. 补充独立可复现用例：多输入及 `-map`、转封装 `-c copy`、时间范围和选项作用域、简单与复杂过滤图、字幕和 metadata、管道、网络与不可用设备、坏输入、各构建配置和系统平台。
6. 必须补充 `ffprobe`、`ffplay` 和所有主要库的公共接口及兼容性验收，含代表性支持的格式、编解码器、过滤器、协议和外部依赖；不能把上面几个样例当作全量验收。
7. 若宣称性能，使用明确版本、构建配置、目标设备、输入、输出、并发、命令和统计口径实测；没有目标或未测量则明确写未知。
8. 按 `VIDEO-FILTERS-OBSERVED.md` 和 `FFmpegSamples` 视频用例，以独立 RGB24 数据检查裁剪、邻近缩放、无效裁剪、帧率升降、FFV1 转码的像素、帧顺序和时间戳；记录目标构建能力及未运行项。通过样本不代表完整视频过滤或跨平台兼容。

## 多音轨和默认标记的可复现用例

在空的临时目录，生成一秒、两个 PCM 音轨的 Matroska 文件：

```sh
ffmpeg -v error -f lavfi -i 'sine=frequency=440:duration=1' -f lavfi -i 'sine=frequency=880:duration=1' -map 0:a -map 1:a -c:a pcm_s16le -ac:a:0 1 -ac:a:1 2 -disposition:a:0 default -disposition:a:1 0 input.mkv
ffmpeg -v error -i input.mkv -c:a copy auto.mkv
ffmpeg -v error -i input.mkv -map 0:a:1 -c:a copy selected.mkv
ffprobe -v error -show_entries stream=index,codec_name,channels:stream_disposition=default -of json input.mkv
ffprobe -v error -show_entries stream=index,codec_name,channels -of json auto.mkv
ffprobe -v error -show_entries stream=index,codec_name,channels -of json selected.mkv
```

本机 n9.0.2 的输入为单声道默认流和双声道非默认流；自动输出为单声道，显式 `-map 0:a:1` 输出为双声道。再生成仅把默认标记换到第二流的独立文件并自动映射，输出应为双声道；以 `-f hash -hash sha256` 分别计算输入选择流与复制输出编码数据的哈希并比较。该样例所需的 lavfi、Matroska 和 PCM 必须可用，结果不推广到未测试的格式、选项和构建。

## 多输入、负向和可选映射

接续上面的 `input.mkv`：再生成一条双声道 PCM 流的 `other.mkv`。使用 `ffmpeg -v error -f lavfi -i 'sine=frequency=660:duration=1' -c:a pcm_s16le -ac 2 other.mkv`。

```sh
ffmpeg -v error -i input.mkv -i other.mkv -c:a copy auto-two.mkv
ffmpeg -v error -i input.mkv -map 0:a -map -0:a:0 -c:a copy negative.mkv
ffmpeg -v error -i input.mkv -map '0:v?' -map 0:a:1 -c:a copy optional.mkv
ffmpeg -v error -i input.mkv -map 0:v -c:a copy missing.mkv
```

前 3 条应成功：`auto-two.mkv` 的音频编码流哈希与 `other.mkv` 相同；`negative.mkv` 和 `optional.mkv` 的音频编码流哈希均与 `input.mkv` 的第二条音轨相同。第四条在输入没有视频时应失败，检查 stderr，记录退出码且确认未创建输出；本机 Linux n9.0.2 返回 234，不将其推广到其他平台。再用不存在的输入文件索引加 `?`，应失败，不能将可选流误当成可选输入文件。

## ffprobe JSON 字段与错误

在隔离目录生成双音轨文件（要求构建支持 lavfi、PCM 与 Matroska）：

```sh
ffmpeg -v error -f lavfi -i 'sine=frequency=440:sample_rate=8000:duration=0.25' -f lavfi -i 'sine=frequency=660:sample_rate=16000:duration=0.25' -map 0:a -map 1:a -c:a pcm_s16le -metadata 'title=Probe Sample' -metadata:s:a:0 language=eng -metadata:s:a:1 language=jpn -disposition:a:0 default -disposition:a:1 0 probe.mkv
ffprobe -v error -show_entries 'stream=index,codec_name,sample_rate,channels:stream_tags=language:stream_disposition=default' -of json probe.mkv
ffprobe -v error -select_streams a:1 -show_entries 'stream=index,codec_name,sample_rate,channels:stream_tags=language:stream_disposition=default' -of json probe.mkv
ffprobe -v error -show_entries 'format=format_name,duration:format_tags=title' -of json probe.mkv
ffprobe -v error -show_error -of json missing.mkv
```

解析 JSON 并按 `PROBE-AND-PLAY.md` 检查字段值和 JSON 类型；第二条查询只返回索引 1。分别检查 `v` 和 `a:5` 为成功的空流结果，`a:bogus` 为错误；`stream=` 不显示字段，未知字段不会凭空出现。最后一条命令应返回非零并提供错误 section 与诊断；再用 `-v quiet` 核对 JSON 保留而 stderr 为空。自动化样本见标准库验收仓库的 `FFmpegSamples`；只有指定候选 ffmpeg 与 ffprobe 并运行后才能作为重建验收。

## 复杂音频过滤图

以下命令在空目录中执行，要求构建支持列出的滤镜、WAV 和 16 位 PCM：

```sh
ffmpeg -v error -f lavfi -i 'sine=frequency=440:sample_rate=8000:duration=0.25' -c:a pcm_s16le a.wav
ffmpeg -v error -f lavfi -i 'sine=frequency=880:sample_rate=8000:duration=0.125' -c:a pcm_s16le b.wav
ffmpeg -v error -i a.wav -i b.wav -filter_complex '[0:a]volume=0.5[a0];[1:a]volume=0.5[a1];[a0][a1]amix=inputs=2:duration=shortest:normalize=0[mix]' -map '[mix]' -c:a pcm_s16le mix.wav
ffmpeg -v error -i a.wav -filter_complex '[0:a]asplit=2[a][b];[a]volume=0[sil];[b]volume=1[orig]' -map '[sil]' -c:a pcm_s16le silent.wav -map '[orig]' -c:a pcm_s16le original.wav
```

以独立 WAV 读取器检查全部输出为单声道、8000 Hz、16 位 PCM。输入分别是 2000 和 1000 个样本，混音是 1000 个样本且不全为零；每个混音样本与相同位置两个输入样本之和的一半相差最多 1 个整数单位。`silent.wav` 有 2000 个全零样本；`original.wav` 的 2000 个样本与 `a.wav` 完全相同。

```sh
ffmpeg -v error -i a.wav -filter_complex '[0:a]volume=1[out]' -map '[missing]' -c:a pcm_s16le missing.wav
ffmpeg -v error -i a.wav -filter_complex '[0:a]asplit=2[out][unused]' -map '[out]' -c:a pcm_s16le unconnected.wav
ffmpeg -v error -i a.wav -filter_complex '[0:a]volume=0.5[out]' -map '[out]' -c:a copy copy.wav
ffmpeg -v error -i a.wav -filter_complex '[0:a]volume=1[out]' -map '[out]' first.wav -map '[out]' second.wav
```

这四条都应失败且产生诊断；记录退出状态，不假定错误时是否创建文件。不要只凭文件存在判断成功。

```sh
ffmpeg -v error -i a.wav -filter_complex '[0:a]volume=0' -c:a pcm_s16le auto-filter.mkv
ffmpeg -v error -i a.wav -filter_complex '[0:a]volume=0' -map 0:a -c:a pcm_s16le mapped-filter.mkv
```

检查 `auto-filter.mkv` 只有一条静音音轨；`mapped-filter.mkv` 有两条音轨，按顺序分别为静音和与 `a.wav` 相同的原始样本。以上行为在本机上游工具观察；只有对生成的目标程序运行同样检查才能算重建验收。


## 时间定位与截取

在空的临时目录生成四帧 4×4 RGB24 原始视频，帧率 4 fps。帧 i 的像素 (x,y) 为 (40×i, 50×x, 50×y)，均为 0–255 的 RGB 分量：

```python
from pathlib import Path
frames = [bytes(v for y in range(4) for x in range(4)
                for v in (i * 40, x * 50, y * 50)) for i in range(4)]
Path('source.rgb').write_bytes(b''.join(frames))
```

把选项分别放在输入 -i 之前和输出文件名前，逐字节检查以下结果：

```sh
ffmpeg -ss 0.5 -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -c:v rawvideo -pix_fmt rgb24 -f rawvideo input-seek.rgb
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -ss 0.5 -c:v rawvideo -pix_fmt rgb24 -f rawvideo output-seek.rgb
ffmpeg -t 0.5 -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -c:v rawvideo -pix_fmt rgb24 -f rawvideo input-duration.rgb
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -t 0.5 -c:v rawvideo -pix_fmt rgb24 -f rawvideo output-duration.rgb
ffmpeg -to 0.75 -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -c:v rawvideo -pix_fmt rgb24 -f rawvideo input-end-position.rgb
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -to 0.75 -c:v rawvideo -pix_fmt rgb24 -f rawvideo output-end-position.rgb
ffmpeg -ss 0.5 -t 0.5 -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -c:v rawvideo -pix_fmt rgb24 -f rawvideo input-seek-window.rgb
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -ss 0.5 -t 0.5 -c:v rawvideo -pix_fmt rgb24 -f rawvideo output-seek-window.rgb
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 4x4 -framerate 4 -i source.rgb -t 0.25 -to 0.75 -c:v rawvideo -pix_fmt rgb24 -f rawvideo priority.rgb
```

输入与输出 -ss 0.5 均输出源帧 2、3；输入与输出 -t 0.5 均输出帧 0、1；输入与输出 -to 0.75 均输出帧 0、1、2；两个 seek-window 命令均输出帧 2、3；同时给出 -t 0.25 -to 0.75 时只输出帧 0。对每个输出文件按上面的帧字节逐帧比较。对应自动化用例位于 standard-library-verification/tests/test_recipes.py 的 FFmpegSamples.test_input_output_seek_and_time_bounds；固定上游 n9.0.2 本机运行 11 个 FFmpegSamples 测试方法通过。


### 输入 seek 与 GOP 关键帧

在同一隔离目录另生成 8 帧 16×16 RGB24、4 fps 样本；第 i 帧像素 (x,y) 为 (4×i, 5×x, 5×y)。编码成 GOP 长度为 8 的 MPEG-2，关闭 B 帧并禁用样本触发的额外关键帧。先确认只有第 0 帧是关键帧：

```python
from pathlib import Path
frames = [bytes(v for y in range(16) for x in range(16)
                for v in (i * 4, x * 5, y * 5)) for i in range(8)]
Path('gop-source.rgb').write_bytes(b''.join(frames))
```

```sh
ffmpeg -f rawvideo -pixel_format rgb24 -video_size 16x16 -framerate 4 -i gop-source.rgb -an -c:v mpeg2video -g 8 -bf 0 -sc_threshold 1000000000 -q:v 2 -pix_fmt yuv420p gop.mkv
ffprobe -v error -select_streams v:0 -show_entries frame=key_frame -show_frames -of json gop.mkv
ffmpeg -i gop.mkv -an -fps_mode passthrough -c:v rawvideo -pix_fmt rgb24 -f rawvideo baseline.rgb
ffmpeg -ss 0.5 -i gop.mkv -an -fps_mode passthrough -c:v ffv1 accurate.mkv
ffmpeg -noaccurate_seek -ss 0.5 -i gop.mkv -an -fps_mode passthrough -c:v ffv1 noaccurate.mkv
ffmpeg -ss 0.5 -i gop.mkv -an -c:v copy stream-copy.mkv
ffmpeg -i accurate.mkv -an -fps_mode passthrough -c:v rawvideo -pix_fmt rgb24 -f rawvideo accurate.rgb
ffmpeg -i noaccurate.mkv -an -fps_mode passthrough -c:v rawvideo -pix_fmt rgb24 -f rawvideo noaccurate.rgb
ffmpeg -i stream-copy.mkv -an -fps_mode passthrough -c:v rawvideo -pix_fmt rgb24 -f rawvideo stream-copy.rgb
```

检查关键帧标志为 [1,0,0,0,0,0,0,0]。以每帧 16×16×3 字节切分 baseline.rgb：accurate.rgb 必须逐字节等于第 2–7 帧；noaccurate.rgb 与 stream-copy.rgb 必须逐字节等于完整 8 帧。此对照只覆盖本样本的输入 seek；不外推到非零起始时间戳、长 GOP 不同封装、B 帧、输出 streamcopy 或实时输入。

前述四帧 rawvideo 样本时间戳从零开始；GOP 小节只验证了一个 MPEG-2 单 GOP 输入样本。非零起始时间戳、B 帧、其他 codec/容器、输出 seek 的 streamcopy、实时/不可 seek 来源及跨平台行为仍未覆盖。所有结果只复核本机上游行为，不代表独立重建已通过。
