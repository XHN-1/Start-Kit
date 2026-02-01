# 调试与可视化您的规划器


输出文件提供了一些相关信息，帮助您设计与调试规划器。 
请先熟悉输出 JSON 文件的结构。详见 [Input_Output_Format.md](./Input_Output_Format.md)。

## 使用输出文件调试规划器
调试规划器时，以下属性有助于了解规划器如何协调机器人。以下建议可能有用：
1. 检查有效性：查看 `errors` 和 `scheduleErrors`，识别并处理无效动作或无效调度。
2. 审阅路径：研究 `plannerPaths` 和 `actualPaths` 结果，检查规划器行为是否符合预期。
3. 审阅调度：研究 `plannerSchedule` 和 `actualSchedule` 结果，检查调度器行为是否符合预期。
4. 分析时间：查看 `plannerTimes` 属性，了解您的 entry 在每次计算回合中的耗时。若发现与预期耗时明显偏差，可进一步排查。
5. 其他方式：您还可以通过 `numTaskFinished` 结果及 `events` 和 `tasks` 中完成的任务分析来对比表现并设计更好的规划器。

## 使用输出文件可视化规划器
我们还提供了 [PlanViz](https://github.com/MAPF-Competition/PlanViz) 工具，用于根据输出 JSON 文件可视化您的规划。
PlanViz 会播放 JSON 中智能体的 actualPaths 与 actualSchedule 动画。
actual paths 与 actual schedule 始终有效且无冲突，而 plannerPaths 与 plannerSchedule 可能包含冲突与无效移动。
此时 PlanViz 会用红色高亮有问题的智能体。
您也可以点击错误列表和事件列表中的每一项，跳转到对应时间步。
请注意 PlanViz 不做任何验证或错误检查。
因此，它显示的错误来自 JSON 文件中的记录。
若您手动修改 JSON 文件，错误列表与智能体高亮可能与移动不一致。

更多详情请参阅 [Visualiser Page](https://github.com/MAPF-Competition/PlanViz)。
