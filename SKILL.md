---
name: channels-live-analysis
description: Use when 分析视频号直播复盘、需要登录视频号助手(channels.weixin.qq.com)抓数据/下载直播录屏/做话术节奏分析，或用户提到"直播复盘""视频号直播分析""讲解时间轴""憋单分析"。关键词：视频号助手、数据大屏、复盘回看、直播录屏下载、m3u8、wujie、话术分析、GPM。
---

# 视频号直播复盘分析全流程

> 来源：2026-09-14 蔡姐时尚包包 9-13 场次实战（全流程跑通并交付对外 PDF）。
> 参考实现脚本：`D:/Claude/项目/11 业务合作/公司沟通/20260914-杭州函洋文化传媒-蔡佳乐/直播分析/_构建脚本/`（channels_scrape.py / channels_jiangjie.py / asr_transcribe.py / gen_report_html.py，成熟后可抽提到 `D:/Claude/配置/.claude/tools/`）

## 一、前置条件

| 依赖 | 位置/说明 |
| --- | --- |
| 运营者权限 | 用户微信已被加为目标视频号运营者（手机端接受，14:26 截图那种） |
| 登录态 profile | Playwright 持久化 Edge profile：`D:\Claude\配置\.claude\tools\browser-profiles\edge-channels-v2`，扫码一次长期有效 |
| SenseVoice 模型 | `D:\Claude\配置\.claude\tools\wechat_extraction\models\sensevoice`（sherpa_onnx，int8） |
| ffmpeg | 已装（8.1），转码用 `h264_nvenc`（本机有 N 卡） |
| PDF 链路 | HTML → `python D:/Claude/配置/.claude/tools/md_to_pdf.py in.html out.pdf` → fitz 渲染 PNG 预览检查 |

## 二、标准流程（六步）

```
登录/切号 → 后台抓数(列表+详情CSV+复盘快照) → 录屏m3u8拉流 →
SenseVoice 分块转写 → 时间轴×成交×语速交叉分析 → 对外 PDF 报告
```

1. **登录与切号**：`ctx.clear_cookies()` 强制出二维码 → 用户扫码 → 自动点目标账号名。一微信多号时默认进用户自己的号（如"用户0284223142"），必须确认页面含目标账号名再动手。
2. **抓数**：数据中心→直播数据（列表/场次）；详情页 `statistic/live?mode=detail&objetctId=xxx` 点「下载数据」得商品明细 CSV；复盘页 `dashboardV4/review?objetctId=xxx` 抓讲解时间轴与全文快照。
3. **录屏**：复盘页触发 video 播放，request 监听抓 `timeshift.m3u8`（VOD 类型、6 秒分片、带 sign），ffmpeg 一次拉流双输出：`-c copy 存档.mp4 -vn -ac 1 -ar 16000 -f s16le audio.pcm`。
4. **转写**：PCM 按 30 秒分块过 SenseVoice（8 线程约 2.5 倍实时，3 小时音轨 ≈ 15 分钟），产出带时间戳逐字稿 + 分钟级语速。
5. **交叉分析**：讲解时段×时段成交×语速×话术要素，按第三节框架执行。
6. **报告**：对外交付走 HTML→PDF（文档排版路线），内部分析存项目 md。

## 三、分析框架（方法论核心）

| 模块 | 关键指标/方法 | 判断要点 |
| --- | --- | --- |
| 漏斗三层归因 | 看播→停留→点袋→成交；GPM、UV 价值、点击价值、点击成交率 | 先定位出血层再下结论；优化顺序：货架→停留→逼单 |
| 货品分层 | 利润款/引流款/零单款的曝光占比 vs 成交占比 | 零单款曝光占比 >15% 即结构问题；人群错配三因（信任不迁移/场景冲突/无价格锚点） |
| 讲解节奏 | 时段成交 20 分钟桶（找双峰/断电段）；每分钟效率=时段成交÷讲解时长 | 4-7 分钟是黄金区间；<3 分钟浅讲基本零单；同款返场效率递减，单场 ≤2 次且换利益点 |
| 话术密度 | 六要素频次：互动指令/关注引导/稀缺词/价格锚点/赠品钩子/售后承诺 | 售后承诺常被忽视（"敢不敢"比"值不值"对银发客群更卡） |
| 闭环对比 | 高效段 vs 零单段逐字稿对比 | 四要素闭环（稀缺→分层互动→锚点→售后托底）走完整才放单；赠品不在闭环前插队 |
| 语速匹配 | 分钟语速曲线 vs 人设播参考带 200-300 字/分 | 激情播 300+ 适合互动段；高价款讲解降到 240-260；匹配客群年龄 |
| 风险词 | 不要钱/免费送/最/第一/假一赔/仅限 等计数+首现时间 | 赠品诱导高频=视频号风控敞口，给合规替换表述 |

**对外报告铁律**：统计数字必须有"分析段"（为什么/怎么办），不许只堆指标；口径存疑时注明（如"讲解成交=该时段全场成交，非该品成交"）。

## 四、踩坑清单（全部实战固化，按踩坑顺序）

### 4.1 视频号助手前端结构（wujie 微前端）

- **页面有两个 `<body>`**：子应用以 wujie 降级模式渲染进主 DOM。`locator('body').inner_text()` 直接 strict mode violation。
  → 用 `'\n'.join(page.locator('body').all_inner_texts())` 合并取全文。
- **子应用内容在 shadow DOM 里**：`page.evaluate` 的 `querySelectorAll` 找不到讲解列表条目。
  → 放弃 DOM 定位，滚动时用全文正则解析（穿透 shadow DOM 的 inner_text）。
- **讲解列表是虚拟滚动**：只渲染可见行。→ 边滚动边收集 union（每步 `scrollTop += 400`，到底部稳定 4 轮停止），正则：
  `(\d{2}:\d{2})-(\d{2}:\d{2})\n第(\d+)次讲解\n([^\n]+)\n商品店铺 ([^\n]+)\n商品价格 (¥[\d.]+)\n讲解成交 (¥[\d.]+)`

### 4.2 登录与页面导航

- **登录页文案自带"复盘/数据大屏"字样**，用关键词判登录态会误判通过。→ 真登录判定：页面同时含「目标账号名 + 内容管理」，或内容页特征词且不含「登录视频号助手」。
- **复盘页直接贴 URL 可能报"本场直播已结束，暂无法查看数据大屏"**：原因通常是①账号上下文未初始化（需先进 `channels.weixin.qq.com/platform` 主页或直播列表页再跳）②登错账号。
- **扫码后超时窗口要留足**（≥10 分钟），且窗口可能被其他窗口挡住，超时前提醒用户。

### 4.3 Playwright 持久化 profile 锁死（exitCode=21）

- 强杀 python 进程后 Edge profile 残留 SingletonLock → 下次启动闪退 exitCode=21。
- 处置：删 profile 根目录 `SingletonLock/SingletonSocket/SingletonCookie/lockfile/DevToolsActivePort`；仍不行则**克隆 profile 到新目录**（剔除锁与缓存目录，`Local State` 必须保留否则 cookie 解密失败）。
- Cookies 文件 WinError 32 被占用但 handle64 查不到 → 多为刚杀完的进程在收尾，等 1 分钟重试。
- **禁止 `taskkill //FI "WINDOWTITLE eq *视频号*"`**：会误杀用户日常浏览器的同名标签页。要按命令行特征（profile 路径）精准杀。

### 4.4 ffmpeg 拉流

- **`-headers` 多行字符串在 Git Bash 会被吞掉一层** → 报 `Error number -138`。→ 改用独立参数 `-user_agent "..." -referer "https://channels.weixin.qq.com/"`。
- 视频号回放流是 **HEVC 1080×1920，约 9.6 Mbps**：体积 ≈ 4.3 GB/小时。3 小时场 ≈ 13 GB，下载前告知用户体积；下完用 `h264_nvenc -b:v 1800k -vf scale=720:-2` 出 720p 压缩版（约 1.5-2 GB），原画删不删先问。
- m3u8 是 timeshift VOD，sign 参数有过期风险，抓到地址后**尽快下载**。

### 4.5 ASR 与分析口径

- SenseVoice 整段长音频要分块（30 秒/块），逐块 decode 并记录时间区间；分钟语速=块字数归并。
- ASR 时间戳是分块级精度（30 秒），做"某句对应成交尖峰"的秒级对齐时别过度解读。
- **「讲解成交」口径 = 该讲解时段内全场成交**，不是该商品成交；引用时必须按"时段成交"表述。

### 4.6 报告排版（PDF）

- 长表格跨页：`<thead>` 加 `display: table-header-group` 让表头每页重复；`tr { break-inside: avoid }`。
- 页脚孤儿页：footer 并进最后一个 section 或加 `break-inside: avoid`。
- 每页必查：`fitz` 渲染 55dpi PNG 逐页读图验收，单次最多读 2 张（读图规则）。
- 对外文档走「文档排版路线」design-taste-frontend 适配版；禁 em-dash（—）；单一强调色。

## 五、复用与迭代

- 脚本当前在项目 `_构建脚本/`，第二次复用跑通后抽提到 `D:/Claude/配置/.claude/tools/channels_live/`，SKILL.md 同步更新路径。
- 新增坑位/分析维度时直接补进本文件对应章节，保持「按踩坑顺序」的叙事结构。
