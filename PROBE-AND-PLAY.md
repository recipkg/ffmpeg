# ffprobe 和 ffplay 初步合同

来源：FFmpeg `n9.0.2` 提交 `946fcce07b6dcd0331c8cc609192aeff5e1924f8` 的 `doc/ffprobe.texi` 与 `doc/ffplay.texi`。条目为文档观察；除 `COVERAGE.md` 明确列出的 ffprobe 用例之外未实际验证。

## ffprobe

形式为 `ffprobe [options] input_url`。读取媒体信息，按选中的 writer 组织可解析的 section；有未能打开或识别的输入时返回正退出码。除非使用 `-o` 指定输出，结果写标准输出。容器与流元数据可出现在对应 section。

- `-of`、`-output_format`、`-print_format` 选择 writer，例如 JSON；文档还列出 default、compact、csv、flat、ini、xml。
- `-select_streams` 限制与流有关的展示；`-show_streams`、`-show_format`、`-show_packets`、`-show_frames` 选择相应 section。
- `-show_entries` 以冒号分隔 section，以逗号选择字段；未提供等号的 section 输出所有字段，带空等号的 section 不输出本地字段。不能假定输出字段顺序与请求顺序一致。
- `-show_error` 报告探测失败信息；`-count_frames` 和 `-count_packets` 在对应流 section 报告计数。不同 writer 的精确转义、缺失值及全部选项仍须补齐。

## ffplay

形式为 `ffplay [options] [input_url]`，上游使用 SDL 与 FFmpeg 库播放。`-an`、`-vn`、`-sn` 可分别禁用音频、视频、字幕；`-ss` 定位，通常可能落在最近的可定位点；`-t` 限制播放时长。`-nodisp` 禁用图形显示，`-fs` 以全屏启动，`-loop 0` 表示持续循环。显示设备、音频输出、按键、过滤图、退出行为及异常恢复尚未验收。

不得将命令文档或系统上游测试说成从本配方重建的 `ffprobe` / `ffplay` 已通过验收。
