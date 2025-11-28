# MCP MongoDB Pipeline 通用查询模式

本文档描述了 EveOneCat 各后端服务中使用的通用 MongoDB Pipeline 查询模式，可用于构建 MCP 服务。

## 目录

- [1. 基础分页查询模式](#1-基础分页查询模式)
- [2. 关联查询模式 ($lookup)](#2-关联查询模式-lookup)
- [3. 聚合统计模式](#3-聚合统计模式)
- [4. 全文搜索模式](#4-全文搜索模式)
- [5. 随机采样模式](#5-随机采样模式)
- [6. 动态字段计算模式](#6-动态字段计算模式)
- [7. 对象与数组转换模式](#7-对象与数组转换模式)
- [8. 条件过滤模式](#8-条件过滤模式)
- [9. 动态配置查询模式](#9-动态配置查询模式)

---

## 1. 基础分页查询模式

这是所有服务中最常用的查询模式，用于实现分页获取数据。

### 模式结构

```python
pipeline = []

# 1. 匹配条件 (可选)
if params is not None:
    pipeline.append({
        "$match": params
    })

# 2. 排序 (通常按 _id 倒序)
pipeline.append({
    "$sort": {"_id": -1}
})

# 3. 跳过 (分页)
if index_id is not None:
    pipeline.append({
        "$skip": (index_id - 1) * page_size
    })

# 4. 限制返回数量
pipeline.append({
    "$limit": page_size
})

results = list(collection.aggregate(pipeline))
```

### 使用场景

| 服务 | Collection | 函数 |
|------|------------|------|
| catgoods | Goods | `Goods.get_many()` |
| catgoods | Goods_Batch | `Batch.get_many()` |
| catgoods | Goods_Code | `Code.get_many()` |
| catgoods | Goods_Collection | `Collection.get_many()` |
| catgoods | Goods_Progress | `Progress.get_many()` |
| backend_vote | Contribute_Article | `Contribute_Article.get_many()` |
| backend_whisper | Whisper | `Whisper.get_many()` |
| backend_check | Catcheck | `Catcheck.get_many()` |
| backend_achievement | Achievement | `Achievement.get_many()` |
| backend_options | Option_Global | `Option_Global.get_many()` |

### 带正则匹配的变体

```python
# 名称模糊匹配
if "name" in params:
    pipeline.append({
        "$match": {"name": {"$regex": params["name"]}}
    })
```

---

## 2. 关联查询模式 ($lookup)

用于跨 Collection 关联查询，类似 SQL 的 JOIN 操作。

### 基础模式结构

```python
pipeline = [
    # 1. 关联查询
    {
        "$lookup": {
            "from": "User",           # 关联的目标表
            "localField": "author_id", # 当前表的外键字段
            "foreignField": "_id",     # 目标表的主键字段
            "as": "author"             # 关联结果的字段名
        }
    },
    # 2. 展开数组 (将数组转为单个对象)
    {
        "$unwind": {
            "path": "$author",
            "preserveNullAndEmptyArrays": True  # 允许关联结果为空
        }
    },
    # 3. 字段投影 (选择需要的字段)
    {
        "$project": {
            "title": 1,
            "author._id": 1,
            "author.nickname": 1,
            "author.avatar": 1
        }
    }
]
```

### 常见关联关系

| 服务 | 主表 | 关联表 | 关联字段 | 用途 |
|------|------|--------|----------|------|
| backend_vote | Contribute_Article | User | author_id -> _id | 获取文章作者信息 |
| backend_whisper | Whisper_Mail | User | user_id -> _id | 获取评论用户信息 |
| backend_whisper | Whisper_Mail | Whisper | whisper_id -> _id | 获取whisper内容 |
| backend_achievement | Achievement_histroy | Achievement | achievement_id -> _id | 获取成就详情 |
| catgoods | Goods_Code | Goods | goods_id -> _id | 获取商品信息 |
| catgoods | Goods_Collection_Cards | Goods_Collection | collection_id -> _id | 获取藏品信息 |
| catgoods | Goods_Progress_Advice | Goods_Progress | progress_id -> _id | 获取进度信息 |

### 完整示例 (文章列表带作者)

```python
@staticmethod
def get_many(index_id, page_size, category=0):
    collection = db["Contribute_Article"]
    pipeline = []
    
    # 分类过滤
    if category != 0:
        pipeline.append({"$match": {"category": category}})
    
    # 游标分页
    if index_id is not None:
        pipeline.append({"$match": {"_id": {"$lt": ObjectId(index_id)}}})
    
    # 排序
    pipeline.append({"$sort": {"_id": -1}})
    
    # 启用状态过滤
    pipeline.append({
        "$match": {
            "$or": [
                {"enable": {"$ne": False}},
                {"enable": {"$exists": False}}
            ]
        }
    })
    
    # 限制数量
    pipeline.append({"$limit": page_size})
    
    # 关联作者
    pipeline.append({
        "$lookup": {
            "from": "User",
            "localField": "author_id",
            "foreignField": "_id",
            "as": "author"
        }
    })
    
    # 展开作者
    pipeline.append({
        "$unwind": {
            "path": "$author",
            "preserveNullAndEmptyArrays": True
        }
    })
    
    # 字段投影
    pipeline.append({
        "$project": {
            "title": 1,
            "category": 1,
            "cover_url": 1,
            "article_url": 1,
            "vote_num": 1,
            "sticker": 1,
            "featured": 1,
            "type": 1,
            "enable": 1,
            "author._id": 1,
            "author.avatar": 1,
            "author.nickname": 1
        }
    })
    
    return list(collection.aggregate(pipeline))
```

---

## 3. 聚合统计模式

用于数据聚合统计，如计数、去重统计等。

### 去重计数模式

```python
# 统计每个 whisper 的已收集人数（不同 user_id 的数量）
pipeline = [
    # 1. 匹配目标文档
    {
        "$match": {
            "whisper_id": {"$in": object_ids}
        }
    },
    # 2. 分组并去重
    {
        "$group": {
            "_id": "$whisper_id",
            "count": {"$addToSet": "$user_id"}  # 使用 addToSet 去重
        }
    },
    # 3. 计算去重后的数量
    {
        "$project": {
            "_id": 1,
            "count": {"$size": "$count"}
        }
    }
]
```

### 使用场景

| 服务 | 函数 | 用途 |
|------|------|------|
| backend_whisper | `Whisper_Mail.count_collected_users()` | 统计每个whisper被多少用户收集 |

---

## 4. 全文搜索模式

基于 MongoDB 全文索引进行搜索。

### 前置条件：创建全文索引

```python
# 创建索引时需要包含文本索引
collection.create_index([("title_split", "text")])
```

### 查询模式

```python
pipeline = [
    # 1. 全文搜索匹配
    {
        "$match": {"$text": {"$search": text}}
    },
    # 2. 按相关性或其他字段排序
    {
        "$sort": {"vote_num": -1}
    },
    # 3. 限制返回数量
    {
        "$limit": page_size
    },
    # 4. 关联其他表
    {
        "$lookup": {
            "from": "User",
            "localField": "author_id",
            "foreignField": "_id",
            "as": "author"
        }
    },
    # ... 后续处理
]
```

### 使用场景

| 服务 | 函数 | 用途 |
|------|------|------|
| backend_vote | `Contribute_Article.get_many_fulltext()` | 文章全文搜索 |

**注意**：标题需要预先进行分词处理（使用 jieba），存储为 `title_split` 字段。

---

## 5. 随机采样模式

从集合中随机获取指定数量的文档。

### 模式结构

```python
pipeline = [
    # 1. 匹配条件 (可选)
    {"$match": {"category": category}},
    
    # 2. 启用状态过滤
    {
        "$match": {
            "$or": [
                {"enable": {"$ne": False}},
                {"enable": {"$exists": False}}
            ]
        }
    },
    
    # 3. 随机采样
    {
        "$sample": {"size": page_size}
    },
    
    # 4. 关联查询等后续操作...
]
```

### 使用场景

| 服务 | 函数 | 用途 |
|------|------|------|
| backend_vote | `Contribute_Article.get_many()` (random=1) | 随机获取文章列表 |

---

## 6. 动态字段计算模式

在查询时动态计算字段值。

### 基于数组最新日期排序

```python
pipeline = [
    # 1. 匹配条件
    {"$match": {"whisper_id": ObjectId(whisper_id)}},
    
    # 2. 添加计算字段
    {
        "$addFields": {
            "record_time": {
                "$cond": {
                    "if": {"$gt": [{"$size": {"$ifNull": ["$logs", []]}}, 0]},
                    "then": {"$max": "$logs.date"},  # 获取 logs 中日期最大的
                    "else": "1970-01-01 00:00:00"    # 没有 logs 的记录排到最后
                }
            },
            "has_logs": {
                "$cond": {
                    "if": {"$gt": [{"$size": {"$ifNull": ["$logs", []]}}, 0]},
                    "then": 1,
                    "else": 0
                }
            }
        }
    },
    
    # 3. 按计算字段排序
    {
        "$sort": {
            "has_logs": -1,      # 有 logs 的排在前面
            "record_time": -1,   # 按 logs 中最新日期降序
            "_id": -1            # 如果时间相同，则按 _id 排序
        }
    }
]
```

### 使用场景

| 服务 | 函数 | 用途 |
|------|------|------|
| backend_whisper | `Whisper_Mail.get_many()` | 按最新消息时间排序 |

---

## 7. 对象与数组转换模式

用于处理嵌套对象结构的查询。

### 对象转数组

```python
{
    "$addFields": {
        "log_array": {
            "$objectToArray": "$log"
        }
    }
}
```

### 数组转对象

```python
{
    "$addFields": {
        "log": {
            "$arrayToObject": "$filtered_log"
        }
    }
}
```

### 数组过滤

```python
{
    "$addFields": {
        "filtered_log": {
            "$filter": {
                "input": "$log_array",
                "cond": {
                    "$or": [
                        {"$eq": ["$$this.v.log_key", log_key]},
                        {"$eq": ["$$this.v.log_key", str(log_key)]}
                    ]
                }
            }
        }
    }
}
```

### 数组排序

```python
{
    "$addFields": {
        "filtered_log": {
            "$sortArray": {
                "input": "$filtered_log",
                "sortBy": {"k": -1}  # 按 key（时间戳）倒序
            }
        }
    }
}
```

### 完整示例 (按 log_key 过滤意见)

```python
@staticmethod
def get_by_progress_and_log_key(progress_id, log_key, index_id=None, page_size=100):
    collection = db["Goods_Progress_Advice"]
    pipeline = []
    
    # 匹配 progress_id
    pipeline.append({"$match": {"progress_id": ObjectId(progress_id)}})
    
    # 将 log 对象转换为数组
    pipeline.append({
        "$addFields": {
            "log_array": {"$objectToArray": "$log"}
        }
    })
    
    # 筛选包含指定 log_key 的记录
    pipeline.append({
        "$match": {
            "log_array.v.log_key": log_key
        }
    })
    
    # 过滤 log 数组，只保留匹配的条目
    pipeline.append({
        "$addFields": {
            "filtered_log": {
                "$filter": {
                    "input": "$log_array",
                    "cond": {"$eq": ["$$this.v.log_key", log_key]}
                }
            }
        }
    })
    
    # 对 log 条目按时间戳排序
    pipeline.append({
        "$addFields": {
            "filtered_log": {
                "$sortArray": {
                    "input": "$filtered_log",
                    "sortBy": {"k": -1}
                }
            }
        }
    })
    
    # 分页
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    pipeline.append({"$limit": page_size})
    
    # 重新构造 log 对象
    pipeline.append({
        "$addFields": {
            "log": {"$arrayToObject": "$filtered_log"}
        }
    })
    
    # 移除临时字段
    pipeline.append({
        "$project": {
            "log_array": 0,
            "filtered_log": 0
        }
    })
    
    return list(collection.aggregate(pipeline))
```

### 使用场景

| 服务 | 函数 | 用途 |
|------|------|------|
| catgoods | `Advice.get_by_progress_and_log_key()` | 按 log_key 过滤意见 |
| catgoods | `Advice.get_by_user_progress_and_log_key()` | 获取用户特定意见 |

---

## 8. 条件过滤模式

### 多条件 OR 过滤

```python
# 过滤启用状态
{
    "$match": {
        "$or": [
            {"enable": {"$ne": False}},
            {"enable": {"$exists": False}}
        ]
    }
}
```

### 时间范围过滤

```python
# 过滤有效期内的藏品
{
    "$match": {
        "start_date": {"$lte": datetime.datetime.now(datetime.timezone.utc)},
        "end_date": {"$gte": datetime.datetime.now(datetime.timezone.utc)},
        "freq": {"$gt": 0}
    }
}
```

### 数组元素匹配

```python
# 筛选有用户消息的记录
{
    "$match": {
        "logs": {
            "$elemMatch": {"role": "user"}
        }
    }
}
```

---

## 9. 动态配置查询模式

backend_options 服务提供了一种配置化的查询方式，支持变量替换。

### 配置结构

```json
{
    "global_vars": {
        "custom_var": "value"
    },
    "queries": [
        {
            "name": "query1",
            "collection": "Whisper_Mail",
            "aggregate": true,
            "pipeline": [
                {"$match": {"owned_date": "{{today}}"}},
                {"$group": {"_id": null, "count": {"$sum": 1}}}
            ]
        }
    ],
    "template": "今日收集数：{{query1.count}}"
}
```

### 内置全局变量

```python
default_vars = {
    "today": "2024-01-01",                    # 当天日期
    "today_start": "2024-01-01 00:00:00",     # 当天开始时间
    "today_end": "2024-01-01 23:59:59",       # 当天结束时间
    "today_start_iso": "2024-01-01T00:00:00", # ISO 格式开始时间
    "today_end_iso": "2024-01-01T23:59:59",   # ISO 格式结束时间
    "now": "2024-01-01 12:00:00",             # 当前时间
    "now_iso": "2024-01-01T12:00:00",         # ISO 格式当前时间
    "timestamp": 1704067200,                   # 当前时间戳
    "yesterday": "2023-12-31",                # 昨天日期
    "yesterday_start": "2023-12-31 00:00:00", # 昨天开始时间
    "yesterday_end": "2023-12-31 23:59:59",   # 昨天结束时间
    "yesterday_start_iso": "...",
    "yesterday_end_iso": "..."
}
```

### 变量替换函数

```python
def replace_variables_in_dict(obj, vars_dict):
    """递归替换字典中的变量占位符 {{var_name}}"""
    if isinstance(obj, dict):
        return {key: replace_variables_in_dict(value, vars_dict) 
                for key, value in obj.items()}
    elif isinstance(obj, list):
        return [replace_variables_in_dict(item, vars_dict) for item in obj]
    elif isinstance(obj, str):
        pattern = r'\{\{(\w+)\}\}'
        def replace_match(match):
            var_name = match.group(1)
            return str(vars_dict.get(var_name, match.group(0)))
        return re.sub(pattern, replace_match, obj)
    return obj
```

### 执行查询

```python
def execute_mongo_query(collection_name, query_config, global_vars):
    collection = db[collection_name]
    
    # 替换查询中的变量
    processed_query = replace_variables_in_dict(
        query_config.get("query", {}), 
        global_vars
    )
    
    is_aggregate = query_config.get("aggregate", False)
    
    if is_aggregate:
        # 聚合查询
        pipeline = query_config.get("pipeline", [])
        processed_pipeline = replace_variables_in_dict(pipeline, global_vars)
        results = list(collection.aggregate(processed_pipeline, maxTimeMS=30000))
    else:
        # 普通查询
        projection = query_config.get("projection", None)
        if projection:
            results = list(collection.find(processed_query, projection))
        else:
            results = list(collection.find(processed_query))
    
    return results
```

### 模板渲染

```python
def render_template(template, query_results, global_vars):
    """
    模板支持：
    - {{global_var}} - 全局变量
    - {{query_name.result}} - 查询结果（JSON格式）
    - {{query_name.count}} - 查询结果数量
    - {{query_name.data}} - 查询结果数组（格式化）
    """
    result = template
    
    for query_name, query_data in query_results.items():
        if isinstance(query_data, list):
            # 替换 {{query_name.count}}
            count_pattern = r'\{\{' + re.escape(query_name) + r'\.count\}\}'
            result = re.sub(count_pattern, str(len(query_data)), result)
            
            # 替换 {{query_name.result}} - JSON 格式
            result_pattern = r'\{\{' + re.escape(query_name) + r'\.result\}\}'
            result_json = json.dumps(query_data, ensure_ascii=False, indent=2)
            result = re.sub(result_pattern, result_json, result)
    
    # 替换全局变量
    for var_name, var_value in global_vars.items():
        pattern = r'\{\{' + re.escape(var_name) + r'(?!\.)\}\}'
        result = re.sub(pattern, str(var_value), result)
    
    return result
```

---

## 常用操作符速查表

### 匹配操作符

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `$eq` | 等于 | `{"field": {"$eq": value}}` |
| `$ne` | 不等于 | `{"field": {"$ne": value}}` |
| `$gt` | 大于 | `{"field": {"$gt": value}}` |
| `$gte` | 大于等于 | `{"field": {"$gte": value}}` |
| `$lt` | 小于 | `{"field": {"$lt": value}}` |
| `$lte` | 小于等于 | `{"field": {"$lte": value}}` |
| `$in` | 在数组中 | `{"field": {"$in": [v1, v2]}}` |
| `$nin` | 不在数组中 | `{"field": {"$nin": [v1, v2]}}` |
| `$regex` | 正则匹配 | `{"field": {"$regex": "pattern"}}` |
| `$exists` | 字段存在 | `{"field": {"$exists": true}}` |

### 逻辑操作符

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `$and` | 与 | `{"$and": [{}, {}]}` |
| `$or` | 或 | `{"$or": [{}, {}]}` |
| `$not` | 非 | `{"field": {"$not": {...}}}` |
| `$nor` | 都不 | `{"$nor": [{}, {}]}` |

### 聚合阶段

| 阶段 | 说明 |
|------|------|
| `$match` | 过滤文档 |
| `$sort` | 排序 |
| `$skip` | 跳过文档 |
| `$limit` | 限制数量 |
| `$lookup` | 关联查询 |
| `$unwind` | 展开数组 |
| `$project` | 字段投影 |
| `$group` | 分组聚合 |
| `$addFields` | 添加字段 |
| `$sample` | 随机采样 |

### 聚合表达式

| 表达式 | 说明 |
|--------|------|
| `$sum` | 求和 |
| `$avg` | 平均值 |
| `$max` | 最大值 |
| `$min` | 最小值 |
| `$size` | 数组长度 |
| `$first` | 第一个元素 |
| `$last` | 最后一个元素 |
| `$addToSet` | 添加到集合（去重）|
| `$push` | 添加到数组 |
| `$ifNull` | 空值处理 |
| `$cond` | 条件表达式 |
| `$filter` | 数组过滤 |
| `$sortArray` | 数组排序 |
| `$objectToArray` | 对象转数组 |
| `$arrayToObject` | 数组转对象 |

