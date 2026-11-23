# Bambu Studio 偶发 0xC0000005 稳定性实验（v0.8.3）

> 实验日期：2026-11-22 20:17–21:09（Asia/Shanghai）
> 实验对象：v0.8.3 合并产物 + Studio 02.09.01.70 CLI
> 约束：未修改 `app/`、未 `git commit`；实验脚本与全部证据只写 `_lab/`；Studio 调用全程串行。

## 0. 结论（TL;DR）

1. **复现了真实的 `0xC0000005`**：同一字节的 v0.8.3 合并产物 `P1_whistle_boomerang.3mf`
   （SHA256 `e82f41c0…dc5`，固定不变）连续第 62 次 `--slice` 时进程崩溃，
   退出码 `0xC0000005`，故障模块 `BambuStudio.dll +0x2187460`，
   崩溃发生在写出 `result.json`/G-code **之前**，并新增 38 MB Studio dump。
   minidump 自解析为 `STATUS_ACCESS_VIOLATION`，异常参数 `[0]=0`（读）、
   `[1]=0x26A00004089`（读一个已释放/未映射的堆地址）→ **UAF 类原生内存错误**。
2. **不是我们的 3MF 结构独有**：
   - 原始源文件 `Whistle140db3mf.3mf` 重复切片 45 次也出现 1 次“成功后挂起”；
   - 全部 6 个“一次只改一个因素”的结构变体（去 `assemble`、去 `cut_information.xml`、
     去缩略图、重新压缩/重排 zip、字节相同改名）各 10/10 全部通过，**不能被结构差异触发**；
   - 同一台机器、同一 Studio 版本（02.09.01.70）的历史 WER 档案中已有 **55 份可读的
     `0xC0000005` 崩溃报告**（另有 115 份因权限不可读），故障模块同样集中在
     `BambuStudio.dll`，且 2026-11-18/19 的事件日志还记录了 `0xC0000008`（无效句柄）与
     `Application Hang`——与“反复切片偶发崩溃/卡死”是同一类现象。
3. **可量化量级**：331 次切片中 1 次崩溃（0.30%）、2 次“成功后挂起”（0.60%），
   合并异常率约 **0.9%（约每 100 次 1 次量级）**；Wilson 95% 置信区间约 0.05%–1.7%。
   导出阶段（31 次）未出现任何异常。
4. **对产品的影响**：当前 `tools/studio_verify` 的 3 次重试**只覆盖“未生成输出”**，
   进程崩溃/超时会直接 `break` 不再重试 → GUI“用 Studio 检查/试切片”会被这 ~1%
   的原生缺陷判为失败。**建议在验证层对“崩溃退出/超时”显式重试一次并保留首次失败证据**
   （Lead 已转 engine-core 实现），UI 文案说明“Studio 偶发崩溃，已重试通过”，
   不承诺修复 Studio，也不因此改动 3MF 结构。

## 1. 环境与方法

| 项 | 值 |
|---|---|
| OS | Windows 10.0.26300，32 逻辑处理器 |
| Bambu Studio | 02.09.01.70（`C:\Program Files\Bambu Studio\bambu-studio.exe`，DLL 时间戳 `6ae9bd54`） |
| 便携 Python | `runtime\python.exe`（3.11.0） |
| 并发状态 | 实验全程存在**用户 GUI 实例**（PID 7736，`* Sink_Drain_Insert - BambuStudio`，带未保存修改）。按 Lead 要求**全程未触碰/未关闭**；所有 CLI 调用与其并发。 |
| CLI 调用 | `--export-3mf <out> <in>` 与 `--slice 0 --outputdir <dir> <in>`，cwd=输出目录，串行、单次 45–60 s 超时；超时先 `taskkill /F /T /PID`，仍存活则 `OpenProcess+TerminateProcess` 兜底并确认进程消失。 |
| 隔离 | 每次运行独立 `run-XX` 输出目录（复刻 GUI 每次校验新建目录的行为）；输入文件 SHA256 每次前后校验，`target_unchanged` 全为 true。 |
| 证据 | 每次记录退出码/十六进制、`result.json`、G-code、`%APPDATA%\BambuStudio\log\*.dmp` 新增（名/大小/时间）、残留进程、stdout/stderr、结果写出到进程退出的 teardown 延迟。逐次记录见 `_lab/data/runs.jsonl`（363 条）。 |

**合并产物生成**：`app.service.PreparedMerge`（唯一入口，未改动）：

| 产物 | 组成 / 目标机型 | 字节 | SHA256 |
|---|---|---|---|
| `P1_whistle_boomerang.3mf`（主基线） | whistle + boomerang，默认 A1 | 496,108 | `e82f41c073e3bc405f42b69a02a0a7489f67760c0080864d6cca47dd8dee7dc5` |
| `P2_boomerang_whistle.3mf` | 反向 base | 495,406 | `b821eef2…` |
| `P3_appearance_whistle.3mf` | 外观件 + whistle → A1 | 531,842 | `a6de3f1d…` |
| `P4_eap_whistle.3mf` | eap外壳(X1C) + whistle → A1 | 1,738,743 | `9f7f12c9…` |
| `P1_exported_by_studio.3mf` | P1 经 Studio `--export-3mf` 回写（含缩略图/slice_info 等） | 521,925 | `51433e23fdce3dea53b9fb26e7890af452741af9ee3f88bd49636b3678b9946e` |
| `whistle_exported_by_studio.3mf` | 源文件 whistle 经 Studio 回写（非我们生成） | — | `4d2ef0eb…` |

> 注：`PreparedMerge` 输出不是字节确定的——每次合并会重新生成随机 `p:UUID`/`part uuid`
> （成员内容除 UUID 外逐字节一致）。因此“同一文件重复切片”实验固定使用上表**同一个文件**，
> 并在每次运行前后校验其 SHA256 未变。

**重复实验分四组**：基线（P1 直接 slice / export / Studio 导出件 slice）、对照（原始源文件、
旧产物、人工合并件）、变体隔离（每个变体只改一个因素，各 10 次）、校验层（`studio_verify.verify` 20 次）。

## 2. 成功率表

### 2.1 逐实验明细（`_lab/data/stability_table.md`）

| 分组 | 实验 | n | 进程正常退出 | 0xC0000005 | 成功后挂起/超时 | 新增 .dmp | 单次耗时中位(s) |
|---|---|---|---|---|---|---|---|
| 基线 | P1 合并产物直接 slice（含 1 次预实验） | 61 | 60 | **1** | 0 | 1 | 3.72 |
| 基线 | P1 合并产物 export | 30 | 30 | 0 | 0 | 0 | 1.05 |
| 基线 | Studio 导出件 slice（P1→export） | 60 | 59 | 0 | **1** | 0 | 3.73 |
| 基线 | Studio 导出件 slice（whistle→export，非我们生成） | 30 | 30 | 0 | 0 | 0 | 2.02 |
| 对照 | 源文件 whistle slice | 45 | 44 | 0 | **1** | 0 | 2.11 |
| 对照 | 源文件 boomerang slice | 45 | 45 | 0 | 0 | 0 | 2.22 |
| 对照 | 旧 v0.7 产物 `Boomerang_Whistle_A1.3mf` slice | 15 | 15 | 0 | 0 | 0 | 3.62 |
| 对照 | 人工合并件 `合并.3mf` slice | 15 | 15 | 0 | 0 | 0 | 3.52 |
| 变体 | V1 去 `model_settings` 的 `<assemble>` | 10 | 10 | 0 | 0 | 0 | 3.72 |
| 变体 | V2 去 `Metadata/cut_information.xml` | 10 | 10 | 0 | 0 | 0 | 3.75 |
| 变体 | V3 去 `cut_information.xml` + `<assemble>`（合并产物本无缩略图） | 10 | 10 | 0 | 0 | 0 | 3.62 |
| 变体 | V4 同内容重新 zip（成员顺序反转 + 压缩方式不同，成员内容逐字节同基线） | 10 | 10 | 0 | 0 | 0 | 3.72 |
| 变体 | V5 字节完全相同副本、仅改名 | 10 | 10 | 0 | 0 | 0 | 3.67 |
| 变体 | V7 Studio 导出件删除 5 张缩略图 PNG（对照导出件基线） | 10 | 10 | 0 | 0 | 0 | 3.48 |
| **合计** | — | **363** | **360** | **1** | **2** | **1** | — |

### 2.2 按“是否我们产物”汇总（只看切片）

| 切片对象 | n | 0xC0000005 崩溃 | 成功后挂起 | 异常率 |
|---|---|---|---|---|
| v0.8.3 合并产物（直接 61 + Studio 导出件 60） | 121 | 1 | 1 | 1.65% |
| 原始源文件（whistle 45 + boomerang 45） | 90 | 0 | 1 | 1.11% |
| 旧/人工产物（v0.7 产物 15 + 人工合并件 15） | 30 | 0 | 0 | 0% |
| 结构变体（V1–V5、V7） | 60 | 0 | 0 | 0% |
| Studio 导出件（whistle 源，非我们生成） | 30 | 0 | 0 | 0% |
| **全部切片** | **331** | **1（0.30%）** | **2（0.60%）** | **0.91%** |
| 导出阶段（P1×31、whistle×1） | 32 | 0 | 0 | 0% |

> 统计口径：单次崩溃点估计 0.30%（Wilson 95% CI ≈ 0.05%–1.7%）；“成功后挂起”0.60%。
> 原始源文件也出现异常 → 该量级不是我们产物特有。

### 2.3 `tools.studio_verify.verify(..., slice=True)` 连续 20 次

- 结果 **20/20 `ok=True`，stage=done**，export/slice 均 `attempts=1`，无重试（20 次内含 20 次真实 export + 20 次真实 slice，约 40 次 Studio 阶段调用）。
- 静态核对代码语义：`_run_studio` 返回非 0 退出码时 `_process_error()` 使流程 **`break`，不会重试**；
  只有“进程正常但没写出输出”的路径才会延迟重试（最多 3 次）。因此**当前实现无法吸收
  本次观测到的崩溃/挂起**，GUI 校验会出现 ~1% 的假失败。
- 证据：`_lab/data/verify_repeat.jsonl`、`_lab/data/verify_repeat_summary.json`。

## 3. 异常事件详情

### 3.1 崩溃 A：`0xC0000005`（唯一一次，已跨证据链确认）

| 项 | 值 |
|---|---|
| 时间 / 实验 | 2026-11-22 20:38:22 / `baseline_p1_slice` run-32（第 62 次切片同一文件） |
| 输入 | `_lab/data/products/P1_whistle_boomerang.3mf`，SHA256 `e82f41c0…dc5`，运行前后未变 |
| 退出码 | `3221225477` = `0xC0000005`（STATUS_ACCESS_VIOLATION），运行 7.3 s（正常 3.5–4 s） |
| 产物 | **无 `result.json`、无 G-code**（崩溃在写出前；运行目录只剩 0 字节 stdout/stderr） |
| 事件日志 | Application Error 1000：`bambu-studio.exe 2.9.1.70`，模块 `BambuStudio.dll`，偏移 `0x2187460` |
| minidump | `bambu-studio.exe.171392.dmp`，38,454,951 B；`STATUS_ACCESS_VIOLATION`，RIP=`BambuStudio.dll+0x2187460`，模块数 110（与 CLI 稳态一致），参数 `[0]=0`（读）、`[1]=0x26A00004089` |
| WER | 新增 `AppCrash_bambu-studio.exe_95c5c848…` 报告：故障模块 `BambuStudio.dll`、异常码 `c0000005`、偏移 `0000000002187460`、110 个已加载模块 |
| 证据路径 | `_lab/data/crashes/baseline_p1_slice_run-32/`（`crash_record.json`、`dmp_analysis.json`、`run-dir/`、`wer/…/Report.wer`）；dump 副本 `_lab/data/dmp_evidence/baseline_p1_slice_run-32__bambu-studio.exe.171392.dmp`（SHA256 `907ae317…ba74`） |

### 3.2 挂起 B：`slice_exported_p1` run-30（Studio 导出件）

- `result.json` 已 `Success.`、`plate_1.gcode` 5.74 MB 已写出（20:31:38），**进程此后 56 s 仍未退出**，
  45 s 超时触发杀进程；`taskkill /F /T /PID` 返回
  *“没有此任务的实例在运行”* 但进程实际存活，最终用 `OpenProcess+TerminateProcess` 兜底才消失。
- 无新增 .dmp；无 WER 事件（是挂起不是崩溃）。
- 说明 GUI 侧看到的“卡住/无响应”和“崩溃”可能是两条不同的原生缺陷路径。

### 3.3 挂起 C：`src_whistle_slice` run-19（**原始源文件**）

- 运行 46.41 s（正常 2.0 s），`result.json` 已 `Success.` + G-code 已写出，
  结果写出后到进程退出延迟 **43.93 s**；无新增 .dmp；同样需 TerminateProcess 兜底。
- **关键对照**：异常发生在原始源文件上，与我们的合并结构无关。

### 3.4 历史 WER 证据（同一台机器、同一 Studio 版本）

`_lab/data/historical_crashes.json`（解析 `C:\ProgramData\Microsoft\Windows\WER\ReportArchive|ReportQueue`）：

- 共 170 个 `bambu-studio.exe` 崩溃报告目录，**55 份当前用户可读**，全部为 `0xC0000005`；
  115 份因权限不可读（已计数，未绕过）。
- 时间窗：2026-11-16T11:01Z – 2026-11-18T16:27Z（本地 11/16 19:01 – 11/19 00:27）。
- 按故障模块：`BambuStudio.dll` 35 份（偏移 `0x16396fe`×10、`0x165b783`×24、`0x16264bb`×1）、
  `msvcrt.dll_unloaded` 15 份（BEX64，偏移 `0x3e990`，模块数 20 → 卸载/退出期）、
  `ntdll.dll` 5 份（偏移 `0x94a8`/`0x971b`）。
- 同期 Application Error 日志还记录 `0xC0000008`（STATUS_INVALID_HANDLE，ntdll+0x16a4fa）
  与一次 `Application Hang`（1002，2026-11-19 01:10）——与本次“退出期挂起/崩溃”形态一致。
- 累计说明：`0xC0000005 @ BambuStudio.dll` 在该版本上**长期、多偏移、会成簇出现**，
  对应此前 QA/用户遇到的“反复切片偶发崩溃”，并非 0.8.3 新引入。

## 4. 归因分析：结构 vs Studio 原生

| 假设 | 证据 | 判定 |
|---|---|---|
| 我们的 3MF 结构触发崩溃 | V1 去 `<assemble>`、V2 去 `cut_information.xml`、V3 两者都去、V4 重压缩/重排 zip、V5 字节相同改名、V7 去缩略图：60 次切片 0 异常；P2/P3/P4 其它组合也正常 | **不成立**（本实验无法用结构差异触发；且崩溃点在一次运行内不可复现） |
| Studio 自身偶发原生缺陷 | ① 同一字节文件 61 次中第 62 次崩溃；② **源文件**也出现成功后挂起；③ 历史 55 份同版本 0xC0000005、多模块/多偏移；④ 崩溃为 UAF 读且发生在 `BambuStudio.dll` 内部 | **成立**（输入只决定“是否可切片”，不决定“是否崩溃”） |
| 环境（并发 GUI / 配置锁 / 残留进程） | 实验全程并发用户 GUI；每次运行前后检查无残留 CLI 进程；运行间隔 0.5–1 s 与 0 s（部分批次）均无系统性差异；导出阶段 0 异常 | 不能排除 GUI 并发是**风险放大项**，但无 GUI 的对照未做（用户数据不可动），不足以定论 |
| 磁盘/输出目录残留 | 每次运行独立目录；挂起案例的结果文件先写出 | 不成立 |

## 5. 可执行缓解建议

1. **验证层对“崩溃退出/超时”重试一次，并保留首次失败证据**（最高优先，Lead 已转 engine-core）：
   - 触发条件：退出码非 0（尤其 `0xC0000005`/`0xC0000409`）或阶段超时；
   - 第一次失败：不删 `run-*` 目录，记录命令、退出码、stdout/stderr、dmp 路径；
   - 重试一次（间隔 1–2 s）；成功则报告 `ok=True`，但**必须**带
     `warnings=["Bambu Studio 首次校验崩溃/超时（0xC0000005），已自动重试通过"]`
     与两次尝试的细节，不能静默当成一次成功；
   - 两次都失败才判失败，并保留两次证据。
2. **超时与杀进程**：单阶段超时 45–60 s 即可（当前 240 s 过长，挂起时会白等 4 分钟）；
   超时后先 `taskkill /F /T /PID`，**必须再用 `OpenProcess+TerminateProcess` 校验兜底**
   （本次实测 `taskkill` 会报“没有此任务的实例在运行”却杀不掉挂起进程），并确认 PID 消失。
3. **UI 文案**：GUI“本地 Studio 校验/试切片”失败时区分三类原因并提示：
   `0xC0000005`/其它退出码 = “Bambu Studio 偶发崩溃（Studio 02.09.01.70 已知不稳定，非本文件错误）”，
   超时 = “Studio 无响应，已终止”，无 `result.json` = “Studio 未产出结果”；
   同时给出证据目录。重试成功时显示“Studio 偶发崩溃，已重试通过”。
4. **不要为规避崩溃改 3MF 结构**：变体实验证明当前最小 sidecar 输出已足够；
   保持“无缩略图/无多余 sidecar”的输出即可。
5. **文档与承诺**：README/交付说明写明“Studio 02.09.01.70 在本机约每 100 次切片出现 1 次
   原生崩溃或退出挂起，工具侧已自动重试；不承诺消除 Studio 原生缺陷”。
6. **打包后 QA 抽样**：0.8.4 发布前建议对最终矩阵做 ≥100 次切片抽样，确认异常量级仍在
   ~1% 以内且重试策略生效（并检查无残留 `bambu-studio` 子进程、用户 GUI 不受影响）。

## 6. 未定位 / 未覆盖

1. **触发条件未知**：异常在 40+ 次连续成功后随机出现，与文件（我们产物/源文件/Studio 导出件）、
   运行序号、间隔均无可辨识的确定关系；现有数据不足以拟合触发条件。
2. **UAF 上游代码**：`BambuStudio.dll` 无符号/无 02.09 对应源码，无法定位 `+0x2187460`
   （以及历史 `+0x16396fe/0x165b783` 等）对应函数；只能确认是读已释放/未映射堆地址。
3. **未覆盖的隔离项**（因 Lead 要求收尾、QA 需 Studio 窗口而停止）：
   - “不同工作目录”（换 workroot 根路径）对照——每次运行本来就是不同 `run-*` 子目录，
     但未做跨根路径比较；
   - “清空 `%APPDATA%\BambuStudio\log` 的 dmp 后重测”；
   - “运行间隔”严格对照（0 s vs 5 s 的成组对比）；
   - “关闭并发 GUI 后重测”（用户 GUI 带未保存修改，按要求不可关闭）；
   - 更大样本（>1000 次）以收紧置信区间，及 export 阶段更多样本（本次 32 次）。
4. **统计力度**：崩溃仅 1 次，点估计 0.30%、95% CI 约 0.05%–1.7%；
   “约每 100–300 次 1 次”比单个精确百分比更稳妥。
5. 115 份历史 WER 报告因权限不可读，历史崩溃总数被低估。

## 7. 复现命令与证据索引

```powershell
$p  = 'C:\Users\laobai\Documents\ChatGPT工作\bambu3mf_v083'
$py = "$p\runtime\python.exe"
$env:PYTHONIOENCODING = 'utf-8'

# 1) 生成合并产物 / 结构变体
& $py "$p\_lab\gen_products.py"
& $py "$p\_lab\make_variants.py"

# 2) 重复实验（串行、短超时、失败证据全保留）
& $py "$p\_lab\run_exp.py" --exp baseline_p1_slice --kind slice `
    --target "$p\_lab\data\products\P1_whistle_boomerang.3mf" --n 30 --sleep 0.5 --timeout 45

# 3) 校验层重复（GUI 同款路径）
& $py "$p\_lab\verify_repeat.py" --target "$p\_lab\data\products\P1_whistle_boomerang.3mf" --n 20

# 4) 汇总成功率表
& $py "$p\_lab\aggregate.py"

# 5) 复核崩溃证据
& $py "$p\_lab\parse_dmp.py" "$p\_lab\data\dmp_evidence\baseline_p1_slice_run-32__bambu-studio.exe.171392.dmp"
& $py "$p\_lab\parse_historical_crashes.py"
```

| 证据 | 路径 |
|---|---|
| 逐次运行记录（363 条，含命令/退出码/result/dmp/残留进程/teardown 延迟） | [`_lab/data/runs.jsonl`](../_lab/data/runs.jsonl) |
| 分实验汇总（JSON + Markdown 表） | [`_lab/data/stability_summary.json`](../_lab/data/stability_summary.json)、[`_lab/data/stability_table.md`](../_lab/data/stability_table.md) |
| **本次崩溃完整证据**（记录/分析/run 目录/WER） | [`_lab/data/crashes/baseline_p1_slice_run-32/`](../_lab/data/crashes/baseline_p1_slice_run-32/) |
| 崩溃 minidump（38,454,951 B） | [`_lab/data/dmp_evidence/`](../_lab/data/dmp_evidence/)（SHA256 `907ae317…ba74`） |
| 历史 WER 崩溃解析（55 份可读） | [`_lab/data/historical_crashes.json`](../_lab/data/historical_crashes.json) |
| 合并产物清单 + 哈希 | [`_lab/data/products_manifest.json`](../_lab/data/products_manifest.json)、[`_lab/data/products/`](../_lab/data/products/) |
| 结构变体清单 + 哈希/成员列表 | [`_lab/data/variants/variants_manifest.json`](../_lab/data/variants/variants_manifest.json) |
| verify 重复 20 次明细 | [`_lab/data/verify_repeat.jsonl`](../_lab/data/verify_repeat.jsonl)、[`_lab/data/verify_repeat_summary.json`](../_lab/data/verify_repeat_summary.json) |
| 实验脚本 | [`_lab/run_exp.py`](../_lab/run_exp.py)、[`_lab/lablib.py`](../_lab/lablib.py)、[`_lab/gen_products.py`](../_lab/gen_products.py)、[`_lab/make_variants.py`](../_lab/make_variants.py)、[`_lab/verify_repeat.py`](../_lab/verify_repeat.py)、[`_lab/parse_dmp.py`](../_lab/parse_dmp.py)、[`_lab/parse_historical_crashes.py`](../_lab/parse_historical_crashes.py)、[`_lab/aggregate.py`](../_lab/aggregate.py)、[`_lab/preserve_crash.py`](../_lab/preserve_crash.py) |

**收尾状态**：实验结束后仅剩用户 GUI 进程 PID 7736；本次启动的 Studio CLI 子进程已全部退出
（2 次挂起均经 `TerminateProcess` 兜底确认消失），无残留。未修改 `app/`、未 `git commit`。
