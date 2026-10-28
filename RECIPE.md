# FFmpeg 软件配方（提取中）

```toml
format = "recipkg/1"

[package]
name = "ffmpeg"
version = "0.1.0"
kind = "software"
description = "FFmpeg 9.0.2 多媒体工具和组件的重建配方；当前覆盖不完整。"
entry = "BUILD_PROMPT.md"

[dependencies.quality]
git = "https://github.com/recipkg/recipkg.git"
rev = "41e5b55c6612a9a5a10e499b5ace6a1a3fed0f2b"
subdir = "recipes/software-quality"
```

来源是 [FFmpeg 官方 GitHub 镜像](https://github.com/FFmpeg/FFmpeg)，观察 tag `n9.0.2` 对应提交 `946fcce07b6dcd0331c8cc609192aeff5e1924f8`。上游多许可证，主要是 LGPL 2.1 或更高版本；某些可选组件为 GPL 或其他许可。详情以该提交的 `LICENSE.md` 和对应文件为准。

此版本是提取中的草案，结构合法不代表足以完整重建 FFmpeg。见 `COVERAGE.md`。
