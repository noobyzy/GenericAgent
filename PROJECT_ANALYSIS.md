# 项目分析

## 概述

本文档总结了 GenericAgent 项目的两部分内容：

1. 项目中如何定义和使用 `skill`
2. 该 Agent 框架暴露了哪些内置工具

本分析基于当前仓库源码，重点参考如下文件：

- `README.md`
- `GETTING_STARTED.md`
- `agentmain.py`
- `agent_loop.py`
- `ga.py`
- `memory/memory_management_sop.md`
- `memory/skill_search/`
- `assets/tools_schema.json`

---

## 1. 本项目中的 `skill` 是什么

### 1.1 它不是传统插件系统

在这个仓库里，`skill` 通常不是那种通过 `load_skill("name")` 之类接口单独注册、加载的插件。

相反，主实现把 `skill` 视为记忆系统的一部分：

- L0：元规则
- L2：全局事实
- L3：任务 SOP 与可复用脚本

`README.md` 中描述的设计路径是：

> 解决一个新任务 -> 将执行路径固化为 skill -> 写入记忆层 -> 后续复用

因此，这个项目里 `skill` 的实际含义主要是：

- `memory/*.md` 下可复用的 SOP 文件
- `memory/*.py` 下可复用的辅助脚本
- 记忆索引中用于发现这些能力的指针

---

## 2. skill 背后的记忆结构

分层记忆模型定义在 `memory/memory_management_sop.md` 中。

### 2.1 L0 / L1 / L2 / L3

- **L0**：`memory/memory_management_sop.md`
  - 规定什么信息可以记忆，以及如何更新记忆
- **L1**：`memory/global_mem_insight.txt`
  - 极简关键词索引，用于提升可发现性
- **L2**：`memory/global_mem.txt`
  - 与当前环境相关的稳定事实
- **L3**：`memory/*.md` 与 `memory/*.py`
  - 任务级 SOP 与可复用脚本

对于 `skill` 来说，最关键的一层是 **L3**。

根据 `memory_management_sop.md`：

- `*_sop.md` 用于保存精简 SOP
- `*.py` 用于保存可复用辅助脚本

这意味着项目中的“技能树”本质上主要是一个建立在记忆层上的 SOP 与脚本集合。

---

## 3. skill 在运行时如何被使用

### 3.1 启动时会把记忆指针注入 system prompt

在 `agentmain.py` 中，`get_system_prompt()` 会把 `ga.py` 里的 `get_global_memory()` 结果拼接到 system prompt。

`ga.py:get_global_memory()` 会读取：

- `assets/insight_fixed_structure.txt`
- `memory/global_mem_insight.txt`

这些内容会向模型提示：

- 记忆文件在哪里
- SOP 在哪里
- 全局事实在哪里
- 哪个元规则文件负责约束记忆更新

因此，Agent 在启动后是通过**文件路径和记忆索引来发现 skill**，而不是通过插件注册表来发现。

### 3.2 Agent 通过普通工具读取并执行 skill

在运行时，并不存在一个专门公开的工具例如 `run_skill`。

相反，Agent 的典型流程是：

1. 用 `file_read` 读取某个 memory / SOP 文件
2. 提取其中的关键指令
3. 用 `update_working_checkpoint` 记录短期任务信息
4. 再用 `code_run`、`web_scan`、`web_execute_js`、`file_patch`、`file_write` 等工具完成实际工作

### 3.3 读取 memory / SOP 时会被特殊对待

在 `ga.py` 中，`do_file_read()` 遇到路径里包含 `memory` 或 `sop` 时，会追加提醒：

- 如果决定按该 SOP 执行，就要提取关键点
- 然后把这些关键点写入 working memory

这说明 SOP 在这个项目里不是普通文档，而是 Agent 的一等操作指南。

### 3.4 working memory 会维持当前 skill 的活跃状态

`update_working_checkpoint` 会保存：

- `key_info`
- `related_sop`

之后 `ga.py` 中的 `_get_anchor_prompt()` 会在后续轮次继续把这些内容注入提示词。这样在长任务过程中，当前选中的 skill 不容易被遗忘。

### 3.5 完成后的任务可以沉淀成新的 skill

在有价值的任务结束后，`start_long_term_update()` 会引导 Agent：

- 把稳定事实写入 L2
- 把可复用的任务经验写入 L3 SOP

这是该框架“自我进化”的核心路径之一。

---

## 4. 如何使用已有 skill

### 4.1 用户层面的使用方式

预期的用户体验通常是自然语言触发，例如：

- “执行 web setup sop”
- “读取你的 SOP，然后帮我完成这个任务”
- “配置 ADB 并写入你的记忆”

该框架的设计目标是让 Agent 能先查看自己的记忆和源码，再自行决定该使用哪个 SOP 或脚本。

### 4.2 直接按文件来使用

因为 skill 本质上是文件，所以也可以直接显式引用，例如：

- `memory/web_setup_sop.md`
- `memory/tmwebdriver_sop.md`
- `memory/plan_sop.md`

Qt 前端也体现了这一设计。在 `frontends/qtapp.py` 中，SOP 页面会直接列出 `memory/*.md`。

---

## 5. 如何新增或“安装”一个 skill

### 5.1 主系统没有独立安装器

对于 GenericAgent 主系统来说，新增一个 skill **并不是** 通过某个插件注册 API 去“安装插件”。

标准做法是：

1. 在 `memory/` 下新增 SOP 或辅助脚本
2. 如果希望它更容易被发现，就更新记忆索引
3. 如有必要，把相关环境事实补充到 L2

### 5.2 推荐形式

#### 方式 A：SOP 文件

例如新建：

- `memory/my_task_sop.md`

适合保存：

- 隐藏前置条件
- 重要坑点
- 精炼、可重复执行的流程

#### 方式 B：辅助脚本

例如新建：

- `memory/my_task_helper.py`

适合保存：

- 可复用且不太简单的逻辑
- 不希望 Agent 每次都重新构造的任务自动化流程

### 5.3 让它更容易被发现

如果你希望新的 skill 更容易被 Agent 想起，最好同步更新：

- `memory/global_mem_insight.txt`

因为这个文件会在启动时直接注入到 system prompt 中，它就是主系统的 discoverability index（可发现性索引）。

### 5.4 什么时候更新 L2

如果某个 skill 依赖稳定的环境事实，则应补充到：

- `memory/global_mem.txt`

典型示例：

- 固定服务地址
- 非标准路径
- 与当前环境绑定的配置事实

### 5.5 “安装 skill”的实际含义

在这个代码库里，“安装一个 skill”通常意味着：

- 把 SOP 或脚本放进 `memory/`
- 如有需要，在 `global_mem_insight.txt` 中建立索引
- 让 Agent 去读取并复用它

当前源码中没有看到专门用于 L3 skill 的安装器或注册表。

---

## 6. 独立的 `skill_search` 子包

仓库里还存在一个单独组件：

- `memory/skill_search/`

它**不是** GenericAgent 主运行时的 skill 机制，而是一个可选的外部 skill 检索客户端。

### 6.1 它提供了什么

`memory/skill_search/skill_search/engine.py` 暴露了：

- `search(query, env=None, category=None, top_k=10)`
- `get_stats(env=None)`
- `detect_environment()`

它的作用是通过 HTTP 搜索一个外部 skill 索引。

### 6.2 环境变量

该子包支持：

- `SKILL_SEARCH_API`
- `SKILL_SEARCH_KEY`

源码中的默认 API 地址是：

- `http://www.fudankw.cn:58787`

### 6.3 CLI 用法

CLI 入口为：

- `python3 -m skill_search`

示例命令：

```bash
cd memory/skill_search
python3 -m skill_search --env
python3 -m skill_search "python testing"
python3 -m skill_search "docker deployment" --category devops --top 5
```

### 6.4 Python 用法

由于这个子包位于仓库内部，默认并没有全局安装，所以如果从仓库根目录直接导入，需要先调整 `sys.path`。

示例：

```python
import sys
sys.path.append("/path/to/repo/memory/skill_search")
from skill_search import search

results = search("python send email")
```

### 6.5 与主 skill 机制的重要区别

这个子包是一个**外部 skill 搜索客户端**，并不等同于主系统基于 L3 memory 的 skill 机制。

从当前主流程代码来看，也没有发现它被自动导入并接入主 Agent 运行时。

---

## 7. Agent 框架的内置工具

公开的内置工具定义在：

- `assets/tools_schema.json`
- `assets/tools_schema_cn.json`

对应实现位于 `ga.py`。

该框架一共暴露 **9 个公开内置工具**。

### 7.1 `code_run`

**作用：** 执行代码或 shell / 终端命令

根据 `ga.py` 的实现：

- `type="python"`：会先写入一个临时 Python 文件，再执行
- 非 Windows 环境下使用 `type="bash"`
- Windows 下使用 `type="powershell"`

因此，这个工具承担了框架中的：

- 终端命令执行
- 脚本运行
- 依赖安装
- 测试执行
- Git 操作

典型支持场景包括：

- `git status`
- `pip install requests`
- `npm test`
- `python3 some_script.py`

因此，该框架**没有独立的 terminal tool**，终端能力实际上是通过 `code_run` 提供的。

### 7.2 `file_read`

**作用：** 读取文件内容

支持：

- 指定路径
- 按行偏移读取
- 指定读取行数
- 基于关键字的上下文搜索
- 行号显示

### 7.3 `file_patch`

**作用：** 通过替换唯一文本块来精细修改文件

这是用于局部修改代码或文本的精准编辑工具。

重要特性：

- `old_content` 必须精确匹配
- 匹配结果必须唯一
- 空格、缩进、换行都会影响匹配

### 7.4 `file_write`

**作用：** 写入整份文件内容

支持模式：

- `overwrite`
- `append`
- `prepend`

更适合用于：

- 新建文件
- 大段改写
- 输出生成结果

### 7.5 `web_scan`

**作用：** 查看当前浏览器页面与标签页列表

会返回简化后的页面信息，并支持：

- 标签页列表
- 标签切换
- 纯文本模式

它是浏览器自动化工具链的一部分。

### 7.6 `web_execute_js`

**作用：** 在受控浏览器中执行 JavaScript

这是网页控制的核心工具，主要用于：

- DOM 交互
- 数据提取
- 页面导航逻辑
- 浏览器端自动化

### 7.7 `update_working_checkpoint`

**作用：** 维护短期任务记忆

它会保存当前任务中的重要信息，例如：

- 用户需求
- 关键约束
- 文件路径
- 当前进度
- 关联 SOP 名称

这些内容会在后续轮次继续注入，减少长任务中的上下文丢失。

### 7.8 `ask_user`

**作用：** 中断当前流程并向用户提问

通常在以下情况下使用：

- 任务需要人类决策
- 缺少必要信息
- 自动化流程被阻塞

### 7.9 `start_long_term_update`

**作用：** 开始长期记忆提炼

通常在以下情况下使用：

- 某个任务产出了可复用事实
- 验证出了新的偏好或环境知识
- 某条可复用流程值得沉淀到记忆层

---

## 8. 工具执行是如何被编排的

执行循环定义在 `agent_loop.py` 中。

高层流程如下：

1. 构造 system prompt
2. 把用户输入发送给 LLM
3. LLM 返回工具调用请求
4. `BaseHandler.dispatch()` 把每个工具调用路由到 `ga.py` 中对应的 `do_<tool_name>()`
5. 工具结果再反馈给下一轮模型
6. 循环直到任务完成

如果模型没有显式返回任何工具调用，引擎会走一个内部兜底路径：

- `no_tool`

这不是普通公开工具之一，而是引擎内部行为。

---

## 9. 最重要的实践结论

### 9.1 关于 skill

- skill 的主体是基于 memory 的 SOP 与脚本
- 它们主要存放在 `memory/`
- skill 是通过记忆文件被发现的，而不是通过传统注册表
- 新 skill 的“安装”本质上就是新增文件并建立索引

### 9.2 关于工具

- 框架公开暴露了 9 个内置工具
- `code_run` 是执行终端命令的内置方式
- 文件编辑分为 `file_patch` 和 `file_write`
- 浏览器控制分为 `web_scan` 和 `web_execute_js`
- 记忆持久化则由 `update_working_checkpoint` 与 `start_long_term_update` 负责

### 9.3 关于 `skill_search`

- `memory/skill_search/` 是一个可选的外部 skill 检索客户端
- 它独立于主系统基于 memory 的 skill 机制
- 它可以通过 CLI 或手动 Python 导入来使用，但当前看不到它已自动接入核心运行时

---

## 10. 参考文件列表

- `README.md`
- `GETTING_STARTED.md`
- `agentmain.py`
- `agent_loop.py`
- `ga.py`
- `assets/tools_schema.json`
- `assets/tools_schema_cn.json`
- `assets/insight_fixed_structure.txt`
- `memory/memory_management_sop.md`
- `memory/web_setup_sop.md`
- `memory/tmwebdriver_sop.md`
- `frontends/qtapp.py`
- `memory/skill_search/SKILL.md`
- `memory/skill_search/skill_search/engine.py`
- `memory/skill_search/skill_search/__main__.py`
