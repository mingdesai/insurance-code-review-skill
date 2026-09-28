---
name: no-idempotency-deduction
tags: [idempotency]
plugins: ["../.."]
runs: 2
---
审查这段扣款代码：

```python
async def deduct(order_no, amount):
    db.execute("INSERT INTO deductions(order_no, amount) VALUES (?,?)", (order_no, amount))
```
