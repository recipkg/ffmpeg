# FFmpeg 来源与选择记录

## 候选选择

- 类别：命令行工具 / 多媒体处理。
- 选择理由：补足标准库中通用音视频转换、流选择、探测与过滤图这一类命令行能力；FFmpeg 官方 GitHub 镜像在 2026-10-29 显示约 66.0k stars。星数只用于发现和排序候选，不代表配方完整、质量或可重建性。
- 目标范围：固定版本 FFmpeg 9.0.2 的 `ffmpeg`、`ffprobe`、`ffplay` 与相关库的可观察行为。当前配方只细化了部分 CLI 行为；功能缺口和未验证范围以 `COVERAGE.md` 为准。

## 固定上游

- 上游项目：[FFmpeg/FFmpeg](https://github.com/FFmpeg/FFmpeg)，GitHub 页面说明这是 [git.ffmpeg.org/ffmpeg.git](https://git.ffmpeg.org/ffmpeg.git) 的镜像。
- 固定发布：[`n9.0.2`](https://github.com/FFmpeg/FFmpeg/releases/tag/n9.0.2)，发布页标记 tag 已验证；官方 [下载页](https://ffmpeg.org/download.html) 记录其于 2026-09-18 发布，并在核验日列为 9.0 分支最新稳定版。
- Tag 对象：`ce8f11b9fac4cd7ccfb41ca7c01bc17f5cda8cc3`。
- Tag 指向的源码提交：`946fcce07b6dcd0331c8cc609192aeff5e1924f8`。
- 核验日期：2026-10-29。

## 许可边界

上游 README 将代码库描述为主要采用 LGPL、可选组件采用 GPL。固定提交包含 GPL-2、GPL-3、LGPL-2.1 和 LGPL-3 许可证文本；实际适用许可取决于所用文件、启用组件及构建配置。不能仅凭 FFmpeg 名称或仓库顶层标签推断某个构建产物的许可证。核对入口：[固定提交的 README](https://github.com/FFmpeg/FFmpeg/blob/946fcce07b6dcd0331c8cc609192aeff5e1924f8/README.md)、[LICENSE.md](https://github.com/FFmpeg/FFmpeg/blob/946fcce07b6dcd0331c8cc609192aeff5e1924f8/LICENSE.md) 和对应 `COPYING.*` 文件。完整组件许可矩阵仍未整理。

## 观察范围

提取记录中的源码阅读覆盖根 README、许可证说明、命令行文档开头及流选择、`ffprobe` 和 `ffplay` 的部分选项；具体本机样本、环境、复核日期和缺口见 `COVERAGE.md` 及其链接的专项说明。本轮通过官方发布页、固定 Git tag 元数据和官方仓库页面复核版本与来源；没有下载或克隆完整上游源码，也没有把本轮来源核验写成独立重建或完整 CLI 验收。
