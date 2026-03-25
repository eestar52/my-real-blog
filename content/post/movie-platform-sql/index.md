---
title: "【数据底座】从业务逻辑到取数实战：电影推荐平台核心 SQL 解析"
date: 2026-03-25T18:00:00+08:00
draft: false
weight: 4
tags: ["SQL", "数据分析", "数据产品", "指标定义"]
categories: ["项目集 Portfolio"]
---

在构建了产品需求文档（PRD）、业务流程图以及前端监控大屏之后，数据产品设计的核心最终需要落地到具体的底层数据支撑上。

优秀的数据可视化看板，其背后依赖于准确的埋点设计和高效的数据提取逻辑。本文将以《智能电影数据分析与推荐平台》为核心场景，展示如何通过 SQL 将抽象的业务指标转化为具象的数据查询。

## 1. 底层数据表结构定义 (Schema)

在梳理 SQL 逻辑前，首先需要明确业务数据库的基础表结构。本平台核心依赖于“用户行为日志表”，其简化结构如下：

**表名：`user_behavior_log` (用户行为日志表)**

| 字段名 | 数据类型 | 字段说明 | 枚举值示例 |
| :--- | :--- | :--- | :--- |
| `log_id` | STRING | 日志唯一流水号 | uuid |
| `user_id` | STRING | 用户唯一标识 | u_1001 |
| `movie_id` | STRING | 电影唯一标识 | m_2056 |
| `action_type` | STRING | 行为类型 | exposure(曝光), click(点击), rate(评分) |
| `algorithm_strategy` | STRING | 触发此次曝光的推荐算法 | CF(协同过滤), CB(基于内容) |
| `action_time` | DATETIME | 行为发生时间 | 2026-03-25 10:00:00 |

---

## 2. 核心业务指标的 SQL 实现

### 2.1 业务场景一：推荐模块转化漏斗监控 (大盘 CTR 计算)

**业务需求**：运营大屏需要实时监控不同推荐算法（如协同过滤）的每日点击率（CTR），以评估算法的实际分发效率。
**数据逻辑**：CTR = 指定周期内的总点击次数 / 总曝光次数。

```sql
SELECT 
    DATE(action_time) AS stat_date,
    algorithm_strategy,
    COUNT(DISTINCT CASE WHEN action_type = 'click' THEN log_id END) AS click_uv,
    COUNT(DISTINCT CASE WHEN action_type = 'exposure' THEN log_id END) AS exposure_uv,
    CONCAT(ROUND(COUNT(DISTINCT CASE WHEN action_type = 'click' THEN log_id END) * 100.0 / 
                 COUNT(DISTINCT CASE WHEN action_type = 'exposure' THEN log_id END), 2), '%') AS ctr_rate
FROM 
    user_behavior_log
WHERE 
    action_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY) -- 提取近7天数据
GROUP BY 
    DATE(action_time),
    algorithm_strategy
ORDER BY 
    stat_date DESC, 
    algorithm_strategy;
