# AGENTS.md — FIFA World Cup 2026 Match Highlights

## 概述

这是 FIFA World Cup 2026 比赛集锦视频的存档，不是软件项目。没有代码、没有构建系统。

## 文件命名规范

```
M{赛事编号} {主队}-{客队} {主队比分}-{客队比分} [{YouTube 视频 ID}].mp4
```

- 赛事编号与赛程表 `D:\coding\fifa-world-cup-2026\schedule\match schedule.md` 一致
- 主队在前，比分按主-客方向
- 文件按编号升序排列（等同于比赛时间顺序）

## 补全新视频工作流

1. **下载**：在 `D:\2026-worldcup\` 运行 yt-dlp（保留已下载的，只下新的）
2. **查编号**：对照赛程表找到赛事编号、主队、客队、最终比分
3. **重命名**：按命名规范重命名文件
4. **更新 README**：在"当前已收录"表末尾追加新行

详细命令见 README.md 第 60-108 行。

## 环境依赖

- **yt-dlp** (≥ 2026.6.9) + `[default]` extra：`pip install -U "yt-dlp[default]"`
- **Node.js**：`D:\zoo\nodejs\node.exe`（JS challenge solver）
- **ffmpeg**：音视频合并
- **YouTube cookies**：`D:\download\www.youtube.com_cookies.txt`（通过 "Get cookies.txt LOCALLY" 扩展导出）

## 已知坑

- Chrome 127+ App-Bound Encryption 导致 `--cookies-from-browser chrome` 失效，必须手动导 cookies.txt
- 视频 24-31 是 FIFA 私享视频，无法通过本工作流补齐
- 下载默认 1080p AV1 + Opus，单集约 30-60 MB；可加 `-S "res:720"` 降为 720p
- 同一场集锦被重新上传时（视频 ID 变），保留旧文件并新增行，靠 `[VIDEO_ID]` 区分
