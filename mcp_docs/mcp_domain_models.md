# MCP MongoDB 领域模型文档

本文档描述了 EveOneCat 各后端服务的 MongoDB Collection 结构、索引定义和字段映射关系。

## 目录

- [1. catgoods 服务](#1-catgoods-服务)
- [2. backend_vote 服务](#2-backend_vote-服务)
- [3. backend_whisper 服务](#3-backend_whisper-服务)
- [4. backend_options 服务](#4-backend_options-服务)
- [5. backend_check 服务](#5-backend_check-服务)
- [6. backend_achievement 服务](#6-backend_achievement-服务)
- [7. backend_message 服务](#7-backend_message-服务)
- [8. 共享模型](#8-共享模型)

---

## 1. catgoods 服务

商品、批次、藏品管理服务。

### 1.1 Goods (商品)

**Collection 名称**: `Goods`

**字段定义**:

| 字段 | 类型 | 说明 | 可编辑 |
|------|------|------|--------|
| `_id` | ObjectId | 主键 | 否 |
| `name` | String | 商品名称 | 是 |
| `price` | String | 商品价格 | 是 |
| `cover_url` | String | 封面图片URL | 是 |
| `description` | Object | 商品描述 (用户侧展示) | 是 |
| `metadata` | Object | 商品元数据 (用户侧不展示) | 是 |
| `detail` | Object | 商品详情 (仅单例查询展示) | 是 |

**索引**:
```python
collection.create_index([("name", 1)])
```

---

### 1.2 Goods_Batch (商品批次)

**Collection 名称**: `Goods_Batch`

**字段定义**:

| 字段 | 类型 | 说明 | 可编辑 |
|------|------|------|--------|
| `_id` | ObjectId | 主键 | 否 |
| `batch_key` | String | 批次唯一标识 (格式: EOC + 8位随机字符) | 否 |
| `goods_id` | ObjectId | 关联商品ID | 是 |
| `batch_size` | Integer | 批次大小 | 是 |
| `description` | Object | 批次描述 | 是 |
| `metadata` | Object | 批次元数据 | 是 |
| `is_generated` | Boolean | 是否已生成验证码 | 否 |
| `is_selling` | Boolean | 是否正在销售 | 否 |
| `selling_time` | String | 发售时间 (格式: YYYY-MM-DD) | 否 |
| `created_time` | DateTime | 创建时间 | 否 |
| `updated_time` | DateTime | 更新时间 | 否 |

**索引**:
```python
collection.create_index([("batch_key", 1)], unique=True)
collection.create_index([("goods_id", 1)])
```

**关联关系**:
- `goods_id` -> `Goods._id`

---

### 1.3 Goods_Code (验证码)

**Collection 名称**: `Goods_Code`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `batch_key` | String | 批次标识 |
| `goods_id` | ObjectId | 关联商品ID |
| `collection_id` | ObjectId | 关联藏品ID |
| `code` | String | 验证码 (格式: batch_key + 8位随机 + 8位随机) |
| `access_time` | Integer | 访问次数 |
| `received` | Boolean | 是否已被领取 |
| `receive_id` | ObjectId | 领取用户ID |
| `updated_time` | DateTime | 更新时间 |

**索引**:
```python
collection.create_index([("batch_key", 1)])
collection.create_index([("code", 1)], unique=True)
collection.create_index([("receive_id", 1)])
```

**关联关系**:
- `goods_id` -> `Goods._id`
- `collection_id` -> `Goods_Collection._id`
- `receive_id` -> `User._id`

---

### 1.4 Goods_Collection (藏品定义)

**Collection 名称**: `Goods_Collection`

**字段定义**:

| 字段 | 类型 | 说明 | 可编辑 |
|------|------|------|--------|
| `_id` | ObjectId | 主键 | 否 |
| `name` | String | 藏品名称 | 是 |
| `img_url` | String | 藏品图片URL | 是 |
| `freq` | Integer | 奖池中该实例的数量 | 是 |
| `description` | Object | 藏品描述 | 是 |
| `start_date` | DateTime | 有效开始日期 | 是 |
| `end_date` | DateTime | 有效结束日期 | 是 |
| `created_time` | DateTime | 创建时间 | 否 |
| `updated_time` | DateTime | 更新时间 | 否 |

**索引**:
```python
collection.create_index([("freq", 1)])
collection.create_index([("start_date", 1)])
```

---

### 1.5 Goods_Collection_Cards (藏品卡片实例)

**Collection 名称**: `Goods_Collection_Cards`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `collection_id` | ObjectId | 关联藏品定义ID |
| `index` | Integer | 该藏品的序号 (从1开始自增) |
| `owner_id` | ObjectId | 当前拥有者ID |
| `status` | Integer | 状态: 0=无所属人, 1=拥有态, 2=送出态 |
| `send_from` | ObjectId | 送出人ID (状态2时) |
| `send_to` | ObjectId | 接收人ID (状态2时) |
| `birth_date` | String | 出生日期 (批次发售日期) |
| `owned_date` | String | 拥有日期 |
| `goods_code` | String | 关联的验证码 |
| `created_time` | DateTime | 创建时间 |
| `updated_time` | DateTime | 更新时间 |

**索引**:
```python
collection.create_index([("owner_id", 1)])
collection.create_index([("send_from", 1)])
collection.create_index([("send_to", 1)])
collection.create_index([("collection_id", 1), ("index", -1)])
```

**关联关系**:
- `collection_id` -> `Goods_Collection._id`
- `owner_id` -> `User._id`
- `send_from` -> `User._id`
- `send_to` -> `User._id`

---

### 1.6 Goods_Progress (商品进度)

**Collection 名称**: `Goods_Progress`

**字段定义**:

| 字段 | 类型 | 说明 | 可编辑 |
|------|------|------|--------|
| `_id` | ObjectId | 主键 | 否 |
| `name` | String | 进度名称 | 是 |
| `cover_url` | String | 封面图片URL | 是 |
| `start_time` | String | 开始时间 | 是 |
| `end_time` | String | 结束时间 | 是 |
| `is_show` | Integer | 是否展示: 0=隐藏, 1=显示 | 是 |
| `description` | Object | 进度描述 | 是 |
| `log` | Object | 进度日志 | 是 |

**索引**:
```python
collection.create_index([("name", 1)])
collection.create_index([("is_show", 1), ("_id", -1)])
```

**Redis 键**:
- `goods-progress-banner`: 存储首页置顶的 progress_id

---

### 1.7 Goods_Progress_Advice (进度意见)

**Collection 名称**: `Goods_Progress_Advice`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `progress_id` | ObjectId | 关联进度ID |
| `user_id` | ObjectId | 用户ID |
| `content` | String | 意见内容 |
| `log` | Object | 对话日志 (key为时间戳) |

**log 结构示例**:
```json
{
    "date1704067200000": {
        "role": "user",
        "log_key": "1",
        "content": "用户意见内容"
    },
    "date1704067300000": {
        "role": "cat_sensei",
        "log_key": "1",
        "content": "管理员回复内容"
    }
}
```

**索引**:
```python
collection.create_index([("progress_id", 1), ("user_id", 1)])
collection.create_index([("user_id", 1)])
```

**关联关系**:
- `progress_id` -> `Goods_Progress._id`
- `user_id` -> `User._id`

---

## 2. backend_vote 服务

文章投稿、投票、贴纸管理服务。

### 2.1 Contribute_Article (投稿文章)

**Collection 名称**: `Contribute_Article`

**字段定义**:

| 字段 | 类型 | 说明 | 可编辑 |
|------|------|------|--------|
| `_id` | ObjectId | 主键 | 否 |
| `title` | String | 文章标题 | 是 |
| `title_split` | String | 分词后的标题 (用于全文搜索) | 否 |
| `type` | String | 类型: "video" / "album" | 是 |
| `pics` | Array | 图片URL数组 (album类型) | 是 |
| `author_id` | ObjectId | 作者ID | 否 |
| `vote_num` | Integer | 投票数 | 否 |
| `category` | Integer | 分类: 0=默认 | 是 |
| `featured` | Integer | 精选: 0=否, 1=是 | 否 |
| `sticker` | Object | 贴纸统计 {Unicode: count} | 否 |
| `content` | String | 文章内容 | 是 |
| `cover_url` | String | 封面图片URL | 是 |
| `article_url` | String | 原文链接 | 是 |
| `video_url` | String | 视频URL | 是 |
| `enable` | Boolean | 是否启用 | 否 |
| `retry_times` | Integer | 重试次数 | 否 |
| `created_time` | DateTime | 创建时间 | 否 |
| `updated_time` | DateTime | 更新时间 | 否 |

**sticker 结构示例**:
```json
{
    "U+01F600": 5,
    "U+01F389": 3
}
```

**索引**:
```python
collection.create_index([("author_id", 1), ("_id", -1)])
collection.create_index([("category", 1), ("_id", -1)])
collection.create_index([("vote_num", -1)])
collection.create_index([("category", 1), ("vote_num", -1)])
collection.create_index([("featured", 1), ("_id", -1)])
collection.create_index([("title_split", "text")])
```

**关联关系**:
- `author_id` -> `User._id`

---

### 2.2 Vote (投票用户扩展)

**Collection 名称**: `User` (扩展字段)

**扩展字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `vote` | Array | 投票记录数组 |
| `my_favor_sticker` | Object | 我喜欢的贴纸统计 |
| `contribute_is_banned` | Boolean | 是否被禁止投稿/投票 |

**vote 数组元素结构**:
```json
{
    "time": "2024-01-01",
    "article": ObjectId("...")
}
```

**Redis 键**:
- `token:{token}`: 用户token -> user_id映射
- `contribute-article-sticker:{user_id}`: 用户贴纸记录集合
- `eveonelogin:{user_id}`: 每日登录标记 (TTL: 86400s)

---

### 2.3 Emoji_Map (表情映射)

**Collection 名称**: `Emoji_Map`

**说明**: 存储表情符号与文章的关联关系，用于按表情筛选文章。

---

## 3. backend_whisper 服务

碎碎念、用户信箱管理服务。

### 3.1 Whisper (碎碎念内容)

**Collection 名称**: `Whisper`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `content` | Object | 碎碎念内容 |
| `last_get_time` | String | 最后获取时间 (用于定时展示) |

**content 结构示例**:
```json
{
    "title": "标题",
    "content": "正面内容",
    "img": ["图片URL1", "图片URL2"],
    "background_color": "#ffffff",
    "front_type": "类型",
    "front_html": "自定义HTML",
    "last_get_time": "2024-01-01T12:00",
    "order": 1,
    "reverse": {
        "title": "反面标题",
        "content": "反面内容",
        "background_color": "#000000",
        "img": ["反面图片URL"]
    }
}
```

---

### 3.2 Whisper_Mail (用户信箱)

**Collection 名称**: `Whisper_Mail`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `whisper_id` | ObjectId | 关联碎碎念ID |
| `user_id` | ObjectId | 用户ID |
| `owned_date` | String | 收集日期 (格式: YYYY-MM-DD) |
| `logs` | Array | 对话记录数组 |

**logs 数组元素结构**:
```json
{
    "content": "消息内容",
    "role": "user",  // 或 "cat"
    "date": "2024-01-01 12:00:00"
}
```

**索引**:
```python
collection.create_index([("user_id", 1), ("whisper_id", -1)], unique=True)
collection.create_index([("whisper_id", -1)])
```

**关联关系**:
- `whisper_id` -> `Whisper._id`
- `user_id` -> `User._id`

---

### 3.3 Whisper_Raw (原始碎碎念)

**Collection 名称**: `Whisper_Raw`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `content` | Object | 碎碎念内容 (同 Whisper) |
| `new_id` | String/ObjectId | 关联的新版 Whisper ID |
| `created_date` | String | 创建日期 |

---

## 4. backend_options 服务

配置管理、推送服务。

### 4.1 Option_Global (全局配置)

**Collection 名称**: `Option_Global`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `app` | String | 应用标识 |
| `unit` | String | 配置单元 |
| `content` | Object | 配置内容 |

**索引**:
```python
collection.create_index([("app", 1), ("unit", 1)], unique=True)
```

**常用配置**:
- `app="notify", unit="setting"`: 推送配置
- `app="statistics", unit="{name}"`: 统计查询配置

**推送配置示例**:
```json
{
    "push": {
        "serverjiang": ["token1", "token2"],
        "bark": ["token1"],
        "sms": [
            {
                "appid": "108875",
                "signature": "xxx",
                "to": "19800000000",
                "project": "OoyYw"
            }
        ]
    }
}
```

**统计配置示例**:
```json
{
    "global_vars": {},
    "queries": [
        {
            "name": "today_collect",
            "collection": "Whisper_Mail",
            "aggregate": true,
            "pipeline": [
                {"$match": {"owned_date": "{{today}}"}}
            ]
        }
    ],
    "template": "今日收集数：{{today_collect.count}}"
}
```

---

### 4.2 Option_User (用户配置)

**Collection 名称**: `Option_User`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `app` | String | 应用标识 |
| `unit` | String | 配置单元 |
| `user_id` | ObjectId | 用户ID |
| `content` | Object | 配置内容 |

**关联关系**:
- `user_id` -> `User._id`

---

## 5. backend_check 服务

投票活动管理服务。

### 5.1 Catcheck (投票活动)

**Collection 名称**: `Catcheck`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `title` | String | 活动标题 |
| `type` | String | 类型: "multi" (多选) |
| `detail` | Object | 活动详情 |
| `items` | Object | 投票选项 {key: {name, img, count}} |
| `is_enabled` | Boolean | 是否启用 |

**items 结构示例**:
```json
{
    "option1": {
        "name": "选项1名称",
        "img": "图片URL",
        "count": 10
    },
    "option2": {
        "name": "选项2名称",
        "img": "图片URL",
        "count": 5
    }
}
```

**索引**:
```python
collection.create_index([("type", 1)])
collection.create_index([("is_enabled", 1)])
```

---

### 5.2 Catcheck_history (投票历史)

**Collection 名称**: `Catcheck_history`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `check_id` | ObjectId | 关联活动ID |
| `attend_id` | ObjectId | 参与用户ID |
| `attend_item` | Array | 投票选项列表 |
| `created_time` | String | 投票时间 (格式: YYYY-MM-DD) |

**关联关系**:
- `check_id` -> `Catcheck._id`
- `attend_id` -> `User._id`

---

## 6. backend_achievement 服务

成就管理服务。

### 6.1 Achievement (成就定义)

**Collection 名称**: `Achievement`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `key` | String | 成就唯一标识 |
| `name` | String | 成就名称 |
| `imgurl` | String | 成就图片URL |
| `condition` | Array | 获取条件 |
| `next` | Array | 后继成就 key 列表 |

**condition 结构示例**:
```json
[
    {"have": ["achievement_key1", "achievement_key2"]}
]
```

**索引**:
```python
collection.create_index([("key", 1)], unique=True)
```

---

### 6.2 Achievement_histroy (成就历史)

**Collection 名称**: `Achievement_histroy`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `key` | String | 成就 key |
| `achievement_id` | ObjectId | 关联成就ID |
| `user_id` | ObjectId | 用户ID |
| `time` | String | 获取时间 (格式: YYYY-MM-DD) |

**索引**:
```python
collection.create_index([("user_id", 1), ("achievement_id", 1)], unique=True)
collection.create_index([("achievement_id", -1)])
```

**关联关系**:
- `achievement_id` -> `Achievement._id`
- `user_id` -> `User._id`

**Redis 键**:
- `achivement-async`: 成就处理队列
- `notify_async:{user_id}`: 用户通知队列

---

## 7. backend_message 服务

消息通知服务。

### 7.1 Message (消息)

**Collection 名称**: `Message`

**字段定义**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `from_id` | ObjectId | 发送者ID |
| `to_id` | ObjectId | 接收者ID |
| `ref_app` | String | 关联应用 (如 "Contribute_Article") |
| `ref_id` | String | 关联对象ID |
| `content` | Object | 消息内容 |
| `created_time` | DateTime | 创建时间 |

**关联关系**:
- `from_id` -> `User._id`
- `to_id` -> `User._id`

---

## 8. 共享模型

### 8.1 User (用户)

**Collection 名称**: `User`

**基础字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `_id` | ObjectId | 主键 |
| `nickname` | String | 昵称 |
| `avatar` | String | 头像URL |
| `is_manager` | Boolean | 是否管理员 |
| `mobile` | String | 手机号 |

**扩展字段 (由各服务添加)**:
- `vote`: 投票记录 (backend_vote)
- `my_favor_sticker`: 贴纸统计 (backend_vote)
- `contribute_is_banned`: 是否被禁止 (backend_vote)
- `achievement`: 成就记录 (已废弃，迁移至 Achievement_histroy)

---

## 数据关系图

```
User
  ├── Contribute_Article (author_id)
  ├── Vote (扩展字段)
  ├── Whisper_Mail (user_id)
  ├── Catcheck_history (attend_id)
  ├── Achievement_histroy (user_id)
  ├── Goods_Code (receive_id)
  ├── Goods_Collection_Cards (owner_id)
  ├── Goods_Progress_Advice (user_id)
  ├── Option_User (user_id)
  └── Message (from_id, to_id)

Goods
  └── Goods_Batch (goods_id)
      └── Goods_Code (batch_key)
          └── Goods_Collection (collection_id)
              └── Goods_Collection_Cards (collection_id)

Goods_Progress
  └── Goods_Progress_Advice (progress_id)

Whisper
  └── Whisper_Mail (whisper_id)

Catcheck
  └── Catcheck_history (check_id)

Achievement
  └── Achievement_histroy (achievement_id)
```

---

## 索引汇总

| Collection | 索引 | 类型 |
|------------|------|------|
| Goods | `name` | 普通 |
| Goods_Batch | `batch_key` | 唯一 |
| Goods_Batch | `goods_id` | 普通 |
| Goods_Code | `batch_key` | 普通 |
| Goods_Code | `code` | 唯一 |
| Goods_Code | `receive_id` | 普通 |
| Goods_Collection | `freq` | 普通 |
| Goods_Collection | `start_date` | 普通 |
| Goods_Collection_Cards | `owner_id` | 普通 |
| Goods_Collection_Cards | `send_from` | 普通 |
| Goods_Collection_Cards | `send_to` | 普通 |
| Goods_Collection_Cards | `collection_id, index` | 复合 |
| Goods_Progress | `name` | 普通 |
| Goods_Progress | `is_show, _id` | 复合 |
| Goods_Progress_Advice | `progress_id, user_id` | 复合 |
| Goods_Progress_Advice | `user_id` | 普通 |
| Contribute_Article | `author_id, _id` | 复合 |
| Contribute_Article | `category, _id` | 复合 |
| Contribute_Article | `vote_num` | 普通 |
| Contribute_Article | `category, vote_num` | 复合 |
| Contribute_Article | `featured, _id` | 复合 |
| Contribute_Article | `title_split` | 文本 |
| Whisper_Mail | `user_id, whisper_id` | 唯一复合 |
| Whisper_Mail | `whisper_id` | 普通 |
| Option_Global | `app, unit` | 唯一复合 |
| Catcheck | `type` | 普通 |
| Catcheck | `is_enabled` | 普通 |
| Achievement | `key` | 唯一 |
| Achievement_histroy | `user_id, achievement_id` | 唯一复合 |
| Achievement_histroy | `achievement_id` | 普通 |

