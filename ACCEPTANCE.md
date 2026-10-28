# FFmpeg 验收条件（草案）

验收工具须是根据配方生成的实现；如果还使用系统 FFmpeg，结果仅是上游观察，不是重建验收。所有用例应在隔离临时目录运行，记录二进制版本、构建配置、命令、输出和退出状态。

1. `ffmpeg -version` 输出可识别的版本；`-h` 和能力列表能让用户了解当前构建支持的项目。
2. 用 1 秒、1000 Hz 的 lavfi sine 输入生成 PCM 16-bit WAV；用 `ffprobe -v error -show_entries stream=codec_name,sample_rate,channels:format=duration -of json` 检查 codec 为 `pcm_s16le`、采样率 44100、单声道、时长约 1 秒。该用例要求构建支持所需输入过滤器、WAV 与 PCM。
3. 对已存在文件运行 `-n`，比较前后文件字节或哈希，确认没有覆盖；另测 `-y` 允许覆盖。记录实际退出状态和 stderr。
4. 对不存在的输入执行转换，要求不会产生可误认为完整结果的输出，并记录错误与退出状态。
5. 补充独立可复现用例：多输入及 `-map`、转封装 `-c copy`、时间范围和选项作用域、简单与复杂过滤图、字幕和 metadata、管道、网络与不可用设备、坏输入、各构建配置和系统平台。
6. 必须补充 `ffprobe`、`ffplay` 和所有主要库的公共接口及兼容性验收，含代表性支持的格式、编解码器、过滤器、协议和外部依赖；不能把上面几个样例当作全量验收。
7. 若宣称性能，使用明确版本、构建配置、目标设备、输入、输出、并发、命令和统计口径实测；没有目标或未测量则明确写未知。

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
