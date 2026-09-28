# 多级权限隔离评审清单（组织 / 部门）

评审 Milvus（或其它向量库）检索代码时，按本清单逐条核对。对应设计原则：**Milvus 是候选检索层，权限权威数据在业务系统**。

## 判定速查

| 检查点 | 通过标准 | 命中反模式的严重程度 |
|:---|:---|:---|
| 权限来源 | 可访问范围由服务端从登录身份 + 权限服务算出 | 用前端传参决定权限 → 阻断 |
| 元数据打标 | 入库向量带 tenant_id / org_id / department_id / 可见范围 / 发布状态 / 版本 | 只在文档层粗打标 → 严重 |
| 过滤条件 | 租户必匹配 + 可见范围落在授权集合内 + 仅已发布未撤回版本 | 缺任一条件 → 严重~阻断 |
| 上下级关系 | 由权限服务算出可访问 org/dept ID 集合再传入过滤 | "上级天然可看下级全部" → 阻断 |
| 检索后复核 | chunk_id 回权威目录核对当前权限/发布/撤回 | 无复核 → 严重 |
| 失败策略 | 权限服务不可用 / 范围算不清 → 拒绝或转人工 | 放宽过滤 → 阻断 |

## 反模式与修复

### 反模式 1：前端参数直接决定权限（阻断）

```python
# ❌ 前端传 department_id，后端直接拼进过滤条件
async def search(req):
    expr = f'department_id == {req.department_id}'   # 越权：改个 ID 就能读别的部门
    return collection.search(req.embedding, expr=expr)
```

```python
# ✅ 服务端从登录身份 + 权限服务算可访问范围，前端参数只能缩小
async def search(req, current_user):
    scope = await perm_service.resolve_scope(current_user)   # 权威范围
    allowed_depts = scope.department_ids
    # 前端若传了 department_id，必须是 allowed 的子集，否则忽略或拒绝
    depts = _intersect(req.department_id, allowed_depts) if req.department_id else allowed_depts
    expr = _build_expr(tenant=scope.tenant_id, depts=depts,
                       published=True, versions=scope.active_versions)
    return collection.search(req.embedding, expr=expr)
```

要点：前端传的 org/dept/密级 **只能收窄不能放宽**授权范围。

### 反模式 2：只在文档层粗打标（严重）

同一文档不同片段可能有不同密级或部门可见范围。若入库时只按文档整体打一个 `department_id`，会导致本应受限的片段被召回。

```python
# ✅ 按实际授权给对应片段打标
for chunk in doc.chunks:
    entity = {
        "chunk_id": chunk.id, "document_id": doc.id,
        "tenant_id": doc.tenant_id,
        "org_id": chunk.org_id,                 # 片段级
        "department_id": chunk.department_id,   # 片段级
        "visibility": chunk.visibility,         # 可见范围/密级
        "publish_status": doc.publish_status,   # 已发布/草稿/撤回
        "doc_version": doc.version,
    }
```

### 反模式 3：用上下级关系推断权限（阻断）

```python
# ❌ 认为上级组织的人天然能看所有下级部门
expr = f'org_id >= {user.org_level}'   # 危险假设
```

```python
# ✅ 由权限服务算出用户可访问的 org/dept ID 集合，再传入过滤
scope = await perm_service.resolve_scope(current_user)   # 已考虑跨部门授权、有效期
expr = f'tenant_id == "{scope.tenant_id}" and department_id in {list(scope.department_ids)}'
```

组织/部门有上下级时，可访问集合的计算是权限服务的职责，代码里不能用层级大小硬推。

### 反模式 4：分区替代授权（严重）

按租户分区（partition）是性能/索引手段，**不能替代授权校验**。即使 `partition_name=tenant_x`，仍需完整过滤条件 + 检索后复核。

## 检索后权威复核（必须存在）

```python
# Milvus 返回候选后，回权威目录逐条核对当前状态
hits = collection.search(embedding, expr=expr, limit=k)
allowed = []
for h in hits:
    meta = await catalog.get(h.chunk_id)          # 权威目录/权限服务
    if not await perm_service.can_read(current_user, meta):  # 当前权限
        continue
    if meta.publish_status != "published" or meta.revoked:   # 发布/撤回状态
        continue
    allowed.append(h)
# allowed 才进入重排 → Evidence Pack → 大模型
```

原因：权限或文档状态可能在**索引更新前**已变化。元数据打标负责缩小候选，查询过滤是第一道隔离，权威复核挡住状态变化造成的越权引用。
