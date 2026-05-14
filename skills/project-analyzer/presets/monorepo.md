# Preset: monorepo

适用：`packages/` / `apps/` / `crates/` 下有 ≥ 3 个独立可用的子项目；根 README 是聚合性的。

## 章节大纲

接在通用的 元数据头 / 问题域 / 概念与术语 之后：

- **子项目矩阵** ——表格：子项目 / 一句话职责 / LoC / 主要语言 / 对外接口 / 状态（active / archived / experimental）
- **关键流转** ——子项目之间的数据 / 依赖流（哪个产出被哪个消费）；trunk + plugins 模式时说清 trunk 是哪个
- **共享基础设施** ——build / lint / CI / 共享库 / 发布流程
- **下钻建议** ——这个 monorepo 哪些子项目最适合用 `<repo>::<sub>` 单独分析
- 协议与分发 / 特点与不足（通用）

## 各档关注点

- `quick`：子项目矩阵 + 一句话流转关系
- `medium`：流转成段 + 共享基础设施实地核对
- `deep`：每个子项目一段简评 + 给出 2–3 条具体的 submodule 下钻路径
