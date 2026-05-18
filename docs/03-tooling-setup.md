# 03 · 工具与环境搭建

目标：把第 01/02 章的方法落到趁手的环境上。原则——**AI 处理文本（脚本、.map、objdump 输出、手册节选），不碰硬件；保密链路默认开启。**

---

## 1. 选型：你需要的不是"一个 AI"，而是三条链路

| 链路 | 用途 | 推荐 | 保密等级 |
|---|---|---|---|
| **A. 联网强模型（学习/原理）** | 概念讲解、跨编译器翻译、写自动化脚本、文档 | 高能力对话/编码助手（如 Claude） | 仅脱敏后的"原理性"问题 |
| **B. 本地离线模型（碰真实片段时）** | 万一要贴接近真实的脚本片段做分析 | Ollama + 代码模型（见下） | 完全离线，数据不出机 |
| **C. 命令行 Agent（自动化闭环）** | 让 AI 直接读 .map/.ld、跑脚本、改脚本并自验 | Claude Code（本环境同款 CLI） | 在你本机/受控环境内 |

> 个人学习为主时，90% 走链路 A（脱敏原理问题），需要贴具体片段时切链路 B，做自动化工具时用链路 C。

---

## 2. 链路 C：Claude Code 用于嵌入式（重点）

链接脚本方向没有"运行 App"可言，但 Claude Code 的价值在于**让 AI 直接接触构建产物并形成"分析→改→验证"闭环**：

它能做：
- 读 `.ld/.lsl/.dld`、`.map`、`readelf -S` / `objdump -h -t` / `nm` 的输出文件
- 写并运行 Python/脚本解析 .map、做预算检查、做 map diff（场景 11）
- 跨文件追踪 MemMap 段宏 → 段名 → 链接脚本归拢（场景 8/9）

它不能做（别指望）：
- 烧录、连调试器、跑目标板——硬件在环始终是你的人工步骤
- 替代厂商链接器/配置器——它生成的脚本必须过真实工具链

### 建议项目结构（给 AI 一个安全的"沙盒视图"）
```
your-analysis-workspace/
  inputs/            # 放脱敏后的 .ld / .map / objdump 文本（手动放，受 .gitignore 保护）
  tools/             # AI 生成并你审过的分析脚本（map_budget.py 等）
  rules/             # 段→region 期望规则、预算阈值（yaml）
  reports/           # 脚本产出
  sanitize.py        # 脱敏脚本（先跑它再把内容放 inputs/）
  CLAUDE.md          # 给 Claude Code 的项目规则（见下）
```

### CLAUDE.md 建议内容（让 AI 默认遵守本方向纪律）
```markdown
# 项目规则（链接脚本分析工作区）
- 这是个人学习/分析工作区，禁止假设任何真实地址；未提供的数值一律用 <<CONFIRM:来源>> 占位。
- 禁止生成"可直接量产"的完整链接脚本；只产出片段+注释+验证步骤。
- 任何内存数值类结论必须附"如何用 .map/readelf 独立验证"。
- inputs/ 内容视为脱敏样本，不得据此推断真实客户/项目/芯片。
- 写分析脚本时：纯标准库优先、附 pytest、对畸形 .map 健壮。
```
> 本仓库根目录可放一份这样的 `CLAUDE.md`；它会被 Claude Code 自动加载为长期约束。

### .gitignore（防真实产物误提交）
```
inputs/
reports/
*.map
*.elf
*.ld.real
sanitize_map.json
```

---

## 3. 链路 B：本地离线模型（处理敏感片段时）

当确实需要分析接近真实的脚本/.map 而不能脱净时，用完全离线方案：

```bash
# 安装 Ollama（本地推理，数据不出机）
curl -fsSL https://ollama.com/install.sh | sh

# 拉一个适合代码/结构化文本的模型（按你机器显存选规模）
ollama pull qwen2.5-coder      # 代码与结构化文本较强，规模可选
# 或 ollama pull deepseek-coder-v2 / codellama 等

# 交互
ollama run qwen2.5-coder
```
配合本地编辑器插件（如 Continue/VS Code）把本地模型接入 IDE，链接脚本/`.map` 全程不出网。
**取舍**：本地模型能力弱于云端，适合"敏感但相对机械"的任务（解释片段、找明显错位）；复杂推理仍建议脱敏后走链路 A。

---

## 4. 把构建产物喂给 AI 的标准化脚本

AI 看 `.map` 原文常被格式干扰。先用小脚本归一化再喂，效率与准确率都高。让 AI 自己（场景 11）生成下面这些，你审核后固化到 `tools/`：

| 脚本 | 作用 | 喂给 AI 的产物 |
|---|---|---|
| `dump_layout.sh` | `readelf -S` + `objdump -h` + `nm --size-sort -S` 汇总成一个文本 | 段属性/大小一览，问场景 1/9 |
| `map_summary.py` | 解析 .map → 各 region 用量表 + 关键段地址表（CSV/Markdown） | 干净表格，问场景 5/11 |
| `map_diff.py` | 两个 .map 的段地址/大小漂移 | 变更影响，问场景 12 评审 |
| `sanitize.py` | 客户名/地址/私有符号 → 占位符 + 本地还原映射表 | 脱敏，链路 A 前必跑 |

`sanitize.py` 最小骨架思路（让 AI 据此扩展）：
```
读入文本 → 按规则表（正则: 0x[0-9A-Fa-f]{6,} -> 0xADDR_{n}；
客户/项目关键词 -> CUST/PROJ；白名单外的私有符号 -> SYM_{n}）替换
→ 输出脱敏文本 + 写本地 mapping.json（仅本机，gitignore）供你还原
```

---

## 5. 一次性环境检查清单

- [ ] 链路 A：选定一个强对话/编码助手，已练熟"四要素+禁猜+脱敏"提示。
- [ ] 链路 B：Ollama + 一个代码模型已装、能离线跑。
- [ ] 链路 C：Claude Code 可用，工作区结构 + `CLAUDE.md` + `.gitignore` 就位。
- [ ] `sanitize.py` 可用，并养成"贴前必跑"。
- [ ] `dump_layout.sh` / `map_summary.py` 至少跑通一份真实（本地）样本。
- [ ] 个人 `verify-checklist.md`（第 01 章第 4 节）落地为可勾选文件。

> 环境的目的不是"工具多"，而是**让"脱敏→喂规整文本→AI 出草稿→工具自验"成为零摩擦的默认路径**。
