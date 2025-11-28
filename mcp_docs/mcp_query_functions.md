# MCP MongoDB 查询函数文档

本文档描述了各后端服务中可供 MCP 调用的 MongoDB 查询函数，包括完整的 Pipeline 定义和使用示例。

## 目录

- [1. catgoods 服务](#1-catgoods-服务)
- [2. backend_vote 服务](#2-backend_vote-服务)
- [3. backend_whisper 服务](#3-backend_whisper-服务)
- [4. backend_options 服务](#4-backend_options-服务)
- [5. backend_check 服务](#5-backend_check-服务)
- [6. backend_achievement 服务](#6-backend_achievement-服务)
- [7. backend_message 服务](#7-backend_message-服务)

---

## 1. catgoods 服务

### 1.1 商品查询

#### get_goods_list - 获取商品列表

**Collection**: `Goods`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.name | str | 否 | 名称模糊匹配 |
| params.price | str | 否 | 价格精确匹配 |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    pipeline = []
    
    if params is not None:
        if "name" in params:
            pipeline.append({
                "$match": {"name": {"$regex": params["name"]}}
            })
        if "price" in params:
            pipeline.append({
                "$match": {"price": params["price"]}
            })
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 1.2 批次查询

#### get_batch_list - 获取批次列表

**Collection**: `Goods_Batch`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.batch_size | str | 否 | 批次大小 |
| params.goods_id | str | 否 | 商品ID |
| params.batch_key | str | 否 | 批次标识 |
| params.is_generated | bool | 否 | 是否已生成 |
| params.is_selling | bool | 否 | 是否在售 |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    if "goods_id" in params:
        params["goods_id"] = ObjectId(params["goods_id"])
    
    pipeline = []
    
    if params is not None:
        pipeline.append({"$match": params})
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 1.3 验证码查询

#### get_code_list - 获取验证码列表

**Collection**: `Goods_Code`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.batch_key | str | 否 | 批次标识 |
| params.received | bool | 否 | 是否已领取 |
| params.goods_id | str | 否 | 商品ID |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    pipeline = []
    
    if params is not None:
        pipeline.append({"$match": params})
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

#### get_many_by_receive - 获取用户领取的验证码 (带商品关联)

**Collection**: `Goods_Code`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| receive_id | str | 是 | 用户ID |

**Pipeline**:
```python
def get_many_by_receive(receive_id):
    pipeline = [
        {"$match": {"receive_id": ObjectId(receive_id)}},
        {"$sort": {"_id": -1}},
        {
            "$lookup": {
                "from": "Goods",
                "localField": "goods_id",
                "foreignField": "_id",
                "as": "goods"
            }
        },
        {
            "$unwind": {
                "path": "$goods",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "code": 1,
                "batch_key": 1,
                "goods._id": 1,
                "goods.name": 1,
                "goods.cover_url": 1,
                "goods.description": 1
            }
        }
    ]
    return list(collection.aggregate(pipeline))
```

---

### 1.4 藏品查询

#### get_collection_list - 获取藏品列表

**Collection**: `Goods_Collection`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.name | str | 否 | 名称模糊匹配 |
| params.freq | int | 否 | 频率 |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    pipeline = []
    
    if params is not None:
        if "name" in params:
            pipeline.append({
                "$match": {"name": {"$regex": params["name"]}}
            })
        if "freq" in params:
            pipeline.append({
                "$match": {"freq": params["freq"]}
            })
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

#### get_pool - 获取当前有效的藏品奖池

**Collection**: `Goods_Collection`

**Pipeline**:
```python
def get_pool():
    pipeline = [
        {
            "$match": {
                "start_date": {"$lte": datetime.datetime.now(datetime.timezone.utc)},
                "end_date": {"$gte": datetime.datetime.now(datetime.timezone.utc)},
                "freq": {"$gt": 0}
            }
        },
        {
            "$project": {"freq": 1}
        }
    ]
    results = list(collection.aggregate(pipeline))
    
    # 构建奖池（按 freq 重复添加）
    pool = []
    for item in results:
        for i in range(item["freq"]):
            pool.append(item["_id"])
    return pool
```

---

### 1.5 藏品卡片查询

#### get_many_by_owner - 获取用户拥有的藏品卡片 (带藏品关联)

**Collection**: `Goods_Collection_Cards`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| owner_id | str | 是 | 用户ID |

**Pipeline**:
```python
def get_many_by_owner(owner_id):
    pipeline = [
        {"$match": {"owner_id": ObjectId(owner_id)}},
        {"$sort": {"_id": -1}},
        {
            "$lookup": {
                "from": "Goods_Collection",
                "localField": "collection_id",
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
                "status": 1,
                "index": 1,
                "owner_id": 1,
                "owned_date": 1,
                "birth_date": 1,
                "goods_code": 1,
                "collection._id": 1,
                "collection.name": 1,
                "collection.img_url": 1,
                "collection.description": 1
            }
        }
    ]
    return list(collection.aggregate(pipeline))
```

---

### 1.6 进度查询

#### get_progress_list - 获取商品进度列表

**Collection**: `Goods_Progress`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| admin | int | 是 | 是否管理员视角 (0=只看is_show=1) |
| params.name | str | 否 | 名称模糊匹配 |

**Pipeline**:
```python
def get_many(index_id, page_size, admin, params=None):
    pipeline = []
    
    # 非管理员只能看已展示的
    if admin == 0:
        pipeline.append({"$match": {"is_show": 1}})
    
    if params is not None:
        if "name" in params:
            pipeline.append({
                "$match": {"name": {"$regex": params["name"]}}
            })
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 1.7 意见查询

#### get_advice_by_progress - 获取进度下的所有意见 (管理员)

**Collection**: `Goods_Progress_Advice`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| progress_id | str | 是 | 进度ID |
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |

**Pipeline**:
```python
def get_by_progress(progress_id, index_id=None, page_size=100):
    pipeline = [
        {"$match": {"progress_id": ObjectId(progress_id)}},
        {"$sort": {"_id": -1}}
    ]
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

#### get_advice_by_user - 获取用户的所有意见 (带进度关联)

**Collection**: `Goods_Progress_Advice`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_id | str | 是 | 用户ID |
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |

**Pipeline**:
```python
def get_by_user(user_id, index_id=None, page_size=100):
    pipeline = [
        {"$match": {"user_id": ObjectId(user_id)}},
        {"$sort": {"_id": -1}},
        {
            "$lookup": {
                "from": "Goods_Progress",
                "localField": "progress_id",
                "foreignField": "_id",
                "as": "progress"
            }
        },
        {
            "$unwind": {
                "path": "$progress",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "progress_id": 1,
                "user_id": 1,
                "content": 1,
                "log": 1,
                "progress.name": 1,
                "progress.cover_url": 1
            }
        }
    ]
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

#### get_advice_by_progress_and_log_key - 按 log_key 过滤意见

**Collection**: `Goods_Progress_Advice`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| progress_id | str | 是 | 进度ID |
| log_key | str | 是 | 日志标识 |
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |

**Pipeline**:
```python
def get_by_progress_and_log_key(progress_id, log_key, index_id=None, page_size=100):
    pipeline = [
        {"$match": {"progress_id": ObjectId(progress_id)}},
        
        # 将 log 对象转换为数组
        {
            "$addFields": {
                "log_array": {"$objectToArray": "$log"}
            }
        },
        
        # 筛选包含指定 log_key 的记录
        {
            "$match": {"log_array.v.log_key": log_key}
        },
        
        # 过滤只保留匹配的 log 条目
        {
            "$addFields": {
                "filtered_log": {
                    "$filter": {
                        "input": "$log_array",
                        "cond": {"$eq": ["$$this.v.log_key", log_key]}
                    }
                }
            }
        },
        
        # 按时间戳排序
        {
            "$addFields": {
                "filtered_log": {
                    "$sortArray": {
                        "input": "$filtered_log",
                        "sortBy": {"k": -1}
                    }
                }
            }
        },
        
        # 提取最新时间戳用于文档排序
        {
            "$addFields": {
                "latest_timestamp": {
                    "$toLong": {
                        "$substr": [
                            {"$arrayElemAt": ["$filtered_log.k", 0]},
                            4,
                            -1
                        ]
                    }
                }
            }
        },
        
        {"$sort": {"latest_timestamp": -1, "_id": -1}}
    ]
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    # 重新构造 log 对象
    pipeline.append({
        "$addFields": {"log": {"$arrayToObject": "$filtered_log"}}
    })
    
    # 移除临时字段
    pipeline.append({
        "$project": {
            "log_array": 0,
            "filtered_log": 0,
            "latest_timestamp": 0
        }
    })
    
    return list(collection.aggregate(pipeline))
```

---

## 2. backend_vote 服务

### 2.1 文章查询

#### get_article_many - 获取文章列表 (带作者关联)

**Collection**: `Contribute_Article`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | str | 否 | 游标ID (上一页最后一条的_id) |
| page_size | int | 是 | 每页数量 |
| category | int | 否 | 分类 (0=全部) |
| featured | int | 否 | 精选 (0=全部, 1=精选) |
| author_id | str | 否 | 作者ID |
| random | int | 否 | 随机 (0=否, 1=是) |

**Pipeline**:
```python
def get_many(index_id, page_size, category=0, featured=0, author_id=None, random=0):
    pipeline = []
    
    # 分类过滤
    if category != 0:
        pipeline.append({"$match": {"category": category}})
    
    # 精选过滤
    if featured != 0:
        pipeline.append({"$match": {"featured": featured}})
    
    # 作者过滤
    if author_id is not None:
        pipeline.append({"$match": {"author_id": ObjectId(author_id)}})
    
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
    
    # 随机采样或限制数量
    if random == 1:
        pipeline.append({"$sample": {"size": page_size}})
    else:
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

#### get_article_many_hot - 获取热门文章列表

**Collection**: `Contribute_Article`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | str | 否 | 游标ID |
| page_size | int | 是 | 每页数量 |
| category | int | 否 | 分类 |

**Pipeline**:
```python
def get_many_hot(index_id, page_size, category=0):
    pipeline = []
    
    # 游标分页（基于 vote_num 和 _id 组合排序）
    if index_id is not None:
        article = Contribute_Article.get_by_id(index_id)
        pipeline.append({
            "$match": {
                "$or": [
                    {
                        "$and": [
                            {"vote_num": article.vote_num},
                            {"_id": {"$lt": ObjectId(index_id)}}
                        ]
                    },
                    {"vote_num": {"$lt": article.vote_num}}
                ]
            }
        })
    
    if category != 0:
        pipeline.append({"$match": {"category": category}})
    
    # 按投票数和ID排序
    pipeline.append({"$sort": {"vote_num": -1, "_id": -1}})
    
    # 启用状态过滤
    pipeline.append({
        "$match": {
            "$or": [
                {"enable": {"$ne": False}},
                {"enable": {"$exists": False}}
            ]
        }
    })
    
    pipeline.append({"$limit": page_size})
    
    # 关联作者...
    # (同 get_many)
    
    return list(collection.aggregate(pipeline))
```

#### get_article_fulltext - 全文搜索文章

**Collection**: `Contribute_Article`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| text | str | 是 | 搜索文本 |
| page_size | int | 是 | 返回数量 |

**Pipeline**:
```python
def get_many_fulltext(text, page_size):
    pipeline = [
        {"$match": {"$text": {"$search": text}}},
        {"$sort": {"vote_num": -1}},
        {"$limit": page_size},
        {
            "$lookup": {
                "from": "User",
                "localField": "author_id",
                "foreignField": "_id",
                "as": "author"
            }
        },
        {
            "$unwind": {
                "path": "$author",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "title": 1,
                "category": 1,
                "cover_url": 1,
                "article_url": 1,
                "vote_num": 1,
                "sticker": 1,
                "featured": 1,
                "type": 1,
                "author._id": 1,
                "author.avatar": 1,
                "author.nickname": 1
            }
        }
    ]
    return list(collection.aggregate(pipeline))
```

#### get_article_one - 获取文章详情 (带作者关联)

**Collection**: `Contribute_Article`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | str | 是 | 文章ID |

**Pipeline**:
```python
def get_one(id):
    pipeline = [
        {"$match": {"_id": ObjectId(id)}},
        {
            "$lookup": {
                "from": "User",
                "localField": "author_id",
                "foreignField": "_id",
                "as": "author"
            }
        },
        {
            "$unwind": {
                "path": "$author",
                "preserveNullAndEmptyArrays": True
            }
        },
        {
            "$project": {
                "title": 1,
                "content": 1,
                "cover_url": 1,
                "article_url": 1,
                "video_url": 1,
                "vote_num": 1,
                "sticker": 1,
                "featured": 1,
                "type": 1,
                "enable": 1,
                "pics": 1,
                "author._id": 1,
                "author.avatar": 1,
                "author.nickname": 1
            }
        }
    ]
    results = list(collection.aggregate(pipeline))
    return results[0] if results else None
```

---

## 3. backend_whisper 服务

### 3.1 Whisper 查询

#### get_whisper_list - 获取碎碎念列表

**Collection**: `Whisper`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| filter_enabled | bool | 否 | 是否按时间过滤 |
| params | dict | 否 | 其他过滤条件 |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None, filter_enabled=False):
    pipeline = []
    
    if params is not None:
        pipeline.append({"$match": params})
    
    # 时间过滤（只显示未到期的）
    if filter_enabled:
        now = datetime.now().strftime("%Y-%m-%dT%H:%M")
        pipeline.append({
            "$match": {
                "$or": [
                    {"content.last_get_time": {"$exists": False}},
                    {"content.last_get_time": None},
                    {"content.last_get_time": {"$gt": now}}
                ]
            }
        })
    
    # 按 order 和 _id 排序
    pipeline.append({
        "$sort": {
            "content.order": -1,
            "_id": -1
        }
    })
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 3.2 Whisper_Mail 查询

#### get_whisper_mail_list - 获取用户信箱列表

**Collection**: `Whisper_Mail`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| whisper_id | str | 否 | Whisper ID |
| user_id | str | 否 | 用户ID |
| filter_has_replied | bool | 否 | 只看有用户回复的 |
| sort_by_id | bool | 否 | 按ID排序 (否则按最新消息时间) |

**Pipeline**:
```python
def get_many(index_id, page_size, whisper_id=None, user_id=None, 
             filter_has_replied=False, sort_by_id=False):
    pipeline = []
    
    if whisper_id is not None:
        pipeline.append({"$match": {"whisper_id": ObjectId(whisper_id)}})
    
    if user_id is not None:
        pipeline.append({"$match": {"user_id": ObjectId(user_id)}})
    
    # 过滤有用户消息的记录
    if filter_has_replied:
        pipeline.append({
            "$match": {
                "logs": {"$elemMatch": {"role": "user"}}
            }
        })
    
    if sort_by_id:
        pipeline.append({"$sort": {"_id": -1}})
    else:
        # 按最新消息时间排序
        pipeline.append({
            "$addFields": {
                "record_time": {
                    "$cond": {
                        "if": {"$gt": [{"$size": {"$ifNull": ["$logs", []]}}, 0]},
                        "then": {"$max": "$logs.date"},
                        "else": "1970-01-01 00:00:00"
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
        })
        pipeline.append({
            "$sort": {
                "has_logs": -1,
                "record_time": -1,
                "_id": -1
            }
        })
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    # 关联用户信息 (当没有指定 user_id 时)
    if user_id is None:
        pipeline.append({
            "$lookup": {
                "from": "User",
                "localField": "user_id",
                "foreignField": "_id",
                "as": "user"
            }
        })
        pipeline.append({
            "$unwind": {
                "path": "$user",
                "preserveNullAndEmptyArrays": True
            }
        })
        pipeline.append({
            "$project": {
                "user._id": 1,
                "user.nickname": 1,
                "user.avatar": 1,
                "owned_date": 1,
                "logs": 1,
                "whisper_id": 1
            }
        })
    
    # 关联 Whisper 信息 (当没有指定 whisper_id 时)
    if whisper_id is None:
        pipeline.append({
            "$lookup": {
                "from": "Whisper",
                "localField": "whisper_id",
                "foreignField": "_id",
                "as": "whisper"
            }
        })
        pipeline.append({
            "$unwind": {
                "path": "$whisper",
                "preserveNullAndEmptyArrays": True
            }
        })
        pipeline.append({
            "$project": {
                "user_id": 1,
                "owned_date": 1,
                "logs": 1,
                "whisper._id": 1,
                "whisper.content": 1
            }
        })
    
    return list(collection.aggregate(pipeline))
```

#### count_collected_users - 统计每个 Whisper 的收集人数

**Collection**: `Whisper_Mail`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| whisper_ids | list | 是 | Whisper ID 列表 |

**Pipeline**:
```python
def count_collected_users(whisper_ids):
    object_ids = [ObjectId(wid) if isinstance(wid, str) else wid 
                  for wid in whisper_ids]
    
    pipeline = [
        {"$match": {"whisper_id": {"$in": object_ids}}},
        {
            "$group": {
                "_id": "$whisper_id",
                "count": {"$addToSet": "$user_id"}  # 去重
            }
        },
        {
            "$project": {
                "_id": 1,
                "count": {"$size": "$count"}
            }
        }
    ]
    
    results = list(collection.aggregate(pipeline))
    
    # 转换为字典
    result_dict = {}
    for item in results:
        result_dict[str(item["_id"])] = item["count"]
    
    # 填充未找到的为 0
    for wid in whisper_ids:
        if str(wid) not in result_dict:
            result_dict[str(wid)] = 0
    
    return result_dict
```

---

## 4. backend_options 服务

### 4.1 全局配置查询

#### get_option_global_list - 获取全局配置列表

**Collection**: `Option_Global`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| app | str | 否 | 应用标识 |
| unit | str | 否 | 配置单元 |

**Pipeline**:
```python
def get_many(index_id, page_size, app=None, unit=None):
    pipeline = []
    
    if app is not None:
        pipeline.append({"$match": {"app": app}})
    
    if unit is not None:
        pipeline.append({"$match": {"unit": unit}})
    
    pipeline.append({"$sort": {"app": 1, "unit": 1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 4.2 动态统计查询

#### execute_mongo_query - 执行配置化的 MongoDB 查询

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| collection_name | str | 是 | Collection 名称 |
| query_config | dict | 是 | 查询配置 |
| global_vars | dict | 是 | 全局变量 |

**query_config 结构**:
```json
{
    "query": {},           // 普通查询条件
    "projection": {},      // 输出字段
    "aggregate": false,    // 是否聚合查询
    "pipeline": []         // 聚合管道
}
```

**实现**:
```python
def execute_mongo_query(collection_name, query_config, global_vars):
    collection = db[collection_name]
    
    # 替换变量
    processed_query = replace_variables_in_dict(
        query_config.get("query", {}), 
        global_vars
    )
    
    is_aggregate = query_config.get("aggregate", False)
    
    if is_aggregate:
        pipeline = query_config.get("pipeline", [])
        processed_pipeline = replace_variables_in_dict(pipeline, global_vars)
        results = list(collection.aggregate(processed_pipeline, maxTimeMS=30000))
    else:
        projection = query_config.get("projection", None)
        if projection:
            results = list(collection.find(processed_query, projection))
        else:
            results = list(collection.find(processed_query))
    
    return results
```

**内置全局变量**:
- `{{today}}` - 当天日期
- `{{today_start}}` / `{{today_end}}` - 当天开始/结束时间
- `{{yesterday}}` - 昨天日期
- `{{now}}` - 当前时间
- `{{timestamp}}` - 当前时间戳

---

## 5. backend_check 服务

### 5.1 投票活动查询

#### get_check_list - 获取投票活动列表

**Collection**: `Catcheck`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.header | str | 否 | 标题 |
| params.type | str | 否 | 类型 |
| params.is_enabled | bool | 否 | 是否启用 |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    pipeline = []
    
    if params is not None:
        pipeline.append({"$match": params})
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

## 6. backend_achievement 服务

### 6.1 成就查询

#### get_achievement_list - 获取成就列表

**Collection**: `Achievement`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| params.key | str | 否 | 成就 key |

**Pipeline**:
```python
def get_many(index_id, page_size, params=None):
    pipeline = []
    
    if params is not None:
        pipeline.append({"$match": params})
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    return list(collection.aggregate(pipeline))
```

---

### 6.2 成就历史查询

#### get_achievement_history - 获取用户成就历史 (带成就详情)

**Collection**: `Achievement_histroy`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| index_id | int | 否 | 分页索引 |
| page_size | int | 是 | 每页数量 |
| achievement_id | str | 否 | 成就ID |
| user_id | str | 否 | 用户ID |
| key | str | 否 | 成就 key |

**Pipeline**:
```python
def get_many(index_id, page_size, achievement_id=None, user_id=None, key=None):
    pipeline = []
    
    if achievement_id is not None:
        pipeline.append({"$match": {"achievement_id": ObjectId(achievement_id)}})
    
    if key is not None:
        pipeline.append({"$match": {"key": key}})
    
    if user_id is not None:
        pipeline.append({"$match": {"user_id": ObjectId(user_id)}})
    
    pipeline.append({"$sort": {"_id": -1}})
    
    if index_id is not None:
        pipeline.append({"$skip": (index_id - 1) * page_size})
    
    pipeline.append({"$limit": page_size})
    
    # 关联成就详情
    pipeline.append({
        "$lookup": {
            "from": "Achievement",
            "localField": "achievement_id",
            "foreignField": "_id",
            "as": "achievement"
        }
    })
    
    pipeline.append({
        "$unwind": {
            "path": "$achievement",
            "preserveNullAndEmptyArrays": True
        }
    })
    
    pipeline.append({
        "$project": {
            "user_id": 1,
            "key": 1,
            "time": 1,
            "achievement._id": 1,
            "achievement.name": 1,
            "achievement.imgurl": 1
        }
    })
    
    return list(collection.aggregate(pipeline))
```

---

## 7. backend_message 服务

### 7.1 消息查询

#### get_my_message - 获取用户消息 (带文章和发送者信息)

**Collection**: `Message`

**参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| user_id | str | 是 | 用户ID |

**实现**:
```python
def get_my_message(token):
    id = Profile_Manager.get_id_by_token(token)
    if id == "":
        return []
    
    # 获取消息列表
    messages = Message.get_by_to_id(id)
    
    processed_messages = []
    for msg in messages:
        processed_msg = msg.copy()
        
        # 关联文章信息
        if msg.get("ref_app") == "Contribute_Article" and msg.get("ref_id"):
            article = db["Contribute_Article"].find_one({
                "_id": ObjectId(msg["ref_id"])
            })
            if article:
                processed_msg["article_info"] = {
                    "title": article.get("title", ""),
                    "article_url": article.get("article_url", "")
                }
        
        # 关联发送者信息
        if msg.get("from_id"):
            from_user = db["User"].find_one({
                "_id": ObjectId(msg["from_id"])
            })
            if from_user:
                processed_msg["from_user_info"] = {
                    "nickname": from_user.get("nickname", "")
                }
        
        processed_messages.append(processed_msg)
    
    return processed_messages
```

---

## MCP 工具函数模板

以下是构建 MCP 服务时可复用的函数模板：

### 通用分页查询

```python
async def query_collection(
    collection_name: str,
    match_params: dict = None,
    sort_field: str = "_id",
    sort_order: int = -1,
    skip: int = None,
    limit: int = 20,
    lookup_configs: list = None,
    project_fields: dict = None
) -> list:
    """
    通用 MongoDB 聚合查询
    
    Args:
        collection_name: Collection 名称
        match_params: 匹配条件
        sort_field: 排序字段
        sort_order: 排序方向 (1=升序, -1=降序)
        skip: 跳过数量
        limit: 返回数量
        lookup_configs: 关联配置 [{from, localField, foreignField, as}]
        project_fields: 投影字段
    """
    collection = db[collection_name]
    pipeline = []
    
    # 匹配
    if match_params:
        pipeline.append({"$match": match_params})
    
    # 排序
    pipeline.append({"$sort": {sort_field: sort_order}})
    
    # 分页
    if skip is not None:
        pipeline.append({"$skip": skip})
    pipeline.append({"$limit": limit})
    
    # 关联
    if lookup_configs:
        for config in lookup_configs:
            pipeline.append({"$lookup": config})
            pipeline.append({
                "$unwind": {
                    "path": f"${config['as']}",
                    "preserveNullAndEmptyArrays": True
                }
            })
    
    # 投影
    if project_fields:
        pipeline.append({"$project": project_fields})
    
    return list(collection.aggregate(pipeline))
```

### ObjectId 转换工具

```python
from bson.objectid import ObjectId

def convert_objectid_to_str(obj):
    """递归将 ObjectId 转换为字符串"""
    if isinstance(obj, ObjectId):
        return str(obj)
    elif isinstance(obj, list):
        return [convert_objectid_to_str(item) for item in obj]
    elif isinstance(obj, dict):
        return {key: convert_objectid_to_str(value) for key, value in obj.items()}
    return obj

def convert_str_to_objectid(obj, fields: list):
    """将指定字段的字符串转换为 ObjectId"""
    if isinstance(obj, dict):
        for field in fields:
            if field in obj and isinstance(obj[field], str):
                obj[field] = ObjectId(obj[field])
    return obj
```

