# SeaSeek Facebook 市场情报模块规格

## 产品定位
Facebook 是 SeaSeek 的全球市场情报数据源，不再只是潜在客户搜索入口。模块服务于全球 3C 市场研究：品牌与企业发现、产品趋势、消费者需求、竞争对手、社群与活动、公开广告情报。

## 导航与工作台
一级菜单：Facebook 市场情报。模块内独立 Tab：
1. 总览
2. 企业主页 Pages
3. 帖子 Posts
4. 视频与 Reels
5. 群组 Groups
6. 活动 Events
7. 评论与反馈 Comments
8. 广告情报 Ads
9. 地点与商家 Places
10. 采集任务
11. 数据中心

每个数据类型独立定义采集条件、Schema、Actor 路由、结果映射、详情和原始 JSON。统一任务中心与统一数据中心，不让不同采集类型形成孤岛。

## Actor 接入策略
采用统一调度层 + 多个专用 Actor，不依赖一个 Actor 覆盖所有数据类型。Actor ID 由系统固定管理，普通用户不输入 Actor ID。
- 企业主页：`topapi/facebook-pages-search-scraper`，当前已接入；真实端到端验收仍需完成。
- Posts / Reels / Comments / Groups / Events / Ads / Places：全部标记为待接入。候选 Actor 不能仅凭市场页说明进入可用状态，必须先核验当前 Schema、样本输出、公开/授权边界、费用、失败率与字段质量。

## 统一对象模型（目标）
- Page：Facebook 主页及公开企业资料
- Post：帖子正文、时间、媒体、互动指标、来源 Page/Group
- Video：视频与 Reels 元数据、描述、媒体 URL、可见指标
- Group：公开群组资料与可访问的公开内容
- Event：活动资料、时间、地点、主办方与公开参与指标
- Comment：评论正文、时间、可见作者标识、所属 Post
- Ad：公开广告资料、广告主、创意、文案、落地页、可获取的投放状态
- Place：公开商家/地点资料、类别、地址与来源
- RawRecord：原始 Actor JSON、采集源、Actor 版本、采集时间
- Relationship：Page/Post/Group/Event/Ad/Comment 之间的关联

统一对象需保留 source URL、external ID、collected_at、raw payload、字段可用性状态；按 source + external ID 去重。未知字段不伪造，未公开字段标注 unavailable。

## 统一任务模型（目标）
Task：data_type、query、country/region/location、time_range、filters、limit、status、created_at。
ActorRun：actor_id、run_id、input snapshot、status、started_at、finished_at、usage_cost、error。
状态包括 queued、running、succeeded、failed、cancelled。费用上限在提交前校验，运行后记录真实费用。

## 用户与安全边界
SeaSeek 当前按单人工作台设计，不增加组织、成员、角色或多租户 UI。只采集公开或用户有权访问的数据。API Key 仅服务端保存，不进入浏览器存储、日志或 Git。
现有 Supabase 数据库中已有组织级基础表；本阶段不删除或重构现有表。Facebook 持久化模型落地前先审查现有 RLS 和数据访问路径，避免把单人系统扩展成多租户。

## 当前实现状态
- P0 UI：Facebook 市场情报一级入口、独立数据类型 Tab、企业主页现有表单保留、未接入能力明确标注、统一任务/数据中心入口已建立。
- 仍待完成：持久化任务中心、标准对象表、实体关系、多个 Actor 接入、端到端验收、AI 分析。
- 不得把 UI 占位入口描述为已具备真实采集能力。


## Multi-type collection implementation (2026-11)

The Facebook workspace now has schema-driven runners for the additional public-data types:

| SeaSeek type | Fixed Apify Actor | Fixed mode / search type |
|---|---|---|
| Posts | `coregent/facebook-search-scraper` | `searchType=posts` |
| Videos & Reels discovery | `coregent/facebook-search-scraper` | `searchType=videos` |
| Public Group posts | `pnda/facebook-scraper` | `mode=groupPosts` |
| Public post comments | `pnda/facebook-scraper` | `mode=comments` |
| URL-based public Reels | `pnda/facebook-scraper` | `mode=reels` |
| Events | `coregent/facebook-search-scraper` | `searchType=events` |
| Places | `coregent/facebook-search-scraper` | `searchType=places` |
| Public ads | `techbellys/facebook-ads-scraper` | Meta Ad Library URLs / queries per live schema |

The server discovers each Actor's live OpenAPI schema, generates the form from that schema, validates inputs again before execution, and injects fixed modes/search types. The task center records run metadata in `APIFY_CONFIG`; successful datasets are archived with their raw payload and source metadata. Cookie/session/password/token-like input fields are filtered and rejected. Only public or user-authorized data is in scope.

**Runtime verification status:** code uploaded and deployed as Worker version 37, 100% traffic, with the existing `APIFY_CONFIG` KV binding preserved. The integration environment received HTTP 403 from Apify's account verification endpoint, so live schemas and sample runs are not yet certified. The UI must show an explicit schema/connection error rather than imply success until Apify returns a valid response. Billable runs require an explicit user click.
