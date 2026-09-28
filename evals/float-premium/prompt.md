---
name: float-premium
tags: [amount-precision]
plugins: ["../.."]
runs: 2
---
审查这段保费试算代码：

```python
def calc_premium(base, rate):
    premium = float(base) * float(rate)
    return round(premium, 2)
```
