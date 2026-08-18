# Preset: protocol (协议实现)

适用：引用 RFC / draft 号；存在 wire format / conformance test；命名含 protocol / spec / codec / wire / frame。

## 章节大纲

接在通用的 元数据头 / 问题域 / 概念与术语 之后：

- **协议定位** ——在协议栈中的位置、上下游、与同类协议的关系（如有）；引用的 RFC / draft
- **wire format** ——报文 / 帧 / 包结构、字段含义、版本编码；尽量给一张 ASCII 字段图或字节布局表
- **实现策略** ——状态机、解析器、序列化策略；手写 parser 还是 codegen；零拷贝 / 内存安全策略
- **互操作 / 一致性** ——与官方测试套件的覆盖、已知互操作问题
- 协议与分发 / 特点与不足（通用）

## 各档关注点

- `quick`：协议定位 + wire format 一级骨架
- `medium`：完整 wire format + 解析器主路径
- `deep`：所有状态转换、错误恢复、互操作矩阵、与参考实现的差异
