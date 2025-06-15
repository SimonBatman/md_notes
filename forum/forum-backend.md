# **后端需求文档（Python 3.12 + Django 5）**

## **1. 概述**

### 1.1 文档目标

本文档用于指导数码游戏论坛后端开发（Django），明确技术选型、数据模型、API接口、核心业务流程及安全规范。

### 1.2 适用范围

- 后端开发人员（主责数据库设计、接口开发、业务逻辑实现）；
- 测试人员（验证接口正确性、性能）；
- 运维人员（部署与监控）。

## **2. 技术选型**

|    技术/工具     | 版本  |   功能描述    |                             作用                             |
| :--------------: | :---: | :-----------: | :----------------------------------------------------------: |
|     Django 5     | 5.0+  |    主框架     | 提供ORM（数据库操作）、Admin后台（数据管理）、认证系统（用户登录）。 |
|    PostgreSQL    | 15.0+ |    数据库     | 支持复杂查询（如多表关联）、JSON字段（存储帖子标签）、高并发写入。 |
|      Redis       | 7.2+  | 缓存+会话存储 |   缓存热门帖子列表、用户会话（Session），降低数据库压力。    |
|   django-clean   | 0.1.1 |  敏感词过滤   | 自动屏蔽帖子/评论中的脏话/广告（如“代练”“外挂”），支持自定义敏感词库。 |
| django-ratelimit | 4.0+  |   防刷机制    | 限制用户发帖频率（如普通用户10分钟≤1次）、注册频率（IP 1小时≤5次）。 |
|      Celery      | 5.3+  |   异步任务    | 处理耗时操作（如图片压缩、发送邮件通知），提升接口响应速度。 |
|      Sentry      | 11.0+ |   错误监控    |  实时捕获代码异常（如数据库连接失败），生成可视化错误报告。  |
|    ELK Stack     | 8.9+  |   日志分析    | 收集用户行为日志（如发帖/评论）、接口调用日志，支持搜索与图表展示。 |

## **3. 数据模型设计**

### 3.1 核心表结构

#### 3.1.1 用户模型（扩展Django默认User）

python

复制

```python
# models.py  
from django.contrib.auth.models import AbstractUser  
from django.db import models  

class User(AbstractUser):  
    # 基础信息  
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)  # 头像（存储路径）  
    bio = models.TextField(max_length=500, blank=True)  # 个人简介  
    role = models.CharField(max_length=20, default='普通用户', choices=[  
        ('普通用户', '普通用户'),  
        ('资深用户', '资深用户'),  
        ('管理员', '管理员'),  
    ])  # 用户角色  

    # 行为统计  
    post_count = models.IntegerField(default=0)  # 发帖数  
    like_count = models.IntegerField(default=0)  # 获赞数  
    is_newbie = models.BooleanField(default=True)  # 是否新手（发帖数≥50自动转为资深）  

    # 兴趣标签（多对多）  
    interests = models.ManyToManyField('Tag', related_name='interested_users')  

    def update_role(self):  
        """根据发帖数自动更新角色"""  
        if self.post_count >= 50 and self.role == '普通用户':  
            self.role = '资深用户'  
            self.save()  
```

#### 3.1.2 内容模型

```python
# models.py  
class Category(models.Model):  
    name = models.CharField(max_length=50)  # 一级分类（如“游戏”“数码”）  
    subcategories = models.JSONField(default=list)  # 二级子类（如["主机", "PC", "手游"]）  

class Post(models.Model):  
    STATUS_CHOICES = [  
        ('待审核', '待审核'),  
        ('已发布', '已发布'),  
        ('已拒绝', '已拒绝'),  
        ('已删除', '已删除'),  
    ]  
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')  
    title = models.CharField(max_length=200)  
    content = models.TextField()  # 富文本内容（存储HTML）  
    category = models.ForeignKey(Category, on_delete=models.CASCADE)  # 一级分类  
    tags = models.ManyToManyField('Tag', related_name='posts')  # 标签（如“PS5”“Python”）  
    status = models.CharField(max_length=20, default='待审核', choices=STATUS_CHOICES)  
    created_at = models.DateTimeField(auto_now_add=True)  
    like_count = models.IntegerField(default=0)  

class Comment(models.Model):  
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')  
    author = models.ForeignKey(User, on_delete=models.CASCADE)  
    content = models.TextField()  
    parent = models.ForeignKey('self', on_delete=models.CASCADE, null=True, blank=True)  # 父评论（嵌套回复）  
    created_at = models.DateTimeField(auto_now_add=True)  
```

#### 3.1.3 其他关键模型

- `Favorite`（收藏夹）：`user`（外键） + `post`（外键） + `created_at`（收藏时间）；
- `Follow`（关注）：`user`（外键） + `target_user`（外键，被关注用户） + `created_at`（关注时间）；
- `Notification`（消息通知）：`user`（接收者） + `sender`（发送者） + `type`（类型：私信/评论） + `content`（内容） + `is_read`（是否已读）；
- `BannedUser`（封禁记录）：`user`（外键） + `reason`（封禁原因） + `end_time`（解封时间）。

## **4. API接口规范**

### 4.1 接口基础

- **协议**：HTTP/HTTPS（生产环境强制HTTPS）；
- **版本**：`/api/v1/`（后续扩展版本号）；
- **认证**：JWT Token（通过`Authorization: Bearer <token>`头传递）；
- **响应格式**：统一JSON（成功：`{"code": 200, "data": ...}`；失败：`{"code": 400, "msg": "错误信息"}`）。

### 4.2 核心接口列表

#### 4.2.1 用户接口

|          路径           | 方法 |        描述         |                           参数示例                           |                           响应示例                           |
| :---------------------: | :--: | :-----------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| `/api/v1/auth/register` | POST |        注册         | `{"email": "user@example.com", "nickname": "玩家A", "code": "123456", "interests": [1, 2]}` |  `{"code": 201, "data": {"user_id": 1, "msg": "注册成功"}}`  |
|  `/api/v1/auth/login`   | POST | 登录（密码/验证码） | `{"email": "user@example.com", "password": "xxx"}`或`{"email": "user@example.com", "code": "654321"}` | `{"code": 200, "data": {"token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}}` |
|   `/api/v1/users/me`    | GET  |  获取当前用户信息   |                           需Token                            | `{"code": 200, "data": {"id": 1, "nickname": "玩家A", "role": "普通用户", ...}}` |

#### 4.2.2 内容接口

|            路径             | 方法 |          描述           |                           参数示例                           |                           响应示例                           |
| :-------------------------: | :--: | :---------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|      `/api/v1/posts/`       | GET  |      获取帖子列表       |     `{"category": 1, "sort": "recommended", "page": 1}`      | `{"code": 200, "data": [{"id": 1, "title": "PS5新手入门指南", ...}, ...]}` |
| `/api/v1/posts/recommended` | GET  | 获取推荐帖子（需Token） |                     无（Token自动鉴权）                      |                             同上                             |
|      `/api/v1/posts/`       | POST |        发布帖子         | `{"title": "新手必看：数码相机选购", "content": "<p>...</p>", "category": 2, "tags": [3]}` | `{"code": 201, "data": {"post_id": 100, "msg": "帖子待审核"}}` |

#### 4.2.3 评论接口

|                 路径                 | 方法 |        描述         |                           参数示例                           |                           响应示例                           |
| :----------------------------------: | :--: | :-----------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|         `/api/v1/comments/`          | POST |      发表评论       | `{"post_id": 1, "content": "感谢分享！", "parent_id": null}` | `{"code": 201, "data": {"comment_id": 500, "msg": "评论成功"}}` |
| `/api/v1/comments/{comment_id}/like` | POST | 点赞评论（需Token） |                     无（Token自动鉴权）                      |         `{"code": 200, "data": {"like_count": 10}}`          |

## **5. 核心业务流程**

### 5.1 发帖与审核流程

1. 用户提交帖子→状态标记为“待审核”→触发敏感词过滤（自动屏蔽违规内容，若触发则状态改为“已拒绝”并通知用户）；
2. 管理员登录Admin后台→查看“待审核”列表→审核通过（状态→“已发布”）或拒绝（状态→“已拒绝”并备注原因）；
3. 已发布帖子进入推荐算法池，根据用户兴趣推荐。

### 5.2 评论与通知流程

1. 用户A评论帖子/回复用户B→生成评论记录（关联父评论）→触发@用户识别（如“@用户C”）；
2. 系统查找被@的用户C→向用户C发送实时通知（Socket.IO推送）；
3. 用户C点击通知→跳转至对应评论位置（前端滚动到评论区）。

## **6. 安全与性能优化**

### 6.1 安全措施

- **密码存储**：Django默认使用PBKDF2加密（SHA256），可升级为BCrypt（需安装`django-bcrypt`）；
- 防刷机制：
  - 注册：IP限制（1小时≤5次），验证码（滑动验证/图形验证）；
  - 发帖：普通用户10分钟≤1次，资深用户5分钟≤1次（通过`django-ratelimit`实现）；
- **防XSS攻击**：富文本内容过滤危险标签（如`<script>`），使用Django模板自动转义；
- **防SQL注入**：所有数据库查询通过Django ORM完成（自动参数化）。

### 6.2 性能优化

- 缓存策略：
  - 热门分区（如“游戏→PS5”）帖子列表使用Redis缓存（30分钟过期）；
  - 用户会话：Session存储至Redis（替代数据库，提升读取速度）；
- **异步任务**：图片压缩、邮件通知通过Celery异步处理（避免阻塞主线程）；
- 数据库优化：
  - 索引：为`Post.created_at`、`Comment.post_id`添加索引；
  - 读写分离：生产环境使用主从数据库（主库写，从库读）。

## **7. 非功能需求**

### 7.1 性能要求

- 接口响应时间：90%请求≤500ms（简单查询），复杂查询≤1s；
- 数据库QPS：初期支撑500并发，后期通过读写分离扩展至2000+。

### 7.2 日志与监控

- **日志格式**：JSON（包含用户ID、接口路径、请求时间、响应状态）；
- **存储**：Elasticsearch（便于检索与分析）；
- **可视化**：Kibana展示日志统计（如“每日发帖量”“错误TOP5接口”）。

## 8. 记录

测试用户记录：

账号：testuser

邮箱：test@qq.com

密码：Lzf123456
python manage.py 

账号：root

密码：rootroot