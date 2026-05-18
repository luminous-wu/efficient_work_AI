# 提示词库

复制即用的模板，已按"链接脚本 / 内存布局 / AUTOSAR"方向裁剪。

**用前必做**：贴任何脚本/.map 片段前先跑 `sanitize.py`（见 docs/03）。

| 文件 | 对应场景（docs/02） |
|---|---|
| `_skeleton.md` | 通用四要素骨架，所有提示的母版 |
| `explain-linker-script.md` | 场景 1：读懂陌生链接脚本 |
| `cross-compiler-translate.md` | 场景 3：跨编译器/链接器翻译 |
| `add-memory-section.md` | 场景 4：新增段/region（含禁猜+验证步骤） |
| `linker-error-triage.md` | 场景 5：链接器报错排查 |
| `memmap-trace.md` | 场景 8/9：MemMap↔段链路与变量未进预期段 |
| `map-budget-script.md` | 场景 11：生成内存预算守护脚本 |

每个模板里 `<...>` 是你要替换的占位；`<<CONFIRM:来源>>` 是要求 AI 保留的"待你核实"标记，**不要让 AI 填掉它**。
