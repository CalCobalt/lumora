# 校园碎片：学习 Agent 实施计划

> 版本：v1.0  
> 日期：2026-09-14  
> 定位：先做个人可用的学习 Agent，再补竞赛版材料。  
> 开发节奏：每天 2-4 小时，边学 Flutter 边做。  
> 工期：不锁死 40 天，预估 17-23 周，整体约 5-6 个月。

## 1. 项目目标

### 1.1 产品目标

做一个以错题为核心的学习 Agent：

```text
拍照/上传错题
→ AI 提取题干、答案、解题思路
→ 判断错因和知识点
→ 用户确认或修正
→ 生成复习任务
→ 完成后更新掌握度
→ Agent 重新规划下一次学习
```

首页以“今日计划 + Agent 为什么这样安排”为主，聊天只作为辅助入口。

### 1.2 个人版成功标准

- Android 端可安装、可长期使用。
- 断网时核心功能可用。
- 番茄、打卡、待办、错题和复习数据重启不丢。
- 一道错题可以走完“识别 → 确认 → 复习 → 反馈”闭环。
- 登录后可自动增量同步到 Supabase。

### 1.3 竞赛版成功标准

- 有完整设计报告、系统架构图、数据库图和 README。
- 有 50-100 道真实错题的识别效果数据。
- 有 10 名左右用户的试用反馈。
- 有“无知识库 / 有知识库 / 有知识库+学习记忆”的对照实验。
- 有 40-60 秒预告片和 5-8 分钟完整演示。
- 能回答“创新点、准确率、隐私、断网、成本、为什么不是普通 ChatGPT”等问题。

## 2. 已锁定决策

| 项目 | 决策 |
|---|---|
| 移动端 | Android 优先 |
| Web 端 | 后补核心流程，图片上传代替相机，不要求完整离线对等 |
| 开发预览 | 先利用 Linux 桌面快速调试 UI |
| 客户端 | Flutter + GetX |
| 本地数据库 | Drift/SQLite |
| 后端 | Supabase Auth + PostgreSQL + Storage + Edge Functions |
| 登录 | 邮箱 + 密码 |
| 身份流程 | 游客可完整使用，首次登录时合并本地数据 |
| 同步 | 本地优先，登录且联网时自动增量同步 |
| AI 协议 | 先按 OpenAI 兼容的 chat/completions 设计，后续可替换 Base URL、模型和适配器 |
| Embedding | 单独接入中文 embedding 服务，默认按 1024 维设计 |
| 知识库 | 开放许可资料 + 自建题库 + AI 生成后人工校验 |
| 试点科目 | 高等数学 |
| 架构范围 | 底层支持多学科，第一版只深耕高等数学 |
| Agent 交互 | 首页仪表盘 + Agent 计划卡片，聊天为辅 |

## 3. 明确不做

- 好友、聊天、广场、动态、发帖
- 自动导入完整课程表
- 自定义皮肤和主题商店
- 支付、积分、商城
- 复杂知识图谱和多 Agent 协作
- 直接收录没有许可证的教材、题库和试卷 PDF
- iOS
- Web 完整离线对等
- 后台常驻精确计时和系统级番茄钟
- 实时多人协作
- 模型训练和微调

## 4. 技术架构

### 4.1 客户端分层

```text
lib/
  app/         启动、路由、主题、依赖初始化
  modules/     首页、登录、番茄、打卡、待办、错题、复习、知识库、设置
  data/
    local/     Drift 表、DAO、迁移
    remote/    Supabase 客户端、DTO、远程仓库
    sync/      同步队列、冲突处理、重试
  services/    AI、Embedding、知识检索、Agent、文件压缩
  shared/      通用组件、工具、常量、异常
```

### 4.2 Supabase 侧

```text
supabase/
  migrations/            数据库迁移和 RLS
  functions/
    ai-analyze/          图片/文本 → 结构化错题
    kb-search/           关键词 + 向量检索
    kb-embed/            调用 embedding 服务
    agent-run/           Agent 规划与重规划
  seed/                  知识点、题目、知识片段
```

### 4.3 安全边界

- AI Key 和 embedding Key 只存 Supabase Secrets。
- Flutter 客户端只调用 Edge Function，不直接持有服务端密钥。
- Supabase 所有用户表启用 RLS。
- 知识库内容登录用户只读；用户错题、掌握度和学习记录只能本人读写。
- Storage 使用私有桶：`mistake-images`、`kb-source`。
- 用户可导出和删除自己的数据。

## 5. 核心数据模型

### 5.1 内容与知识结构

| 表 | 作用 | 关键字段 |
|---|---|---|
| `subjects` | 科目 | id、name、description |
| `knowledge_points` | 知识点树 | id、subject_id、parent_id、name、description、difficulty |
| `knowledge_edges` | 前置/关联关系 | from_kp_id、to_kp_id、relation |
| `kb_sources` | 资料来源 | id、title、author、license、url、source_type |
| `kb_chunks` | 可检索内容 | id、kp_id、source_id、title、content、metadata、embedding、embedding_model |
| `questions` | 题库 | id、kp_id、stem、answer、solution、difficulty、type、source_id |
| `question_knowledge` | 题目和知识点多对多 | question_id、kp_id、weight |

### 5.2 用户学习数据

| 表 | 作用 | 关键字段 |
|---|---|---|
| `profiles` | 用户资料 | id、nickname、timezone |
| `focus_sessions` | 专注记录 | id、user_id、start_at、end_at、duration_sec |
| `checkins` | 打卡记录 | id、user_id、date、note |
| `todos` | 待办 | id、user_id、title、completed、sort_order、due_at |
| `mistake_notes` | 错题 | id、user_id、image_path、question、solution、answer、error_type、subject、tags、ai_confidence |
| `mistake_knowledge` | 错题和知识点关联 | mistake_id、kp_id |
| `review_tasks` | 复习任务 | id、user_id、mistake_id、kp_id、due_at、interval_days、status |
| `user_kp_mastery` | 用户知识点掌握度 | user_id、kp_id、mastery、correct_count、wrong_count、last_reviewed_at |
| `learning_events` | 学习事件流 | id、user_id、event_type、kp_id、payload、created_at |

### 5.3 同步公共字段

所有可同步业务表统一包含：

```text
id uuid
user_id nullable
created_at utc
updated_at utc
deleted_at nullable
device_id
```

本地额外使用：

```text
sync_state: local | pending | synced | conflict
```

## 6. 同步规则

1. 本地数据库是主写入口，所有写入先成功落本地。
2. 业务数据和同步队列必须在同一本地事务内写入。
3. 游客数据的 `user_id` 为空。
4. 首次登录时把本地数据的 `user_id` 绑定到当前账号，并全部标记为 `pending`。
5. 登录且联网时自动同步；同步范围默认最近 90 天和全部未同步记录。
6. 删除使用 `deleted_at` 软删除，不做物理删除。
7. 冲突规则：先比较 `updated_at`，较新的记录覆盖较旧记录；时间相同则比较 `device_id`，字典序较大的一方保留。
8. 每次冲突写入 `sync_conflicts`，设置页可查看。
9. 网络失败使用指数退避重试，最大间隔 30 分钟。
10. 首版不做实时推送同步，不做 CRDT。

## 7. 知识库建设

### 7.1 内容来源优先级

1. 自建知识点和例题。
2. 明确开放许可的公开资料。
3. AI 生成后经过人工校验的题目。
4. 无许可证资料只做个人参考，不进入公开版本、竞赛版本和演示数据。

### 7.2 首版规模

```text
知识点：80-120 个
知识片段：300-500 条
题库：150-300 道
高数模块：极限、导数、微分、积分、级数、微分方程
```

### 7.3 导入流程

```text
Markdown / CSV / JSONL
→ 清洗与去重
→ 按知识点或单题切片
→ 补充 metadata 和许可证字段
→ 调 embedding 服务
→ upsert 到 Supabase
```

规则：

- 概念内容每块 300-600 字。
- 一道题一块，不把多道题混在一起。
- 每条数据必须有稳定 `source_id`，重复导入不产生重复记录。
- `kb_chunks.embedding` 默认 1024 维。
- `embedding_model` 和 `source_id` 必须存储。
- 更换 embedding 模型时全量重建，不混用向量。

### 7.4 检索流程

```text
用户问题或错题
→ 提取科目、知识点和难度过滤条件
→ metadata 过滤
→ pgvector 相似度检索 Top 20
→ 关键词/标题二次筛选
→ 取 Top 5-8
→ 交给模型生成答案
→ 返回引用 chunk_id 和来源
```

没有检索到足够依据时，模型必须明确说明知识库依据不足。

## 8. Agent 与 AI 接口

### 8.1 结构化错题输出

```json
{
  "subject": "高等数学",
  "question": "",
  "solution_steps": [],
  "answer": "",
  "error_type": "concept | calculation | reading | method | unknown",
  "knowledge_points": [],
  "difficulty": 1,
  "confidence": 0.0
}
```

### 8.2 Agent 工具

```text
analyze_mistake(image_url)                 识别错题、错因和知识点
search_knowledge(query, filters, top_k)    检索知识点和例题
get_weak_points(user_id, subject)          查询薄弱知识点
update_mastery(user_id, kp_id, result)     更新掌握度
schedule_review(user_id, kp_ids)           生成复习任务
run_agent(goal, context)                   生成或重排今日计划
```

### 8.3 Agent 原则

- Agent 提出计划，用户确认后才创建批量任务。
- AI 识别失败时允许用户手动填写和保存。
- AI 结果必须可编辑。
- 错因分类和复习排期优先使用确定性规则，不全部交给模型。
- 掌握度首版使用规则计算：

```text
初始 50 分
答对 +8
答错 -12
连续答错额外 -8
长期未复习缓慢衰减
范围 0-100
```

## 9. 里程碑与排期

| 阶段 | 预计 | 主要交付 | 验收标准 |
|---|---:|---|---|
| M0 环境与基础 | 1-2 周 | Flutter 环境、Android SDK、项目骨架、GetX、路由、主题 | Linux 能跑，Android 真机或模拟器能安装 |
| M1 本地核心版 | 3-4 周 | 首页、番茄、打卡、待办、手动错题、设置 | 断网可用，重启数据不丢 |
| M2 知识库与 Agent | 3-4 周 | 知识点、题库、错题诊断、复习计划、掌握度 | 完成一次完整 Agent 闭环 |
| M3 云账号与同步 | 3-4 周 | Supabase 登录、云端备份、自动同步、RLS | 游客数据可合并，离线修改可同步 |
| M4 Android 发布版 | 2 周 | 权限、异常、性能、APK、真机回归 | 主流程无 P0/P1 bug |
| M5 Web 演示版 | 2-3 周 | Web 登录、知识库、错题、复习、图片上传 | 可用于演示和答辩 |
| M6 竞赛增强版 | 3-4 周 | 实验数据、用户反馈、设计报告、PPT、视频 | 提交材料完整，可答辩 |

总计约 17-23 周。时间不足时优先保证 M1-M4。

## 10. 测试计划

### 10.1 单元测试

- 番茄计时状态转换
- 待办排序和完成状态
- 掌握度计算
- 复习间隔生成
- 同步冲突选择
- 游客数据合并
- AI JSON 解析和异常回退

### 10.2 集成测试

- 本地 CRUD 后重启
- 断网新增错题，联网后同步
- 两台设备修改同一条数据
- 删除后同步不复活
- 登录、退出、重新登录
- 图片上传失败和重试
- AI 超时、限流、返回格式错误

### 10.3 验收测试

- 50-100 道高数图片识别测试
- 30 条知识检索测试
- Recall@5、引用正确率、响应时间、失败率
- 10 名用户试用
- Android 小屏和大屏布局
- Web 上传图片完整流程

## 11. 风险与处理

| 风险 | 处理 |
|---|---|
| Flutter 基础不足 | M0 只做骨架和基础页面，不碰同步和 AI |
| Android SDK 未安装 | M0 第一时间安装并跑通真机 |
| 知识库资料版权不清 | 只纳入开放许可、自建或人工校验内容 |
| AI 输出不稳定 | 固定 JSON Schema，失败降级为手动填写 |
| AI 成本过高 | 图片压缩、限制上下文、缓存、失败只重试一次 |
| 自动同步冲突复杂 | 只做 LWW + 软删除，不做 CRDT |
| Web 和本地数据库兼容 | Web 放到 M5，接受能力和离线差异 |
| 范围膨胀 | 新功能统一进入 backlog，当前里程碑完成前不插队 |

## 12. 前 14 天任务

| 天 | 任务 |
|---|---|
| D1 | 安装 Android Studio、Android SDK、模拟器，跑通 `flutter doctor` |
| D2 | 创建 Flutter 项目，同时运行 Linux 和 Android |
| D3 | 接入 GetX、路由、主题，完成启动页和登录占位页 |
| D4 | 完成底部导航和 6 个空页面 |
| D5 | 建立 modules、data、sync、shared 目录 |
| D6-7 | 学 Drift，建 Todo、Checkin、FocusSession 三张表 |
| D8-9 | 完成待办增删改查 |
| D10-11 | 完成番茄计时和本地记录 |
| D12-13 | 完成打卡日历基础 UI |
| D14 | 真机跑完整流程，修 bug，提交第一个可运行版本 |

## 13. 砍功能顺序

1. Web 完整功能
2. 自动同步，改手动同步
3. 动画和美化
4. 高级标签和复杂统计
5. iOS

不能砍：

- 本地数据不丢
- 错题闭环
- AI 失败降级
- RLS 和密钥隔离
- README、演示视频、测试数据
