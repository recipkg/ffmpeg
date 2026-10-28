# ffprobe 和 ffplay 初步合同

来源：FFmpeg `n9.0.2` 提交 `946fcce07b6dcd0331c8cc609192aeff5e1924f8` 的 `doc/ffprobe.texi` 与 `doc/ffplay.texi`。条目为文档观察；除 `COVERAGE.md` 明确列出的 ffprobe 用例之外未实际验证。

## ffprobe

形式为 `ffprobe [options] input_url`。读取媒体信息，按选中的 writer 组织可解析的 section；有未能打开或识别的输入时返回正退出码。除非使用 `-o` 指定输出，结果写标准输出。容器与流元数据可出现在对应 section。

- `-of`、`-output_format`、`-print_format` 选择 writer，例如 JSON；文档还列出 default、compact、csv、flat、ini、xml。
- `-select_streams` 限制与流有关的展示；`-show_streams`、`-show_format`、`-show_packets`、`-show_frames` 选择相应 section。
- `-show_entries` 以冒号分隔 section，以逗号选择字段；未提供等号的 section 输出所有字段，带空等号的 section 不输出本地字段。不能假定输出字段顺序与请求顺序一致。
- `-show_error` 报告探测失败信息；`-count_frames` 和 `-count_packets` 在对应流 section 报告计数。不同 writer 的精确转义、缺失值及全部选项仍须补齐。

### JSON 字段、流选择和错误样本

本机 Linux 上的 `ffmpeg` / `ffprobe` 均报告 n9.0.2，构建启用 GPL 和 version3；本机二进制与固定源码的精确构建关系未验证。用 lavfi 创建一个 0.25 秒 Matroska：两条单声道 PCM s16le 音轨分别为 8000 Hz 和 16000 Hz，语言标签 `eng` / `jpn`，第一条为 default，容器标题 `Probe Sample`。具体可执行测试见 `ACCEPTANCE.md` 和标准库验收仓库。

- `-show_entries stream=index,codec_name,sample_rate,channels:stream_tags=language:stream_disposition=default -of json` 输出 `streams` 数组，分别含索引 0、1，`codec_name` 为 `pcm_s16le`、`sample_rate` 是字符串 `"8000"` / `"16000"`、`channels` 是数字 1；嵌套 `disposition.default` 为数字 1 / 0，`tags.language` 为 `eng` / `jpn`。该样本 JSON 还含空数组 `programs` 与 `stream_groups`；格式空格不属于合同。
- 添加 `-select_streams a:1` 只显示第二条流；`-select_streams v` 或 `a:5` 在该文件中成功并给出空 `streams`。无效选择器 `a:bogus` 失败并写 stderr，不应把语法错误伪装成空结果。
- `-show_entries format=format_name,duration:format_tags=title -of json` 的 `format` 包含 `format_name` 为 `matroska,webm`、`duration` 为字符串 `"0.250000"` 和 `tags.title` 为 `Probe Sample`。`-show_entries stream=` 的样本输出空 JSON 对象；`stream=index,unknown_field` 忽略未知字段，只显示索引。以上为所列构建及输入的样本，其他 demuxer、版本、writer 和标签情况仍待验证。
- 输入文件不存在时，`-show_error -of json` 在该 Linux 样本返回退出码 1，stdout 包含 `error.code = -2` 和非空 `error.string`，`-v error` 同时写 stderr；`-v quiet` 抑制 stderr 但保留 JSON 错误与非零退出。不要将该数值或语言相关错误文本推广到所有平台。

## ffplay

形式为 `ffplay [options] [input_url]`，上游使用 SDL 与 FFmpeg 库播放。`-an`、`-vn`、`-sn` 可分别禁用音频、视频、字幕；`-ss` 定位，通常可能落在最近的可定位点；`-t` 限制播放时长。`-nodisp` 禁用图形显示，`-fs` 以全屏启动，`-loop 0` 表示持续循环。显示设备、音频输出、按键、过滤图、退出行为及异常恢复尚未验收。

不得将命令文档或系统上游测试说成从本配方重建的 `ffprobe` / `ffplay` 已通过验收。
