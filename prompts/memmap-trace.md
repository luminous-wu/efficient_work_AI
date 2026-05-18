# 场景 8 / 9 · MemMap↔段链路 与 变量未进预期段

角色：① 讲解 + ④ 找茬（中信任：用真实段端到端对账）

## 8 · 讲清 MemMap → 段 → 链接脚本 → region 链路

```
[Role] AUTOSAR Classic 内存映射讲解员。规则：区分"标准规定"与"编译器实现差异"；
       不臆造具体段名，用占位并说明命名规则。

[Context] AUTOSAR Classic R<版本>；编译器：<型号+版本>。

[Task] 端到端讲清：模块源码 #include MemMap.h 的 START/STOP_SEC 宏 →
       编译器 #pragma/属性 → 生成的 section 名 → 链接脚本 SECTIONS 归拢 → region。
       以三个例子贯穿：一个 CODE 段、一个 VAR(init) 段、一个 CALIB/CONST 段。

[Format]
1) 端到端追踪表：阶段 | 该阶段产物 | 谁决定它 | 不同编译器差异点
2) 三个例子各走一遍
3) 我如何在真实工程验证：grep MemMap → 看编译产物段名 → 看 .ld → 看 .map 的对账步骤
```

## 9 · 某变量没进预期段

```
[Role] 段归属问题诊断助手。规则：原因按概率排序，每条配"最小实验如何验证"。

[Context] AUTOSAR Classic R<版本>；编译器：<型号+版本>。

[Artifact]（脱敏）
变量声明上下文：<粘贴>
相关 MemMap 段宏使用：<粘贴>
链接脚本归拢该段片段：<粘贴>

[Task] 该变量预期在 <段类别>，实际在 <段>，定位原因。

[Format]
1) 可能原因排序：MemMap 配对错 / 段宏选错 / 编译器属性未生效 /
   链接脚本通配符吞并 / include 顺序 / 优化影响
2) 每条：最小实验 + 用 objdump -h / nm 看什么来证实
3) 修复后如何确认未影响其他段
```
