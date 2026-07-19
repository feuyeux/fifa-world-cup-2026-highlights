---
name: add-new-highlight
description: Pull new FIFA World Cup 2026 highlight videos from the official YouTube playlist into D:\2026-worldcup. MANDATORY dedup gate: ALWAYS `--flat-playlist` first, build (existing VIDEO_ID) ∪ (new playlist VIDEO_ID) set diff, and download ONLY the diff via single-video `yt-dlp <url>` calls — NEVER `--yes-playlist` (the playlist re-uploads already-archived matches with new IDs, e.g. M67-M72 appeared as fresh playlist entries after M73-M79 were first released). Direct download to final filename `M{NN} {Home}-{Away} {H}-{A} [{ID}].mp4` (no intermediate "NN - Highlights" state). Append row to README.md. Triggered by: 拉新视频 / 补全集锦 / 同步播放列表 / refresh highlights.
---

# 拉取 FIFA World Cup 2026 比赛集锦新视频

## 触发条件

用户说 **playlist 范围 vs disk 范围**：FIFA 官方 playlist 公开 100 条全部已收录（M01-M72 + M73-M102）。M36 / M80 是 playlist **外**的 FIFA 频道公开上传，需要绕过 playlist 用单条 `--dump-single-json` 验证 description 归属后走单条下载。"拉新视频"/"补全集锦"/"同步播放列表"/"刷新一下集锦"等。

## 关键约定（必须遵守）

- **⚠️ 第 0 步：先 `--flat-playlist` 比对 VIDEO_ID 再决定下哪些**。FIFA playlist 会**回填已收录的旧比赛换新 ID**（实测：拉 M73-M79 时，playlist #8/9/10 是 M71/M72/M69 的新 ID 重传，#11/12/13 是 M70/M68/M67 的新 ID 重传 — 共 6 条"假新"）。如果直接 `--yes-playlist`，会把已收录的 6 场当成新比赛再下一遍，浪费带宽 + 落一堆重复文件。**正确流程**：先 `yt-dlp --flat-playlist --print "%(id)s\t%(title)s"` 拿全 playlist → 与 `existing VIDEO_ID` 做集合差集 → **只对差集里那 N 个 video_id 各跑一次单条 `yt-dlp <url>`**。`--no-overwrites` 防不住"新 ID 重传"，必须靠 ID 集合差集。
- **去重判据 = YouTube 视频 ID**，写在文件名 `[\w_-]{11}` 段里。**不**用"主队+比分"判重——多场可能同主队或同比分，ID 才是唯一键。
- **下载即终态命名**：用 yt-dlp 的 `-o` 直接落 `M{NN} {主队}-{客队} {比分} [{VIDEO_ID}].mp4`，**不**先下到 `NN - Highlights ｜ ...` 中间态再 os.rename。中途断电/中断也不会留 .part 残骸。
- **默认用 `-S "res:720"`** 而非默认 1080p：1080p AV1 + Opus 单条 40-60MB、且并发时单进程速度被压到 50-75 KiB/s；720p 单条 17-35MB、5 并发单进程 ~600 KiB/s、120s 内下完，画质对比赛集锦完全够用。
- **主队在前、比分按主-客方向**（不是胜-负方向）。`M18 Iraq-Norway 1-4` 这种主队输了的，比分仍是 `1-4`。视频标题里 FIFA 是"胜者在前、胜-负方向"，重命名时要从赛程表主客顺序反过来。
- **playlist_index ≠ 赛程 M{NN}**。实测 playlist 第 1 条 = M24 (Uzbekistan-Colombia)，第 24 条 = M01 (Mexico-South Africa)——FIFA 把最近比赛排最前。**必须用视频标题里的两支队伍名查赛程表来定位 M{NN}**，不能假设 playlist_index = M 编号。
- 编号 01–72 = 赛程表前 72 场小组赛。赛程表：`D:\coding\fifa-world-cup-2026\schedule\match schedule.md`，段 `## 全部 72 场 小组赛（按日期排序）`（**4 列：日期/主队/客队/小组，无 `#` 列** — M{NN} 用行号 1-based 推，第 1 行 = M01，第 N 行 = M{N}）。**注意**：文件后面还有 `## A 第一组` 起的"分组详情"段（5 列但第 5 列是"球场"不是"小组"）——解析时**只**读 `## 全部 72 场` 段，限定到下一个 `## ` 标题。
- 播放列表里私享/已删的条目 `--flat-playlist` 拿到的标题是 `NA`，无法定位到 M{NN}，跳过并报告。
- 同一场被重新上传时（视频 ID 变）保留旧文件 + 新增一行，README 靠 `[VIDEO_ID]` 区分。

## 工作流

### 1. 列出已下载的 [VIDEO_ID] 集合

```python
# 走 Windows 宿主：execute_code + subprocess.run([...], shell=False)
# 注意：patch / read_file 工具对 D: 路径间歇性 File not found，读写用 open()
import os, re
DOWNLOAD_DIR = r'D:\2026-worldcup'
existing = {}
for f in os.listdir(DOWNLOAD_DIR):
    m = re.search(r'\[([\w-]{11})\]\.mp4$', f)
    if m: existing[m.group(1)] = f
```

### 2. 拿 playlist 元数据（不下载，秒回）

```python
import subprocess
r = subprocess.run([
    r'D:\garden\anaconda3\Scripts\yt-dlp.exe',
    '--no-update', '--flat-playlist',
    '--print', '%(id)s\t%(playlist_index)d\t%(title)s',
    '--no-warnings',
    'https://www.youtube.com/watch?v=gy8h5JJ2b_E&list=PLBRLtDhTHh5o',
], capture_output=True, text=True, shell=False, timeout=60)
entries = []
for ln in r.stdout.splitlines():
    if '\t' in ln:
        vid, idx, title = ln.split('\t', 2)
        entries.append((int(idx), vid, title))
```

私享/已删的条目 title 字段会是字面 `NA`。

### 3. 解析每条 title → 定位 M{NN}

英文队名 → 中文队名映射（已收录的 23 个文件能反推出大部分，缺的在赛程表里捞）：

```python
en2cn = {
    'Mexico': '墨西哥', 'South Africa': '南非',
    'Korea Republic': '韩国', 'Korea DPR': '朝鲜', 'Czechia': '捷克',
    'Canada': '加拿大', 'Bosnia and Herzegovina': '波黑',
    'USA': '美国', 'Paraguay': '巴拉圭',
    'Haiti': '海地', 'Scotland': '苏格兰',
    'Australia': '澳大利亚', 'Türkiye': '土耳其', 'Turkey': '土耳其',
    'Brazil': '巴西', 'Morocco': '摩洛哥',
    'Qatar': '卡塔尔', 'Switzerland': '瑞士',
    "Côte d'Ivoire": '科特迪瓦', 'Ivory Coast': '科特迪瓦',
    'Ecuador': '厄瓜多尔',
    'Germany': '德国', 'Curacao': '库拉索', 'Curaçao': '库拉索',
    'Netherlands': '荷兰', 'Japan': '日本',
    'Sweden': '瑞典', 'Tunisia': '突尼斯',
    'Saudi Arabia': '沙特阿拉伯', 'Uruguay': '乌拉圭',
    'Spain': '西班牙', 'Cabo Verde': '佛得角', 'Cape Verde': '佛得角',
    'IR Iran': '伊朗', 'Iran': '伊朗', 'New Zealand': '新西兰',
    'Belgium': '比利时', 'Egypt': '埃及',
    'France': '法国', 'Senegal': '塞内加尔',
    'Iraq': '伊拉克', 'Norway': '挪威',
    'Argentina': '阿根廷', 'Algeria': '阿尔及利亚',
    'Austria': '奥地利', 'Jordan': '约旦',
    'Ghana': '加纳', 'Panama': '巴拿马',
    'England': '英格兰', 'Croatia': '克罗地亚',
    'Portugal': '葡萄牙', 'Congo DR': '刚果民主共和国', 'DR Congo': '刚果民主共和国',
    'Uzbekistan': '乌兹别克斯坦', 'Colombia': '哥伦比亚',
}
```

标题正则（覆盖 "Highlights | Home N-A Away | ..." 和 "Kylian Mbappe Stars! |  France 3-1 Senegal | Match Highlights" 两种格式）：

```python
import re
title_re = re.compile(r'([A-Za-z][A-Za-z\s\.\-\']+?)\s+(\d+)-(\d+)\s+([A-Za-z][A-Za-z\s\.\-\']+?)\s*(?:\||$)')
# 例外格式："Lionel Messi Hat-trick Beats Algeria" 这种没显式比分的得用队名匹配
```

对每条 title：
1. 用正则抽 (home_en, h_score, a_score, away_en)
2. 查 en2cn → (home_cn, away_cn)
3. 在赛程表 `## 全部 72 场 小组赛` 段里找 `主=home_cn 且 客=away_cn` **或** `主=away_cn 且 客=home_cn`，定位 M{NN}
4. 记下 `is_home_first`（赛程表主队是否在 title 的 home_en 位置）
5. 查 `existing` —— 命中就跳过

### 4. 赛程表解析（只读 `## 全部 72 场 小组赛` 段）

```python
import subprocess
schedule = subprocess.run(['cmd', '/c', 'type', r'D:\coding\fifa-world-cup-2026\schedule\match schedule.md'],
                          capture_output=True, text=True, shell=False).stdout
lines = schedule.splitlines()
start = end = None
for i, ln in enumerate(lines):
    if ln.strip().startswith('## 全部 72 场'):
        start = i
    elif start is not None and ln.startswith('## ') and i > start:
        end = i; break
rows = []
# 只读 `## 全部 72 场 小组赛` 段（4 列：日期/主/客/组，**无 #**）
for ln in lines[start:end]:
    m = re.match(r'^\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|\s*([A-L])\s*\|\s*$', ln)
    if m:
        rows.append((m.group(2).strip(), m.group(3).strip(), m.group(4).strip(), m.group(1).strip()))
# rows[idx] = (主队, 客队, 小组, BJT) — idx 0-based 对应 M{idx+1}
```

### 5. 用 git-bash 跑 yt-dlp，下到最终文件名

**为什么用 git-bash**：因为 yt-dlp 的 JS challenge solver 路径在 README 工作流里就指定了 git-bash 行为（README 第 60-108 行）。**关键陷阱**：`terminal` 工具跑在 Linux 沙箱里看不到 D:，必须用 `subprocess.run([r'D:\Program Files\Git\usr\bin\bash.exe', '-lc', '...'])` 走 Windows 宿主。

```python
import subprocess
gb = r'D:\Program Files\Git\usr\bin\bash.exe'

# 例：M24 乌兹别克斯坦-哥伦比亚 1-3 [cjsFUxVHAX0]
# 注意：-o 路径用 git-bash POSIX 风格 /d/2026-worldcup/...，不能写成 D:\...
# 然后 bash -c "cd /d/2026-worldcup && yt-dlp ..."
fname = f'M24 Uzbekistan-Colombia 1-3 [cjsFUxVHAX0].mp4'
gb_output = f'/d/2026-worldcup/{fname}'

bash_cmd = f'''cd /d/2026-worldcup && yt-dlp --no-update \\
  --js-runtimes "node:/d/zoo/nodejs/node.exe" \\
  --cookies "/d/download/www.youtube.com_cookies.txt" \\
  -f "bv*+ba/b" --merge-output-format mp4 \\
  -o '{gb_output}' \\
  "https://www.youtube.com/watch?v=cjsFUxVHAX0"'''

r = subprocess.run([gb, '-lc', bash_cmd], capture_output=True, text=True, shell=False, timeout=300)
print("rc:", r.returncode)
print(r.stdout[-500:])  # tail
```

逐个下载未命中 ID 的视频（**不要用 `--yes-playlist`**——一次下一个，便于核对）。

### 6. 追加 README 行

用 `with open(fp, 'r', encoding='utf-8')` 读，定位最后一行 `| M{NN} |` 后插入新行；同步改 `## 当前已收录（xx / xx 公开）` 标题和 `**未收录**` 段描述。

```python
fp = r'D:\2026-worldcup\README.md'
with open(fp, 'r', encoding='utf-8') as f:
    content = f.read()
lines = content.splitlines()
# 找最后一行 | M{NN} |
last_idx = -1
for i, ln in enumerate(lines):
    if re.match(r'^\| M\d{2} \|', ln):
        last_idx = i
new_row = f'| M{NN} | {home_cn} - {away_cn} | {h_score}-{a_score} | {group} | {bjt} | `M{NN} {home_en}-{away_en} {h_score}-{a_score} [{vid}].mp4` |'
lines.insert(last_idx + 1, new_row)
# 写回
with open(fp, 'w', encoding='utf-8') as f:
    f.write('\n'.join(lines) + '\n')
```

### 7. 报告

输出：
- 本次新落地的 VIDEO_ID 列表
- 每个新文件：编号 / 主客 / 比分 / 视频 ID / 大小 / 时长（`ffprobe`）
- 已下载总数 (xx/72)
- 仍未收录：私享/已删条数 + 赛程 M25+ 待 FIFA 上传

## 验证

- `ffprobe` 验证新文件时长 60–240s、码流正常
- `os.listdir(DOWNLOAD_DIR)` 数 .mp4 数 == README 表 `| M{NN} |` 行数
- 无 `.part` / `.ytdl` 残骸
- 无误建子目录（如 `D:\d\`、`D:\d\2026-worldcup\` 等）——这是 git-bash 路径错配的典型症状

## 已知坑

- **git-bash MSYS 路径转换会吞掉 `/d/...` 前导斜杠**：当 `-o "/d/2026-worldcup/${fname}"` 传给 yt-dlp 时，MSYS 自动把 `/d/foo` 改成 `\doo`（相对路径），文件会落到 `D:\D6-worldcup\`（一个鬼目录）。**修复**：脚本顶部加 `export MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*'`，并且 `-o` 用相对文件名（`cd /d/2026-worldcup` 后直接 `-o "${fname}"`），不写绝对路径。验证：日志里看到 `Destination: \d\...` 立即停手。
- **`terminal` 工具跑在 Linux 沙箱看不到 D:**。所有 ls/cd/yt-dlp 不能用 `cd /d/...`，必须用 `subprocess.run([r'D:\Program Files\Git\usr\bin\bash.exe', '-lc', '...'])` 走 Windows 宿主 + git-bash。文件读写/glob/listdir 用 `execute_code` 走 Windows 宿主。
- **`patch` / `read_file` 工具对 D: 路径间歇性 File not found**（memory 提过，路径含非 ASCII 段时易触发）。读写 README/AGENTS.md 用 `with open(path, 'r/w', encoding='utf-8')`，不要用 `read_file` / `patch`。
- **git-bash `-o` 路径必须是 `/d/2026-worldcup/...` POSIX 风格**，不能写 `D:\...` Windows 风格。如果只给单文件名（无路径）会落到 git-bash 的 `pwd` 目录（实测是 `/d/2026-worldcup`）——OK 的；但**绝对不要**写 `D:\...` 反斜杠形式，yt-dlp 会当成相对路径建出 `D:\D\...` 这种鬼目录。
- **下载前 `cd /d/2026-worldcup`**：保证中间文件（`.part`）和最终文件都落在正确目录。如果 yt-dlp 报 "Destination: \d\..." 那种反斜杠开头的路径，说明 cwd 不对或 `-o` 写错。
- **JS runtime**: `--js-runtimes "node:/d/zoo/nodejs/node.exe"` 在 git-bash 下要给 MSYS 风格路径（用 `/d/...`）；在 cmd 下要给 Windows 风格（用 `D:\...`）。**cookies 失效但视频仍能下**：M24 那次 cookies 已失效，yt-dlp 给了 WARNING，但下载 rc=0 成功。说明对公开集锦 YouTube 不强校验。
- **playlist_index ≠ 赛程 M{NN}**——FIFA 把最近比赛排最前。M24 (Uzbekistan-Colombia) 在 playlist index 1，M01 (Mexico-South Africa) 在 playlist index 24。**必须用视频标题里的两支队伍查赛程表来定位 M{NN}**。
- **赛程表有两段容易混**：`## 全部 72 场 小组赛`（5 列：# 日期 主 客 小组）vs `## A 第一组` 起的分组详情段（5 列：# 日期 主 客 **球场**）。**只**读"全部 72 场"段，第 5 列才是小组。
- **yt-dlp 标题里"胜者在前、比分胜-负"**——视频写 `Colombia 3-1 Uzbekistan`、赛程表写 `乌兹别克斯坦 vs 哥伦比亚`、比分按主-客方向 = `1-3`。**重命名时要从赛程表主客顺序反过来**，主队输了的场次比分方向也要翻。
- **私享/已删条目** `--flat-playlist` 拿到的 title 是字面 `NA`，无法定位 M{NN}，跳过并报告。
- **同一场被重新上传**（视频 ID 变）保留旧文件 + 新增一行，README 靠 `[VIDEO_ID]` 区分；不要去重删旧。
- **淘汰赛阶段（1/16、1/8、半决赛、决赛）继续 M{NN} 编号续编**（72 场小组赛后从 M73 起），文件名前缀永不嵌 stage 标签（R16/QF/SF/F 都不写进文件名），stage 区分靠 README 表格的"阶段"列。点球胜负比分格式用 `(胜者总)主-客(胜者总)` — 例：M75 `Germany-Paraguay (3)1-1(4)`、M76 `Netherlands-Morocco (2)1-1(3)`，与 FIFA YouTube 标题里 FIFA 写的格式一致。
- **`--cookies` 间歇性报 `FileNotFoundError` 但 cookies 文件实际存在**（实测 M53 那次：路径 `D:\download\www.youtube.com_cookies.txt` 1609 bytes 在的，yt-dlp `cookies.py:1305` 还是抛 No such file）。**症状**：stderr 出现 `FileNotFoundError: [Errno 2] No such file or directory: '/d/download/www.youtube.com_cookies.txt'`，视频流本身能下到 47% 左右时进程 rc=1 退出、留下 `.part` 残骸。**根因疑似**：yt-dlp 早期 cookies 解析阶段路径转换异常，与 git-bash 的 MSYS 路径转发有关。**修复**：下次批量下载时直接 `--no-cookies`（公开集锦不要求登录，YouTube 对未登录请求只发低码率但仍可下载）。如果必须保留 cookies，单独文件用 `with open(p,'rb').read()` 校验实际可读后重试；或换 Windows 原生 cmd 跑 yt-dlp 而不是 git-bash。
- **yt-dlp 退出阶段 `save_cookies()` 也可能抛 `FileNotFoundError` → rc=1**，但此时 download + Merger 都已完成，**视频文件正常落地**。**判定标准**：`os.path.exists(目标.mp4)` 且大小 > 5 MB 即视为成功，不要被 rc=1 误导去重下。根因同上（MSYS 路径转换在 `__exit__` 阶段再次异常）。**配套**：`os.path.exists` 大小 > 5 MB 视为成功时**仍然要 `taskkill //F //IM yt-dlp.exe`** 清理可能残留的 .part（虽然本次测试几乎不发生）。
- **yt-dlp 并发数 5 最优**：单进程 ~600 KiB/s；10 并发被压到 50-75 KiB/s 反而更慢，且 5 分钟 timeout 容易把 .part 残骸留一半。脚本里 5 个 Popen/terminal background 同时跑，120s 内全部下完 17-25 MB 的 720p 集锦。如果想做"一次性补很多"，分批 5 个是平衡点。


## 已知补传案例（2026-07-19 M36 / M80）

FIFA 官方 playlist `PLBRLtDhTHh5o` 已稳定 100 条、0 NA，但 FIFA 频道**另**有 2 条通过 playlist **外**的频道上传收录：
- M36 突尼斯-日本 0-4（2026/06/22 12:00 BJT），vid `V7z6uzD91Ao`（2026-07-19 复测时 playlist 仅 100 条、YT 头"104 个视频"为旧值）
- M80 英格兰-刚果民主共和国 2-1（2026/07/02 00:00 BJT），vid `T6MYUQgpCv8`

**识别 / 补传流程**：

```bash
# 1. --dump-single-json 拿 description 第二行（{H} v {A} | FIFA WC 2026 | {date} | Post-game 模板）
yt-dlp --no-update --dump-single-json --no-warnings "https://www.youtube.com/watch?v=<id>"
# 检查: channel='FIFA' + availability='public' + description 日期与赛程 BJT-1 一致 + upload_date 在比赛次日

# 2. 直接单条 yt-dlp 下到终态文件名（git-bash 三件套 + --no-cookies）
cd /d/2026-worldcup && yt-dlp --no-update --no-cookies \
  --js-runtimes "node:/c/Program Files/nodejs/node.exe" \
  -f "bv*+ba/b" --merge-output-format mp4 -S "res:720" \
  -o "M{NN} {Home}-{Away} {H-A} [{VID}].mp4" \
  "https://www.youtube.com/watch?v=<id>"

# 3. 追加 README 表格行 + 更新"未收录"段（72/72、16/16、0 NA 注释）
```

**注意事项**：
- 不要被 YouTube playlist 网页头"104 个视频"误导去 `--flat-playlist` 找第 101-104 条；实测仍只返 100 条。差额的 4 条里 2 条对应 M36/M80（已补传），剩 2 条可能是 system/test 视频。
- `--no-cookies` 走公开访问对 FIFA 频道公开集锦 100% 够用；不要带 `--cookies` 走 MSYS 路径转换，否则 `__exit__` 误报 + `D:\d\2026-worldcup\` 鬼目录。
- 文件名按赛程表主客顺序（home-away），比分按主客方向（home-away 视角），不翻转。
