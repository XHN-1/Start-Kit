# Start-Kit

## 加入竞赛

使用 GitHub 账号登录[竞赛网站](http://www.leagueofrobotrunners.org/)，我们将自动为您创建一个私有的 GitHub 提交仓库。
该仓库是您提交代码的地方。在「我的提交」页面，您可以点击「My Repo」打开您的 GitHub 提交仓库页面。

## 克隆您的提交仓库

将您的提交仓库克隆到本地。该仓库包含入门代码，帮助您准备提交。

```
$ git clone git@github.com:your_submission_repo_address
$ cd your_submission_repo
```

## 编译 start-kit

### 依赖项

- [cmake >= 3.16](https://cmake.org/)
- [libboost >= 1.49.0](https://www.boost.org/)
- 使用 Python 接口的用户建议安装 Python3 >= 3.11 和 [pybind11](https://pybind11.readthedocs.io/en/stable/) >=2.10.1。

在 Ubuntu 或 Debian Linux 上安装依赖：
```shell
sudo apt-get update
sudo apt-get install build-essential libboost-all-dev python3-dev python3-pybind11 
```

在 Mac OS 上建议使用 [Homebrew](https://brew.sh/) 安装依赖。

### 编译

使用 `compile.sh`：
```shell
./compile.sh
```

使用 cmake： 
```shell
mkdir build
cmake -B build ./ -DCMAKE_BUILD_TYPE=Release
make -C build -j
```

## 运行 start kit

使用以下命令运行 start-kit： 
```shell
./build/lifelong --inputFile the_input_file_name -o output_file_location
```

例如：
```shell
./build/lifelong --inputFile ./example_problems/random.domain/random_32_32_20_100.json -o test.json
```

更多帮助信息：
```shell
./build/lifelong --help
```

## Windows 用户
如果您是 Windows 用户，使用本 start-kit 最直接的方式是使用 WSL（Windows 子系统 for Linux）。请按以下步骤操作：
1. 安装 WSL，请参考 [https://learn.microsoft.com/en-us/windows/wsl/install](https://learn.microsoft.com/en-us/windows/wsl/install)
2. 在 WSL 中打开终端，执行以下命令安装必要工具（CMake、GCC、Boost、pip、Pybind11）：
```shell
sudo apt-get update
sudo apt-get install cmake g++ libboost-all-dev python3-dev python3-pip
pip install pybind11-global numpy
```
3. 使用上述命令编译 start-kit。

虽然理论上可以使用 Cygwin、Mingw 和 MSVC 运行本 start-kit，但相比使用 WSL 会更为复杂。您可能需要自行配置环境。

如果您使用 Docker，另一种选择是在 Docker 环境中开发和测试您的 Python 实现。您可以在本地机器上复现评测环境。更多详情请参阅 [在 Docker 中测试](./Prepare_Your_Submission.md#test-in-docker) 小节。

## 升级您的 Start-Kit

如果您的私有 start-kit 副本仓库是在 start-kit 升级之前创建的，您可以运行脚本 `./upgrade_start_kit.sh` 将 start-kit 升级到最新版本。

您可以查看 `version.txt` 了解当前 start-kit 的版本。

`upgrade_start_kit.sh` 会检查哪些文件被标记为需要升级，并从 start-kit 拉取这些文件。它会拉取并暂存文件，但不会提交。这样您可以在提交前审阅更改。

对于 [Parepare_Your_Planner.md](./Prepare_Your_Submission.md) 中规定为不可修改的文件，您应始终提交其更改。

⚠️ 但请注意，start-kit v2.1.0 对 `task_pool` 引入了所需的 API 变更。这需要对您的实现进行小幅修改以适配新 API。  
该变更也会影响 `src/Entry.cpp` 中 `update_goal_locations` 函数的实现，因此升级脚本会拉取新版本的 `src/Entry.cpp` 并可能覆盖您的修改。您可以使用 `git diff` 比较差异，并决定是否撤销部分修改或部分接受该文件的更改。 

升级脚本不会触碰大多数参赛者的实现文件。
但 `python/pyMAPFPlanner.py`、`python/pyTaskScheduler.py`、`inc/MAPFPlanner.h`、`inc/TaskScheduler.h`、`src/MAPFPlanner.cpp`、`src/TaskScheduler.cpp`、`default_planner/planner.cpp` 和 `default_planner/scheduler.cpp` 中的示例实现会随新 API 和补充文档一起更新。您可能需要查看这些文件的变更。 

## 输入输出说明

请参阅 [Input_Output_Format.md](./Input_Output_Format.md)。

## 准备您的规划器

请参阅 [Prepare_Your_Submission.md](./Prepare_Your_Submission.md)。

## 调试与可视化您的规划器
我们提供了一个用 Python 编写的可视化工具：[https://github.com/MAPF-Competition/PlanViz](https://github.com/MAPF-Competition/PlanViz)。
它可以可视化 start-kit 程序的输出，帮助参赛者调试实现。 

更多信息请参阅项目网站。文档 [Debug_and_Visualise_Your_Planner](./Debug_and_Visualise_Your_Planner.md) 也提供了解读和诊断规划器输出的实用提示。

## 提交说明

请参阅 [Submission_Instruction.md](./Submission_Instruction.md)。


