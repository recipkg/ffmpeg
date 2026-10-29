# 来源、证据与缺口

- 上游：https://github.com/FFmpeg/FFmpeg ，tag `n9.0.2`，源码提交 `946fcce07b6dcd0331c8cc609192aeff5e1924f8`。
- 阅读范围：根 README.md、LICENSE.md、`doc/ffmpeg.texi` 开头、流选择章节，以及 `doc/ffprobe.texi`、`doc/ffplay.texi` 的部分选项和官方命令文档；未审阅完整源码、测试套件及各组件文档。
- 本机观察：Arch 系统 `ffmpeg version n9.0.2`，构建配置包含 `--enable-gpl` 与 `--enable-version3`。使用 lavfi sine 生成 1 秒 WAV；本机 ffprobe JSON 显示 `pcm_s16le`、44100 Hz、1 通道、时长 1.000000 秒。对不存在输入返回 254。本机覆盖保护提示目标已存在并返回 0；其余平台与场景未验证。
- 未完成：完整 CLI 选项、过滤器、协议、编解码器、封装格式、时间戳、错误退出、`ffprobe` 和 `ffplay`、七个库的公开 API/ABI、全平台运行、性能、安全及许可证组件矩阵。上游测试套件、代表性源码及运行时仍需细读并逐项验证。
- 本机观察不是独立重建，也不证明其他构建具有相同能力。安装的上游依赖、操作系统、设备和外部服务没有打入配方。

- 多音轨实际对照：使用本机 n9.0.2 构造 Matroska 文件，含 44100 Hz、PCM 16-bit 的单声道 440 Hz 与双声道 880 Hz 音轨；默认标记在第一轨时自动选择单声道，默认标记在第二轨时自动选择双声道。分别显式映射两轨并以 FFmpeg hash muxer SHA-256 对比对应编码流，复制输出与输入匹配。源码 `fftools/ffmpeg_mux_init.c` 的 `map_auto_audio` 有每输入评分与跨输入比较步骤，文档概述遗漏默认标记影响；无多输入比较实验。

- 多输入/映射对照：本机 n9.0.2 以 A（默认单声道、非默认双声道）和 B（双声道）作为两个独立 Matroska 输入；自动输出与 B 的编码数据 SHA-256 相同。`-map 0:a -map -0:a:0`、`-map 0:v? -map 0:a:1` 输出均与 A 的第二轨哈希相同；不存在的视频流且不加 `?` 返回 234，stderr 指出无匹配，未创建输出。无效输入索引 `-map 1:a?` 本机也返回 234、未创建输出。

- 音频过滤图：本机 n9.0.2 用两条 8000 Hz PCM 音频混合，逐样本与输入的半幅和比较；用 `asplit` 输出静音和原音两个 WAV；验证不存在、重复映射的标签、未连接的输出和过滤后 streamcopy 均失败。无标签静音输出与显式映射的原音可共存于首个 Matroska 文件。详见 `FILTERGRAPH-AUDIO.md` 和验收用例。当前只有本机构建及所写的场景得到验证；未覆盖通用图语法、所有时间戳、重采样、布局和其他过滤器。

- 视频样本：本机 n9.0.2 使用独立生成的 4×4 RGB24 像素，验证裁剪、邻近放大、两种无效裁剪、帧率升降及 FFV1/Matroska 转码，逐帧比对像素并检查两个时间戳。详见 `VIDEO-FILTERS-OBSERVED.md`；其他像素格式、视频滤镜、编码器、性能、硬件及重建未验证。

- `ffprobe` JSON：本机 n9.0.2 对双音轨合成 Matroska 验证字段类型、标签、默认标记、流筛选、空字段选择和不存在文件的错误 section；命令和边界见 `PROBE-AND-PLAY.md`。只证明所列上游构建与样本，其他格式、writer、协议、计数、实时性及重建尚未验收。

- 2026-10-29 自动样本复核：在 Omarchy Linux x86_64、Python 3.14.8 上执行 `python3 -m unittest discover -s tests -p test_recipes.py -k FFmpegSamples -v`；默认目标为 `/usr/bin/ffmpeg` 和 `/usr/bin/ffprobe`，两者均报告 n9.0.2。9 项通过：自动音轨选择与 stream copy、复杂音频混合/分支与映射错误、多输入/负向/可选流映射、ffprobe JSON 字段/筛选/错误、未标记过滤输出、RGB24 裁剪与邻近缩放、帧率升降选帧、FFV1 解码像素及时间戳。测试用例各自在临时目录生成素材。此结果是固定版本上游工具的复核，不是配方重建或其他发行构建的验收。


## 时间定位与截取迭代

- 从固定上游手册提取输入/输出 `-ss`、`-t`、`-to` 的位置语义、默认 accurate seek 对转码与 streamcopy 的区别，以及 `-t` 和 `-to` 并用时的优先级。专门说明选项作用于下一个输入或输出文件。
- 在本机上游 FFmpeg/ffprobe n9.0.2 上用 4 帧、4 fps、4×4 RGB24 rawvideo 对照输入/输出 seek、输入/输出 duration、end position、seek window 和 `-t` 优先级；11 个 `FFmpegSamples` 测试方法全部通过，其中新增时间用例逐字节核对帧内容。
- 增加 8 帧 MPEG-2 单 GOP 样本：`ffprobe` 确认只有第 0 帧为关键帧。输入 `-ss 0.5` 默认转码解码结果逐字节等于完整解码基线的第 2–7 帧；`-noaccurate_seek` 转码和输入 streamcopy 均等于全部 8 帧基线。测试同时验证编码关键帧布局，避免把 rawvideo 的快速精确 seek 误当成 GOP 行为。
- 这是固定上游程序的观察，不是独立重建验收。非零起始时间戳、复杂 GOP、多种容器、不可 seek/live 输入和平台差异仍未覆盖。
