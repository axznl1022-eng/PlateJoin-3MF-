# Bambu 3MF 工坊 v0.8 — 团队实施契约（唯一事实来源）

> 所有成员动工前必须完整读完本文件。文件路径均相对项目根：
> `C:\Users\laobai\Documents\deepseek\bambu3mf_v08`
>
> 便携 Python：`C:\Users\laobai\Downloads\Bambu3MF_Merger_Windows_v0_7\Bambu3MF_Merger_Windows_v0_7\runtime\python.exe`
> （无第三方包；只用标准库。GUI 用 Tkinter 8.6。）
> 已安装 Bambu Studio：`C:\Program Files\Bambu Studio`（02.09.01.70，含官方预设与中文翻译，可用于真实验证）。
> 源码 zip：`C:\Users\laobai\Downloads\BambuStudio-02.09.01.70.zip`（已抽取关键文件到 `research/source/`）。

## 1. 目标与边界

用户痛点：拓竹单盘一次只能打印一个物体；把第二个模型导入 Bambu Studio 时，
- “打开为项目”→ 覆盖原项目，只剩一份参数；
- “导入几何”→ 丢掉第二个模型的打印参数。

v0.7 已证明：可以按对象把源项目的工艺参数写成对象覆盖，几何/变换原样搬运。
v0.8 在此基础上：

1. **界面重做**：现代化布局、卡片层次、打印板图形预览、可过滤参数差异表、状态流清晰。
2. **自动迁移不同机型**：目标机型可选，机器级参数按官方机台预设整体迁移，床型/喷嘴/打印范围做兼容检查并报告。
3. **兼容性提升**：旧版本项目（字段少）按当前官方缺省值补全；多板输入可选板；多耗材槽合并与引用重映射；跨版本字段处理。
4. **可验证**：内部复核 + 调用已安装 Studio CLI（导出/切片）做自动验收，报告里给出验证等级。

硬性边界（不得违反）：
- 原始输入文件只读；输出绝不覆盖任何现有文件；失败不留半成品。
- 不改动任何模型的 build/component 变换字符串（逐字符保持），不重采样网格。
- 不承诺“物理打印效果完全一致”；报告必须如实标注验证等级。
- 不联网、不连打印机、不自动切片（除非用户显式勾选本地校验）。

## 2. 目录与文件所有权（写作用域）

| 路径 | 负责人 | 说明 |
|---|---|---|
| `app/core/archive.py` | engine-core | 3MF 读写、安全校验、原子写 |
| `app/core/project.py` | engine-core | 项目/对象/打印板/侧文件解析模型 |
| `app/core/merge.py` | engine-core | 合并编排、输出装配、内部校验 |
| `app/service.py` | engine-core | **UI 唯一入口**（见 §4） |
| `app/report.py` | engine-core | overview 文本 / HTML / JSON 报告 |
| `app/core/__init__.py` | engine-core | 空或轻量导出 |
| `app/core/catalog.py` | engine-features | 字段目录、缺省补全、帮助查询 |
| `app/core/presets.py` | engine-features | 发现已安装 Studio、机台预设解析、嵌入回退 |
| `app/core/machines.py` | engine-features | 机型迁移计划、床型映射、兼容判定 |
| `app/core/palette.py` | engine-features | 耗材槽并集/去重/引用重映射/涂色策略 |
| `app/ui/*`、`app/startup.py` | ui-dev | 主题、组件、主窗口、启动器 |
| `app/resources/catalog.json`、`help_zh.json` | source-analyst | 源码派生的字段目录 / 官方中文说明 |
| `app/resources/machines.json`、`presets_fallback.json` | preset-analyst | 机台预设快照 / 回退 |
| `research/**`、`docs/SOURCE_NOTES*.md`、`docs/PRESET_NOTES.md` | 各自分析员 | 依据与结论 |
| `tests/**`、`tools/studio_verify.py`、`docs/VALIDATION_V08.md`、`docs/TEST_REPORT.md` | qa | 测试与 Studio 自动验收 |
| `docs/CONTRACTS.md`、`README*.md`、`CHANGELOG_v0_8.md`、`使用说明.md`、打包 zip | Lead | 契约、文档、打包 |

规则：**只写自己作用域内的文件**。需要别人文件改动/接口调整，发消息给 Lead 或对应 owner，不得直接改。
临时脚本放 `_recon/`（不入 git）、产物放自己目录。旧文件 `app/merge3mf.py`、`app/schema.py`、`app/native_catalog.json`、
`app/parameter_help.py`、`app/app.py`、`app/test_*.py` 由 Lead 在集成时删除，不要在此工作期间删除它们
（engine-core / ui-dev 可先并存新实现）。

## 3. 数据产物 schema（分析员必须严格按此输出）

### 3.1 `app/resources/catalog.json`（source-analyst）

从源码静态提取，**不得猜测**。字段：

```json
{
  "schema": 1,
  "studio_version": "02.09.01.70",
  "source_sha256": {"src/libslic3r/PrintConfig.cpp": "...", "src/libslic3r/PrintConfig.hpp": "...", "src/libslic3r/Preset.cpp": "..."},
  "scopes": {
    "process": ["layer_height", "..."],
    "machine": ["printer_model", "..."],
    "filament": ["filament_type", "..."],
    "object":  ["layer_height", "..."],
    "region":  ["wall_loops", "..."]
  },
  "options": {
    "layer_height": {
      "type": "ConfigOptionFloat",
      "default": 0.2,
      "scope": "process",
      "object_scope": "object",        // "object" | "region" | null，来自 PrintObjectConfig/PrintRegionConfig 静态成员
      "min_version": null,              // 源码有版本门槛时填，否则 null
      "label_en": "Layer height",
      "label_zh": "层高",               // 来自已安装 zh_CN BambuStudio.mo；无则 null
      "tooltip_en": "…",
      "tooltip_zh": "…"
    }
  },
  "renames": {"旧字段": "新字段"},        // 仅在源码/版本迁移逻辑中有明确依据时填写
  "deprecated": ["..."],                 // 有依据的废弃字段
  "notes": {"…": "依据说明"}
}
```

要求：
- `default` 必须是 JSON 可表达值（数值/布尔/字符串/数组/null）；`ConfigOptionFloatOrPercent` 等给源码默认的合理 JSON 形式并在 `notes` 说明。
- 所有 `options` 覆盖 PrintConfigDef 的**全部键**（约 600+），不是只覆盖对象参数。
- 帮助中文优先取已安装 `resources/i18n/zh_CN/BambuStudio.mo`（可用 Python `gettext` 解析），
  回退到英文 `tooltip_en`；不得保留乱码（v0.7 的 catalog 中文是 mojibake，必须修掉）。
- 另出 `app/resources/help_zh.json`：`{key: {"label": "...", "tooltip": "..."}}`（官方原文，供 UI 帮助弹窗）。
- `research/source/` 下放抽取的 C++ 文件；`docs/SOURCE_NOTES.md` 写清每个结论的源码文件+行号依据。

### 3.2 `app/resources/machines.json`（preset-analyst）

从已安装 Studio 的 `resources/profiles`（BBL.json + machine/*.json + process/*.json 继承链）解析：

```json
{
  "schema": 1,
  "studio_version": "02.09.01.70",
  "source_mode": "installed",
  "vendors": {"BBL": {"name": "Bambu Lab"}},
  "machines": {
    "Bambu Lab A1 0.4 nozzle": {
      "name": "Bambu Lab A1 0.4 nozzle",
      "vendor": "BBL",
      "printer_model": "Bambu Lab A1",
      "printer_variant": "0.4",
      "nozzle_diameter": ["0.4"],
      "printer_structure": "i3",
      "printable_area": ["0x0", "256x0", "256x256", "0x256"],
      "printable_height": 256,
      "bed_exclude_area": [],
      "bed_types": ["Textured PEI Plate", "..."],
      "machine_key_count": 300,
      "values": {"printer_model": "Bambu Lab A1", "...": "解析后的完整 machine-scope 键值"}
    }
  },
  "common": {
    "machine": {"…fdm_machine_common 解析结果…"},
    "process": {"…fdm_process_common…"},
    "filament": {"…fdm_filament_common…"}
  },
  "process_presets": {"0.20mm Standard @BBL A1": {"layer_height": 0.2, "inherits": "…"}},
  "filament_presets_sample": {}
}
```

要求：只收录 BBL 厂商（可选其它厂商但体积受控），`machines.json` 目标 < 8MB；解析要正确处理 `inherits` 链、
`include`、`instantiation`、`setting_id`、`from: system/user`。同时输出探测结果：Studio 可执行文件路径、版本、预设根目录。
`docs/PRESET_NOTES.md` 记录继承链解析规则与坑（如 `fdm_machine_common` 的 `nozzle_diameter` 覆盖）。

### 3.3 `app/resources/presets_fallback.json`

当用户机器上没有可读的 Studio profiles 时使用：至少包含 BBL 主流机型（A1/A1 mini/P1P/P1S/X1C/X1E/H2D 等 0.4 喷嘴）的
`printable_area/height/nozzle/structure/bed_types` 与 `machine` 常用键。可裁剪，但必须标注 `"source_mode": "embedded", "reduced": true`。

## 4. 服务层 API（UI 只依赖这一层；engine-core 实现）

```python
# app/service.py
VERSION = "0.8.0"

class MergeError(ValueError):
    """可读的中文错误，UI 直接展示 describe_error(e)。"""

def describe_error(error: BaseException) -> str: ...

def summarize(path) -> dict:
    """文件行。失败抛 MergeError。字段：
    path(str 绝对), file, studio_version, printer_model, printer_variant,
    printer_settings_id, layer_height, plates(int), plate_list([{id,name,objects}]),
    objects(int, 选中板对象数/默认板), config_keys(int),
    filament([{index,type,colour,settings_id}]), warnings([str])
    """

def target_machines() -> list[dict]:
    """[{id, name, printer_model, variant, nozzle([str]), printable_area, printable_height,
        structure, bed_types, vendor, source("installed"|"embedded")}]"""

def machine_source_info() -> dict:
    """{mode: "installed"|"embedded", studio_path, studio_version, machine_count, message}"""

def default_options() -> dict:
    """见 §6 MergeOptions"""

def overview(report: dict) -> str:      # 概览文本（中文）
def html_report(report: dict) -> str:   # 自包含 HTML

class PreparedMerge:
    def __init__(self, paths, *, base_index=0, target_machine=None,
                 plates=None, options=None):
        """paths: [str|Path]；plates: {源路径str: 1-based 板号} 或 None；
        options: dict（缺省 default_options()）。构造时完成全部分析，结果在 self.report。
        失败抛 MergeError（内部已写 blocked 报告到内存，不落盘用户目录）。"""
    report: dict
    requires_acceptance: bool
    def export(self, destination) -> list[Path]:
        """写出 3MF + .report.json + .report.html；返回路径列表。已存在/等于输入则报错。"""
    def close(self): ...
    def __enter__/__exit__: ...
```

## 5. 报告 JSON schema v2（UI 与报告共用；字段名固定）

```json
{
  "schema": 2, "tool_version": "0.8.0", "generated_at": "ISO8601",
  "status": "created | created_with_warnings | blocked_conflicts | blocked",
  "error": null,
  "target": {
    "machine": {"id": "...", "printer_model": "...", "variant": "0.4", "nozzle": ["0.4"],
                "printable_area": ["..."], "printable_height": 256, "structure": "i3", "bed_types": ["..."]},
    "bed_type": "Textured PEI Plate",
    "preset_source": "installed | embedded",
    "studio_version": "02.09.01.70"
  },
  "base": {"file": "A.3mf", "index": 0},
  "files": [{"file","path","sha256","studio_version","printer_model","printer_variant",
             "layer_height","bed_type","objects","plates","plate_selected",
             "config_keys","missing_keys","completed_keys","filament_types":["PLA"],
             "filament_count":1,"warnings":["..."]}],
  "objects": [{"source_file","source_object_id","output_object_id","name","part_count",
               "instances":[{"source_instance_id","transform","final_transform","bbox",
                             "footprint":[[x,y],...],"on_bed":true,"out_of_bed":false}],
               "explicit":{...},"converted":{...},"preserved":"full|converted|base",
               "notes":["..."]}],
  "machine_migration": {"performed": false, "from": [{"file","machine"}], "to": {...},
    "key_changes": [{"key","from","to","reason":"machine_preset|bed_type|nozzle|area|identity"}],
    "warnings": ["..."], "blocked_reasons": ["..."]},
  "palette": {"performed": false, "target_slots": 1,
    "sources": [{"file","slots":[{"index","type","colour","settings_id"}],"mapping": {"1":1}}],
    "deduped": [], "added": [], "paint_policy": "identity_only|remapped|refused|none",
    "warnings": []},
  "conflicts": [{"source","object_name","key","scope":"process|object|region|filament|machine|plate|metadata",
    "source_value","output_value","status":"converted_to_object|filled_default|base_value|dropped|blocked",
    "severity":"info|warning|blocking","help_key":true,
    "base_value": "可选；仅 converted_to_object 时有意义，表示基础项目全局值（output_value 是写入对象的源值）",
    "note": "可选；人类可读说明，例如‘已写入对象覆盖（源值）；基础项目全局值…’"}],
  "metadata": [{"source","file","status":"converted|dropped|blocked","note":"..."}],
  "layout": {"plate": {"area": ["..."], "height": 256},
    "possible_overlaps": [{"a":"obj@inst","b":"obj@inst","bbox_overlap":"3D"}],
    "out_of_bed": [{"object_id","instance","reason":"x|y|z"}]},
  "verification": {"archive": "passed|failed", "studio_load": "not_run|passed|failed|skipped",
    "studio_slice": "not_run|passed|failed|skipped", "details": ["..."],
    "note": "studio_load=passed 要求：Studio export 成功 且 对象 metadata/值全部保持（缺失或数值被改写都算 failed）"},
  "summary": {"files": 2, "objects": 2, "converted_fields": 4, "filled_defaults": 0,
              "warnings": 0, "blocking": 0}
}
```

UI 只读这些字段；新增字段必须向后兼容。`severity=blocking` 决定 `requires_acceptance` 与严格模式。

## 6. 合并语义（engine-core + engine-features 共同遵守）

`MergeOptions`（`service.default_options()` 原样返回）：

```json
{
  "allow_machine_migration": true,   // target_machine 与 base 不同时是否迁移；false 则视差异为冲突
  "allow_nozzle_mismatch": false,    // 喷嘴直径不同默认阻止；true 仅报告不阻止（不推荐）
  "fill_defaults": true,             // 用当前官方缺省补全旧文件缺失键
  "merge_palette": true,             // 耗材槽合并/去重/追加
  "strict_slot_fingerprint": false,  // true=同色但逐槽参数不同也拆成不同槽；false=按身份去重并 warning
  "paint_policy": "auto",            // auto: 需要重排且存在涂色引用时阻止；refuse: 直接阻止
  "strict": false,                   // 存在 warning/blocking 时禁止导出
  "verify_studio": false             // prepare 时调用已安装 Studio CLI 做导出校验（较慢）
}
```

语义规则：

- **R1 只读/原子**：输入只读；输出用临时文件+`os.replace`；目标已存在或等于输入 → 拒绝。
- **R2 单板输出**：每份输入选一块板（默认 1 号）。只合并该板对象；其它板写入报告 `metadata`。允许多板输入文件。
- **R3 几何**：模型文件（`3D/Objects/*.model`）复制并重映射资源 ID/路径/part id/UUID；
  build item 的 `transform` 字符串逐字符保持；顶点/三角面/涂色/接缝/支撑属性原样。
- **R4 全局配置**：以基础项目 config 为载体 →
  1) `fill_defaults`：源项目缺失键按 `catalog` 缺省值补全（仅用于比较；输出 config 也补全，保证新版本键齐全）；
  2) 机型迁移：`target_machine` 指定且与 base 不同 → 用目标机台预设值替换全部 machine-scope 键
     （`printer_model/printer_variant/printer_structure/nozzle_diameter/printable_area/printable_height/bed_exclude_area/printer_settings_id`
     等），`curr_bed_type` 按目标机可用床型映射（不可用→保留并 warning）；喷嘴不同→默认 blocking，除非 `allow_nozzle_mismatch`；
  3) 全局工艺差异（非对象可转换键）：source 与 base 不同 → 记 conflict；`strict`/未接受时 blocking，接受后用 base 值；
  4) 耗材：调色板并集（见 R7）。
- **R5 对象覆盖**：对象/区域可转换键（`catalog.is_object_convertible`）在“源解析值 ≠ 基础解析值且对象无显式覆盖”时，
  写成对象 metadata（`converted_to_object`）；对象已有显式覆盖则原样保留（`full`）。零件/层范围的既有覆盖不动。
- **R6 侧文件**：`layer_config_ranges.xml`、`layer_heights_profile.txt`、`cut_information.xml`、`custom_gcode_per_layer.xml`
  按 1-based 模型序号重映射；`filament_sequence.json`、缩略图、slice_info、plate_*.json 等缓存丢弃并报告；
  v0.7 已支持的未知打印元数据 fail-closed 规则保留。
- **R7 耗材**：
  - 槽定义 = 以 `filament_type/filament_colour/filament_settings_id` + 全部 `filament_*` 数组逐项值构成的指纹；
    相同指纹去重（映射到同一目标槽）；不同则追加；
  - 重写全局 `filament_*` 数组（按槽重排）、`filament_settings_id`、`wipe_tower_*` 等槽相关数组、plate 的 `filament_map*`；
  - 重映射对象/零件 metadata 的 `extruder`、`wall_filament` 等 FILAMENT_REFS，以及 `layer_config_ranges` 内的耗材键；
  - 涂色/多材料面数据（`paint_color` 等）：分析员给出编码结论前一律按“若映射非恒等且存在涂色引用 → blocking”处理；
    映射恒等时无需改写。
- **R8 布局**：仅计算/报告：包围盒、重叠对、是否超出目标机打印范围/高度；**绝不自动移动**。
- **R9 验证**：输出重新读回校验（ZIP/XML/计数/变换逐字符/引用）；`verify_studio=true` 时调用
  `tools/studio_verify.py` 的库函数（不经子进程也可）做 Studio 导出校验。报告 `verification` 如实填写，未跑= `not_run`。
- **R10 失败**：任何异常 → `status=blocked`、`error` 中文说明；不产生输出文件。

## 7. UI 规格（ui-dev）

窗口：默认 1200x800（按 DPI 缩放，最小 1024x700），标题「Bambu 3MF 工坊 v0.8」，中文界面，无模态弹窗刷屏。

布局（自上而下）：
1. **HeaderBar**：左侧品牌色块 + 标题 + 副标题「同盘合并 · 参数随模型迁移」；
   右侧 Studio 检测徽章（`已检测 Studio 02.09.01.70` / `未检测到，将使用内置预设`）+ 版本号。
2. **主体左右分栏**（可随窗口缩放，左 360px 固定/右自适应）：
   - 左「① 源文件」卡片：`＋ 添加 3MF`、`移除`、`清空`；文件列表（Treeview：文件名/机型/对象/板/层高），
     选中行下方显示路径、板号下拉（多板文件才有）、耗材 chips；
   - 右「② 合并设置」卡片：基础项目下拉、目标机型下拉（含 `与基础项目相同` 选项）、
     复选框：`补全旧版缺省参数`、`自动合并耗材槽`、`严格模式`、`本地 Studio 校验`；
     显示机型一致性提示条（相同/需迁移/硬件不兼容）。
3. **操作条**：`分析项目` 主按钮 + `选择位置并导出 3MF`（分析后才可用）+ 进度条 + 状态文本。
4. **Notebook（四个页签）**：
   - `概览`：4 个统计卡（文件/对象/迁移键/警告）+ 风险横幅 + 对象列表。
   - `打印板预览`：Canvas 画目标机打印范围（多边形）、各对象 footprint（带编号与名称）、重叠高亮红、超界高亮橙；缩放适配；图例；未分析时给占位提示。
   - `参数差异`：顶部筛选（分类 chips：全部/对象参数/全局工艺/耗材/机型迁移；搜索框）；表格列
     来源/对象/参数(中文+字段名)/原值/输出/处理/说明[?]；行 tag 着色（转换/补全/沿用/阻止/信息）；
     下方详情面板；`?`/双击/F1 弹官方中文说明。
   - `验证与报告`：验证等级表（ZIP/Studio 加载/Studio 切片）、报告摘要、`打开 HTML 报告`按钮。
5. **StatusBar**：就绪/处理中/完成/错误 + 导出路径。

风格：浅色现代卡片风；主色 Bambu 绿 `#00A85A`；背景 `#F3F6F4`；卡片 `#FFFFFF` 边框 `#E2E8E4`；
正文 `#16221C`、次要 `#5C6B63`；警告 `#B7791F`、错误 `#C0392B`；圆角不可用时用留白+细边框；
字体 `Microsoft YaHei UI` 10/11pt，标题 15-16pt 粗体；统一间距 8/12/16/20。禁用态降低对比但可读。
所有耗时操作异步（threading + queue + `after` 轮询），按钮状态机严谨，异常用 Banner 展示 `describe_error`。


### 7.1 v0.8.4 实际界面（以此为准）

- 左侧两个独立框：**① 主文件**（只选一份，点击「选择 / 更换」；点击文件名看详情）与 **② 加入文件**（可多份，列表只显示加入文件）。
- 清空语义：「清空加入文件」只清加入列表、保留主文件；「清空全部」清主文件+加入文件；均不删除磁盘文件。
- 更换主文件：旧主文件自动保留到加入列表；新主文件若已在加入列表则移出列表成为主文件；重选同一文件只更新摘要。三种入口（文件对话框 / 双击加入行 / 异步 set_main_file）规则一致。
- 右侧「合并设置」只保留打印机、合并选项、一致性提示；不再重复文件列表。主文件由 `_base_path` 标识，`_base_index()` 按路径反查。
- 长文件名：主文件标签省略+Tooltip 全文；加入列表悬停单元格显示完整名称/机型/路径。
- 校验失败：界面进入专用状态「校验未通过 · 已禁止导出」，不显示“接受差异”开关；导出按钮禁用（service/core 另有独立闸门）。
## 8. 测试与 Studio 自动验收（qa）

语料（`C:\Users\laobai\Downloads`，测试常量集中定义，缺失则 skip）：

| 文件 | 关键特征 |
|---|---|
| `Whistle140db3mf.3mf` | A1/02.09/1 对象/PLA/层高 0.2、壳层 2/2 |
| `superang_boomerang(3).3mf` | A1/02.09/1 对象/PLA/壳层 3/5、缩放+旋转 |
| `eap外壳.3mf` | X1C/02.05/2 对象 → 迁移到 A1 |
| `外观件.3mf` | A1 mini/02.00/7 对象/PETG → 迁移 + 耗材合并 |
| `瓦尔塔_EVA01_MW发布.3mf` | A1/02.08/6 对象/4 板/4 耗材槽 |
| `Removable+poop+bucket+(A1,A1+mini)+one+color+.3mf` | 01.10/368 键旧文件 → 缺省补全 |
| `Thunderstorm_V4.3mf` | A1/02.09/1 对象 |

`tools/studio_verify.py`：CLI 封装（stdout 打印中文摘要，`--json` 输出机器可读）：
- 探测 `C:\Program Files\Bambu Studio\bambu-studio.exe`；
- `--export-3mf`：临时目录运行 `bambu-studio.exe --export-3mf <tmp>\export.3mf <input.3mf>`，超时 240s，
  解析 `<tmp>\result.json`（`error_string=="Success."` and `return_code==0`），并解析导出 3mf 校验对象数/目标机型/对象 metadata；
- `--slice`：`--slice 0 --outputdir <tmp>\slice`，检查 `result.json` 与 `plate_1.gcode` 存在；
- 输出结论 JSON：`{ok, stage, error, objects, warnings, config_subset, gcode}`。
- **串行执行**（Studio 配置目录可能被锁）；不要启动 GUI 窗口模式；超时杀进程。

测试要求：
- `tests/` 用 `unittest`，`python -m unittest discover -s tests -v` 可全跑；
- 单元：字段目录/缺省补全/机型迁移/调色板映射/报告 schema/几何变换保持/失败路径；
- 集成（真实语料）：默认合并、X1C→A1 迁移、A1mini+PETG 调色板合并、旧版补全、多板选择；
  每个产出用 `studio_verify --export-3mf` 至少一次，关键 4 例加 `--slice`；
- 输入哈希前后不变、输出不覆盖、输出可被内部校验 + Studio CLI 重新加载；
- 汇总到 `docs/TEST_REPORT.md`（命令、结果、失败原因、未验证事项）。

## 9. 协作与交付

- 每项任务完成后：在共享任务板 `complete`，并向 Lead 发消息：产物路径、关键结论、测试证据（命令+结果）、已知限制。
- 遇到接口不一致/需要改别人文件：立即消息 Lead，不要自行改。
- 提交信息/文档用中文为主；代码注释可中英混排。
- 交付物：`bambu3mf_v08/`（app + runtime + launcher + 文档），最终由 Lead 打包 zip；
  验收标准：`startup.py --self-test` 通过、测试套件全绿、真实语料合并 + Studio 校验通过、新 UI 截图可正常打开。

