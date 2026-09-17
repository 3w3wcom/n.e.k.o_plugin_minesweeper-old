### ！！！
### 注意： 该文件中的文本/注释 完全由ai编写，请注意辨别
### ！！！


# 合作扫雷 (minesweeper)

内置扫雷（`game\minesweeper.py`）会把关键事件转述给 N.E.K.O. 的陪伴角色，让她像在和你一起玩。
游戏本体与运行环境**随插件一起打包**，别人只需安装插件即可开始游戏（无需自装 Python）。

## 结构

```
minesweeper\                   插件本体（打包对象，约 34 MB）
├── plugin.toml
├── __init__.py
├── Ai README.md
└── game\
    ├── minesweeper.py          游戏本体（唯一源码）
    ├── minesweeper_plugin.py   游戏侧发送端
    ├── window.json             自动生成：记录四个窗口上次关闭时的位置
    └── python\                 随包内置的便携 Python（含 tkinter/tcl）
```

> `window.json` 由游戏在关闭各窗口时自动写入（`main`/`rank`/`help`/`dev` 各一格），
> 下次打开时恢复位置；删掉它即可重置为默认位置。

> `game\minesweeper.py` 就是**唯一源码**，没有另行维护的副本——要改游戏直接改这里即可。

## 启动游戏

安装插件后，**对角色说“启动扫雷”（或“开始扫雷 / 来一局 / 开一局扫雷”）**，
她会调用工具 `start_minesweeper` 打开扫雷窗口；也可以在插件详情 → **入口点** 手动触发。

- 插件用 `subprocess` 启动 **`game/python/python.exe game/minesweeper.py`**，
  **不在宿主进程里创建窗口**（N.E.K.O. 的冻结运行时缺 Tcl 脚本库，直接建 Tk 会让插件进程崩溃）。
- 事件通过 `127.0.0.1:39001` 回环发回插件（游戏是客户端，插件是服务端）。
- 插件停止或重载时，会终止游戏子进程。
- 若启动失败，`start_minesweeper` 会返回 `is_error: true` 与原因（子进程 stderr 尾部）。

## 加载方式

N.E.K.O. 只扫描两个插件根：

| 根 | 路径 |
|---|---|
| 用户安装根 | `%LOCALAPPDATA%\N.E.K.O\.neko-plugin-installations\plugins\<id>\` |
| 内置插件根 | `<游戏目录>\resources\bin\plugin\plugins\<id>\` |

把**整个** `minesweeper` 文件夹（**要连 `game\python\` 一起**）复制到上面任一位置的
`minesweeper\` 下，然后在插件管理器点 **重载全部**（或重启 N.E.K.O.）。

> 注意：不要把整个开发目录拖进“导入插件包”，包根目录必须直接含
> `plugin.toml`（否则会报 `nestedRoot` / `manifestMissing`）。要生成可拖入导入界面的
> `.neko-plugin`，见下方“打包”。

## 打包成 .neko-plugin（导入界面用）

插件在列表中出现后：

1. 左侧勾选 **合作扫雷(自带迷你版扫雷)**
2. 右侧“包管理” → **检查 / 校验**
3. → **构建**（产物在 `%LOCALAPPDATA%\N.E.K.O\.neko-plugin-packages\`，自带 `manifest.json` 与哈希）
4. 把生成的 `.neko-plugin` 拖入 **导入插件包** → **导入**
5. 需要的话再点 **安装**

## 验证 / 诊断

1. 触发 **start_minesweeper**：应返回 `started: true, pid: ...` 并弹出游戏窗口。
   若失败会返回 `is_error: true` 与原因（子进程 stderr 尾部）。
2. 对角色说“启动扫雷”，她应调用工具并打开游戏。
3. 触发 **bridge_status**：可读 `port / listening / summary / last_event /
   game_running / game_pid / game_error`。
4. 玩一局，随机触发下列事件。
5. 游戏窗口菜单栏点 **帮助**：查看玩法提示与实时「插件状态」。

## 事件与行为

| 事件 | 触发 | 处理 |
|---|---|---|
| `state` | 每次 AI 回合 | 更新缓存里的棋盘（不推送） |
| `new_game` | 开局 | **respond**「新的一局扫雷开始了，你和人类一起排雷：9x9，共 10 颗雷。」 |
| `opening` | 开局第一手翻开 | 缓存记事「**你/人类**开局翻开了 N 格」 |
| `big_open` | 一次翻开 ≥2 格 | 缓存记事「**你/人类**翻开了 N 格」 |
| `risky_open` | 翻开数字 ≥3 的格子 | 缓存记事「**你/人类**翻出了数字 K」 |
| `streak` | 连续安全格 ≥2 | 缓存记事（当前累计值） |
| `player_flag` | 人类插旗 | 缓存记事 |
| `ai_doubt` | 你起疑 | **respond** |
| `ai_unflag` | 人类拔你的旗 | **respond** |
| `endgame` | 剩 ≤3 格未翻开 | **respond** |
| `idle` | 人类 45s 无操作 | 缓存记事 |
| `mine_hit` | 踩雷 | 缓存记事／与 `lose` 合并时随 `lose` |
| `win` | 胜利 | **respond**「胜利，用时 N 秒」（与 `new_record` 合并） |
| `lose` | 失败 | **respond**「…踩到雷了，本局结束，标记正确 N 个雷」（与 `mine_hit` 合并；**不报秒数**） |
| `new_record` | 破最佳 | 随 `win` 合并 |

所有推送用 `visibility=[]`：原始事件不显示在聊天里，只进入她的上下文。
文案为客观陈述，视角约定：游戏里的 AI 队友 = 「你」，另一方默认称「人类」
（称呼由 `__init__.py` 的 `HUMAN_LABEL` 控制，可改成「对方」「玩家」「主人」等）。

**归属感设定**（A/B/C）：
- `new_game` 明确点出「你和人类一起排雷」；
- 记事里的动作带上主语（`_who()` → 「你」/「人类」）；
- 每局第一次汇总额外带一行「（本局：你和人类合作排雷）」。

### 省上下文设计（缓存 + 按需拉取）

**平时什么都不推送。** 插件把最近 **2 步**（每步 = 一个 0.35s 窗口的事实短句）与
最新**棋盘**缓存在本进程内存里；更早的步骤直接丢弃（棋盘本身是累积状态）。
**新开一局会清空缓存**，避免串上一局的内容。

**白名单说话事件**（开局 / 起疑 / 拔旗 / 残局 / 胜负）触发时，只发那一句 `respond`：

```
[respond] 人类把你插在第9行第5列的旗拔掉了。
```

- 该推送带 `coalesce_key="ms_speak"`：队列中同 key 的消息**最新覆盖旧的**，避免堆积。
- 推送带 `target_lanlan`（从调用上下文捕获的角色名；未捕获到则为 None）。
- 战况汇总**不主动推送**：她可用 **`@llm_tool` 工具 `minesweeper_status`** 在生成回复时
  **按需拉取**（返回最近两步 + 当前棋盘），无时序风险。
- 汇总推送的实现 `_push_summary()` 仍保留，但在 `_flush_pending` 里**被注释停用**；
  需要恢复时取消注释即可（`SUMMARY_DEDUP_SECONDS` 等一并保留）。
- **不读取对话内容**，因此插件描述里无需附加隐私声明。
- 重要度见 `_RANK`；合并特例 `mine_hit`+`lose`、`win`+`new_record`。

## 调参

游戏侧 `minesweeper.py` 顶部常量：

```python
NEKO_OPEN_MIN = 2               # 一次翻开 ≥ 此格数就写进记事
NEKO_RISKY_NUMBER = 3           # 高风险数字阈值
NEKO_STREAK_MIN = 2             # 连续安全格 ≥ 此值才写
NEKO_ENDGAME_SAFE_LEFT = 3      # 残局提示阈值
NEKO_IDLE_SECONDS = 45          # 挂机提醒秒数
```

插件侧 `minesweeper/__init__.py` 可调：`COALESCE_WINDOW`（合并窗口）、
`CACHE_STEPS`（缓存步数，默认 2）、`SUMMARY_DEDUP_SECONDS`（汇总去重秒数）、
`_SPEAK`（说话白名单）、`HUMAN_LABEL`（称呼）。

## 端口

默认 `127.0.0.1:39001`。被占用时同时改两处：

- `minesweeper/plugin.toml` 的 `[minesweeper] port`
- `minesweeper_plugin.py` 的 `PORT`

## 隐私

- 事件只在 `127.0.0.1` 回环传输，不落盘、不联网。
- 但经 `push_message` 后，文本会进入对话，也就是会发给你在 N.E.K.O. 里配置的
  模型服务商（除非使用本地模型）。因此事件只含棋盘坐标与胜负等通用信息。
