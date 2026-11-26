# 实践课5：Pi Agent工具调用与会话恢复 maoenqi

- GitHub 登录名：Mao123415
- [实践报告：practive-05.md](practive-05.md)
- [小组代码仓库](https://github.com/MangDui/pi-hunnu-2026)

报告包含单轮工具调用的代码级时序图、恢复旧会话与新建会话的对照图，以及两阶段实际运行结果和源码定位。

## 图表与证据

- [图一：单轮工具调用](img/figure-1-tool-call.drawio.png) · [可编辑源文件](img/figure-1-tool-call.drawio)
- [图二：会话恢复对照](img/figure-2-session-compare.drawio.png) · [可编辑源文件](img/figure-2-session-compare.drawio)
- [任务 A 运行证据](task-a-evidence.json)
- [任务 B 运行证据](task-b-evidence.json)

任务 A 实际执行 read 一次、调用模型两次。任务 B 中，恢复会话能给出交接单里的完整命令，新会话说明缺少命令，两次均未调用工具。
