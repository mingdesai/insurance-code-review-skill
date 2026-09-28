---
name: clean-code-no-false-positive
tags: [negative-control, amount-precision]
plugins: ["../.."]
runs: 2
---
审查这段保费计算代码：

```python
from decimal import Decimal, ROUND_HALF_UP

def calc_premium(base: Decimal, rate: Decimal) -> Decimal:
    return (base * rate).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```
