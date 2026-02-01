# 更新日志

Version 2.1.2 - 2025-01-19
----------------------------
修复：
- 若 entry 在未重新分配新任务的情况下放弃某智能体的任务，Task Manager 不执行该放弃命令的问题。
- 升级文件列表缺少默认规划器文件的问题。

Version 2.1.1 - 2024-11-18
----------------------------
修复：
- 解的成本被重复计算的问题。
- 将任务重新分配给 id 更小的智能体时，`task->agent_assigned` 被错误地重置为 -1 而非该更小 id 智能体的问题。  

新增：
- 说明默认规划器 anytime 行为的文档。


Version 2.1.0 - 2024-11-15
----------------------------
新增：
- 在 `SharedEnvironment` 中新增 `new_tasks` API，用于通知 entry 当前时间步新发布的任务。
- 在 `SharedEnvironment` 中新增 `new_freeagents` API，用于通知 entry 当前时间步新释放的机器人（刚完成所分配任务的机器人）。
- 新增 `--logDetailLevel` 选项，用于指定日志文件的详细程度。

变更：
- `SharedEnvironment` 中的 API `vector<Task> task_pool` 已移除。请改用以 `task_id` 为键的 `unordered_map<int, Task> task_pool`。
- 文档已更新以反映 API 变更。
- 竞赛系统现会等待 entry 返回并记录超时次数，再进入仿真器。避免 entry 使用仿真器上未记录的时间。
- 输出 JSON 会记录 entry 超时次数、无效调度次数和无效动作次数。
- 默认 Scheduler 现已使用新 API 进行任务调度。
- 默认 Entry 中的 `update_goal_locations` 已更新为使用新的 `task_pool` API。（升级时请注意审阅 `Entry.cpp` 的变更，并决定如何将变更适配到您的 entry 实现。）
- 更新 Python 绑定以支持更新后的 API。
- 更新示例 Python 调度器以使用新 API。（升级时请注意审阅 `pyTaskScheduler.py` 的变更，并决定如何将变更适配到您的调度器实现。）
- 将 `task_id` 设为 `-1` 表示智能体无分配任务，会丢弃任何已有但未开启的任务。但对已开启任务的智能体赋 `-1` 会导致无效调度错误。

Version 2.0.0 - 2024-10-2
----------------------------

新增：
- 支持任务调度器
- 默认规划器与默认任务调度器实现
- 支持综合赛道
- 新赛道的 Python 支持
- 新增 Evaluation_Environment.md，说明评测环境详情

变更：
- 时间限制单位改为 `ms`
- 2024 竞赛的 start-kit 输入输出格式更新，支持带优先约束的任务
- 新功能相关文档更新
- 更新 RunInDocker.sh 以复现 2024 竞赛使用的评测环境
- 评测环境支持 PyTorch 与 CUDA
- 将 Prepare_Your_Planner.md 重命名为 Prepare_Your_Submission.md


Version 1.1.5 - 2023-11-21
----------------------------
修复：
- 未通过 CLI 设置日志文件时程序崩溃的紧急修复。

Version 1.1.4 - 2023-11-18
----------------------------
修复：
- 预处理超时导致段错误的 bug。预处理超时现会以退出码 124 终止程序。
- 规划时间超限时读取 env->goal_locations 导致段错误的 bug。

变更：
- 文档更新以说明预处理超时行为。
- 文档更新以更好说明在线超时行为。特别说明：若超过给定规划时间限制，在 `plan()` 调用期间 env->curr_timestep 可能会增加。

Version 1.1.3 - 2023-10-24
----------------------------
新增：
- 新增 Working_with_Preprocessed_Data.md，说明如何使用预处理数据。
- 新增 Debug_and_Visualiser_Your_Planner.md，说明如何用 JSON 输出配合 PlanViz 进行调试与可视化。
  
变更：
- 输入参数新增 `OutputScreen` 选项，用于选择输出 JSON 文件的详细程度。
- Readme、Parepare_Your_Planner 和 compile.sh 建议在仓库根目录下运行 start-kit。
- 简化 cout 与日志文件中重复输出
- 文档更新（在 Prepare_Your_Planner.md 中增加坐标系相关描述）
- 文档更新（在 Input_Output_Format.md 中增加 `OutputScreen` 对应说明）
- 主进程（仿真）终止时终止所有进程。

修复：
- 在仓库根目录运行 start-kit 时 Python 找不到已编译 MAPF 模块的问题。
- pybind 缺少 fie_storage_path 的问题。

Version 1.1.2 - 2023-08-29
----------------------------
修复：
- git 历史中解析 'T' 符号提交缺失的问题。


Version 1.1.1 - 2023-08-27
----------------------------
新增：
- 每张地图增加更多示例问题

变更：
- 文档更新（增加地图符号与任务分配策略相关描述）：Input_Output_Format.md
- 文档更新（增加 Windows 用户说明）：README.md
- 文档更新（增加地图坐标系相关描述）：Prepare_Your_Planner.md
- 从 start kit 中移除未使用类文件：Validator.h Validator.cpp

修复：
- 示例任务文件中起点与终点不连通的问题。
- 检测地图符号 'T' 的问题。
- 智能体验证超出地图边界的问题。

Version 1.1.0 - 2023-08-04
----------------------------
新增：
- 新增 changelog.md 用于跟踪版本变更
- 仓库域增加更多示例问题

变更：
- 文档更新（增加更多描述）：Input_Output_Format.md、Prepare_Your_Planner.md 
- README.md 中增加升级说明
- 问题文件中任务数小于队伍规模时在 logger 中增加警告
- 将 py_driver.cpp 与 drive.cpp 合并为一个
- 简化 Python 绑定的 CMakeList，支持通过 cmake 选项指定 Python 版本/可执行文件。
- Python 解释器路径增加 ./python 与 ../python。支持通过 config.json 指定自定义路径。

修复：
- 仓库 large 示例任务文件仅含 1000 个任务（小于队伍规模）、需替换为 10000 个任务的问题。
- 边冲突检查导致整数溢出的 bug。

Version 1.0.0 - 2023-07-13
----------------------------
项目首次发布
