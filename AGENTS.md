# AGENTS.md — FIFA World Cup 2026 Match Highlights

## 概述

这是 FIFA World Cup 2026 比赛集锦视频的存档，不是软件项目。没有代码、没有构建系统。

## 文件命名规范

```
M{赛事编号} {主队}-{客队} {主队比分}-{客队比分} [{YouTube 视频 ID}].mp4
```

- **赛事编号 M{N} 跨阶段续编**：72 场小组赛后从 M73 起继续接 1/16、1/8、半决赛、决赛。文件名前缀**永远**只有 `M{数字}`，**不**嵌 stage 标签（如 `R16`/`QF`/`SF`/`F`）。stage 区分靠 README 表格列。
- **点球胜负比分**用 `(胜者总)主-客(胜者总)` 格式，与 FIFA YouTube 标题一致（例：`Germany-Paraguay (3)1-1(4)`）。
- 赛事编号与赛程表 `D:\coding\fifa-world-cup-2026\schedule\match schedule.md` 一致
- 主队在前，比分按主-客方向（不是胜-负方向）
- 文件按编号升序排列（等同于比赛时间顺序）

## 去重规则（防 playlist 回填陷阱）

FIFA playlist 会**回填已收录的旧比赛换新 ID**，直接 `--yes-playlist` 必然下出重复文件。**正确流程**：

1. 先 `yt-dlp --flat-playlist --print "%(id)s\t%(title)s"` 拿全 playlist 元数据
2. 与 `D:\2026-worldcup\*.mp4` 已落地文件名的 `[VIDEO_ID]` 做集合差集
3. 只对差集里的 video_id 各跑一次**单条** `yt-dlp <url>`，下到终态文件名
4. **`--no-overwrites` 防不住"新 ID 重传"**，必须靠 ID 集合差集

详细命令见 README.md 第 60-108 行 + SKILL.md（`add-new-highlight`）。

## 环境依赖

- **yt-dlp** (≥ 2026.6.9) + `[default]` extra：`pip install -U "yt-dlp[default]"`
- **Node.js**：`D:\zoo\nodejs\node.exe`（JS challenge solver）
- **ffmpeg**：音视频合并
- **YouTube cookies**：`D:\download\www.youtube.com_cookies.txt`（通过 "Get cookies.txt LOCALLY" 扩展导出）

## 已知坑

- Chrome 127+ App-Bound Encryption 导致 `--cookies-from-browser chrome` 失效，必须手动导 cookies.txt
- 公开 FIFA 集锦不强制登录（`--no-cookies` 默认开）；MSYS 路径转换坑下保留 `--cookies` 易触发 `__exit__` FileNotFound 误报。
- 视频下载默认 720p av1/opus（`S "res:720"`），单集约 15-25 MB、3-5x 快于 1080p；如需 1080p 把 `-S "res:720"` 去掉即可。
- 同一场集锦被重新上传时（视频 ID 变），保留旧文件并新增行，靠 `[VIDEO_ID]` 区分。
- **playlist 范围外补传**：FIFA 官方 playlist `PLBRLtDhTHh5o` 公开 100 条全部收录；M36 / M80 走 FIFA 频道**非 playlist** 公开上传（vid `V7z6uzD91Ao` / `T6MYUQgpCv8`），用 `--dump-single-json` 验证 description + channel 后单条 `yt-dlp` 下到终态文件名。详见 `SKILL.md`（add-new-highlight）末尾"已知补传案例（2026-07-19 M36 / M80）"。
