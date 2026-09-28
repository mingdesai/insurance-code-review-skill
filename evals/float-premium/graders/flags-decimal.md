---
type: llm
criteria: |
  评审必须指出用 float 参与金额/保费计算是高危（阻断）问题，应改用 Decimal；
  并指出 round() 舍入方向不受控、应用 quantize + 显式 rounding。
  识别出 float 问题并建议 Decimal 即算通过。
---
