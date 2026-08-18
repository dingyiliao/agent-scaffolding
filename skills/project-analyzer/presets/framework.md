# Preset: framework (框架 / SDK)

适用：库形态但通过 trait / interface / abstract class 强加架构约束；明确暴露扩展点（middleware / plugin / hook）；用户代码需要"塞进"框架的生命周期。

## 章节大纲

接在通用的 元数据头 / 问题域 / 概念与术语 之后：

- **设计哲学** ——核心抽象（最重要的 1–3 个 trait / interface / 概念）；与同类框架（如 axum vs actix、Express vs Fastify）的差异点
- **扩展点矩阵** ——表格：扩展点 / 形式（trait / hook / middleware / decorator）/ 触发时机 / 典型用途
- **内部机制** ——框架如何把扩展点串起来、生命周期管理、错误传播、类型推断或反射的代价
- **典型用法** ——minimal "hello world" + 一个非平凡场景（中间件链、自定义提取器之类）
- 协议与分发 / 特点与不足（通用）

## 各档关注点

- `quick`：列核心抽象 + 扩展点矩阵（一级）
- `medium`：设计哲学成段 + 跟踪一个扩展点从声明到调用的完整路径
- `deep`：内部机制完整、与同类对比成段、覆盖错误传播 / 生命周期边界 case
