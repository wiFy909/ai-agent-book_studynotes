# 实验复现记录

> [!IMPORTANT]
> **分层许可证说明**
>
> 自本仓库首次引入 [`LICENSE-NOTES.md`](LICENSE-NOTES.md) 的提交起，
> 本文件中由 `wiFy909` 独立撰写的实验过程记录、结果分析、结论和复盘采用
> [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)。
> 实验代码、配置、日志、原始证据、模型输出、事实数据、上游材料和第三方内容
> 不在该许可范围内，继续适用 [Apache License 2.0](LICENSE)
> 或其各自的许可证。详细范围和历史版本说明见
> [`LICENSE-NOTES.md`](LICENSE-NOTES.md)。

## Experiment 1-1：上下文消融实验

本记录来自本机对 Chapter 1 上下文消融实验的复现。实验目标是检查五种上下文模式对同一数值任务的影响，并判断结果是否支持作者关于 history、reasoning、tool calls、tool results 的结论。

### 结果来源

- 复现时间：`2026-07-31T05:14:13.008909+00:00`
- 证据文件：[chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json)
- 最新指针：[chapter1/context/validation/latest.json](chapter1/context/validation/latest.json)，内容与本次 `evidence.json` 一致
- 哈希文件：[chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.sha256](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.sha256)
- `evidence.json` 的 SHA-256：`a5cc38c3dc950a576e158db96109799188f58287fd7c2dd3f8c78cdfce82796d`
- 仓库基线：`personal/main`，commit `ab356891c2470252de7839f18ad1d0b37c213063`，运行时 `worktree_dirty=true`
- 凭据来源：`MOONSHOT_API_KEY`；证据文件没有记录 API Key 值

复现命令：

```bash
python run_experiment_1_1.py --provider kimi --model kimi-k3 --max-iterations 5 \
  --output-dir validation/real_kimi_k3_20260731T051135Z
```

正确答案判定目标来自证据文件中的 `expected_numbers`：

- 年度总额：`9,602,895.73 USD`
- 季度平均：`2,400,723.93 USD`

### 实验任务

任务要求把四个季度收入统一换算成 USD，然后计算年度总额和季度平均。任务明确要求使用工具观测值，不允许模型自行估算汇率。

输入数据：

| Quarter | 原始金额 |
|---|---:|
| Q1 | `2.5 million USD` |
| Q2 | `2.1 million EUR` |
| Q3 | `1.8 million GBP` |
| Q4 | `380 million JPY` |

### 实验结果

| Mode | Success | Correct | Iterations | Tool Calls | Time | 结果摘要 | 来源 |
|---|---:|---:|---:|---:|---:|---|---|
| `full` | true | true | 3 | 4 | 32.95s | 算出 `$9,602,895.73` 和 `$2,400,723.93` | [evidence.json#L42-L809](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json#L42-L809) |
| `no_history` | false | false | 5 | 15 | 57.09s | 到达迭代上限，没有最终答案；重复调用换汇工具 | [evidence.json#L813-L2056](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json#L813-L2056) |
| `no_reasoning` | true | true | 3 | 4 | 29.53s | 算出 `$9,602,895.73` 和 `$2,400,723.93` | [evidence.json#L2060-L2829](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json#L2060-L2829) |
| `no_tool_calls` | true | false | 1 | 0 | 12.72s | 只说会调用工具计算，实际没有工具行动，也没有数值答案 | [evidence.json#L2833-L2932](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json#L2833-L2932) |
| `no_tool_results` | true | false | 2 | 3 | 24.73s | 调用了换汇工具，但工具结果被隐藏；最终承认无法计算 | [evidence.json#L2936-L3431](chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json#L2936-L3431) |

注意：这里的 `success` 只表示 agent loop 返回了 `final_answer` 文本，不等于任务数值正确。正确性以 `behavior.canonical_answer_correct` 为准。

### 我的分析

这次复现实验支持一个更窄的结论：在这道固定换汇和聚合任务里，保留上一轮的 reasoning trace 不是必要条件。`no_reasoning` 与 `full` 一样完成了 4 次工具调用，得到同样的正确数值，而且耗时略低。

但这不等于模型完全不需要 reasoning。Kimi K3 在 `no_reasoning` 模式下仍然产生了 reasoning tokens；该模式移除的是历史消息里的 `reasoning_content`，不是关闭模型内部推理能力。因此更准确的说法是：这道任务不依赖把推理轨迹作为后续上下文保留。

`no_tool_calls` 和 `no_tool_results` 的 `success=true` 容易误读。它们都有 `final_answer`，所以被框架标为成功；但两个结果都没有包含目标数值，因此 `canonical_answer_correct=false`。这说明“有最终回答”不能替代“完成任务”，尤其是在数值任务和工具任务中。

`no_history` 的失败比较明确。模型反复调用同一组换汇工具，到 5 次迭代上限仍没有给出最终答案。这支持作者关于 history 对避免重复行动和维持进度很重要的判断。

### 对作者结论的修正或补充

1. 关于 `reasoning`：作者的“移除 reasoning 必然导致矛盾或错误”，这次复现不完全支持。可改为：复杂任务中 reasoning trace 可能帮助规划，但对于步骤清楚、工具输出确定的换汇聚合任务，保留 reasoning trace 未必带来正确性收益。

2. 关于 `tool calls`：移除工具定义后，模型没有工具行动。表面 `success=true` 只是因为它输出了一句计划性回答；实际没有完成任务。因此工具调用能力仍是任务完成的必要条件。

3. 关于 `tool results`：移除工具结果后，模型能发起工具调用，但无法读取观测结果。这次它没有重复调用工具，而是停止并说明无法计算。作者若写成“必然重复行动”，可改为“可能重复行动，也可能承认缺少观测；共同点是无法得到可验证的正确数值答案”。

4. 关于实验指标：复现实验应同时报告 `success` 和 `canonical_answer_correct`。只看 `success` 会把 `no_tool_calls` 与 `no_tool_results` 误判为任务成功。

### 快速复查命令

查看五个实验臂的完整答案和正确性：

在仓库根目录运行：

```bash
jq '.arms[] | {mode, success, correct: .behavior.canonical_answer_correct, final_answer}' \
  chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json
```

校验证据文件哈希：

在仓库根目录运行：

```bash
shasum -a 256 chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.json
cat chapter1/context/validation/real_kimi_k3_20260731T051135Z/evidence.sha256
```
