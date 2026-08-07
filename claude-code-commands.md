# Claude Code 指令速查手册(按使用频率排序)

> 本文档覆盖 Claude Code 的斜杠命令、CLI 启动参数、键盘快捷键与输入前缀,按"最常用 → 最不常用"排序。
> 基于官方文档(code.claude.com/docs,2026-08 快照)与本机 `claude --help` 整理。
> 说明:排序依据是典型日常使用场景,非官方统计数据;插件的命令未收录;`/review`、`/cost` 等均为别名。

---

## 目录(按频率分层)

| 层级 | 频率 | 命令 |
|---|---|---|
| 一 | 几乎每次会话 | `/help` `/clear` `/compact` `/context` `/init` `/model` `/config` `/permissions` `/usage` `/memory` `/status` `/copy` |
| 二 | 经常使用 | `/resume` `/rewind` `/diff` `/export` `/mcp` `/hooks` `/login` `/logout` `/keybindings` `/terminal-setup` `/theme` `/color` `/doctor` `/release-notes` `/rename` `/bug` |
| 三 | 代码质量(按需) | `/code-review` `/security-review` `/simplify` `/run` `/verify` |
| 四 | 进阶效率 | `/add-dir` `/cd` `/autocompact` `/fork` `/branch` `/agents` `/skills` `/plan` `/effort` `/fast` `/loop` `/schedule` `/tasks` `/background` `/btw` `/goal` `/focus` `/debug` `/subtask` `/workflows` `/plugin` |
| 五 | 少用 / 一次性 | `/import` `/desktop` `/voice` `/sandbox` `/insights` `/recap` `/install-github-app` `/install-slack-app` `/web-setup` `/remote-control` `/teleport` `/autofix-pr` `/ultrareview` `/ide` `/chrome` `/mobile` `/setup-bedrock` `/setup-vertex` `/usage-credits` `/upgrade` `/exit` 等 |

---

# 第一层:几乎每次会话都用

## 1. `/help`
列出所有可用命令。刚上手时第一个该敲的命令。

```
/help
```

## 2. `/clear`(别名:`/reset`、`/new`)
清空上下文,开启全新对话。上下文快满时最常用。

```
/clear            # 直接新开
/clear <名字>     # 清空并给新会话起名
```

## 3. `/compact`
把当前对话总结压缩,释放上下文空间。比 `/clear` 温和——压缩后 Claude 仍记得要点。

```
/compact                  # 默认压缩
/compact 只保留需求与结论  # 附带压缩方向
```

## 4. `/context`
以彩色网格直观查看当前上下文占用情况,并给出优化建议。上下文管理第一站。

```
/context      # 查看占用
/context all  # 显示全部细节
```

## 5. `/init`
初始化项目:Claude 扫描代码后生成 `CLAUDE.md` 项目指南,后续会话自动加载,大幅提升跨会话一致性。

```
/init
```

## 6. `/model`
切换当前会话的模型,并保存为以后新会话的默认模型。

```
/model                    # 列出可选模型
/model opus               # 切换(如 opus / sonnet / fable 等)
```

## 7. `/config`(别名:`/settings`)
打开设置界面(信任、权限、模型、主题、编辑器、hooks 等),也可直接传 `key=value` 写入。

```
/config                   # 打开设置
/config theme dark        # 直接设置某项
```

## 8. `/permissions`(别名:`/allowed-tools`)
管理工具的 allow / ask / deny 权限规则,是权限弹窗变多的解药。

```
/permissions              # 打开权限管理
```

## 9. `/usage`(别名:`/cost`、`/stats`)
查看本会话费用、用量上限与活动统计。关注预算时必用。

```
/usage
```

## 10. `/memory`
管理长期记忆:查看、编辑记忆条目,开关自动记忆。Claude 会据此在后续会话记住你的偏好。

```
/memory
```

## 11. `/status`
打开设置界面的状态页:版本号、当前模型、账号、连接状态等。排查环境问题先看这里。

```
/status
```

## 12. `/copy [N]`
把最近一条助手回复复制到剪贴板(传数字取倒数第 N 条)。

```
/copy      # 复制上一条回复
/copy 2    # 复制倒数第二条
```

---

# 第二层:经常使用

## 13. `/resume [会话]`(别名:`/continue`)
按 ID 或名字恢复历史会话,或打开会话选择器。隔天继续干活的首选。

```
/resume               # 打开选择器
/resume 20260808-xx   # 恢复指定会话
```

## 14. `/rewind`
把对话(及代码改动)回退到之前的某个节点,或从某条消息重新总结。误操作后悔药。

```
/rewind
```

## 15. `/diff`
打开交互式 diff 查看器,浏览未提交的改动和每一轮对话产生的差异。提交前过一遍的好工具。

```
/diff
```

## 16. `/export [文件名]`
把当前对话导出为纯文本。

```
/export 会话记录.txt
```

## 17. `/mcp`
管理 MCP 服务器连接与 OAuth 认证。

```
/mcp                          # 管理列表
/mcp reconnect <server>       # 重连某个服务器
/mcp enable|disable [server|all]
```

## 18. `/hooks`
查看 hooks 配置(工具事件钩子),可开关、删除。自动化工作流(如每次 Bash 前检查)从这里管理。

```
/hooks
```

## 19. `/login` / `/logout`
登录 / 登出 Anthropic 账号(影响用量计费、云端功能等)。

```
/login
/logout
```

## 20. `/keybindings`
直接打开你的快捷键配置文件(JSON)。

```
/keybindings
```

## 21. `/terminal-setup`
配置终端键位,让 `Shift+Enter` 换行等快捷键在 VS Code、Cursor、Alacritty、Zed 等终端里生效。

```
/terminal-setup
```

## 22. `/theme` / `/color`
`/theme` 切换颜色主题;`/color` 只改当前会话的提示条颜色。

```
/theme
/color blue
```

## 23. `/doctor`(别名:`/checkup`)
环境体检:自动诊断并修复安装/配置问题。异常时先跑一遍。

```
/doctor
```

## 24. `/release-notes`
以交互式版本选择器查看更新日志。

```
/release-notes
```

## 25. `/rename [名字]`
重命名当前会话(会显示在提示条上,方便 `/resume` 时辨认)。

```
/rename 重构登录模块
```

## 26. `/bug [描述]`(别名:`/share`)
报告 bug 或分享当前对话。

```
/bug 工具调用偶尔卡死
```

---

# 第三层:代码质量与评审

## 27. `/code-review`(别名:`/review`)
审查当前工作区改动(或指定 PR / 分支 / 路径),按严重度给出 bug 与清理建议。带等级与自动修复选项。

```
/code-review                # 审查当前改动
/code-review high           # 高强度审查
/code-review --fix          # 审查后直接修复
/code-review 42             # 审查 PR #42
/code-review main           # 审查某分支
```

## 28. `/security-review`
专门针对当前分支改动做安全漏洞审查(注入、密钥泄露、权限问题等)。

```
/security-review
```

## 29. `/simplify`
审查改动代码的复用、简化、效率问题并直接应用修复。只讲质量,不查 bug。

```
/simplify
```

## 30. `/run`
真正启动并驱动项目应用,亲眼验证某个改动是否生效(不只是跑测试)。

```
/run
```

## 31. `/verify`
构建并运行应用,确认改动确实达成了目标。比单跑单元测试更可靠。

```
/verify
```

---

# 第四层:进阶效率

## 32. `/add-dir <路径>`
把另一个目录加入本会话的工具访问范围。

```
/add-dir ../shared
```

## 33. `/cd <路径>`
把当前会话迁移到新工作目录。

```
/cd /Users/me/projects/api
```

## 34. `/autocompact [auto|tokens]`
设置上下文多满时自动压缩。

```
/autocompact auto      # 自动
/autocompact 200k      # 200k tokens 触发
```

## 35. `/fork` / `/branch`
`/fork [提示]` 把对话复制到一个新的后台会话,两边互不影响;`/branch [名字]` 为当前对话开一条分支,尝试另一个方向而不丢失原线。

```
/fork 用另一种方案重做
/branch 激进重构版
```

## 36. `/agents` / `/skills`
`/agents` 提示你让 Claude 创建/管理子代理(或直接编辑 `.claude/agents/`);`/skills` 列出所有可用技能。

```
/agents
/skills
```

## 37. `/plan [描述]`
直接从提示符进入计划模式(先规划再动手)。

```
/plan 重构这个模块
```

## 38. `/effort [等级|auto]`
设置模型的推理努力等级(low → max,以及 ultracode;`auto` 恢复默认)。

```
/effort high
```

## 39. `/fast [on|off]`
开关快速模式(用 Opus 加快输出速度,不降级为小模型)。

```
/fast
```

## 40. `/loop [间隔] [提示]`(别名:`/proactive`)
让同一个提示按间隔自动重复执行,适合轮询、持续跟进类任务。

```
/loop 5m 检查 CI 状态
```

## 41. `/schedule [描述]`(别名:`/routines`)
创建、更新、列出或运行"例行任务"(云端执行)。

```
/schedule 每天早晨跑一遍测试
```

## 42. `/tasks`
查看和管理当前会话的后台工作(含已完成的子代理)。

```
/tasks
```

## 43. `/background [提示]`(别名:`/bg`)
把当前会话转为后台代理继续跑,腾出终端。

```
/background 继续处理剩下的文件
```

## 44. `/btw [问题]`
问个边角小问题,不会污染对话历史。

```
/btw 这个函数的复杂度是多少
```

## 45. `/goal [条件|clear]`
设定目标,让 Claude 跨轮次持续工作直到满足条件。

```
/goal 所有测试通过
/goal clear
```

## 46. `/focus`
切换专注视图(只显示最近提示、工具调用摘要、最终回复),减少刷屏。

```
/focus
```

## 47. `/debug [描述]`
开启调试日志并基于会话日志排查问题。

```
/debug 工具调用超时
```

## 48. `/subtask <任务>`
派发一个继承完整对话的 fork 子代理,在后台并行干活。

```
/subtask 先把这个接口的文档写完
```

## 49. `/workflows`
打开工作流进度视图:观看、暂停、恢复或保存多代理工作流。

```
/workflows
```

## 50. `/plugin`
管理插件(安装、卸载、更新)。

```
/plugin
```

---

# 第五层:少用 / 一次性 / 团队协作

## 51. `/import [codex|gemini] [--dry-run] [--yes]`
把其他编码代理(OpenAI Codex、Google Gemini CLI)的配置迁移进来。

```
/import codex --dry-run
```

## 52. `/desktop`(别名:`/app`)
把当前会话迁移到 Claude Code 桌面应用继续。

```
/desktop
```

## 53. `/voice [hold|tap|off]`
开关语音输入(长按 / 点按说话)。

```
/voice hold
```

## 54. `/sandbox`
切换沙箱模式(命令在受限环境中执行)。

```
/sandbox
```

## 55. `/insights`
生成一份分析报告,统计你的 Claude Code 会话习惯与效率。

```
/insights
```

## 56. `/recap`
随时生成当前会话的一句话总结。

```
/recap
```

## 57. `/install-github-app`
为仓库安装 Claude GitHub App(可顺带配置 GitHub Actions)。

```
/install-github-app
```

## 58. `/install-slack-app`
安装 Claude Slack 应用(打开浏览器 OAuth)。

```
/install-slack-app
```

## 59. `/web-setup`
用本地 `gh` 凭据把 GitHub 账号连接到 Web 版 Claude Code。

```
/web-setup
```

## 60. `/remote-control`(别名:`/rc`)· `/teleport`
`/remote-control` 让当前会话可被 claude.ai 远程控制;`/teleport` 把 Web 端会话拉回终端。

```
/remote-control
/teleport
```

## 61. `/autofix-pr [提示]`
派一个 Web 版会话盯住当前 PR,CI 失败或评审留言时自动推送修复。

```
/autofix-pr 按评审意见修复
```

## 62. `/ultrareview [PR|分支]`
在云端沙箱里跑一次深度的多代理代码评审。

```
/ultrareview
```

## 63. `/ide` · `/chrome`
`/ide` 管理 IDE 集成;`/chrome` 配置 Claude 在 Chrome 里的集成。

```
/ide
/chrome
```

## 64. `/mobile`(别名:`/ios`、`/android`)
显示二维码,下载移动端 Claude 应用。

```
/mobile
```

## 65. `/setup-bedrock` · `/setup-vertex`
配置 Amazon Bedrock / Google Vertex 的认证、区域与模型。

```
/setup-bedrock
```

## 66. `/usage-credits`
配置用量额度(原 `/extra-usage`)。

```
/usage-credits
```

## 67. `/upgrade` · `/passes` · `/stickers` · `/radio` · `/powerup`
升级套餐、送朋友一周免费额度、订购贴纸、打开 lo-fi 电台、通过互动课程发现新功能——都是"锦上添花"型。

```
/upgrade
```

## 68. `/advisor [model|off]`
开启顾问工具:关键时刻让第二个模型提供建议。

```
/advisor sonnet
```

## 69. `/design-login` · `/design-sync`
授权 claude.ai 账号访问设计系统,并把仓库的 React 设计系统同步上传。

```
/design-login
```

## 70. `/tui [default|fullscreen]`
切换终端渲染器并重启。

```
/tui fullscreen
```

## 71. `/statusline`
配置状态栏显示内容。

```
/statusline
```

## 72. `/scroll-speed`
交互式调节鼠标滚轮速度。

```
/scroll-speed
```

## 73. `/reload-plugins` / `/reload-skills`
不重启就重新加载插件 / 重扫技能目录。

```
/reload-skills
```

## 74. `/privacy-settings`
查看并更新隐私设置。

```
/privacy-settings
```

## 75. `/remote-env`
选择云代理的默认环境。

```
/remote-env
```

## 76. `/heapdump`
导出 JS 堆快照,诊断高内存占用。

```
/heapdump
```

## 77. `/exit`(别名:`/quit`)· `/stop`
`/exit` 退出 CLI;`/stop` 停止当前后台会话。

```
/exit
```

---

# 已移除 / 已改名的命令(别再用)

| 命令 | 状态 | 替代 |
|---|---|---|
| `/pr-comments` | v2.1.91 移除 | 直接让 Claude 看 PR 评论 |
| `/vim` | v2.1.92 移除 | `/config` → Editor 模式 |
| `/ultraplan` | 移除 | 计划模式 `/plan` |
| `/extra-usage` | 改名 | `/usage-credits` |
| `/feedback` | 保留,但 `/bug` 已是主入口 | `/bug` |

---

# 输入前缀速查(输入框第一字符)

| 前缀 | 作用 |
|---|---|
| `/` | 命令或技能菜单 |
| `!` | Shell 模式:直接跑命令,输出进会话,Claude 会回应(如 `! git status`) |
| `@` | 文件路径补全 |
| `:` | Emoji 快捷码 |
| `?` | 空输入时按 `?` 开关快捷键帮助面板 |
| `\` + `Enter` | 任意终端里快速换行转义 |

---

# 键盘快捷键(按常用度排序)

## 最常用

| 快捷键 | 作用 |
|---|---|
| `Ctrl+C` | 中断 Claude(空输入时再按一次退出) |
| `Esc` | 中断 Claude / 关闭对话框 |
| `Shift+Enter` | 输入换行(部分终端需先 `/terminal-setup`) |
| `Ctrl+R` | 反向搜索命令历史 |
| `↑` / `↓` 或 `Ctrl+P` / `Ctrl+N` | 浏览命令历史 / 移动光标 |
| `Ctrl+O` | 打开对话记录视图(详细工具调用,再按 `?` 看全屏快捷键) |
| `Shift+Tab` | 循环切换权限模式(默认 / acceptEdits / plan / auto / bypass) |
| `Ctrl+D` | 退出会话 |

## 常用

| 快捷键 | 作用 |
|---|---|
| `Option+P` / `Alt+P` | 切换模型(不清空当前输入) |
| `Option+T` / `Alt+T` | 开关扩展思考 |
| `Option+O` / `Alt+O` | 开关快速模式 |
| `Ctrl+T` | 开关 Claude 的任务清单(待办列表) |
| `Ctrl+B` | 查看后台运行任务(tmux 用户按两次) |
| `Ctrl+L` | 重绘屏幕(快速连按两次 = `/clear`) |
| `Ctrl+X` `Ctrl+K` | 停止所有后台子代理(3 秒内按两次确认) |
| `Ctrl+G` / `Ctrl+X` `Ctrl+E` | 在默认编辑器里打开当前输入 |
| `Esc` `Esc` | 清空输入草稿;空输入时打开回退菜单 |
| `Ctrl+S` | 暂存 / 恢复输入 |
| `Ctrl+Z` | 挂起 Claude Code(Unix,`fg` 恢复) |

## 文本编辑

| 快捷键 | 作用 |
|---|---|
| `Ctrl+A` / `Ctrl+E` | 光标到行首 / 行尾 |
| `Ctrl+K` / `Ctrl+U` | 删到行尾 / 删到行首(存入剪贴板) |
| `Ctrl+W` | 删前一个词 |
| `Ctrl+Y` | 粘贴删除的文本 |
| `Alt+Y` | 循环粘贴历史 |
| `Alt+B` / `Alt+F` | 光标按词移动(前 / 后) |
| `Ctrl+_` / `Ctrl+Shift+-` | 撤销上次输入编辑 |
| `Ctrl+V` / `Cmd+V` | 粘贴剪贴板图片为 `[Image #N]` 引用 |

---

# CLI 启动参数(终端直接调用,按常用度排序)

## 最常用

| 参数 | 作用 |
|---|---|
| `-p, --print "提示"` | 非交互一次性模式,适合脚本/管道:`claude -p "解释这段代码"` |
| `-c, --continue` | 继续当前目录最近一次会话 |
| `-r, --resume [id]` | 恢复指定会话或打开选择器 |
| `--model <model>` | 指定模型(`claude --model sonnet "..."` 或用别名 `fable` / `opus`) |
| `--permission-mode <mode>` | 权限模式:`acceptEdits` / `auto` / `bypassPermissions` / `manual` / `dontAsk` / `plan` |
| `--allowedTools <工具...>` | 允许的工具白名单(如 `Bash(git *) Edit`),配合 `--disallowedTools` 黑名单 |
| `--add-dir <目录...>` | 追加可访问目录 |
| `-n, --name <名字>` | 给会话起名 |
| `--output-format <text|json|stream-json>` | 配合 `-p` 控制输出格式 |
| `--max-budget-usd <金额>` | 本次调用费用上限(`-p` 模式) |

## 常用

| 参数 | 作用 |
|---|---|
| `--system-prompt <提示>` / `--append-system-prompt <提示>` | 自定义 / 追加系统提示 |
| `--settings <文件|json>` | 加载额外设置 |
| `--mcp-config <配置>` | 从 JSON 加载 MCP 服务器 |
| `--agent <agent>` / `--agents <json>` | 指定 / 定义子代理 |
| `--effort <level>` | 推理努力等级(low → max) |
| `--debug [filter]` / `--debug-file <路径>` | 调试模式 / 调试日志文件 |
| `--safe-mode` | 禁用全部自定义项(CLAUDE.md、技能、插件、hooks、MCP 等),排查配置故障 |
| `--from-pr [值]` | 恢复链接到某 PR 的会话 |
| `--fork-session` | 恢复时新建会话 ID,不覆盖原会话 |
| `--fallback-model <model>` | 主模型过载时自动降级(`-p` 模式) |
| `--bg, --background` | 作为后台代理启动,配合 `claude agents` 管理 |
| `--session-id <uuid>` | 指定会话 ID |

## 少用

| 参数 | 作用 |
|---|---|
| `--plugin-dir <路径>` / `--plugin-url <url>` | 加载本地 / 远程插件 |
| `--json-schema <schema>` | 结构化输出校验 |
| `--input-format <text|stream-json>` | 流式输入(配合流式输出) |
| `--ide` / `--chrome` / `--no-chrome` | IDE / Chrome 集成控制 |
| `--teleport [会话]` / `--remote-control [名字]` | Web 端协作 |
| `--tmux` | 为工作树创建 tmux 会话 |
| `--betas <betas>` | 附加 beta 头(仅 API key 用户) |
| `--bare` | 极简模式:跳过 hooks、LSP、插件等 |
| `--dangerously-skip-permissions` | 跳过全部权限检查(仅限可信沙箱) |
| `--disable-slash-commands` | 禁用全部技能 |
| `--safe-mode` 之外: `--strict-mcp-config`、`--setting-sources`、`--file`、`--ax-screen-reader`、`--cloud`、`--prompt-suggestions`、`--no-session-persistence`、`--include-partial-messages`、`--forward-subagent-text`、`--include-hook-events`、`--replay-user-messages`、`--exclude-dynamic-system-prompt-sections` | 流式输出 / 高级场景 |

---

# 附:权限模式一览

| 模式 | 行为 |
|---|---|
| 默认(default) | 敏感操作逐个询问 |
| `acceptEdits` | 自动接受文件编辑,其他操作询问 |
| `plan` | 只规划,不执行(配合 `/plan`) |
| `auto` | 自动接受常见安全操作 |
| `bypassPermissions` | 跳过全部权限检查(谨慎) |

会话中用 `Shift+Tab` 循环切换,或启动时用 `--permission-mode` 指定。

---

*文档生成日期:2026-08-08 · 若命令行为与文档不符,以 `claude --help` 和 `/help` 输出为准。*
