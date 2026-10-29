# FFmpeg · Recipkg

**多媒体处理工具和组件：提炼音视频转换、流选择、过滤图及相关接口。**

[标准库](https://github.com/recipkg/standard-library) · [配方清单](RECIPE.md) · [来源记录](SOURCES.md) · [构建要求](BUILD_PROMPT.md) · [验收条件](ACCEPTANCE.md) · [覆盖状态](COVERAGE.md) · [时间定位与截取](SEEK-DURATION-OBSERVED.md)

![从软件与依赖整理配方，再交给 Agent 重建](https://raw.githubusercontent.com/recipkg/recipkg/main/assets/recipkg-cover.png)

本仓库保存软件重建配方，不是上游软件的源码或官方发行版。上游版本、固定提交、来源证据和许可证边界见 [RECIPE.md](RECIPE.md) 与 [SOURCES.md](SOURCES.md)。配方仍在提取中；已经观察的行为、未验证内容及重建状态见 [COVERAGE.md](COVERAGE.md)。

## 使用

打包会收集配方及其固定的递归依赖，`build` 输出交给 Agent 的完整提示词；它不会调用模型或执行生成的软件。实际重建后应运行 [ACCEPTANCE.md](ACCEPTANCE.md) 中的验收。归档检查通过不等于软件已重建。

```sh
recipe_dir="$(mktemp -d)"
recipkg pack . -o "$recipe_dir/ffmpeg.recipe.tar.gz"
recipkg check "$recipe_dir/ffmpeg.recipe.tar.gz"
recipkg build "$recipe_dir/ffmpeg.recipe.tar.gz" --lang zh-CN --output "$recipe_dir/build.md"
printf '提示词：%s\n' "$recipe_dir/build.md"
```
