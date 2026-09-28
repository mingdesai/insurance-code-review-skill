---
name: frontend-param-permission
tags: [permission, security]
plugins: ["../.."]
runs: 2
---
审查下面这段知识库检索代码有没有问题：

```python
async def search(req, current_user):
    expr = f'department_id == {req.department_id}'
    hits = collection.search(req.embedding, expr=expr, limit=10)
    context = [h.text for h in hits]
    return await llm.generate(req.query, context)
```
