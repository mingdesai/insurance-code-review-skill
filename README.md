# insurance-code-review-skill

保险核心业务代码审查 Agent Skill，面向 Python、FastAPI、SQLAlchemy 与 Milvus（向量库），检查金额精度、幂等、状态流转、多级权限隔离与涉密数据安全、检索链路越权兜底、知识库效果验证。Chinese Agent Skill for insurance code review.

## 结构

```
SKILL.md                              # 技能入口：5 个评审维度 + 分级 + 输出模板
references/
  permission-isolation.md            # 组织/部门多级权限隔离（Milvus 打标 + 检索过滤 + 检索后复核）
  classified-docs.md                 # 涉密/非涉密共存的双层安全兜底校验
  kb-validation.md                   # 知识库效果验证（标准化问答对 + 三层指标）
使用说明.md                           # 安装、触发、使用与评测指南
设计文档.md                           # 设计来源：三个安全/验证主题的原始论述
evals/                                # claude plugin eval 评测套件（每个子目录一个用例）
  <case>/prompt.md                    #   输入代码 + 运行配置
  <case>/graders/*.md                 #   断言（tool_used / llm 评分）
```

## 评审维度

1. **金额精度** — 全程 `Decimal`、舍入受控、元/分单位一致、分摊兜差额。
2. **幂等** — 写操作幂等键 + DB 唯一约束兜底，重试/并发安全。
3. **状态流转** — 显式状态机、迁移前置校验、乐观锁防并发双改、终态不可逆。
4. **权限与数据安全** — 权限服务端计算不信前端；检索前过滤 + 检索后权威复核双层；失败即拒绝不放宽。详见 `references/permission-isolation.md`、`references/classified-docs.md`。
5. **知识库效果验证** — 分"检索找对证据 / 回答忠于证据 / 守住业务边界"三层看指标。详见 `references/kb-validation.md`。

## 触发

用户说"审查/评审这段保险业务代码""看下这个理赔/保单/核保接口有没有问题""检查金额计算/幂等/权限过滤""审一下 RAG 检索的权限隔离"，或粘贴相关代码求评审时触发。

## 边界

产出**技术控制层面**的评审结论。文档定密、授权策略、标准答案由业务与安全管理方确定；本 Skill 只评审这些规则在代码里是否落实到位。
