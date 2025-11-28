# MCP 业务分析查询文档

本文档提供常见业务分析场景的 MongoDB 查询模式，用于支持模糊需求查询，如"哪个成就最受欢迎"、"最新的murmur数据"等。

## 目录

- [1. Murmur (Whisper) 分析](#1-murmur-whisper-分析)
- [2. 成就分析](#2-成就分析)
- [3. 投稿文章分析](#3-投稿文章分析)
- [4. 用户活跃度分析](#4-用户活跃度分析)
- [5. 藏品分析](#5-藏品分析)
- [6. 投票活动分析](#6-投票活动分析)
- [7. 时间维度查询工具](#7-时间维度查询工具)

---

## 1. Murmur (Whisper) 分析

### 1.1 获取最新的 N 条 Murmur

**场景**: "最新的murmur卡片数据如何？"

**Collection**: `Whisper`

```python
def get_latest_murmurs(limit=10):
    """获取最新的 N 条 murmur"""
    pipeline = [
        {"$sort": {"_id": -1}},
        {"$limit": limit},
        {
            "$project": {
                "_id": 1,
                "content.title": 1,
                "content.content": 1,
                "content.img": 1,
                "content.order": 1,
                "content.last_get_time": 1
            }
        }
    ]
    return list(db["Whisper"].aggregate(pipeline))
```

---

### 1.2 Murmur 总体统计

**场景**: "现在murmur的数据是怎样的？"

**Collection**: `Whisper`, `Whisper_Mail`

```python
def get_murmur_overview():
    """获取 murmur 总体统计"""
    whisper_collection = db["Whisper"]
    mail_collection = db["Whisper_Mail"]
    
    # 总数
    total_count = whisper_collection.count_documents({})
    
    # 今日新增
    today = datetime.now().strftime("%Y-%m-%d")
    # 注意: Whisper 没有 created_date 字段，可以用 _id 的时间戳估算
    today_start = ObjectId.from_datetime(
        datetime.strptime(today, "%Y-%m-%d")
    )
    today_count = whisper_collection.count_documents({
        "_id": {"$gte": today_start}
    })
    
    # 总收集次数
    total_collections = mail_collection.count_documents({})
    
    # 今日收集次数
    today_collections = mail_collection.count_documents({
        "owned_date": today
    })
    
    return {
        "total_murmurs": total_count,
        "today_new_murmurs": today_count,
        "total_collections": total_collections,
        "today_collections": today_collections
    }
```

---

### 1.3 最受欢迎的 Murmur (收集人数排行)

**场景**: "哪个murmur被收集最多？"

**Collection**: `Whisper_Mail`

```python
def get_most_collected_murmurs(limit=10):
    """获取收集人数最多的 murmur"""
    pipeline = [
        # 按 whisper_id 分组，统计不同用户数
        {
            "$group": {
                "_id": "$whisper_id",
                "collect_count": {"$addToSet": "$user_id"}
            }
        },
        # 计算去重后的数量
        {
            "$project": {
                "_id": 1,
                "collect_count": {"$size": "$collect_count"}
            }
        },
        # 按收集数降序排序
        {"$sort": {"collect_count": -1}},
        # 取前 N 条
        {"$limit": limit},
        # 关联 Whisper 获取详情
        {
            "$lookup": {
                "from": "Whisper",
                "localField": "_id",
                "foreignField": "_id",
                "as": "whisper"
            }
        },
        {
            "$unwind": {
                "path": "$whisper",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "whisper_id": "$_id",
                "collect_count": 1,
                "title": "$whisper.content.title",
                "content": "$whisper.content.content"
            }
        }
    ]
    return list(db["Whisper_Mail"].aggregate(pipeline))
```

---

### 1.4 按日期统计 Murmur 收集趋势

**场景**: "最近一周每天有多少人收集murmur？"

**Collection**: `Whisper_Mail`

```python
def get_murmur_collection_trend(days=7):
    """获取最近 N 天的收集趋势"""
    from datetime import timedelta
    
    # 计算起始日期
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    start_date_str = start_date.strftime("%Y-%m-%d")
    
    pipeline = [
        # 过滤日期范围
        {
            "$match": {
                "owned_date": {"$gte": start_date_str}
            }
        },
        # 按日期分组统计
        {
            "$group": {
                "_id": "$owned_date",
                "collection_count": {"$sum": 1},
                "unique_users": {"$addToSet": "$user_id"}
            }
        },
        # 计算唯一用户数
        {
            "$project": {
                "date": "$_id",
                "collection_count": 1,
                "unique_user_count": {"$size": "$unique_users"}
            }
        },
        # 按日期排序
        {"$sort": {"date": 1}}
    ]
    return list(db["Whisper_Mail"].aggregate(pipeline))
```

---

## 2. 成就分析

### 2.1 成就获得数量排行榜

**场景**: "哪个成就的获得数量最多？"

**Collection**: `Achievement_histroy`

```python
def get_most_achieved(limit=10):
    """获取被获得最多的成就排行"""
    pipeline = [
        # 按成就 key 分组统计
        {
            "$group": {
                "_id": "$key",
                "achievement_id": {"$first": "$achievement_id"},
                "count": {"$sum": 1}
            }
        },
        # 按数量降序排序
        {"$sort": {"count": -1}},
        # 取前 N 条
        {"$limit": limit},
        # 关联成就详情
        {
            "$lookup": {
                "from": "Achievement",
                "localField": "achievement_id",
                "foreignField": "_id",
                "as": "achievement"
            }
        },
        {
            "$unwind": {
                "path": "$achievement",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "key": "$_id",
                "count": 1,
                "name": "$achievement.name",
                "imgurl": "$achievement.imgurl"
            }
        }
    ]
    return list(db["Achievement_histroy"].aggregate(pipeline))
```

---

### 2.2 成就总体统计

**场景**: "成就系统的整体情况如何？"

**Collection**: `Achievement`, `Achievement_histroy`

```python
def get_achievement_overview():
    """获取成就系统总体统计"""
    achievement_collection = db["Achievement"]
    history_collection = db["Achievement_histroy"]
    
    # 成就定义总数
    total_achievements = achievement_collection.count_documents({})
    
    # 已颁发成就总数
    total_issued = history_collection.count_documents({})
    
    # 获得成就的用户数
    pipeline = [
        {"$group": {"_id": "$user_id"}},
        {"$count": "unique_users"}
    ]
    result = list(history_collection.aggregate(pipeline))
    unique_users = result[0]["unique_users"] if result else 0
    
    # 今日颁发数量
    today = datetime.now().strftime("%Y-%m-%d")
    today_issued = history_collection.count_documents({"time": today})
    
    return {
        "total_achievements": total_achievements,
        "total_issued": total_issued,
        "unique_users_with_achievements": unique_users,
        "today_issued": today_issued,
        "avg_achievements_per_user": round(total_issued / unique_users, 2) if unique_users > 0 else 0
    }
```

---

### 2.3 最稀有的成就 (获得人数最少)

**场景**: "哪些成就最稀有？"

**Collection**: `Achievement_histroy`, `Achievement`

```python
def get_rarest_achievements(limit=10):
    """获取最稀有的成就（获得人数最少）"""
    pipeline = [
        # 按成就分组统计
        {
            "$group": {
                "_id": "$key",
                "achievement_id": {"$first": "$achievement_id"},
                "count": {"$sum": 1}
            }
        },
        # 按数量升序排序
        {"$sort": {"count": 1}},
        # 取前 N 条
        {"$limit": limit},
        # 关联成就详情
        {
            "$lookup": {
                "from": "Achievement",
                "localField": "achievement_id",
                "foreignField": "_id",
                "as": "achievement"
            }
        },
        {
            "$unwind": {
                "path": "$achievement",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "key": "$_id",
                "count": 1,
                "name": "$achievement.name",
                "imgurl": "$achievement.imgurl"
            }
        }
    ]
    return list(db["Achievement_histroy"].aggregate(pipeline))
```

---

### 2.4 用户成就完成度

**场景**: "某个用户的成就完成情况？"

**Collection**: `Achievement`, `Achievement_histroy`

```python
def get_user_achievement_progress(user_id):
    """获取用户成就完成度"""
    achievement_collection = db["Achievement"]
    history_collection = db["Achievement_histroy"]
    
    # 总成就数
    total = achievement_collection.count_documents({})
    
    # 用户已获得数
    achieved = history_collection.count_documents({
        "user_id": ObjectId(user_id)
    })
    
    # 用户已获得的成就列表
    pipeline = [
        {"$match": {"user_id": ObjectId(user_id)}},
        {"$sort": {"time": -1}},
        {
            "$lookup": {
                "from": "Achievement",
                "localField": "achievement_id",
                "foreignField": "_id",
                "as": "achievement"
            }
        },
        {"$unwind": "$achievement"},
        {
            "$project": {
                "key": 1,
                "time": 1,
                "name": "$achievement.name",
                "imgurl": "$achievement.imgurl"
            }
        }
    ]
    achievements = list(history_collection.aggregate(pipeline))
    
    return {
        "total_achievements": total,
        "achieved_count": achieved,
        "completion_rate": f"{round(achieved / total * 100, 1)}%" if total > 0 else "0%",
        "achievements": achievements
    }
```

---

## 3. 投稿文章分析

### 3.1 文章总体统计

**场景**: "投稿数据整体情况如何？"

**Collection**: `Contribute_Article`

```python
def get_article_overview():
    """获取投稿文章总体统计"""
    collection = db["Contribute_Article"]
    
    # 总文章数
    total = collection.count_documents({})
    
    # 启用的文章数
    enabled = collection.count_documents({
        "$or": [
            {"enable": {"$ne": False}},
            {"enable": {"$exists": False}}
        ]
    })
    
    # 精选文章数
    featured = collection.count_documents({"featured": 1})
    
    # 按类型统计
    pipeline = [
        {
            "$group": {
                "_id": "$type",
                "count": {"$sum": 1}
            }
        }
    ]
    type_stats = {item["_id"]: item["count"] for item in collection.aggregate(pipeline)}
    
    # 总投票数
    pipeline = [
        {
            "$group": {
                "_id": None,
                "total_votes": {"$sum": "$vote_num"}
            }
        }
    ]
    result = list(collection.aggregate(pipeline))
    total_votes = result[0]["total_votes"] if result else 0
    
    return {
        "total_articles": total,
        "enabled_articles": enabled,
        "disabled_articles": total - enabled,
        "featured_articles": featured,
        "type_distribution": type_stats,
        "total_votes": total_votes
    }
```

---

### 3.2 热门文章排行榜

**场景**: "哪些文章投票最多？"

**Collection**: `Contribute_Article`

```python
def get_top_voted_articles(limit=10, category=0):
    """获取投票最多的文章"""
    pipeline = []
    
    # 分类过滤
    if category != 0:
        pipeline.append({"$match": {"category": category}})
    
    # 启用状态过滤
    pipeline.append({
        "$match": {
            "$or": [
                {"enable": {"$ne": False}},
                {"enable": {"$exists": False}}
            ]
        }
    })
    
    # 按投票数排序
    pipeline.append({"$sort": {"vote_num": -1}})
    pipeline.append({"$limit": limit})
    
    # 关联作者
    pipeline.append({
        "$lookup": {
            "from": "User",
            "localField": "author_id",
            "foreignField": "_id",
            "as": "author"
        }
    })
    pipeline.append({
        "$unwind": {
            "path": "$author",
            "preserveNullAndEmptyArrays": True
        }
    })
    
    pipeline.append({
        "$project": {
            "title": 1,
            "vote_num": 1,
            "cover_url": 1,
            "type": 1,
            "featured": 1,
            "author.nickname": 1
        }
    })
    
    return list(db["Contribute_Article"].aggregate(pipeline))
```

---

### 3.3 最活跃的投稿作者

**场景**: "哪些用户投稿最多？"

**Collection**: `Contribute_Article`

```python
def get_top_contributors(limit=10):
    """获取投稿最多的作者"""
    pipeline = [
        # 按作者分组
        {
            "$group": {
                "_id": "$author_id",
                "article_count": {"$sum": 1},
                "total_votes": {"$sum": "$vote_num"},
                "featured_count": {
                    "$sum": {"$cond": [{"$eq": ["$featured", 1]}, 1, 0]}
                }
            }
        },
        # 按文章数排序
        {"$sort": {"article_count": -1}},
        {"$limit": limit},
        # 关联用户信息
        {
            "$lookup": {
                "from": "User",
                "localField": "_id",
                "foreignField": "_id",
                "as": "user"
            }
        },
        {
            "$unwind": {
                "path": "$user",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "user_id": "$_id",
                "nickname": "$user.nickname",
                "avatar": "$user.avatar",
                "article_count": 1,
                "total_votes": 1,
                "featured_count": 1,
                "avg_votes": {
                    "$round": [{"$divide": ["$total_votes", "$article_count"]}, 1]
                }
            }
        }
    ]
    return list(db["Contribute_Article"].aggregate(pipeline))
```

---

### 3.4 最热门的贴纸

**场景**: "哪些表情贴纸使用最多？"

**Collection**: `Contribute_Article`

```python
def get_top_stickers(limit=10):
    """获取使用最多的贴纸"""
    pipeline = [
        # 过滤有贴纸的文章
        {"$match": {"sticker": {"$exists": True, "$ne": {}}}},
        # 将 sticker 对象转为数组
        {
            "$project": {
                "stickers": {"$objectToArray": "$sticker"}
            }
        },
        # 展开数组
        {"$unwind": "$stickers"},
        # 按贴纸分组统计
        {
            "$group": {
                "_id": "$stickers.k",
                "total_count": {"$sum": "$stickers.v"},
                "article_count": {"$sum": 1}
            }
        },
        # 按使用次数排序
        {"$sort": {"total_count": -1}},
        {"$limit": limit},
        {
            "$project": {
                "sticker": "$_id",
                "total_count": 1,
                "article_count": 1
            }
        }
    ]
    return list(db["Contribute_Article"].aggregate(pipeline))
```

---

## 4. 用户活跃度分析

### 4.1 用户活跃度概览

**场景**: "用户活跃情况如何？"

**Collections**: 多个

```python
def get_user_activity_overview(user_id):
    """获取用户活跃度概览"""
    user_oid = ObjectId(user_id)
    
    # 投稿数
    article_count = db["Contribute_Article"].count_documents({
        "author_id": user_oid
    })
    
    # 获得的总投票数
    pipeline = [
        {"$match": {"author_id": user_oid}},
        {"$group": {"_id": None, "total": {"$sum": "$vote_num"}}}
    ]
    result = list(db["Contribute_Article"].aggregate(pipeline))
    total_votes_received = result[0]["total"] if result else 0
    
    # 收集的 murmur 数
    murmur_collected = db["Whisper_Mail"].count_documents({
        "user_id": user_oid
    })
    
    # 获得的成就数
    achievement_count = db["Achievement_histroy"].count_documents({
        "user_id": user_oid
    })
    
    # 参与的投票活动数
    check_participated = db["Catcheck_history"].count_documents({
        "attend_id": user_oid
    })
    
    # 拥有的藏品数
    collection_cards = db["Goods_Collection_Cards"].count_documents({
        "owner_id": user_oid,
        "status": 1
    })
    
    return {
        "user_id": user_id,
        "articles_posted": article_count,
        "votes_received": total_votes_received,
        "murmurs_collected": murmur_collected,
        "achievements_earned": achievement_count,
        "votes_participated": check_participated,
        "collection_cards_owned": collection_cards
    }
```

---

### 4.2 每日活跃用户统计

**场景**: "每天有多少活跃用户？"

**Collection**: `Whisper_Mail` (作为活跃指标之一)

```python
def get_daily_active_users(days=30):
    """获取每日活跃用户数（基于 murmur 收集）"""
    from datetime import timedelta
    
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days)
    start_date_str = start_date.strftime("%Y-%m-%d")
    
    pipeline = [
        {"$match": {"owned_date": {"$gte": start_date_str}}},
        {
            "$group": {
                "_id": "$owned_date",
                "active_users": {"$addToSet": "$user_id"}
            }
        },
        {
            "$project": {
                "date": "$_id",
                "active_user_count": {"$size": "$active_users"}
            }
        },
        {"$sort": {"date": 1}}
    ]
    return list(db["Whisper_Mail"].aggregate(pipeline))
```

---

## 5. 藏品分析

### 5.1 藏品总体统计

**场景**: "藏品系统整体情况？"

**Collections**: `Goods_Collection`, `Goods_Collection_Cards`

```python
def get_collection_overview():
    """获取藏品总体统计"""
    collection_def = db["Goods_Collection"]
    collection_cards = db["Goods_Collection_Cards"]
    
    # 藏品定义总数
    total_definitions = collection_def.count_documents({})
    
    # 已发放的藏品卡片总数
    total_cards = collection_cards.count_documents({})
    
    # 有效期内的藏品
    now = datetime.now(datetime.timezone.utc)
    active_collections = collection_def.count_documents({
        "start_date": {"$lte": now},
        "end_date": {"$gte": now}
    })
    
    # 拥有藏品的用户数
    pipeline = [
        {"$match": {"status": 1}},
        {"$group": {"_id": "$owner_id"}},
        {"$count": "unique_owners"}
    ]
    result = list(collection_cards.aggregate(pipeline))
    unique_owners = result[0]["unique_owners"] if result else 0
    
    return {
        "total_collection_types": total_definitions,
        "total_cards_issued": total_cards,
        "active_collections": active_collections,
        "unique_card_owners": unique_owners
    }
```

---

### 5.2 最稀有的藏品

**场景**: "哪些藏品发放数量最少？"

**Collection**: `Goods_Collection_Cards`

```python
def get_rarest_collections(limit=10):
    """获取最稀有的藏品（发放数量最少）"""
    pipeline = [
        # 按藏品分组统计
        {
            "$group": {
                "_id": "$collection_id",
                "issued_count": {"$sum": 1}
            }
        },
        # 按数量升序
        {"$sort": {"issued_count": 1}},
        {"$limit": limit},
        # 关联藏品详情
        {
            "$lookup": {
                "from": "Goods_Collection",
                "localField": "_id",
                "foreignField": "_id",
                "as": "collection"
            }
        },
        {
            "$unwind": {
                "path": "$collection",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "collection_id": "$_id",
                "issued_count": 1,
                "name": "$collection.name",
                "img_url": "$collection.img_url"
            }
        }
    ]
    return list(db["Goods_Collection_Cards"].aggregate(pipeline))
```

---

## 6. 投票活动分析

### 6.1 投票活动总体统计

**场景**: "投票活动的参与情况？"

**Collections**: `Catcheck`, `Catcheck_history`

```python
def get_check_overview():
    """获取投票活动总体统计"""
    check_collection = db["Catcheck"]
    history_collection = db["Catcheck_history"]
    
    # 活动总数
    total_checks = check_collection.count_documents({})
    
    # 已启用的活动
    enabled_checks = check_collection.count_documents({"is_enabled": True})
    
    # 总参与次数
    total_participations = history_collection.count_documents({})
    
    # 参与过投票的用户数
    pipeline = [
        {"$group": {"_id": "$attend_id"}},
        {"$count": "unique_users"}
    ]
    result = list(history_collection.aggregate(pipeline))
    unique_participants = result[0]["unique_users"] if result else 0
    
    return {
        "total_activities": total_checks,
        "enabled_activities": enabled_checks,
        "total_participations": total_participations,
        "unique_participants": unique_participants
    }
```

---

### 6.2 最受欢迎的投票活动

**场景**: "哪个投票活动参与人数最多？"

**Collection**: `Catcheck_history`

```python
def get_most_popular_checks(limit=10):
    """获取参与人数最多的投票活动"""
    pipeline = [
        # 按活动分组
        {
            "$group": {
                "_id": "$check_id",
                "participant_count": {"$sum": 1}
            }
        },
        # 按参与数排序
        {"$sort": {"participant_count": -1}},
        {"$limit": limit},
        # 关联活动详情
        {
            "$lookup": {
                "from": "Catcheck",
                "localField": "_id",
                "foreignField": "_id",
                "as": "check"
            }
        },
        {
            "$unwind": {
                "path": "$check",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "check_id": "$_id",
                "participant_count": 1,
                "title": "$check.title",
                "type": "$check.type",
                "is_enabled": "$check.is_enabled"
            }
        }
    ]
    return list(db["Catcheck_history"].aggregate(pipeline))
```

---

## 7. 时间维度查询工具

### 7.1 通用日期范围查询

```python
from datetime import datetime, timedelta

def get_date_range(range_type="today"):
    """
    获取日期范围
    
    Args:
        range_type: "today", "yesterday", "this_week", "last_week", 
                   "this_month", "last_month", "last_7_days", "last_30_days"
    
    Returns:
        (start_date_str, end_date_str)
    """
    now = datetime.now()
    today = now.replace(hour=0, minute=0, second=0, microsecond=0)
    
    if range_type == "today":
        start = today
        end = now
    elif range_type == "yesterday":
        start = today - timedelta(days=1)
        end = today - timedelta(seconds=1)
    elif range_type == "this_week":
        start = today - timedelta(days=today.weekday())
        end = now
    elif range_type == "last_week":
        start = today - timedelta(days=today.weekday() + 7)
        end = today - timedelta(days=today.weekday()) - timedelta(seconds=1)
    elif range_type == "this_month":
        start = today.replace(day=1)
        end = now
    elif range_type == "last_month":
        first_of_this_month = today.replace(day=1)
        end = first_of_this_month - timedelta(seconds=1)
        start = (first_of_this_month - timedelta(days=1)).replace(day=1)
    elif range_type == "last_7_days":
        start = today - timedelta(days=7)
        end = now
    elif range_type == "last_30_days":
        start = today - timedelta(days=30)
        end = now
    else:
        raise ValueError(f"Unknown range_type: {range_type}")
    
    return (start.strftime("%Y-%m-%d"), end.strftime("%Y-%m-%d"))
```

---

### 7.2 通用时间范围统计

```python
def count_by_date_field(collection_name, date_field, range_type="last_7_days"):
    """
    按日期字段统计数量
    
    Args:
        collection_name: Collection 名称
        date_field: 日期字段名 (如 "owned_date", "time", "created_time")
        range_type: 日期范围类型
    """
    start_date, end_date = get_date_range(range_type)
    
    collection = db[collection_name]
    
    pipeline = [
        {
            "$match": {
                date_field: {
                    "$gte": start_date,
                    "$lte": end_date
                }
            }
        },
        {
            "$group": {
                "_id": f"${date_field}",
                "count": {"$sum": 1}
            }
        },
        {"$sort": {"_id": 1}},
        {
            "$project": {
                "date": "$_id",
                "count": 1,
                "_id": 0
            }
        }
    ]
    
    return list(collection.aggregate(pipeline))
```

---

## 8. MCP 自然语言查询映射

以下是常见自然语言查询到函数的映射表：

| 自然语言查询 | 对应函数 | 参数 |
|-------------|---------|------|
| "最新的murmur" | `get_latest_murmurs()` | `limit=10` |
| "murmur整体数据" | `get_murmur_overview()` | - |
| "最受欢迎的murmur" | `get_most_collected_murmurs()` | `limit=10` |
| "murmur收集趋势" | `get_murmur_collection_trend()` | `days=7` |
| "哪个成就获得最多" | `get_most_achieved()` | `limit=10` |
| "成就整体情况" | `get_achievement_overview()` | - |
| "最稀有的成就" | `get_rarest_achievements()` | `limit=10` |
| "用户成就完成度" | `get_user_achievement_progress()` | `user_id` |
| "文章整体数据" | `get_article_overview()` | - |
| "热门文章" | `get_top_voted_articles()` | `limit=10` |
| "活跃作者排行" | `get_top_contributors()` | `limit=10` |
| "热门贴纸" | `get_top_stickers()` | `limit=10` |
| "用户活跃度" | `get_user_activity_overview()` | `user_id` |
| "每日活跃用户" | `get_daily_active_users()` | `days=30` |
| "藏品整体情况" | `get_collection_overview()` | - |
| "最稀有的藏品" | `get_rarest_collections()` | `limit=10` |
| "投票活动情况" | `get_check_overview()` | - |
| "最受欢迎的投票" | `get_most_popular_checks()` | `limit=10` |

---

## 9. 示例：构建问答式 MCP 工具

```python
def answer_analytics_question(question: str) -> dict:
    """
    根据自然语言问题返回分析结果
    
    示例问题：
    - "现在murmur的数据是怎样的？"
    - "哪个成就的获得数量最多？"
    - "最新的murmur卡片数据如何？"
    """
    question_lower = question.lower()
    
    # Murmur 相关
    if "murmur" in question_lower or "碎碎念" in question_lower:
        if "最新" in question_lower:
            return {
                "query_type": "latest_murmurs",
                "data": get_latest_murmurs(10)
            }
        elif "整体" in question_lower or "数据" in question_lower or "怎样" in question_lower:
            return {
                "query_type": "murmur_overview",
                "data": get_murmur_overview()
            }
        elif "受欢迎" in question_lower or "收集" in question_lower:
            return {
                "query_type": "most_collected_murmurs",
                "data": get_most_collected_murmurs(10)
            }
    
    # 成就相关
    if "成就" in question_lower:
        if "最多" in question_lower or "排行" in question_lower:
            return {
                "query_type": "most_achieved",
                "data": get_most_achieved(10)
            }
        elif "稀有" in question_lower or "最少" in question_lower:
            return {
                "query_type": "rarest_achievements",
                "data": get_rarest_achievements(10)
            }
        elif "整体" in question_lower or "情况" in question_lower:
            return {
                "query_type": "achievement_overview",
                "data": get_achievement_overview()
            }
    
    # 文章相关
    if "文章" in question_lower or "投稿" in question_lower:
        if "热门" in question_lower or "最多" in question_lower:
            return {
                "query_type": "top_articles",
                "data": get_top_voted_articles(10)
            }
        elif "整体" in question_lower:
            return {
                "query_type": "article_overview",
                "data": get_article_overview()
            }
    
    return {
        "query_type": "unknown",
        "message": "无法理解的问题，请尝试更具体的描述",
        "suggestions": [
            "最新的murmur",
            "murmur整体数据",
            "哪个成就获得最多",
            "热门文章排行"
        ]
    }
```

这个函数可以作为 MCP 工具的入口，将自然语言问题映射到具体的查询函数。

