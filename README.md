## 说明

目前该插件由 AI 辅助完成。

本地环境（`*\N.E.K.O\.neko-plugin-installations\plugins`）下的插件运行测试暂时没有问题。
如有问题请联系我，我再去问 AI（（

## 已知问题

目前插件分为两部分：插件入口 `__init__.py`，以及游戏本体 `game\minesweeper.py`（新版扫雷）。

由于游戏窗口由 tkinter 绘制，因此依赖 Tcl/Tk。此外我还不清楚如何让游戏随插件一同启动，所以需要一个可被调用的小型 Python 脚本，放在 `game\python` 目录下。

## 当前计划

正在阅读开发文档（`N.E.K.O/plugin/neko_plugin_cli/docs/package-format.md` 以及 https://project-neko.online/zh-CN/plugins/ ）。

目前进度是「AI 已读，我还没读」，所以插件可能还有不少地方不符合规范。
