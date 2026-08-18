# 勘误与补充记录

> [!IMPORTANT]
> **分层许可证说明**
>
> 自本仓库首次引入 [`LICENSE-NOTES.md`](LICENSE-NOTES.md) 的提交起，
> 本文件中由 `wiFy909` 独立撰写的勘误说明、补充解释和判断采用
> [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)。
> 被勘误文件的原文、代码、命令、事实数据、上游材料和第三方内容不在该许可
> 范围内，继续适用 [Apache License 2.0](LICENSE) 或其各自的许可证。
> 详细范围和历史版本说明见 [`LICENSE-NOTES.md`](LICENSE-NOTES.md)。

这个文件记录学习过程中发现的项目文件勘误或补充。条目只记录已经验证过的内容；命令、路径、配置键和日志原文保持原样。

## 上游版本核对（2026-08-18）

> [!NOTE]
> 已对照上游 `bojieli/ai-agent-book` main 分支最新提交 `5388f951`（大版本迭代后）复核下方两条勘误：
>
> 1. `interactive` 模式说明——**原作者未修复**：上游新版 `README.md` 仍保留旧式"注释在代码块内"的写法，未拆分终端命令与交互输入。你的勘误（下方条目 1）保留有效，且已在本次仓库同步时**三方合并进上游新版 README**（英文 `Start Interactive Mode`、中文 `启动交互模式` 两段，见下文更新后的行号）。
> 2. `--provider` 默认值与 API Key 匹配关系——**原作者部分修复**：上游 `main.py` 的 `--help` 文本已补充环境变量说明（见 [`main.py:1126`](chapter1/context/main.py#L1126)），但 `README.md` 仍未在样例任务段说明默认值匹配关系，这部分仍靠你的勘误（下方条目 2，已合入 README 的 Provider note / Provider 说明）。

## 2026-07-31

### 1. `interactive` 模式说明容易混淆终端命令与模型交互输入

**涉及文件**：[chapter1/context/README.md](chapter1/context/README.md)（英文 `#### 4. Start Interactive Mode` 段 [L205](chapter1/context/README.md#L205)；中文 `#### 4. 启动交互模式` 段 [L716](chapter1/context/README.md#L716)；集成测试说明在 [L178](chapter1/context/README.md#L178)）

**原问题**：`README.md` 的交互模式说明把两类输入放得太近：一类是在普通终端里运行的启动命令，例如 `python main.py --provider kimi --mode interactive`；另一类是在程序进入交互模式后，在 `[KIMI]>` 这类提示符下输入的任务或命令。读者容易在 `[KIMI]>` 里再次粘贴启动命令。

**已验证事实**：

- `python main.py --provider kimi --mode interactive` 应在普通 shell 提示符下运行，用来启动程序并连接 Kimi API。
- 看到 `[KIMI]>` 后，程序已经进入模型交互输入区。这时输入的内容会作为用户消息发给模型，不会被当成本地 shell 命令执行。
- 如需回到普通终端提示符，应先在交互模式里输入 `quit`。
- 代码中的 `interactive_mode()` 也确认了退出命令是 `quit`。

**已修正说明**：

- 在 [Kimi / DeepSeek 集成测试说明](chapter1/context/README.md#L178) 下补充：连接成功后已经进入模型交互提示符，不要再次粘贴启动命令。
- 在 [启动交互模式说明](chapter1/context/README.md#L205) 中拆分普通终端命令和交互模式命令：`bash` 代码块只放启动命令，交互输入改为带说明的表格。
- **2026-08-18 更新**：上述修改已随本次仓库同步合并进上游新版 README（上游自己未修复此点），英文段 L205、中文段 L716。

### 2. `single` 样例任务未说明 `--provider` 默认值与 API Key 匹配关系

**涉及文件**：[chapter1/context/README.md](chapter1/context/README.md)（英文 [Run Sample Tasks](chapter1/context/README.md#L236) 段 Provider note [L252](chapter1/context/README.md#L252)；中文 [运行样例任务](chapter1/context/README.md#L745) 段 Provider 说明 [L761](chapter1/context/README.md#L761)）、[chapter1/context/main.py](chapter1/context/main.py#L1125)

**原问题**：`README.md` 的样例任务段同时给出 `python main.py --mode single` 和 `python main.py --mode single --provider doubao`，但没有说明前者并不会自动使用读者已经配置好的其他提供商 API Key。读者如果只配置了 Kimi、DeepSeek、SiliconFlow 或 Zhipu 等非 Doubao Key，不加匹配的 `--provider` 就会按默认 `doubao` 查找 `ARK_API_KEY`，从而失败；只有另配 `OPENROUTER_API_KEY` 时才会进入 OpenRouter 兜底路径。

**已验证事实**：

- `main.py` 的 CLI 参数把 `--provider` 默认值设为 `doubao`。
- 不指定 `--provider` 时，程序不会扫描已经配置了哪个提供商的 API Key，而是按默认 `doubao` 读取 `ARK_API_KEY`。
- 若默认提供商 Key 缺失但存在 `OPENROUTER_API_KEY`，程序会走 OpenRouter 兜底；两者都没有时退出并提示缺少 API Key。
- 配置了 `MOONSHOT_API_KEY`（也兼容旧变量 `KIMI_API_KEY`）、`DEEPSEEK_API_KEY`、`SILICONFLOW_API_KEY` 或 `ZHIPU_API_KEY` 时，需要显式传入对应的 `--provider kimi|deepseek|siliconflow|zhipu`。

**已修正说明**：

- 在英文 [Run Sample Tasks](chapter1/context/README.md#L236) 段和中文 [运行样例任务](chapter1/context/README.md#L745) 段的代码块下补充 Provider 说明。
- 说明中明确：`--provider` 默认值是 `doubao`；CLI 不会自动选择已经配置 API Key 的提供商；非 Doubao Key 需要显式传入匹配的 `--provider`；OpenRouter 只在配置 `OPENROUTER_API_KEY` 时作为兜底路径。
- **2026-08-18 更新**：上游 `main.py` 的 `--help` 已自行补充环境变量说明（[`main.py:1126`](chapter1/context/main.py#L1126)：*"LLM 提供商（默认：doubao；openrouter 或缺失主 key 时经 OpenRouter 兜底；ollama 为本地免费）"*），即条目 2 的**后半部分（OpenRouter 兜底说明）原作者已修复**；但 README 样例任务段的 Provider note 仍未补充，保留你的勘误（已合并进上游新版 README，英文 L252、中文 L761）。
