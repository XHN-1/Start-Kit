# 准备您的参赛作品
要运行程序，请参阅 [README.md](./README.md) 下载 start-kit 并完成编译。 


## 系统概览

<img src="./image/lorr2024_system_overview.jpg" style="margin:auto;display: block;max-width:800px"/>

上图展示了 start-kit 的系统概览。
在每个时间步：
1. 竞赛系统调用 `Entry` 获取提议的调度与规划，并传入存储当前机器人与调度状态的 `SharedEnv`。在内部，`Entry` 调用 `Scheduler` 计算提议的调度，并调用 `Planner` 计算提议的规划。
2. `Entry` 返回后，提议的规划会交给 Simulator 进行验证；若提议的规划有效，Simulator 会执行该规划并返回结果状态（当前状态）。
3. 当前状态与提议的调度随后交给 Task Manager 验证提议的调度，并检查哪些任务有进展或已完成。Task Manager 还会向系统发布新任务。
4. 返回的当前调度、当前状态、新发布的任务及其他信息会在下一时间步通过 `SharedEnv` 传给 `Entry`。

### 重要概念

在开始前，请熟悉代码库中的以下概念：
- 坐标系：机器人在地图上的位置是一个元组 (x,y)，其中 x 表示机器人所在的行，y 表示所在的列。第一行（最上行）的 x = 0，第一列（最左列）的 y = 0。可视化说明见[此处](./image/coordination_system.pdf)。
- 地图：地图是一个 `int` 的向量，索引由位置的 (行, 列) 线性化得到：(行 * 地图总列数) + 列，取值为 1 表示不可通行，0 表示可通行。
- `State`（定义于 `inc/States.h`）：包含当前位置（地图位置索引）、当前时间步和当前朝向（0:东，1:南，2:西，3:北）的状态。
- `Task`（定义于 `inc/Tasks.h`）：一个任务包含多个差事，存放在 `locations` 中，以及 `task_id`、被分配机器人的 id `agent_assigned` 和下一个未完成差事的索引 `idx_next_loc`。每个差事是地图上的一个位置，须按顺序依次访问。
  若任务的第一个差事尚未完成（即 `idx_next_loc=0`），可将任务重新分配给其他机器人。
  但一旦某机器人**开启**其分配到的任务（即完成该任务的第一个差事），该任务就不能再分配给其他机器人。
- `Action` 枚举（定义于 `inc/ActionModel.h`）：四种动作用起始动作编码为：FW - 前进，CR - 顺时针旋转，CCR - 逆时针旋转，W - 等待，NA - 未知动作。
- `Entry` 类是与 start-kit 主仿真的接口。每个时间步，主仿真会调用 Entry 的 compute() 函数以获取每个机器人的下一任务与下一调度。Entry 的 compute 会先调用任务调度器为每个机器人调度下一任务，再调用规划器规划下一动作。

  
### SharedEnv
`SharedEnvironment` API 提供计算调度与动作所需的信息。该数据结构（在 `inc/MAPFPlanner.h`、`inc/Entry.h` 和 `inc/TaskScheduler.cpp` 中定义为 `env`）描述仿真设置与当前时间步的状态：
-  `num_of_agents`：`int`，队伍总人数。
-  `rows`：`int`，地图行数。
-  `cols`：`int`，地图列数。
-  `map_name`：`string`，地图文件名。
-  `map`：`int` 的向量，存储地图。  
-  `file_storage_path`：`string`，用于指定文件存储路径，参见「本地预处理与大文件」小节。
-  `goal_locations`：`pair<int,int>` 的向量的向量：每个机器人当前的目标位置，即已调度任务的第一个未完成差事。第一个 int 是目标位置，第二个 int 表示任务被分配的时间步。
-  `current_timestep`：`int`，仿真器中的当前时间步。*请注意，在 `plan()` 调用期间 current_timestep 可能会增加。当规划器在给定时间步超出时间限制时会发生这种情况。*
-  `curr_states`：`State` 的向量，当前时间步每个机器人的当前状态。
-  `plan_start_time`：`chrono::steady_clock::time_point`，记录 `Entry::compute()` 被调用的确切时间；`Entry::compute()` 应在 plan_start_time 之后不超过 `time_limit` 毫秒内返回提议的规划与调度。
- `curr_task_schedule`：`vector<int>`，Task Manager 返回的当前调度。`curr_task_schedule[i]` 表示机器人 `i` 被调度的 `task_id`。`-1` 表示该机器人没有已调度的任务。
- `task_pool`：`unordered_map<int, Task>`，该 `unordered_map` 存储所有已发布但未完成的任务，以 `task_id` 为键。
- `new_tasks`：`vector<int>`，当前时间步新发布任务的 `task_id`。
- `new_freeagents`：`vector<int>`，当前时间步新释放的机器人（任务已完成的机器人）的 id。

  
## Entry 集成


### 理解默认 entry
在 `src/Entry.cpp` 中可找到默认的 entry 实现。在 `Entry::compute()` 中，默认 entry 先调用默认调度器。调度器完成后，机器人可能被分配新任务，其目标位置（每个机器人已调度任务的下一个差事）会写入 `env->goal_locations` 供规划器使用。
随后，entry 调用默认规划器计算机器人的动作。
时间限制会同时提供给默认调度器和规划器。在默认调度器与规划器内部，可见它们各自使用时间限制的一半。

#### 默认调度器
在 `src/TaskScheduler.cpp` 中可找到默认任务调度器，其调用的函数在 `default_planner/scheduler.cpp` 中进一步定义。
- 默认调度器的预处理函数（见 `scheduler.cpp` 中的 `schedule_initialize()`）调用 `DefaultPlanner::init_heuristics()`（见 `default_planner/heuristics.cpp`）初始化全局启发式表，用于存储不同位置间的距离。这些距离在仿真过程中按需计算。调度器用这些距离估计给定机器人在给定任务上的完成时间。
- 默认调度器的调度函数（见 `scheduler.cpp` 中的 `schedule_plan()`）实现贪心调度：每次调用 `schedule_plan()` 时，遍历每个尚未分配任务的机器人；对每个被遍历的机器人，遍历尚未分配给任何机器人的任务，将 makespan（从机器人当前位置依次经过任务所有差事的行程距离）最小的任务分配给该机器人。

#### 默认规划器
在 `src/MAPFPlanner.cpp` 中可找到默认规划器实现，其调用的函数在 `default_planner/planner.cpp` 中进一步定义。默认规划器与默认调度器共享同一启发式距离表。其 `initialize()` 准备必要的数据结构和全局启发式表（若调度器未初始化）。其 `plan()` 为当前时间步计算无碰撞动作。

默认规划器实现的 MAPF 规划器是 Traffic Flow Optimised Guided PIBT 的变体，参见 [Chen, Z., Harabor, D., Li, J., & Stuckey, P. J. (2024, March). Traffic flow optimisation for lifelong multi-agent path finding. In Proceedings of the AAAI Conference on Artificial Intelligence (Vol. 38, No. 18, pp. 20674-20682)](https://ojs.aaai.org/index.php/AAAI/article/view/30054/31856)。规划器先为每个机器人优化交通流分配，再按优化后的交通流使用 [Priority Inheritance with Backtracking](https://www.sciencedirect.com/science/article/pii/S0004370222000923) (PIBT) 计算无碰撞动作。更详细的技术报告将稍后提供。

> ⚠️ **注意** ⚠️，默认规划器是 anytime 算法，即用于计算解的时间会直接影响解的质量。 
> 当分配时间较少时，默认规划器行为类似 vanilla PIBT。当分配时间较多时，默认规划器会更新交通成本并增量重算引导路径以改善到达时间。
> 若您参加的是 Scheduling Track，请记住 `schedule_plan()` 越早返回，留给默认规划器的时间就越多。
> 更好的规划与更好的调度都会显著影响您在排行榜上的表现。 
> 如何在这两个组件之间分配时间是成功策略的重要一环。

## 各赛道的实现内容

- 调度赛道（Scheduling Track）：
您需要实现自己的调度器，它将与默认规划器配合使用。详见 [实现您的调度器](./Prepare_Your_Submission.md#implement-your-scheduler) 小节。

- 规划赛道（Planning Track）：
您需要实现自己的规划器，它将与默认调度器配合使用。详见 [实现您的规划器](./Prepare_Your_Submission.md#implement-your-planner) 小节。

- 综合赛道（Combined Track）：
您需要同时实现自己的规划器与调度器。您也可以修改 entry 以满足需求。详见 [实现您的规划器](./Prepare_Your_Submission.md#implement-your-planner)、[实现您的调度器](./Prepare_Your_Submission.md#implement-your-scheduler) 和 [实现您的 entry](./Prepare_Your_Submission.md#implement-your-entry)。

### 实现您的调度器

若您只参加调度赛道，可忽略 [实现您的规划器](./Prepare_Your_Submission.md#implement-your-planner) 和 [实现您的 entry](./Prepare_Your_Submission.md#implement-your-entry)。 
请阅读 [Entry 集成](./Prepare_Your_Submission.md#Entry-Integration) 以理解默认规划器与默认 Entry 的工作方式。

实现调度器的起点是查看 `src/TaskScheduler.cpp` 和 `inc/TaskScheduler.h`。
- 实现您自己的预处理函数 `TaskScheduler::initialize()`。 
- 实现您自己的调度函数 `TaskScheduler::plan()`。`plan` 的输入为时间限制和一个整数向量引用作为结果调度。结果调度中第 i 个整数表示分配给第 i 个机器人的任务索引。
- 不要修改 `TaskScheduler::initialize()` 和 `TaskScheduler::plan()` 的定义。除此以外，可自由向 `TaskScheduler` 类添加新成员/函数。
- 不要重写任何与操作系统相关的函数（如信号处理函数）。
- 不要干扰程序运行（如栈操作等）。

每个时间步，调度器可通过 `SharedEnvironment` API 在 `env->task_pool` 中读取可分配给机器人的任务。
调度器应向仿真环境为每个机器人返回一个任务调度。调度写入 `plan()` 的输入参数 `proposed_schedule` 向量。机器人 `i` 的调度 `proposed_schedule[i]` 为一个已开放任务的 `task_id`。以下情况为无效调度：
- 同一任务被分配给多个智能体，
- 包含已完成任务，
- 已被某智能体开启的任务被重新分配给另一智能体。
此外，将 `task_id` 设为 `-1` 表示该智能体无分配任务，会丢弃任何已有但未开启的任务。但对已开启任务的智能体赋 `-1` 会导致无效调度错误。

  
若调度器返回无效的 `proposed_schedule`，该调度会被拒绝，`current_schedule` 保持不变。


### 实现您的规划器

若您只参加规划赛道，可忽略 [实现您的调度器](./Prepare_Your_Submission.md#implement-your-scheduler) 和 [实现您的 entry](./Prepare_Your_Submission.md#implement-your-entry)。 
请阅读 [Entry 集成](./Prepare_Your_Submission.md#Entry-Integration) 以理解默认调度器与默认 Entry 的工作方式。

实现的起点是 `src/MAPFPlanner.cpp` 和 `inc/MAPFPlanner.h`。可参考 `src/MAPFPlanner.cpp` 中的示例。
- 在提供给您的 `MAPFPlanner::initialize()` 中实现您的预处理。 
- 在提供给您的 `MAPFPlanner::plan()` 中实现您的规划器。
- 不要修改 `MAPFPlanner::initialize()` 和 `MAPFPlanner::plan()` 的定义。除此以外，可自由向 `MAPFPlanner` 类添加新成员/函数。
- 不要重写任何与操作系统相关的函数（如信号处理函数）。
- 不要干扰程序运行（如栈操作等）。

每个规划回合结束时，您需为每个机器人向仿真环境返回一个 `Action`。动作写入作为 `plan()` 引用参数的 `actions` 向量。`actions[i]` 应为机器人 `i` 的有效 `Action`，不得将智能体移动到障碍物，且不得与任何其他机器人产生边或顶点冲突。若规划器返回任何无效 `Action`，该时间步所有智能体将等待。
 
与调度器类似，规划器可访问 `SharedEnvironment` API。您需通过该 API 读取系统当前状态。默认 `Entry` 实现会将有分配任务的智能体的下一目标位置写入 `env->goal_locations`。规划器可参考这些目标位置计算智能体的下一动作，也可通过 `env->curr_task_schedule` 获取详细任务调度与任务信息。

### 实现您的 entry
参加综合赛道的参赛者可自由修改 `Entry`、`MAPFPlanner` 和 `TaskScheduler` 以满足需求。
您需要实现自己的 `Entry::initialize()` 和 `Entry::compute()`，且不得修改它们的定义。除此以外，可自由向 `Entry` 类添加新成员/函数。
`Entry::compute()` 需要计算机器人的任务调度与动作。默认 entry 通过分别调用调度器和规划器完成，但这不是强制要求。
若参加综合赛道，允许修改 `Entry::compute()` 函数。

### 默认规划器与调度器的时间参数

每个时间步，我们会要求您的规划器在给定的 `time_limit`（毫秒）内为每个机器人计算下一个有效动作。`time_limit` 作为 `Entry.cpp` 中 `compute()` 的输入参数传入，再传给 `TaskScheduler::plan()` 和 `MAPFPlanner::plan()`。注意，对 `TaskScheduler::plan()` 和 `MAPFPlanner::plan()` 而言，当前时间步的起始时间为 `env->plan_start_time`，即调度器与规划器应在 `env->plan_start_time` 加上 `time_limit` 毫秒之前返回动作。这是软限制：若在 `time_limit` 届满前未返回动作，仿真会继续，所有机器人将原地等待到下一规划回合。

默认调度器与默认规划器按顺序运行。默认调度器使用 `time_limit/2` 作为计算调度的时限。
默认规划器在调度器返回后使用剩余时间计算无碰撞动作。

您仍可部分控制默认调度器与默认规划器的时间行为。
文件 `default_planner/const.h` 中定义了若干控制调度器与规划器时间的参数：
- `PIBT_RUNTIME_PER_100_AGENTS` 指定 PIBT 每 100 个机器人计算无碰撞动作所需的毫秒数。默认规划器用时间限制减去 PIBT 动作时间得到交通流分配的结束时间，将剩余时间留给 PIBT 返回动作。
- `TRAFFIC_FLOW_ASSIGNMENT_END_TIME_TOLERANCE` 指定交通流分配过程结束时间的容差（毫秒）。默认规划器会在交通流分配结束时间前这么多毫秒结束交通流分配阶段。
- `PLANNER_TIMELIMIT_TOLERANCE` MAPFPlanner 会从默认规划器的时间限制中减去该值。
- `SCHEDULER_TIMELIMIT_TOLERANCE` TaskScheduler 会从默认调度器的时间限制中减去该值。

允许修改这些参数的值以增减相关组件的用时。

### 不可修改的文件

除各赛道相关的实现文件及下文提到的可修改文件外，start-kit 中大部分文件不可修改，且必须确保不干扰其功能。 
更多细节请参阅 [Evaluation_Environment.md](./Evaluation_Environment.md)。


## 构建

实现规划器后，需要编译您的提交以进行本地测试与评测。
本节说明编译系统的工作方式及如何指定依赖。

### Compile.sh

- 评测系统会在竞赛服务器上执行 `compile.sh` 来构建您的程序。
- 评测系统会查找并执行 `./build/lifelong` 进行评测。
- 请确保 `compile.sh` 生成的可执行文件名为 `lifelong`，且位于 `build` 目录下。
- 默认情况下 `compile.sh` 会构建 C++ 接口实现。要构建 Python 实现（见下文），请删除 `# build exec for cpp` 之后的命令，并取消注释 `# build exec for python` 之后的命令。
- 您可根据实现需要调整 `compile.sh`。
- 允许根据需求自定义 `compile.sh` 和 `CMakeLists.txt`，但必须确保不干扰 start-kit 功能，且所有相关功能（尤其不可修改文件中的实现）均被编译。

### 依赖

实现中可自由使用第三方库或其他依赖。可通过以下方式指定：
- 将依赖包含在提交仓库中，
- 在 apt.txt 中列出依赖包。这些包须能在 Ubuntu 22 上通过 apt-get 安装。

## Python 接口
我们基于 pybind11 为 Python 用户提供了 Python 接口。

依赖：[Pybind11](https://pybind11.readthedocs.io/en/stable/)

pybind11 绑定实现在 `python/common`、`python/default_planner`、`python/default_scheduler`、`python/user_planner/` 和 `python/user_scheduler/` 下。
这些实现使用户实现的 Python 调度器能与默认 C++ 规划器配合，用户实现的 Python 规划器能与默认 C++ 调度器配合。
使用 Python 接口时，只需在以下文件中实现您的规划器和/或调度器：
+ `python/pyMAPFPlanner.py`：用户在此实现基于 Python 的 MAPF 规划算法，并为每个机器人返回动作列表。
+ `python/pyTaskScheduler.py`：用户在此实现基于 Python 的任务调度算法，并为每个机器人返回提议调度（任务 ID 列表）。

### 赛道配置与编译

竞赛各赛道使用不同的 Python 与 C++ 实现组合：
- 调度赛道：start-kit 使用 `Python 调度器` 与 `C++ 默认规划器`。
- 规划赛道：start-kit 使用 `Python 规划器` 与 `C++ 默认调度器`。
- 综合赛道：start-kit 同时使用 `Python 规划器` 与 `Python 调度器`。

在本地测试实现时，需在 start-kit 根目录下使用 `./python/set_track.bash` 配置对应赛道。该脚本会将所有必要的 Python 绑定文件放入 `./python/tmp` 以供编译。

综合赛道：
```shell
./python/set_track.bash combined
```
调度赛道：
```shell
./python/set_track.bash scheduler
```
规划赛道：
```shell
./python/set_track.bash planner
```

然后编辑 compile.sh，确保仅包含以下内容：
```shell
mkdir build
cmake -B build ./ -DCMAKE_BUILD_TYPE=Release -DPYTHON=true
make -C build -j
```

使用以下命令编译并测试：
```shell
./compile.sh
./build/lifelong --inputFile ./example_problems/random.domain/random_32_32_20_100.json -o test.json
```
编译完成后，程序会在相对于当前工作目录的 `./python` 或 `../python` 下查找 `pyMAPFPlanner` Python 模块和 `pyTaskScheduler.py`。此外，可在 `config.json` 中指定路径，并使用 cmake 选项 `-DCOPY_PY_PATH_CONFIG=ON`（会将 `config.json` 复制到目标构建目录），使程序在指定目录下查找 `pyMAPFPlanner`。

也可通过 `-DPYBIND11_PYTHON_VERSION` 指定 Python 版本，或通过 `-DPYTHON_EXECUTABLE` 指定具体 Python 安装路径。

例如：
```shell
cmake -B build ./ -DCMAKE_BUILD_TYPE=Release -DPYTHON=true -DPYBIND11_PYTHON_VERSION=3.6

# 或

cmake -B build ./ -DCMAKE_BUILD_TYPE=Release -DPYTHON=true -DPYTHON_EXECUTABLE=path/to/python
```

Python 包也可在评测服务器上通过 `pip` 安装，因此可在 `pip.txt` 中列出需要安装的包。
例如，默认 `pip.txt` 包含：
```
torch
pybind11-global>=2.10.1
numpy
```

## 评测

规划器实现并编译完成后，即可进行本地测试与评测。
评测系统使用官方 PyTorch 镜像 [pytorch/pytorch:2.4.1-cuda11.8-cudnn9-devel](https://hub.docker.com/layers/pytorch/pytorch/2.4.1-cuda11.8-cudnn9-devel/images/sha256-ebefd256e8247f1cea8f8cadd77f1944f6c3e65585c4e39a8d4135d29de4a0cb?context=explore) 作为 Docker 基础镜像构建评测环境，其中已包含 GPU 驱动、CUDA、cuDNN 等必要 GPU 软件。我们在此环境中官方支持并测试了 `Pytorch`，其他框架在评测环境中可能可用也可能不可用。

更多细节请参阅 [Evaluation_Environment.md](./Evaluation_Environment.md)。

### 本地测试
`example_problems` 目录下提供了多种测试问题。可使用其中任意 JSON 输入文件进行测试。
评测结果会写入您通过命令行参数 `--output_file_location` 指定的输出文件。
输入问题与输出结果的格式详见 [Input_Output_Format.md](./Input_Output_Format.md)。

### 在 Docker 中测试
评测系统在作为沙箱的 Docker 容器中构建并执行您的实现。
为确保实现能按预期构建与运行，可在本地构建 Docker 容器。

首先，在您的机器上安装最新版 Docker：[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)。
然后，使用我们提供的 `RunInDocker.sh` 脚本构建容器。 
本节剩余部分说明脚本用法及如何使用 Docker 容器。

#### 使用 `RunInDocker.sh`

* 在代码库根目录执行 `./RunInDocker.sh`。该脚本会根据 `pytorch/pytorch:2.4.1-cuda11.8-cudnn9-devel` 自动生成 Dockerfile 并构建镜像。
* 但镜像 `pytorch/pytorch:2.4.1-cuda11.8-cudnn9-devel` 仅支持 linux/amd64 系统/架构。若您使用其他系统/架构（例如 Arm CPU 的 MacOS）且不使用 GPU，可使用 [ubuntu:jammy](https://hub.docker.com/layers/library/ubuntu/jammy/images/sha256-58148fb210e3d70c972b5e72bdfcd68be667dec91e8a2ed6376b9e9c980cd573?context=explore) 作为替代，通过指定基础镜像运行脚本：`./RunInDocker.sh ubuntu:jammy`。
* 脚本会将您的代码复制到 Docker 环境，使用 apt-get 安装 `apt.txt` 中的依赖、使用 pip 安装 `pip.txt` 中的 Python 包，并用 `compile.sh` 编译代码。
* 脚本结束后您即处于容器内。
* 此时可在容器内运行已编译的程序。
* 镜像名 `<image name>` 为 `mapf_image`，容器名 `<container name>` 为 `mapf_test`。
* 默认工作目录为 `/MAPF/codes/`。
* 您可以在容器内测试与评测您的实现。
* 使用命令 `exit` 退出容器。

#### 启动已有容器：
  * 后台运行：`docker container start <container name>`
  * 交互式：`docker container start -i <container name>`

#### 在容器外执行命令
若 Docker 容器在后台启动，可从容器外执行命令（将容器视为可执行对象）。
 
  * 使用前缀：`docker container exec <container name> `加上要执行的命令，例如：
  ```shell
  docker container exec mapf_test ./build/lifelong --inputFile ./example_problems/random.domain/random_20.json -o test.json
  ``` 
 
  * 所有输出保存在容器内。可从容器复制文件。例如：`docker cp mapf_test:/MAPF/codes/test.json ./test.json`，将 `test.json` 复制到当前工作目录。

## 预处理与大文件存储

每次评测开始前，我们允许您的 entry 对每张地图有 30 分钟的预处理时间，用于加载支持文件并初始化支持数据结构。 
`preprocess_time_limit` 作为参数传入您 entry 的 `initialize()` 函数；默认会调用 MAPFPlanner 和 TaskScheduler 的 `intialize()`。若 entry 的预处理超过 `preprocess_time_limit`，规划器将失败，仿真会以 **退出码 124** 终止。 

更多细节请参阅 [Working_with_Preprocessed_Data.md](./Working_with_Preprocessed_Data.md) 中的文档。
