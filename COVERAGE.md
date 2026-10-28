# 来源、证据与缺口

- 上游：https://github.com/FFmpeg/FFmpeg ，tag `n9.0.2`，源码提交 `946fcce07b6dcd0331c8cc609192aeff5e1924f8`。
- 阅读范围：根 README.md、LICENSE.md、`doc/ffmpeg.texi` 开头及官方命令文档；未审阅完整源码、测试套件及各组件文档。
- 本机观察：Arch 系统 `ffmpeg version n9.0.2`，构建配置包含 `--enable-gpl` 与 `--enable-version3`。使用 lavfi sine 生成 1 秒 WAV；本机 ffprobe JSON 显示 `pcm_s16le`、44100 Hz、1 通道、时长 1.000000 秒。对不存在输入返回 254。本机覆盖保护提示目标已存在并返回 0；其余平台与场景未验证。
- 未完成：完整 CLI 选项、过滤器、协议、编解码器、封装格式、时间戳、错误退出、`ffprobe` 和 `ffplay`、七个库的公开 API/ABI、全平台运行、性能、安全及许可证组件矩阵。上游测试套件、代表性源码及运行时仍需细读并逐项验证。
- 本机观察不是独立重建，也不证明其他构建具有相同能力。安装的上游依赖、操作系统、设备和外部服务没有打入配方。
