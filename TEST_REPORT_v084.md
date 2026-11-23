# Bambu 3MF 工坊 v0.8.4 — QA 测试报告（本地 + Studio 回归）

> 生成时间：2026-11-22（Asia/Shanghai）
> 项目根：`C:\Users\laobai\Documents\ChatGPT工作\bambu3mf_v083`
> （v0.8.3 代码基线 + 0.8.4 版本/校验 enforce 收尾；运行时 `service.VERSION = 0.8.4`）
> Bambu Studio：`02.09.01.70`；便携 Python：`runtime\python.exe` 3.11.0（仅标准库）。
> 职责：QA 只改 `tests/**`、`docs/`，未修改 `app/`、`tools/`、`startup.py`。

## 1. 结论摘要

| 阶段 | 命令 | 结果 | 说明 |
|---|---|---|---|
| 基线（改动前，0.8.3） | `QA_STUDIO=0 python -m unittest discover -s tests -v` | **124 项 OK / skipped=15**（65s） | 与 Lead 提供基线一致，可重复 |
| 本地全套（0.8.4，新增 69 项） | `QA_STUDIO=0 python -m unittest discover -s tests -v` | **193 项 OK / skipped=15**（102s） | 178 项实际执行通过；15 项仅因关闭 Studio 而跳过，**不计入通过** |
| 全量现有回归 | 同上 | 原 124 项全部保留通过 | 无因新改动失败 |
| 真实 Studio 回归 | `QA_STUDIO=1 QA_SLICE=1 ...` + `tools/qa_matrix.py` | **193 项 OK / skipped=1**（181s）；矩阵 **8/8** | 见 §7；skip 为“无 blocking 冲突可测”，非 Studio 跳过；无原生崩溃/超时 |

## 2. 环境与基线

```powershell
cd 'C:\Users\laobai\Documents\ChatGPT工作\bambu3mf_v083'
$env:QA_STUDIO='0'; $env:PYTHONIOENCODING='utf-8'
& .\runtime\python.exe -m unittest discover -s tests -v
# 0.8.3 基线：Ran 124 tests ... OK (skipped=15)
# 0.8.4 最终：Ran 193 tests ... OK (skipped=15)
```

- 基线日志：`_recon_v083_baseline.log`
- 0.8.4 本地最终日志：`_recon_v083_local_v084.log`
- 15 项 skip 全部是 `QA_STUDIO=0` 关闭的真实 Studio 校验/切片用例
  （`test_studio_verify` / 集成产物 Studio 校验 / layer_height 切片等）；跳过项不计入通过率。

## 3. 新增测试清单（69 项）

| 文件 | 项数 | 覆盖点 |
|---|---|---|
| `tests/test_file_selection_state.py` | 21 | ①/② 文件框状态与清空语义（UI 层，Tk 隐藏窗口 + FakeBridge + 同步 worker） |
| `tests/test_verification_gate.py` | 13 | `verify_studio` / `verify_slice` 校验失败阻断、accept 不可绕过、报告字段、核心层不写盘、失败后恢复 |
| `tests/test_export_options.py` | 19 | 导出报告格式组合/默认值/去重/非法值、报告内容、防覆盖、`suggest_output_name` 递增、ServiceBridge 透传 |
| `tests/test_param_semantics.py` | 8 | `converted_to_object` 冲突行 `output_value` / `base_value` 语义与输出对象 metadata 一致性（双向 base、object_name 一致性） |
| `tests/test_studio_verify.py`（新增 RetrySemanticsTests） | +8 | Studio 原生崩溃/超时重试语义（伪进程：崩溃/超时/协议错误/三次失败/证据落盘） |

### 3.1 主文件 / 加入文件状态（21 项）

依据 ui-fixer 最终接口（`_base_path` + `_base_index()`、source_tree 仅显示加入文件）：

- **先选主文件才能加加入文件**：空态调用 `add_files()` 不弹对话框、rows 保持空、state=empty、加入按钮禁用。
- **选择主文件**：rows=[主文件]、`_base_path`/`_main_full_name`/`main_name` 正确、加入列表为空、计数“1 个文件”、分析禁用、清空加入禁用、清空全部可用。
- **加入多份文件**：追加到 rows 末尾；加入列表 iid 恒为 rows 索引且不含主文件；计数/按钮状态正确；分析按钮在 ≥2 文件时可用。
- **重复选择同一文件**：`add_files` 与更换主文件均不产生重复行。
- **更换主文件**：旧主文件自动保留为加入文件、新主文件若原在加入列表则从列表移除（文件数不变、无重复）；重选同一路径只刷新摘要、不打乱顺序。
- **set_base_row**：双击加入行切换主文件，旧主文件保留，列表同步。
- **移除**：只移除选中的加入文件；主文件不可被移除（外部误选给出提示且不删）。
- **清空加入文件**：`clear_join_files()` → rows 仅剩主文件、state=ready、列表空、分析禁用；之后仍可继续加入文件；旧别名 `clear_button` 等价于清空加入。
- **清空全部**：`clear_files()` → rows 空、`_base_path=None`、state=empty、计数 0、主文件显示“尚未选择主文件”、全部操作按钮禁用。
- **忙碌态保护**：busy 状态下加文件/清空/换主文件均不执行。
- **失败路径**：更换主文件读取失败 → 保留原列表与主文件、Banner 错误、state 恢复。
- **旧控件已移除**：不存在 `base_combo` / `secondary_files_var` / `_detached_main`。
- **长文件名**：主文件标签可截断，但 `_main_full_name` 与悬停提示保留完整名/路径；加入列表行 tooltip 含完整文件名+路径。
- **analyze 使用当前主文件**：切换主文件后 `bridge.prepare(paths, base_index=...)` 的 base_index/paths 正确。
- **校验选项联动**：勾选“真实切片校验”自动包含回读校验（verify_studio=True）；两者均可独立关闭。

### 3.2 校验失败阻断（13 项）

通过 mock `core.merge.run_studio_verification`（不调用真实 Studio）验证：

- export 失败 / 参数保持失败（stage=preserve）/ 切片失败 / 校验工具不可用 → `status=blocked_conflicts`，`verification.studio_load`、`studio_slice`、`studio_result` 如实填写，冲突中恰好一条 `studio_verification` blocking 行，报告 schema v2 校验通过。
- 任何失败下 `export(destination)`、`export(destination, accept_conflicts=True)`、`options.accept_conflicts=True` **都抛 MergeError 且不写输出**。
- 切片失败不会把已完成的回读改成失败：`studio_load=passed`、`studio_slice=failed`。
- 核心入口 `merge_projects` 校验失败时返回 blocked_conflicts 报告、**不写 3MF**（只写 blocked 报告）；成功时正常写输出。
- 校验成功：可导出，`studio_load=passed`、未请求切片时 `studio_slice=not_run`、请求切片成功时 `passed`。
- 两项校验都关闭：不调用 Studio、均 `not_run`、正常导出。
- 失败被阻断后，取消校验重新分析可恢复正常导出（未验证输出）。
- 连续两次分析（成功/失败）互不影响。

### 3.3 导出选项 / 命名防覆盖（19 项）

- `report_formats=None`（旧脚本不传）→ 仍输出 3MF + `.report.json` + `.report.html` 两份报告。
- `()` → 只 3MF；`('json',)` / `('html',)` → 对应一份；`('json','html')` → 三份；重复格式去重。
- 非法格式、错误后缀 → 中文 MergeError，且不留下 3MF/报告半成品。
- 目标 3MF 已存在、同名报告已存在、目标等于输入 → 拒绝且原文件内容哈希不变。
- 报告内容：JSON `output` 指向目标、`tool_version` 为 `0.8.x`；HTML 为自包含页面。
- `suggest_output_name`：两份文件默认 `主模型+另一个模型_合并.3mf`；单文件/多文件（`等N份`）；非法字符替换；模型或任一报告已存在时递增 `_2`/`_3`；空列表回退 `模型_合并.3mf`。
- `ServiceBridge.export` 将 `report_formats` 透传给 `PreparedMerge.export`。

### 3.4 参数差异语义（8 项）

engine-core 最终冻结口径为“每个实际写入对象覆盖的 (对象, 参数) 一行”：

- `converted_to_object` 行：`source`=来源文件、`object_name`=对象名、`source_value`=源项目解析值、`output_value`=实际写入对象 metadata 的精确字符串、`base_value`=主文件全局值、`note` 说明对象覆盖来源；schema v2 通过。
- 每一行都能在报告 objects 的 `converted` 与输出 3MF 对象 metadata 中找到同值（双向：base=whistle 与 base=boomerang 两种方向）。
- `object_name` 存在时与报告 object 一一对应；同一（source, key, object_name）无重复行；源值与基础值相同的键不产生 converted 行；数值/字符串比较带容差。

### 3.5 Studio 原生崩溃/超时重试语义（8 项，伪进程）

`tools/studio_verify.py` 0.8.4 对**进程级失败**（崩溃退出码、超时、未生成文件）最多尝试 3 次，
协议级失败（进程正常退出但 `result.json` 报项目错误）不重试。测试用 patch 的伪 `_run_studio` 验证：

- **第 1 次崩溃 0xC0000005 → 第 2 次成功**：`ok=True`、`export.attempts=2`、
  `retries=[{stage:'export', attempt:1, exit_hex:'0xC0000005', will_retry:true,...}]`、
  `export.error=None`（错误只保留在 retries）；`result.warnings` /
  `verification.details` 含“Bambu Studio 原生偶发崩溃/超时…已重试 1 次后通过（不代表 Studio 崩溃已修复）”。
- **超时后重试成功**：`retries[0].timed_out=True`，提示“无响应/超时（已杀进程树）”。
- **连续 3 次崩溃**：`ok=False`、`stage=export`、`attempts=3`、`retries` 2 条、
  `error` 含“共尝试 3 次 + 首次 exit_hex(0xC0000005)”；每次 `runs`/日志、`verification.json`、
  `failed-input.3mf`（输入副本）均保留。
- **协议级失败不重试**：`attempts=1`、`retries=[]`、仅 1 次进程调用、错误为“未返回 Success”。
- **干净首次成功**：`attempts=1`、无 retry/无“原生偶发”警告。
- **切片阶段**同样支持“崩溃后重试成功”与“3 次失败”；导出阶段成功结果不被切片失败改写。
- `verification.json` 落盘包含最终 `attempts/retries`，可作为 Studio 原生不稳定证据。

## 4. 现有 124 项回归

- 新增测试未修改既有测试文件；基线与新增后全套结果均为 **OK**。
- 原有 `test_export_choices.py`（v0.8.2 导出格式）与 `test_verification_reliability.py`（校验可靠性/隔离/证据）继续通过。
- engine-core 专项（非本轮 QA 交付）：`_lab/verify` 63 tests OK；内部回读 8 类篡改探针全部检出、converted `output_value` 与对象 metadata 0 不一致（报告 `_lab/verify/REPORT.md`）。
- ui-fixer 自检 81/81、布局自检 8/8 通过。

## 5. 已知问题 / 失败分析

- 本地（`QA_STUDIO=0`）**无失败**。新增测试编写过程中修正了测试自身的两个口径问题：
  1) 默认命名递增测试共享临时目录导致跨 subTest 污染（改为独立临时目录）；
  2) 参数语义最初假设“每来源文件一行”，engine-core 冻结为“每（对象, 参数）一行”；当前断言两种粒度均成立（按 (source,key,object_name) 查找），最终代码下已复跑通过。
- `README.md` 明确 Bambu Studio **偶发 0xC0000005 仍未解决**。studio-lab 稳定性实验
  （`docs/STUDIO_STABILITY.md`，363 次运行：360 正常 / 1 崩溃 / 2 成功后挂起）：
  **331 次切片中 1 次 0xC0000005（0.30%）、2 次“成功后挂起”（0.60%）**，导出阶段 31 次 0 异常；
  崩溃为 `BambuStudio.dll+0x2187460` 的 UAF 类原生错误（38MB dump），且 6 个结构变体各 10/10
  通过，说明**不是 3MF 结构差异触发**。Wilson 95% 置信区间约 0.05%–1.7%。
  因此本地 `QA_STUDIO=0` 的 193 项通过**不等于**含真实 Studio 全绿；Studio 结果单列见 §7。
- **本次 QA Studio 矩阵未遇到原生 0xC0000005/超时**（0 崩溃、0 挂起、0 原生重试）；
  唯一 retry 出现在 M2（真实几何冲突）：Studio 切片以**非零退出码 `0xFFFFFF9B`（-101）**
  拒绝 “G-code conflicts”，verify 重试逻辑把它当作进程级失败共尝试 3 次（retries=2），
  最终错误被包成“共尝试 3 次…原生崩溃/超时”文案，**实际是确定性项目冲突，不应重试/不应叫原生崩溃**。
  这是本次唯一观察到的重试误分类（不影响安全，矩阵仍按“真实几何冲突”归类通过）；
  建议后续把“result.json 已含明确项目错误”视为协议级失败（即使退出码非 0）不再重试。
- UI 已提供 **verify_failed 专用状态**（ui-fixer 81/81 自检）：校验失败时不再提示“接受差异后可导出”，
  导出按钮禁用 + service/core 独立闸门兜底；旧说法（“可能仍提示接受差异”）已不适用。

## 6. 未验证事项

1. **真实 Studio 全矩阵已执行**（§7）；但 Studio 原生偶发崩溃/挂起（0xC0000005）根因未修复，
   只能靠重试掩盖，不能视为“Studio 全绿”。
2. `0xC0000005` 崩溃根因与复现条件（README 明示未解决；实验室统计见 `docs/STUDIO_STABILITY.md`）。
3. GUI 手动验收（对象树/参数差异/打印板预览/保存重开）仍需人工按 `docs/VALIDATION_V08.md` 执行。
4. 多喷嘴机型、真实涂色/MMU、超大文件性能同既有报告限制。
5. 导出“默认文件名”在 UI 端到端点击流程（文件对话框）未自动化，仅验证 `suggest_output_name` 纯函数。

## 7. 真实 Studio 回归（已执行）

环境：Bambu Studio 02.09.01.70；Studio 调用串行、单阶段超时 60s（`QA_STUDIO_TIMEOUT=60`）、
失败自动重试最多 3 次；全程只杀自己启动的 CLI 子进程；**用户 GUI（PID 7736）未触碰**；
结束时仅剩该用户 GUI，无 QA 残留进程。运行期间 studio-lab 已结束，无并发实验。

```powershell
$env:QA_STUDIO='1'; $env:QA_SLICE='1'; $env:QA_STUDIO_TIMEOUT='60'
$env:PYTHONIOENCODING='utf-8'
& .\runtime\python.exe -m unittest discover -s tests -v
# -> Ran 193 tests ... OK (skipped=1)  181.26s
& .\runtime\python.exe tools\qa_matrix.py --out _recon_v083_studio
# -> 结果：8/8 通过
```

### 7.1 全套结果（含真实 Studio）

- **193 项 OK / skipped=1**：唯一的 skip 是 `test_export_without_acceptance_when_required`
  （本组语料无 blocking 冲突可测），**不是 Studio 跳过**；所有 Studio 校验/切片用例均实际执行并通过。
- 真实 Studio 调用中**未出现 0xC0000005、未出现超时、未发生原生重试**；
  所有 export 首次尝试 `attempts=1`、`retries=[]`。

### 7.2 真实语料矩阵（`tools/qa_matrix.py`，8/8）

| 用例 | export | slice | G-code / 说明 |
|---|---|---|---|
| M1 whistle+boomerang | **Success.** rc=0（attempts=1） | **Success.** rc=0（attempts=1） | 214 层 / [0.2, 0.4] |
| M2 eap(X1C→A1)+whistle | **Success.** rc=0 | 真实几何冲突（预期） | Studio 以非零 `0xFFFFFF9B`(-101) 报 G-code conflicts；**已按协议级失败处理（1 次，不重试）**，见 §7.4 |
| M3 外观件(A1 mini/PETG→A1)+whistle | **Success.** rc=0 | **Success.** rc=0 | 874 层 / [0.2, 0.4] |
| M4 旧版 368 键+whistle | **Success.** rc=0 | **Success.** rc=0 | 270 层 / [0.2, 0.4] |
| M5 瓦尔塔第 2 板+whistle | **Success.** rc=0 | 按契约不切片 | — |
| M6 对象 layer_height=0.12 注入 | **Success.** rc=0 | **Success.** rc=0 | **376 层** / [0.04, 0.08, **0.12**, 0.2, 0.4]（214→376，证明对象覆盖被执行） |
| M7 v0.7 变换对照 | — | — | build 变换集合完全一致 |
| M8 人工基准 `合并.3mf` | **Success.** rc=0 | **Success.** rc=0 | 214 层；对象覆盖差异 `[]` |

- 汇总：**export 7/7 Success**（矩阵中 7 个需要 export 的用例；M7 不产生产物）；
  **slice 5/6 Success**（M1/M3/M4/M6/M8 成功；M2 为预期真实几何冲突），M5 按契约不切。
- 导出复核：对象数、机型 A1/0.4、对象/零件 metadata 保留、build 变换逐字符全部保持；
  与 studio-lab 的导出阶段 31/31 无异常结论一致。
- 原始证据：`_recon_v083_studio/<用例>/studio/run-*/result.json`（export）、
  `run-*/slice/result.json` + `plate_1.gcode`（slice）、`verification.json`（完整 attempts/retries/runs/logs）。

### 7.3 Studio 原生不稳定（实验结论引用）

引用 `docs/STUDIO_STABILITY.md`（studio-lab 363 次运行）：

- **331 次切片：1 次 `0xC0000005`（0.30%）、2 次“成功后挂起/超时”（0.60%）**；
  export 阶段 31 次 0 异常；合计异常率约 0.9%（Wilson 95% CI 约 0.05%–1.7%）。
- 崩溃退出码 `0xC0000005`，模块 `BambuStudio.dll+0x2187460`，minidump 解析为
  `STATUS_ACCESS_VIOLATION`（读已释放/未映射堆地址）→ **UAF 类 Studio 原生内存错误**；
  6 个结构变体（去 assemble / 去 cut_information / 去缩略图 / 重新 zip / 同字节改名等）
  各 10/10 通过，源文件 whistle 45 次也有 1 次成功后挂起 → **不是本工具的 3MF 结构触发**。
- 本次 QA 矩阵未触发该原生问题（0 崩溃/0 挂起）；若未来触发，`studio_verify` 会重试最多 3 次、
  保留每次 `runs` 与 `verification.json`，并在 warnings 写明“原生偶发崩溃/超时…已重试 N 次后通过
  （不代表 Studio 崩溃已修复）”。**结论：Studio 原生偶发崩溃/挂起未修复，只能重试兜底。**

### 7.4 重试误分类：已发现并修复

M2 的“G-code conflicts”是确定性项目/几何冲突（report.layout 已提示重叠），但 Studio 以
**非零退出码 `0xFFFFFF9B`(-101)** 退出且照常写出 `result.json`（error_string 为 conflicts）。
旧版 `studio_verify` 把“非零退出”当进程级失败，重试 3 次（attempts=3、retries=2），
文案误写成“原生崩溃/超时”，并多耗约 70s。

**修复（Lead 补丁，已复验）**：若本次 attempt 已写出 `result.json` 且 `error_string/return_code`
明确是项目级错误，按**协议级失败**处理——不重试、不使用原生崩溃文案；只有“无 result.json 的
非零退出/超时”才视为 Studio 原生崩溃/挂起并重试。复验 M2：`slice attempts=1, retries=0`，
错误为“result.json 返回项目错误（协议级失败，不重试）”，耗时 9.1s（原 ~70s）。
本地 193 项测试复跑通过（`_recon_v083_local_afterfix.log`）。

### 7.5 跳过口径

- 含 Studio 全套：193 项，skipped=1（业务条件 skip，非 Studio）；**跳过不计为通过**。
- 本地 `QA_STUDIO=0`：193 项，skipped=15（全部为 Studio 关闭类）；**跳过不计为通过**。

## 8. 证据文件

| 文件 | 内容 |
|---|---|
| `_recon_v083_baseline.log` | 基线（0.8.3）124 项（skipped=15）日志 |
| `_recon_v083_local_v084.log` | 0.8.4 本地全套 193 项（skipped=15）日志 |
| `_recon_v083_local1.log` | 新增测试后中间轮 177 项日志（历史留档） |
| `_recon_v083_studio_suite.log` | **含真实 Studio 全套 193 项（skipped=1）完整日志**（181s） |
| `_recon_v083_studio_matrix.log` | `tools/qa_matrix.py` Studio 矩阵运行日志（8/8） |
| `_recon_v083_studio/matrix_results.json`、`matrix_table.md` | 矩阵汇总证据（export/slice 字段、层数、层高） |
| `_recon_v083_studio/<用例>/studio/run-*/verification.json` | 每例 verify 原始结果（attempts/retries/runs/warnings） |
| `_recon_v083_studio/<用例>/studio/run-*/result.json`、`slice/result.json`、`slice/plate_1.gcode` | Studio 原始 export/slice 证据 |
| `docs/STUDIO_STABILITY.md` | studio-lab 363 次稳定性实验（0.30% 崩溃 / 0.60% 挂起 / 结构变体 10/10） |
| `docs/TEST_REPORT.md` | v0.8.0 既有测试报告（保留，未覆盖） |

## 9. 最终结论

1. **本地（无 Studio）**：`QA_STUDIO=0` → **193 项 OK / skipped=15**（178 项实跑通过）；
   基线 124 项全部保留通过；新增 69 项覆盖主/加入文件状态、校验阻断、导出选项/命名、参数语义、
   Studio 原生崩溃/超时重试语义。**15 项 skip 不计入通过**。
2. **含真实 Studio 全套**：`QA_STUDIO=1 QA_SLICE=1` → **193 项 OK / skipped=1**（181s）；
   唯一 skip 为业务条件（无 blocking 冲突），非 Studio 跳过；真实调用中无 0xC0000005/超时。
3. **真实语料矩阵**：8/8 通过；export 7/7 `Success.`；slice 5/6 `Success.`，
   M2 为**真实几何重叠**拒绝（report.layout 已提示，用户移动对象后可切片），M5 按契约不切片。
   对象级 layer_height=0.12 被切片器真实执行（214→376 层，G-code 出现 0.12）。
4. **Studio 原生稳定性**：实验统计切片 0.30% 崩溃（0xC0000005 UAF）+ 0.60% 成功后挂起，
   结构变体与源文件均可触发 → **非本工具 3MF 结构问题，Studio 原生缺陷未修复**；
   `studio_verify` 以最多 3 次重试 + 完整证据兜底，不掩盖、不宣称修复。
5. **遗留风险**：M2 类“非零退出但 result.json 已报项目错误”被重试误分类（见 §7.4）；
   GUI 手动验收尚未执行；Studio 原生偶发崩溃仍需用户重试。
6. 本报告所有数字以实际运行日志为准（`Ran 193 tests`）；未执行 `git commit`。



