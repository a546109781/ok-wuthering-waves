# Agent 指南 / AGENTS.md

本仓库 `ok-ww`（ok-wuthering-waves）是一个基于图像识别的《鸣潮》自动化工具，使用 [ok-script](https://github.com/ok-oldking/ok-script) 框架开发，运行于 Windows，仅通过图像识别与 UI 模拟（PostMessage）操作游戏，不读取内存、不修改游戏文件。本文是本仓库唯一的 agent 指令入口（`.claude/`、`CLAUDE.md` 已被 gitignore）。

## Python 与虚拟环境

- 本仓库仅支持 **Python 3.12**。
- 运行任何 Python 命令前，优先使用仓库内的本地虚拟环境；没有本地 `.venv` 时才回退到 `python`。
  - Windows / PowerShell：优先 `.\.venv\Scripts\python.exe`。
  - POSIX shell：优先 `./.venv/bin/python`。
- **直接调用解释器**，例如 `.\.venv\Scripts\python.exe -m pytest`，不要依赖 shell 激活虚拟环境。
- 安装或更新依赖（运行时）：

  ```powershell
  .\.venv\Scripts\python.exe -m pip install -r requirements.txt --upgrade
  ```

- 开发/测试依赖（`requirements-dev.txt`，含 `polib`、`pytest`）：

  ```powershell
  .\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt --upgrade
  ```

- 运行程序：`.\.venv\Scripts\python.exe main.py`（release）/ `main_debug.py`（debug）。
- 顶层 `ok` 包（`from ok import ...`）来自 pip 依赖 `ok-script`，**不是仓库内的目录**（已 gitignore），不要在仓库里查找或新建它。

## 项目结构速查

- `config.py`：中心配置，构造 `config` 字典传给 `ok.OK(config)`。包含任务注册（`onetime_tasks` / `trigger_tasks`）、`scene`、`custom_tabs`、OCR、模板匹配、游戏窗口、分辨率、各语言链接、`my_app`（指向 `src/globals.Globals`）。**新增任务必须在此注册**。
- `src/task/`：自动化任务。
  - 一次性任务（用户启动、完成后自禁用）：继承项目基类 `BaseWWTask` / `WWOneTimeTask` / `BaseCombatTask`，最终源自 `ok.BaseTask`，注册于 `config.py` 的 `onetime_tasks`（如 `DailyTask`、`FarmEchoTask`、`TacetTask`、`ForgeryTask`、`GardenTask` 等）。
  - 触发任务（按间隔反复运行的背景任务）：继承 `TriggerTask`（如 `AutoCombatTask`、`AutoPickTask`、`AutoLoginTask`、`SkipDialogTask` 等），注册于 `trigger_tasks`。
- `src/char/`：角色战斗自动化，每个角色一个 `BaseChar` 子类，实现 `do_perform()`（上场连招）。`BaseChar.py` 提供通用 API；`CharFactory.py` 在 `_char_dict_raw` 中把 `Labels.<char>` 映射到角色类与定位（`CharType`、`Elements`）；`CustomCharLoader.py` 加载用户自定义的 `.wwcombo`。
- `src/scene/WWScene.py`：游戏场景状态定义（`WWScene`，注册为 `scene`）。
- `src/combat/CombatCheck.py`：战斗状态检测。
- `src/gui/`：自定义 GUI 标签页；注意 `CharacterCodeTab.py` 为**生成文件**（已 gitignore），不要手工编辑或提交。
- `src/Labels.py`：特征名 / 角色标签常量（开发者维护资产）。
- `src/globals.py`：`Globals` 单例（通过 `og.my_app` 访问），持有 YOLO 声骸检测模型与登录状态等。
- `assets/`：`coco_annotations.json`（特征 / 模板标注清单，被 `config.py` 引用）、`echo_model/`（YOLO ONNX 权重）、`images/`（模板图）。
- `i18n/`：gettext 翻译目录，含 `es_ES`、`ja_JP`、`ko_KR`、`zh_CN`、`zh_TW`，每个下 `LC_MESSAGES/ok.po`（+ 编译产物 `ok.mo`）。英文为源语言，`i18n/en_US/` 已 gitignore。
- `tests/`：`unittest` 测试与 `tests/images/` 测试图片。
- `ok_templates/`：git 子模块（COCO 标注模板），需要时 `git submodule update --init`。
- `main.py` / `main_debug.py`：入口，均将 `config` 交给 `ok.OK(config).start()`。

## 关键代码模式（硬性约束）

### 任务（`src/task/`，参考 `ok-script-tasks` skill）
- `__init__` **首行**调用 `super().__init__()`。
- 在 `__init__` 中设置 UI 元数据：`name`、`description`、`default_config`、`config_description`、`config_type`；配置通过 `self.default_config` 声明、`after_init()` 之后用 `self.config.get(...)` 读取，**不要绕开 `Config`**。
- 使用框架 API：`self.log_info` / `self.log_warning` / `self.info_set` / `self.wait_until` / `self.next_frame` / `self.sleep` / `self.click_relative` / `self.find_one` / `self.wait_click_feature` / `self.ocr` / `self.wait_ocr`，**避免自旋轮询**，`sleep` 要短，优先 `wait_until` / `wait_ocr` / `wait_click_feature`。
- `TriggerTask` 需设置 `trigger_interval`，`default_config['_enabled']` 要有意为之；`run()` 仅在做了实质性工作后返回真值。
- 新增/修改任务类后，在 `config.py` 对应列表里注册。
- 配置 key 优先使用**稳定的英文**（会落盘成持久化 JSON）；中文通过 `config_description` 或翻译系统补。

### 角色（`src/char/`，参考 `ok-ww-characters` skill）
- 新角色：参考相近角色的实现 → 阅读 `BaseChar` 与技能说明 → 新建 `src/char/<Name>.py` 继承 `BaseChar`（或 `Healer` 等），实现 `do_perform()` → **在 `CharFactory.py` 中 import 并加入 `_char_dict_raw`**（含 `Labels.<char>`、冷却、`liberation_cd`、`ring_index` 等）。
- 优先复用 `BaseChar` API：`click_resonance`（共鸣技能）、`click_liberation`（共鸣解放）、`click_echo`（声骸）、`heavy_attack`（重击）、`continues_normal_attack`（普攻连点）、`heavy_click_forte`、`is_forte_full`、`has_long_action`、`f_break`、`switch_next_char` 等；循环要有超时上限，配合 `self.task.next_frame()` / `self.sleep(...)`。
- 术语对照：`echo`=声骸、`resonance`=共鸣技能、`forte`=共鸣回路、`liberation`=共鸣解放、`click`=普通攻击、`heavy`=重击。
- **`Labels` 枚举名、`assets/coco_annotations.json`、模板图为开发者维护资产，不得擅自编造新标签名**；需要新标签或新模板时，须明确告知开发者补充，不要在代码里假设它们已存在。
- 测试：仅在已有截图或识别/可用性逻辑变动时增改测试，参考 `tests/TestChar.py`。

### 国际化（参考 `ok-script-i18n` skill）
- 源语言为英文，需翻译的字符串用 `self.tr("...")` 包裹（任务 `name`/`description`/`default_config` 的 key 与值、`config_description`、`config_type` 选项等）。
- 翻译目录约定：`i18n/<locale>/LC_MESSAGES/ok.po`；用 glob（`i18n/*/LC_MESSAGES/ok.po`）发现 locale，**不要硬编码语言列表**。
- 编辑 `.po` 后**必须编译生成 `ok.mo`**（用 skill 内的 `task_i18n_helper.py compile`）；`msgid` 必须与源串完全一致；不要加入仅用于日志的字符串；保留译者注释、flags 与顺序。

## 测试

- 测试框架为 Python 内置 `unittest`（测试基类多用 ok-script 的 `TaskTestCase`）。仓库未配置 `pytest.ini`/`conftest.py`/ruff/mypy。
- 全量测试（PowerShell）：`.\run_tests.ps1`，会逐个 `tests/*.py` 用解析到的 venv 解释器跑 `python -m unittest`。
- 单个测试：

  ```powershell
  .\.venv\Scripts\python.exe -m unittest tests\TestChar.py
  ```

- CI（`.github/workflows/test.yml`）：在 PR 打开/更新时于 `windows-latest` + Python 3.12 上逐文件运行 `python -m unittest`；运行前会**清除所有 token/密钥环境变量**，并强制 `core.autocrlf false`、`core.eol lf`、UTF-8 输出。提交 PR 前请在本地跑相关测试并在描述中附结果。

## Git 与提交流程

- **远程仓库**：
  - `origin`：当前开发者 fork（`a546109781/ok-wuthering-waves`），所有日常推送的目标。
  - `upstream`：原项目仓库（`ok-oldking/ok-wuthering-waves`），仅用于同步上游更新（`git fetch upstream` 后合并/变基），**不要向 upstream 推送**。
- **即时提交推送**：本地完成一组有意义的修改后，**立即提交并推送到 `origin`**，不要在本地堆积大量未提交改动。提交前先 `git status` 确认只包含预期文件；按需新建分支，提交信息遵循下方语言与风格约定。
  ```bash
  git add <预期文件>
  git commit -m "<中文提交信息>"
  git push origin HEAD
  ```
- 默认分支：`master`。提交信息以**中文**为主（与历史风格一致），简洁，不使用 conventional-commit 前缀（参见近期提交如 `修复特征测试图片路径`、`优化爱达穗`、`支持战斗前切换治疗`）。
- 改动范围与 PR 门禁（详见 `CONTRIBUTING.md`）：
  - Bug 修复、文档/翻译修正、小型兼容性修复：可直接提交 PR。
  - **新功能、重构、架构调整、角色逻辑大改、任务行为大改、显著改变现有体验的修改：须先联系作者或在社区讨论，确认方向后再开始。**
  - 不确定是否属于"大改"：先联系作者。
- PR 须保持聚焦，不混入无关格式化/重构；用户可见行为变更需附截图/录屏；改动配置/任务/角色/识别逻辑时需说明适用场景与可能影响。
- **不要提交**：日志、缓存、临时文件、个人配置、生成文件（包括 `configs/`、`logs/`、`screenshots/`、`cache/`、`onnxocr/`、`models`、`issue*.json`、`assets/result.json`、`src/gui/CharacterCodeTab.py`、`i18n/en_US/`、`.venv` 等）。
- 行尾：仓库使用 LF（`core.eol lf`），不要引入 CRLF 改动。
- 发布与打 tag：由 `deploy` skill 处理（稳定 `vX.Y.Z`，预发布 `-beta.N` / `-alpha.N`）。**不要重写、移动或删除已有 tag**；不要仅凭本地 tag 推断发布成功；仅在用户明确要求发布时才推送。

## 可用 Skill 指引

遇到以下任务时优先调用对应 skill（细节以 skill 内文档为准）：

- `use-local-venv`：任何 Python 命令的解释器选择与虚拟环境处理。
- `ok-script-tasks`：新增/修改 `ok-script` 任务类（`BaseTask` 一次性、`TriggerTask` 触发）、配置 UI 元数据、`config.py` 注册。
- `ok-ww-characters`：新增/修改/审查 `src/char` 角色类、`CharFactory` 注册、角色连招逻辑。
- `ok-script-i18n`：任务/角色相关字符串的翻译新增、同步、修复与 `.mo` 编译。
- `deploy`：完成修改后的提交、按规则计算并创建下一个版本 tag、推送发布。

## 联系与参考

- 开发者 QQ 群：`926858895`；Discord：https://discord.gg/vVyCatEBgA 。
- 入门与免责声明见 `README.md`（含中文版与 `README_en.md` / `README_ja.md` / `README_zh_TW.md`）；贡献流程见 `CONTRIBUTING.md`。
