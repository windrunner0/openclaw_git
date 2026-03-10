# OpenClaw 实战培训手册

## 目录
1. [OpenClaw 简介](#一openclaw-简介)
2. [核心功能与架构](#二核心功能与架构)
3. [实战案例：股票策略分析系统](#三实战案例股票策略分析系统)
4. [技能开发指南](#四技能开发指南)
5. [常见问题与解决方案](#五常见问题与解决方案)

---

## 一、OpenClaw 简介

### 1.1 什么是 OpenClaw？

OpenClaw（龙虾）是一个开源的 AI 智能体平台，专为构建自动化工作流和智能助手而设计。它支持多通道消息收发、定时任务、技能扩展等核心能力。

**核心特点：**
- 🤖 多模型支持（支持 OpenAI、Claude、Google 等主流模型）
- 📱 多通道集成（QQ、微信、飞书、Discord、Telegram 等）
- ⏰ 定时任务与 Cron 调度
- 🔌 可扩展的技能系统
- 🌐 Web 浏览与搜索能力
- 💾 记忆与上下文管理

### 1.2 典型应用场景

| 场景 | 说明 |
|------|------|
| 金融投研 | 自动收集市场数据、生成策略报告、定时推送 |
| 智能客服 | 多渠道自动回复、知识库问答 |
| 内容运营 | 定时发布、自动采集、内容生成 |
| 个人助理 | 日程管理、信息聚合、智能提醒 |

---

## 二、核心功能与架构

### 2.1 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  QQ Bot  │  │飞书 Bot  │  │ Discord  │  │Telegram │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
├─────────────────────────────────────────────────────────┤
│                      Core Runtime                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  Skills  │  │  Memory  │  │  Tools   │  │  Agent  │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
├─────────────────────────────────────────────────────────┤
│                      Provider Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ OpenAI   │  │ Claude   │  │  Google  │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

### 2.2 关键配置文件

#### `AGENTS.md` - 身份定义
```markdown
# AGENTS.md - Your Workspace

## First Run
- 读取 `BOOTSTRAP.md`（如果存在）
- 删除 BOOTSTRAP.md

## Every Session
1. 读取 `SOUL.md` — 系统人格定义
2. 读取 `USER.md` — 用户信息
3. 读取 `memory/YYYY-MM-DD.md` — 今日上下文
4. 读取 `MEMORY.md` — 长期记忆（主会话）

## Memory 管理
- **Daily notes:** `memory/YYYY-MM-DD.md` — 每日原始日志
- **Long-term:** `MEMORY.md` — 长期记忆（安全敏感）

注意：MEMORY.md 只在主会话加载，群聊不加载
```

#### `SOUL.md` - 系统人格
定义 AI 的核心行为准则和安全策略：
- Prompt 注入防御
- 敏感操作确认机制
- 反泄露输出规范
- 技能/插件安全审查

#### `HEARTBEAT.md` - 定时检查
```markdown
# 检查系统事件

## 早盘策略报告生成
当收到 systemEvent 包含"生成并发送今日A股早盘策略报告"时：
1. 收集最新市场数据
2. 生成完整的早盘策略报告（8个部分）
3. 通过 feishu_doc 创建飞书文档
4. 通过 message 发送到飞书用户
5. 通过 msmtp 发送到邮箱
```

---

## 三、实战案例：股票策略分析系统

### 3.1 项目背景

构建一个自动化 A 股策略分析系统，实现：
- 每日早盘自动生成策略报告
- 量化选股（三因子策略）
- 多渠道推送（邮件、飞书、QQ）
- Web 页面展示

### 3.2 核心代码实现

#### 3.2.1 三因子策略选股脚本

```python
#!/usr/bin/env python3
"""
三因子策略选股脚本
- 龙虎榜统计
- 连阳趋势数据
- 涨停统计
"""

import akshare as ak
import pandas as pd
from datetime import datetime, timedelta

# 参数配置
STAT_DAYS = 20  # 近N天统计龙虎榜和涨停
TREND_MIN_DAYS = 3  # 连涨最少天数
TREND_MAX_DAYS = 5  # 连涨最多天数
TREND_MIN_PCT = 5  # 连涨涨幅最小值（%）
TREND_MAX_PCT = 30  # 连涨涨幅最大值（%）
ZT_MIN_COUNT = 1  # 涨停次数最低要求

# 时间区间
today = datetime.today()
end_date = today.strftime("%Y%m%d")
start_date = (today - timedelta(days=STAT_DAYS)).strftime("%Y%m%d")

# 1. 龙虎榜统计
detail_df = ak.stock_lhb_detail_em(start_date=start_date, end_date=end_date)
lhb_stat = (
    detail_df.groupby(["代码", "名称"])
    .size()
    .reset_index(name="龙虎榜次数")
)

# 2. 连阳趋势数据
lxsz_df = ak.stock_rank_lxsz_ths()
lxsz_df = lxsz_df.rename(columns={"股票代码": "代码", "股票简称": "名称"})
lxsz_df["趋势涨幅"] = lxsz_df["连续涨跌幅"]

# 筛选3~5天连涨且涨幅5%~30%
lxsz_df = lxsz_df[
    (lxsz_df["连涨天数"] >= TREND_MIN_DAYS) &
    (lxsz_df["连涨天数"] <= TREND_MAX_DAYS) &
    (lxsz_df["趋势涨幅"] >= TREND_MIN_PCT) &
    (lxsz_df["趋势涨幅"] <= TREND_MAX_PCT)
]

# 3. 涨停统计
zt_list = []
for d in pd.date_range(start_date, end_date):
    try:
        zt = ak.stock_zt_pool_em(date=d.strftime("%Y%m%d"))
        if not zt.empty:
            zt_list.append(zt[["代码", "名称"]])
    except:
        continue

zt_df = pd.concat(zt_list)
zt_stat = zt_df.groupby(["代码", "名称"]).size().reset_index(name=f"近{STAT_DAYS}日涨停次数")
zt_stat = zt_stat[zt_stat[f"近{STAT_DAYS}日涨停次数"] >= ZT_MIN_COUNT]

# 4. 合并三因子
final_df = lhb_stat.merge(lxsz_df, on=["代码", "名称"], how="inner")
final_df = final_df.merge(zt_stat, on=["代码", "名称"], how="inner")

# 5. 过滤 ST / 转债 / 北交所
code = final_df["代码"].astype(str)
name = final_df["名称"]
final_df = final_df[
    (~name.str.contains("ST", na=False)) &
    (~code.str.startswith(("8", "9"))) &
    (~code.str.startswith(("110", "113", "123", "127", "128")))
]

# 6. 构建综合评分
final_df["综合评分"] = (
    final_df["龙虎榜次数"] * 3 +
    final_df["连涨天数"] * 2 +
    final_df[f"近{STAT_DAYS}日涨停次数"] * 5
)

final_df = final_df.sort_values("综合评分", ascending=False)
```

#### 3.2.2 定时任务配置

```python
# stock_cron_job.py
#!/usr/bin/env python3
"""
A股早盘策略报告自动生成脚本
每日早上8点自动执行
"""
import json
from datetime import datetime

def main():
    print(f"[{datetime.now()}] 开始生成早盘策略报告...")
    print("系统事件：生成今日A股早盘策略报告")
    
    # 记录执行日志
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "event": "stock_strategy_report",
        "status": "completed"
    }
    
    with open("/workspace/projects/workspace/cron_job_log.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")
    
    print(f"[{datetime.now()}] 早盘策略报告任务完成")
    return 0
```

#### 3.2.3 Web 页面展示

```html
<!-- three-factor-strategy.html -->
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>三因子策略选股报告</title>
    <style>
        /* 渐变背景 */
        body {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 900px;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            margin: 0 auto;
            overflow: hidden;
        }
        
        .header {
            background: linear-gradient(135deg, #4caf50 0%, #2e7d32 100%);
            color: white;
            padding: 40px;
            text-align: center;
        }
        
        /* 表格样式 */
        table {
            width: 100%;
            border-collapse: collapse;
        }
        
        thead {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        
        th, td {
            padding: 12px;
            text-align: center;
            border-bottom: 1px solid #eee;
        }
        
        tr:nth-child(even) {
            background-color: #f8f9fa;
        }
        
        tr:hover {
            background-color: #e3f2fd;
            transition: background-color 0.3s;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📊 三因子策略选股报告</h1>
            <div>统计日期：2026年3月10日</div>
        </div>
        <!-- 内容区域 -->
    </div>
</body>
</html>
```

### 3.3 Git 版本控制

```bash
# 初始化仓库
cd /workspace/projects/workspace/openclaw_git
git init

# 添加远程仓库
git remote add origin https://github.com/username/repo.git

# 提交代码
git add .
git commit -m "feat: 添加三因子策略选股报告页面"
git push origin main
```

---

## 四、技能开发指南

### 4.1 技能目录结构

```
skills/
└── your-skill/
    ├── SKILL.md          # 技能说明文档
    ├── scripts/          # 脚本文件
    │   └── run.py
    └── config.json       # 配置文件
```

### 4.2 SKILL.md 模板

```markdown
---
name: your-skill-name
description: 一句话描述技能功能
homepage: https://example.com
metadata:
  openclaw:
    emoji: 🎨
    requires:
      bins: ["python3", "node"]
---

# 技能名称

## 快速开始

### 基本用法
```bash
python3 {baseDir}/scripts/run.py --param value
```

## 参数说明

| 选项 | 描述 | 默认值 |
|------|------|--------|
| --param | 参数说明 | default |

## 输出格式

描述输出结果的格式和示例。
```

### 4.3 常用技能示例

#### Web 搜索技能
```bash
npx ts-node skills/coze-web-search/scripts/search.ts \
  -q "Python programming" \
  --time-range 1d \
  --format markdown
```

#### 图片生成技能
```bash
npx ts-node skills/coze-image-gen/scripts/gen.ts \
  --prompt "A futuristic city at sunset" \
  --size 2K
```

---

## 五、常见问题与解决方案

### 5.1 GitHub 推送失败

**问题：** `Permission to user/repo.git denied`

**原因：** GitHub Token 权限不足

**解决：**
1. 登录 GitHub → Settings → Developer settings → Personal access tokens
2. 生成新 Token，确保勾选 `repo` 权限
3. 更新远程 URL：
```bash
git remote set-url origin https://TOKEN@github.com/user/repo.git
```

### 5.2 QQ 消息发送失败

**问题：** 消息发送后用户收不到

**排查步骤：**
1. 检查 QQ Bot 配置是否正确
2. 确认用户已添加 Bot 为好友
3. 查看网关日志：`openclaw gateway status`
4. 测试最小消息：先发送纯文本，再尝试富媒体

### 5.3 定时任务不执行

**问题：** Cron job 没有按时触发

**排查步骤：**
1. 检查 HEARTBEAT.md 配置
2. 验证 cron 表达式是否正确
3. 查看日志文件确认任务是否被触发
4. 手动执行测试：`python3 stock_cron_job.py`

### 5.4 内存搜索无结果

**问题：** `memory_search` 返回空结果

**原因：** 记忆功能未启用或文件不存在

**解决：**
1. 确认 MEMORY.md 或 memory/*.md 文件存在
2. 检查文件内容是否相关
3. 手动搜索验证内容是否存在

---

## 附录

### A. 常用命令速查

```bash
# 网关管理
openclaw gateway status
sh /workspace/projects/scripts/start.sh    # 启动
sh /workspace/projects/scripts/restart.sh  # 重启
sh /workspace/projects/scripts/stop.sh     # 停止

# Git 操作
git status
git add .
git commit -m "message"
git push origin main
git pull --rebase origin main

# 文件操作
ls -la /workspace/projects/workspace/
read /path/to/file
write /path/to/file "content"
edit /path/to/file "old" "new"
```

### B. 资源链接

- OpenClaw 文档：https://docs.openclaw.ai
- GitHub 仓库：https://github.com/openclaw/openclaw
- 技能市场：https://clawhub.com
- 社区 Discord：https://discord.com/invite/clawd

---

**文档版本：** v1.0  
**最后更新：** 2026年3月10日  
**作者：** 策略君
