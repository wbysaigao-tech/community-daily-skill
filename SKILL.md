---
name: xdf-community-daily
description: 「集团总公司」（新东方之家微信小程序）社区每日活跃自动化：纯 HTTP 接口方案，每天自动发帖 + 评论 + 点赞。涵盖接口契约、抓包取 token、水贴审核规避、结果判定与异常处理。当用户提到集团总公司、新东方之家、社区每日活跃、发帖积分任务时使用。
---

# Skill：集团总公司（新东方之家小程序）社区每日活跃自动化

核心思路：**纯 HTTP 接口直连网关，不走 UI 自动化**。所有路径、参数名、鉴权均从 `api_config.json` 读取（由抓包生成），代码不硬编码契约，接口变更时只改配置。

> 本文为 2026-11-26 最终整理版，已合并 2026-11-05 重建至今的全部迭代与踩坑记录。

## 1. 任务定义

「集团总公司」（微信小程序「新东方之家」）社区，每天自动完成一次活跃动作：

- **每天发 1 帖**：话题按天轮换（4 个话题，day-of-year % 4），文案从每话题 3 套标题+正文的池子里按天选取
- **该帖 5 条评论**：从每话题 6 条自然评论池按天轮换选取
- **该帖 5 次点赞**：点赞 targetId 为评论 id，targetType=2

**内容红线（必须遵守）**：只发吃喝、徒步、生活治愈、萌宠等纯生活内容，**绝不涉及公司/工作相关话题**（金点子建议、夸同事等均属危险内容，曾因此返工）。

### 积分规则（实测）

- 评论积分每日上限 **2 分**（commentPointDayLimit=2），超出部分 action 照常执行但不加分
- 点赞积分上限 **5 分**（likePointDayLimit=5）
- 发帖积分以响应 `pointsEarned` 为准

## 2. 目录结构

```
wx_auto/
├── src/xdf_api.py          # 核心：纯 HTTP 接口客户端（日常执行入口，--daily）
├── src/extract_api.py      # 从 MITM 抓包自动提取接口/鉴权，生成 api_config.json
├── src/mitm.py             # mitmproxy addon：抓包落盘 /tmp/mitm_flows.jsonl（仅首次抓 token 用）
├── api_config.json         # 鉴权 + 接口契约（token 存这里，切勿外传）
├── api_config.template.json# 占位模板
├── run_daily_api.sh        # 日常执行脚本（自动 no_proxy 直连 + 日志落盘）
├── capture_api.sh          # 一次性抓包取证脚本
├── logs/YYYY-MM-DD.log     # 运行日志（按天追加）
└── SKILL.md                # 本文档
```

## 3. 接口契约

网关：`https://gateway.xdf.cn`

### 鉴权三件套（缺一不可，缺任一 → 401/403）

- `u2at`
- `u2st`
- `Authorization`（形如 `U2 2410035_e_xxx_SE261124134801`，`_SE` 后缀疑似含时间戳，token 有生命周期，过期属正常现象）

### 公共请求头

- `xweb_xhr: 1`
- `Referer: https://servicewechat.com/wx2ef052b12371bf1b/175/page-frame.html`
- 微信小程序 UA（MicroMessenger + miniProgram/wx2ef052b12371bf1b）

### 端点（以 api_config.json 的 endpoints 为准，当前实测值）

| key | 方法 | 路径 | 编码 |
|---|---|---|---|
| topic_list | GET | `/member-server/staff/community/topic/select-options` | - |
| posts_list | POST | `/member-server/staff/community/feed/recommend/list` | json |
| post_create | POST | `/member-server/staff/community/post/publish` | **multipart** |
| comment_create | POST | `/member-server/staff/community/comment/publish` | json* |
| like | POST | `/member-server/staff/community/like/toggle` | json |

\* comment/publish 实测 JSON 与 multipart 服务端均兼容，代码用 JSON。

公共参数（query/body 均带）：`schoolId`、`orgId`（按各人真实抓包值填写，不要照抄别人的）。

### 返回约定

`{"code":0,"msg":"success","data":{...}}`；发帖返回 `data.postId` + `machineReviewPass` + `tipMessage` + `pointsEarned`；评论返回 `data.commentId`。

## 4. 日常执行（核心命令）

```bash
cd <wx_auto 目录>
./run_daily_api.sh                 # 推荐：自动 no_proxy 直连 + 日志落盘 logs/当日.log
# 或
python3 src/xdf_api.py --daily
```

### 其他命令

```bash
python3 src/xdf_api.py --list-topics          # 查看话题列表（也用于验证 token/契约）
python3 src/xdf_api.py --list-posts           # 查看推荐帖子
python3 src/xdf_api.py --post "标题" "内容" --topic 今天吃点啥   # 手动发单帖
python3 src/xdf_api.py --config api_config.json --daily          # 指定配置
```

## 5. 结果判定标准（当前 1 帖版本）

- 退出码 **0**；
- 日志含 1 个 postId，且审核行 `过审=True`；
- 5 条评论均有 commentId；
- 5 次点赞完成。

**话题轮换顺序**（day-of-year % 4）：今天吃点啥(210009) → 进山充电(210012) → 生活小治愈(210077) → 我家毛孩子(180006)。话题 ID 失效时更新 `xdf_api.py` 的 `TOPIC_ID_MAP`（可用 `--list-topics` 核对最新 id）。

## 6. 异常处理（按退出码）

### 退出码 3 = TOKEN_EXPIRED（HTTP 401/403 或 auth 字段为空）

token 过期属正常现象，**必须由用户本人在微信小程序里重新抓包**，禁止任何自动抓包/自动登录：

1. 打开微信小程序「新东方之家」，进社区随便做一个操作（看话题列表即可）
2. 用抓包工具取发往 `gateway.xdf.cn` 请求头里的 `u2at` / `u2st` / `Authorization` 三个值
3. 更新 `api_config.json` 的 `auth` 字段（endpoints 无需重抓）；或直接运行 `./capture_api.sh`

### 退出码 2 = API_ERROR

- **Connection refused / 网络抖动**：直接重跑一次 `./run_daily_api.sh` 再判定
- **业务错误**（code≠0）：看返回体 msg 定位。已知案例：
  - `帖子当前状态不可评论`：帖子被机器审核转人工期间不可评论，等审核通过后再跑评论，或当日放弃
  - 发帖 `过审=False`（水贴/重复内容）：见第 8 节规避

### 首次配置（api_config.json 不存在）

见第 7 节抓包流程。

## 7. 抓包流程（首次配置 / token 失效）

**前提**：`pip3 install mitmproxy requests`；CA 证书 `~/.mitmproxy/mitmproxy-ca-cert.pem` 已装入系统钥匙串并勾选"始终信任"。

```bash
./capture_api.sh
# 流程：启动 MITM :8888 → 设置系统代理 → 你在微信里打开小程序手动完成完整操作
# （进社区 → 看话题列表 → 发 1 帖 → 评论 1 条 → 点赞 1 次）→ 回车 → 恢复直连 → 自动生成 api_config.json
```

**⚠️ 关键坑：capture_api.sh 里写的是 `python3 src/mitm.py`，但 mitm.py 是 mitmproxy addon，直接 python3 跑不会启动代理服务！** 必须用 mitmdump 加载：

```bash
mitmdump -s src/mitm.py -p 8888 &
```

然后用 `networksetup -setwebproxy/-setsecurewebproxy "Wi-Fi" 127.0.0.1 8888` 设系统代理，小程序操作完成后 `extract_api.py --flows /tmp/mitm_flows.jsonl --out api_config.json --base api_config.template.json` 提取，最后务必恢复系统直连并**删除 `/tmp/mitm_flows.jsonl`（内含 token）**。

**extract_api.py 的已知缺陷**（2026-11-24 实测）：

1. 响应 body 是 brotli 压缩时 b64 解出来是乱码，接口识别会失败 → 需手动解压分析
2. authorization 请求头可能是**小写** `authorization`，按大写 key 匹配会漏 → 手动补

所以抓包后**务必人工核对 api_config.json**：auth 三件套齐全、endpoints 五个 path 非空、params 的 schoolId/orgId 是自己账号的真实值。

## 8. 水贴审核规避（血泪教训）

机器审核（machineReviewPass）会判水贴/重复内容，命中后帖子转人工、不可评论、可能不加分：

- ❌ 固定文案（如每天「我要领鸡蛋」）→ 必判水贴
- ❌ 标题/正文与历史发过的帖子**完全相同**（包括测试帖）→ 会判重复
- ✅ 每话题准备 ≥3 套标题+正文 + ≥6 条评论，按 day-of-year 轮换
- ✅ 文案写成自然生活口吻（有细节、有场景），不要口号式
- ✅ 换文案后先用 `--post` 发一条测试帖验证过审，再改回池子

已知接口**无法删除已发的帖子**，测试帖和被判水贴的帖子会留在社区，需在小程序里手动删。

## 9. 关键技术要点（踩坑全集）

- 鉴权是**三个头**，不是单一 token——缺任一个都 401/403
- 发帖 multipart/form-data（手工构造 boundary），评论/点赞/列表 JSON，别搞混
- 点赞 `targetId` 是**评论 id**，`targetType=2` 表示评论（不是帖子）
- **直连网关必须 no_proxy**（run_daily_api.sh 已内置 `export no_proxy="*"` + unset 代理），否则走 MITM/系统代理导致 Connection refused
- 话题列表接口失败时用内置 id 兜底（TOPIC_ID_MAP），任务不因列表抖动中断
- requests.Session 设 `trust_env=False` 防环境代理干扰
- Authorization 的 `_SE...` 后缀疑似含时间戳，token 有生命周期，过期属正常现象
- 日志按天落盘 `logs/YYYY-MM-DD.log`，脚本退出码透传给 crontab/自动化判定

## 10. 自动化接入与维护建议

- crontab / 定时自动化只调 `./run_daily_api.sh`，按退出码 + 日志判定结果（第 5、6 节）
- token 抓取必须由用户本人完成，自动化任务只负责"运行 + 判定 + 汇报"
- 话题池/文案池改动后：`py_compile` 校验 → `--post` 发测试帖验证过审 → 再恢复正式轮换
- 维护记录写在 SKILL.md 末尾或独立 CHANGELOG，注明日期与改动点

## 11. 维护记录

- 2026-11-05 本机重建项目骨架（代码按文档逻辑重写，契约全部走配置）
- 2026-11-24 token 抓取成功（mitmdump 方案 + 手动修复 brotli/小写头两个提取缺陷），任务首次跑通
- 2026-11-24 固定文案被判水贴 → 改为话题文案池按天轮换
- 2026-11-24 内容全面调整为纯生活话题（4 话题），移除全部公司/工作相关内容
- 2026-11-25 按"两帖话题不同 + 间隔 1 分钟"改造；随后改为**每日只发 1 帖**（现行规则）
- 2026-11-26 标题与历史测试帖重复被判水贴（过审=False 转人工，评论失败）→ 再次验证第 8 节规避规则；本文档据此整理为最终版
