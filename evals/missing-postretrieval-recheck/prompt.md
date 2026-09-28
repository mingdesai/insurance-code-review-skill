---
name: missing-postretrieval-recheck
tags: [permission, security, rag]
plugins: ["../.."]
runs: 2
---
审查这段检索代码，重点看权限安全：

```python
async def answer(query, user):
    scope = await perm_service.resolve_scope(user)
    expr = _build_expr(tenant=scope.tenant_id, depts=scope.department_ids)
    hits = collection.search(embed(query), expr=expr, limit=10)
    context = [h.text for h in hits]
    return await llm.generate(query, context)
```
