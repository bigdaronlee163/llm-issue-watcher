# llm-issue-watcher

监控一批 LLM 相关开源仓库的新 Issue，并通过 [ntfy.sh](https://ntfy.sh) 推送到手机。整套系统完全构建在免费服务之上：

- **GitHub Actions** — 运行检查脚本
- **ntfy.sh** — 免费、免注册的手机推送
- **cron-job.org** — 外部定时触发（比 GitHub 自带的 schedule 可靠）

不需要自己的服务器、数据库或任何账号体系。

## 工作原理

```
cron-job.org（每 30 分钟）
   │  POST GitHub dispatch API
   ▼
GitHub Actions 运行 check_new_issues.py
   │  1. 读取 state/last_check.txt（上次检查时间戳）
   │  2. 逐仓库调用 Search API：created:>=上次时间
   │  3. 过滤：剔除 PR + 标签筛选（优先仓库全放行）
   │  4. 有新 Issue → POST 到 ntfy.sh
   ▼
手机 ntfy App 收到推送
   │
   └── 新时间戳写回 state/last_check.txt，由 bot 自动 commit 回仓库
```

每次运行只查询"上次检查时间之后新开"的 Issue（增量检测），因此绝不重复推送。

## 监控范围与规则

- 默认监控 27 个仓库（完整列表见 [check_new_issues.py](check_new_issues.py) 的 `REPOS`），覆盖：LLM 推理/服务、训练/微调、模型量化压缩、Agent 框架、数据管线、向量检索
- 默认只推送带 `good first issue` 或 `help wanted` 标签的 Issue（由 `ONLY_CONTRIBUTOR_LABELS` 控制）
- **优先仓库例外**：`ALL_ISSUES_REPOS` 中列出的仓库推送**所有**新 Issue（当前为 `vllm-project/llm-compressor`）
- 只监控"新开"的 Issue，不监控评论或状态变化

## 自己部署一份

### 1. 创建仓库

新建仓库，放入 `check_new_issues.py` 与 `.github/workflows/notify-new-issues.yml`，按需修改 `REPOS` 列表。

### 2. 手机端订阅 ntfy topic

安装 ntfy App（iOS: App Store；Android: 推荐 F-Droid 版，推送更即时），订阅一个带随机后缀的 topic（topic 名相当于"密码"，不要太简单）。

### 3. 配置仓库 Secret

仓库 Settings → Secrets and variables → Actions → New repository secret：

| Name | Value |
|---|---|
| `NTFY_TOPIC` | 你订阅的 topic（与 App 中**一字不差**，区分大小写） |

### 4. 配置定时触发（cron-job.org）

先生成一个 Fine-grained PAT（只授权本仓库，权限选 **Actions: Read and write**），然后在 cron-job.org 新建任务：

| 配置项 | 值 |
|---|---|
| URL | `https://api.github.com/repos/<owner>/<repo>/actions/workflows/notify-new-issues.yml/dispatches` |
| Method | POST |
| Headers | `Authorization: Bearer <PAT>`、`Accept: application/vnd.github+json`、`Content-Type: application/json` |
| Body | `{"ref":"main"}` |
| Schedule | 每 30 分钟（`*/30 * * * *`） |

### 5. 验证

在 Actions 页面手动 Run workflow，日志出现 `Checked N repos since ...: found N new issue(s).` 即为正常。

## 常见问题

- **收不到推送？** 按顺序排查：App 是否已订阅 → topic 与 Secret 是否一字不差（注意大小写）→ 系统通知权限
- **提交历史出现 `Update last check timestamp [skip ci]`？** 正常现象，这是 bot 在每次运行后回写时间戳
- **个别仓库偶发漏查？** GitHub Search API 限流 30 次/分钟，监控的仓库总数建议不超过 30 个

## 致谢 Acknowledgements

本项目的核心设计与实现参考自 **[rohan9446/llm-issue-watcher](https://github.com/rohan9446/llm-issue-watcher)**，衷心感谢原作者的开源分享。

本仓库在其基础上做了如下调整：

- 修复 workflow 中脚本调用路径（原 `scripts/check_new_issues.py` 与脚本实际位置不符）
- 按个人关注方向（LLM 模型量化）调整监控仓库列表
- 新增 `ALL_ISSUES_REPOS` 优先仓库机制：指定仓库推送所有新 Issue
- 增补中文说明文档
