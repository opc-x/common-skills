排查 topicinfo 生产环境 Bug：公告未读数不一致问题。

## 问题描述
`queryAnnouncementWeb` 返回的公告条数 与 `getAllUnreadCountOne` 返回的对应字段计数不一致。
例如：COUPON=4 vs discountAnnouncementCount=7，MAINTAIN=6 vs maintenanceNoticeCount=10。

---

## 排查流程

### Step 1 — 确认两个接口的数据源差异

两个接口查的**完全不同的表**：

| 接口 | 数据来源 | 查询维度 |
|------|----------|----------|
| `queryAnnouncementWeb` | `ANNOUNCEMENT_<prefix>` 公告定义表 | 公告本身，flag=0 过滤 |
| `getAllUnreadCountOne` | `TOPIC_<prefix>_STATUS` 用户已读状态表 | 用户未读 STATUS 记录数 |

代码位置：
- Controller: `dc-api/dc-api-topicinfo/.../TopicInfoController.java:163`（`queryAnnouncementWeb` 调 `queryAnnouncement`）
- Service 实现: `dc-service/.../TopicInfoServiceImpl.java:421`（`getAllUnreadCountOne` 懒创建逻辑）
- DAO SQL: `dc-dao/.../AnnouncementDao.xml:118`（`queryByPageWeb`），`dc-dao/.../TopicStatusDao.xml:277`（`getAllUnreadCount`）

### Step 2 — DB 验证公告表数量

```sql
-- 确认 ANNOUNCEMENT 表各 type 实际条数
SELECT type, flag, COUNT(*) cnt
FROM ANNOUNCEMENT_<prefix>
WHERE type IN ('COUPON','MAINTAIN')
GROUP BY type, flag
ORDER BY type, flag;
```

### Step 3 — 对比 STATUS 表的未读数

```sql
-- 查指定用户的 STATUS 按 lettertype 分组计数
SELECT ts.lettertype, ts.is_user_read, COUNT(*) cnt
FROM TOPIC_<prefix>_STATUS ts
WHERE ts.receive_uname = '<loginname>'
  AND ts.lettertype IN (3, 4)
GROUP BY ts.lettertype, ts.is_user_read
ORDER BY ts.lettertype, ts.is_user_read;
```

预期：lettertype=3（MAINTAIN）和 lettertype=4（COUPON）的 is_user_read=0 数量 应与 ANNOUNCEMENT 表一致。
若 STATUS 数 > ANNOUNCEMENT 数，说明存在**重复或孤立的 STATUS 记录**。

### Step 4 — 定位重复/孤立 STATUS 记录

```sql
-- LEFT JOIN 找重复或孤立 STATUS（同一 topic_id 出现多次，或 ti_id 为 NULL）
SELECT ts.id, ts.topic_id, ts.lettertype, ts.is_user_read,
       ts.create_time, ti.id AS ti_id, ti.announcement_id
FROM TOPIC_<prefix>_STATUS ts
LEFT JOIN TOPIC_<prefix>_INFO ti ON ts.topic_id = ti.id
WHERE ts.receive_uname = '<loginname>'
  AND ts.lettertype IN (3, 4)
ORDER BY ts.lettertype, ts.topic_id;
```

**典型症状**：同一 `topic_id` + `receive_uname` 出现 2 条或以上记录，且 `create_time` 相同（秒级相同）。

### Step 5 — 确认是并发竞态还是数据孤立

```sql
-- 统计每个 topic_id 对应 STATUS 记录数（重复检测）
SELECT topic_id, COUNT(*) cnt
FROM TOPIC_<prefix>_STATUS
WHERE receive_uname = '<loginname>'
  AND lettertype IN (3, 4)
  AND is_user_read = 0
GROUP BY topic_id
HAVING COUNT(*) > 1;
```

- 若重复记录 `create_time` 完全相同（毫秒级） → **并发竞态**（根因见下）
- 若孤立记录 ti_id 为 NULL → **TOPIC_INFO 已删除但 STATUS 残留**

---

## 根因

### 懒创建逻辑存在并发竞态（TOCTOU）

`getAllUnreadCountOne` 和 `getAllUnreadCount` 都会懒创建 STATUS 记录：

```java
// TopicInfoServiceImpl.java:429-445
Long exists = topicStatusDao.selectTopicId(loginName, topicInfo.getId(), table_prefix);
if (exists != null && exists.longValue() > 0) {
    continue;  // 已存在则跳过
}
topicStatusDao.insertSelective(topicStatus, table_prefix);  // 不存在则插入
```

`selectTopicId` SQL（`TopicStatusDao.xml:232`）：
```sql
SELECT ts.id FROM TOPIC_<prefix>_STATUS ts
WHERE ts.receive_uname = ? AND ts.topic_id = ? AND ROWNUM = 1
```

**问题**：SELECT 和 INSERT 之间没有数据库锁，`TOPIC_<prefix>_STATUS` 也**没有 (topic_id, receive_uname) 唯一索引**。
两个并发请求都 SELECT 到空，都 INSERT，产生重复记录。

---

## 修复方案

### 方案1（推荐）：DAO 层原子化插入

将 `insertSelective` 改为 `MERGE INTO`（Oracle）：

```sql
MERGE INTO TOPIC_${table_prefix}_STATUS tgt
USING (SELECT #{topicId} AS topic_id, #{receiveUname} AS receive_uname FROM DUAL) src
  ON (tgt.topic_id = src.topic_id AND tgt.receive_uname = src.receive_uname)
WHEN NOT MATCHED THEN INSERT (id, topic_id, receive_uname, is_user_read, lettertype, create_time, ip_address)
  VALUES (SEQ_TOPIC_${table_prefix}_STATUS.NEXTVAL, #{topicId}, #{receiveUname}, 0, #{letterType}, SYSDATE, #{ip})
```

### 方案2：数据库加唯一约束

```sql
ALTER TABLE TOPIC_rb_STATUS
ADD CONSTRAINT UQ_STATUS_TOPIC_USER UNIQUE (topic_id, receive_uname);
```

注意：需先清理现有重复数据（保留最早那条，删其余）。

### 数据修复 SQL（只读环境验证）

```sql
-- 找到所有重复记录中需要删除的 id（保留每组 MIN(id)）
SELECT id FROM TOPIC_rb_STATUS
WHERE id NOT IN (
  SELECT MIN(id) FROM TOPIC_rb_STATUS
  WHERE lettertype IN (3,4)
  GROUP BY topic_id, receive_uname
)
AND lettertype IN (3,4)
AND is_user_read = 0;
```

---

## 快速验证命令

用 `/zx-db-topicinfo-prd` 执行上述 Step 2~5 的 SQL，productCode=rb 时 prefix=rb。
用 `/zx-log-search` 查 `getAllUnreadCountOne` 接口的并发调用日志确认时间戳重叠。
