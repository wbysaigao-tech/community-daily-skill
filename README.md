# community-daily-skill

「集团总公司」（新东方之家微信小程序）社区每日活跃自动化 Skill。

每天自动完成：1 帖（4 个生活话题按天轮换）+ 5 条评论 + 5 次点赞，纯 HTTP 接口直连网关，无 UI 自动化。

## 内容

- `SKILL.md` — 完整 Skill 文档：接口契约、抓包流程、水贴审核规避、结果判定与异常处理、踩坑全集
- 配套代码（xdf_api.py / extract_api.py / mitm.py / run_daily_api.sh / capture_api.sh）在本地私有项目中，如需开源请自行整理后放入本仓库

## 快速上手

1. 阅读并按 `SKILL.md` 第 2 节搭好目录
2. 首次配置：按第 7 节抓包生成 `api_config.json`（token 必须由本人抓取）
3. 日常执行：`./run_daily_api.sh`，按第 5 节判定结果

## ⚠️ 注意

- 文档中不含任何真实 token；`api_config.json` 含个人鉴权信息，切勿提交到任何仓库
- 仅用于个人账号的日常活跃任务，请遵守所在社区的内容规范
