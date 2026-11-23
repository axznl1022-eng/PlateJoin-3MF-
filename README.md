# PlateJoin · 3MF 拼盘助手

把几份拓竹 3MF 项目合到同一块打印板，尽量保留每个模型原来的打印设置。

Merge several Bambu Studio 3MF projects onto one build plate while keeping each model’s supported print settings.

**Windows 10/11 x64 · 1.0 · 本地处理 / Runs locally**  
[中文](#中文) · [English](#english) · [开发交接 / Developer history](https://github.com/axznl1022-eng/PlateJoin-3MF-/blob/main/DEVELOPER_HISTORY.md)

## 中文

**什么时候用它？** 比如你下载了一个收纳盒和一个挂钩，想放在一盘打印。盒子需要 5 层顶壳，挂钩只需要 2 层。拼盘助手会尽量让各模型保留自己的设置，把只能全盘共用的差异列出来，方便你在打印前核对。

### 从两份模型到一次切片

1. 解压安装包，双击 `PlateJoin-1.0-Setup.exe`。安装后从开始菜单打开 **PlateJoin**。如果装过早先的 1.0 安装版，要换用这一版，请先从 Windows「已安装的应用」卸载旧版。
2. 在 **① 主文件** 选一份带打印配置的 3MF。选你想沿用其打印机和整盘设置的那一份。在 **② 加入文件** 添加其余项目。
3. 点 **分析项目**。在「打印板预览」看模型有没有重叠或超出边界，在「参数差异」看每项设置最终用了哪个值。
4. 点 **保存合并文件**，得到一份新的、可编辑的 `.3mf`。在 Bambu Studio 打开，调整摆放，查看切片预览后再打印。原文件保留在原处。

### 哪些设置听谁的？

| 界面提示或选项 | 对打印项目有什么影响 |
|---|---|
| 使用主文件的值 | 这项设置整盘只能选一个值。例如两份项目的擦料塔设置不同，合并后采用你在 ① 里选的那份。 |
| 保留模型自己的设置 | 支持按模型设置的项目可以各用各的值。所以上例中的盒子可以保留 5 层顶壳，即使主文件设的是 2 层。 |
| 用 Studio 检查文件 / 试切片 | 可选，需要本机装有 Bambu Studio。勾选的检查失败时会阻止导出。试切片不启动打印机。 |
| 不另存报告 / HTML / JSON | 默认只保存 3MF。需要把差异发给别人看可选 HTML，需要用程序读取可选 JSON，也可以两种都存。 |

### 适用范围

- 输入和输出都是 **Bambu Studio 打印项目（.3mf）**。仅凭扩展名无法判断一个 3MF 是否带有打印配置。STL、OBJ 和 G-code 无法直接作为合并输入。
- 程序保留模型位置。两份项目的模型可能恰好都在板中央，合并后需要在 Studio 里移开。软件不自动排版。
- 多喷嘴、复杂 AMS 和超大项目的验证有限。面涂色遇到不支持的耗材槽换序时会阻止合并。
- 基本合并可以离线使用。开发测试主要使用 Bambu Studio 02.09.01.70。[测试记录](https://github.com/axznl1022-eng/PlateJoin-3MF-/blob/main/TEST_REPORT_v084.md)

## English

**When would I use it?** Say you want to print a storage box and a hook from two downloaded projects. The box needs five top shell layers and the hook needs two. PlateJoin can keep those settings per model and shows which settings must be shared across the plate.

### Quick start

1. Extract the installer ZIP and run `PlateJoin-1.0-Setup.exe`. Open **PlateJoin** from Start. If you installed an earlier 1.0 build, uninstall it from Windows Installed apps before installing this build.
2. Under **① 主文件 (Main project)**, choose the project whose printer and shared settings you want to use. Under **② 加入文件 (Additional projects)**, add the others.
3. Select **分析项目 (Analyze projects)**. Check the plate preview for overlaps or models outside the bed, then review the source and output values in the settings comparison.
4. Select **保存合并文件 (Save merged file)**. Open the new editable 3MF in Bambu Studio, arrange the models and review the slice preview before printing. Your original files stay intact.

### Settings, checks and reports

Supported settings such as top shell layers can follow each model. Settings shared by the entire plate, such as certain prime tower settings, use the main project’s value.

The default output is a single 3MF. Choose HTML for a readable report, JSON for processing, or both. With Bambu Studio installed, you can also enable a read-back check or a test slice. A failed requested check blocks export. A test slice never sends a print job.

### Compatibility

PlateJoin works locally with Bambu Studio 3MF projects that contain print settings. STL, OBJ and G-code are unsupported as merge inputs. Models retain their positions and may overlap until you arrange them in Studio. Multi-nozzle setups, complex AMS configurations and very large projects have limited test coverage. Unsupported filament-slot changes on face-painted models block the merge.

## 开发与反馈 / Development and feedback

[开发历史与修复记录 / Developer history](https://github.com/axznl1022-eng/PlateJoin-3MF-/blob/main/DEVELOPER_HISTORY.md) · [合并规则 / Contracts](https://github.com/axznl1022-eng/PlateJoin-3MF-/blob/main/CONTRACTS.md) · [Studio 验证记录 / Validation notes](https://github.com/axznl1022-eng/PlateJoin-3MF-/blob/main/STUDIO_STABILITY.md)

反馈时请附工具版本、Studio 版本、复现步骤和报错信息。  
When reporting a problem, include both version numbers, the steps to reproduce it and the error message.

独立社区工具，与 Bambu Lab 无官方关联。第三方许可随软件附带，项目整体许可情况见开发交接文档。  
Independent community tool, unaffiliated with Bambu Lab. Third-party notices are bundled. See the developer history for project licensing status.
<div align="center">

**🐾 既然你看到这里了，那就告诉你 🐾**
**我是一个什么代码都不会的小猫 🐱**
**本期作品是由 DeepSeek V4.1 flash 1017 和 ChatGPT 6GPT-6 Astra 交叉完成制作以及绝大多数测试！**

</div>
> <img width="472" height="472" alt="xiaomao" src="https://github.com/user-attachments/assets/39410fcc-ec64-4af2-9289-875f849a3f31" />

