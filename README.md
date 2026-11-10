# Claude Fable 5.2 中转 API：国内直连 Anthropic 前沿模型，5.2 和 5-1 怎么切

Fable 5.2 是 Anthropic 2026 年 11 月 9 日发布的两个前沿模型之一（另一个是 Mythos 5.2）。官方给它的定位很明确：不是日常主力，是啃硬骨头的那一档——开放式的技术难题、跨语言的软件迁移、长周期无人值守的重构。日常写代码和知识工作，官方建议还是先用 Opus 5.5。

官方 `api.anthropic.com` 在国内连不上，这个仓库说的是怎么通过 Token173（https://token173.net）的国内入口调 Fable 系列，Anthropic 原生协议和 OpenAI 兼容协议两条路都能走。

## 先说清楚现在的状态

Fable 5.2 刚发布，站点上的模型清单还在同步。写这篇的时候，Token173 上能直接调的是：

| 模型 ID | 状态 |
|---|---|
| `claude-fable-5-1` | 现在就能用，5.2 的上一代 |
| `claude-fable-5` | 现在就能用，更早一版，便宜 |
| `claude-fable-5-2` | 5.2 上架后的调用名，以 `/v1/models` 或后台模型页的实际名称和单价为准 |

同一个底层模型，Fable 版带网络安全和生物安全护栏，所以能对所有用户开放；Mythos 版是去掉部分护栏的版本，只对通过验证计划的机构开放。这一点决定了你在中转站上基本只会看到 Fable 系列。

## 5.2 相对 5.1 改了什么

| 项目 | Fable 5.1 | Fable 5.2 |
|---|---|---|
| 官方定价 | — | 输入 $8 / 百万 tokens，输出 $40 / 百万 tokens |
| 快速模式 | 无 | 输入 $16 / 输出 $80，token 生成速度最高提升 2.5 倍 |
| 输出速度 | 基准 | 比 5.1 快最多 30% |
| FrontierCode 2.0 | 32.4%（Xhigh） | 49.1%（Xhigh） |
| FrontierSWE v2 | 56.3% | 69.9% |
| AutomationBench | 31.4% | 51.4% |
| Humanity's Last Exam（带工具） | 65.6% | 74.2% |
| OSWorld 2.1（计算机操作） | 83.5% | 89.3% |
| Chartography（读图表） | 44.8% | 73.4% |
| BenchCAD 2.1（硬件图纸） | 44.4% | 83.5 |
| WANDR（大任务检索，relaxed） | 68.7% | 82.1% |

同样的难度，5.2 用的 token 更少。官方给的对照是：FrontierCode 2.0 上，5.2 用 High 档的分数和 Opus 5.5 的最高档基本持平，单任务成本大约是一半；拉到 Xhigh 档，49.1% 是已发布模型里的最高分。

一个可参考的真实案例：Anthropic 内部让 1000 个 agent 协作，30 小时把费马大定理完整形式化到 Lean，人类干预很少；同样配置下 Opus 5.5 在 18 小时只完成 26%，之后基本不动了。

## 什么时候该用它

| 你的情况 | 选哪个 |
|---|---|
| 日常写代码、改 bug、写文档 | `claude-opus-5-5`，别浪费钱 |
| 要迁移一个跑了五年的大型服务（比如 Python 到 Rust） | `claude-fable-5-2` |
| 需求本身就是模糊的，要靠模型自己想清楚再动手 | `claude-fable-5-2` |
| 要跑十几个小时没人盯着的大任务 | `claude-fable-5-2` + 高 effort 档 |
| 让便宜模型遇到难题时回来问它 | 把 Fable 当 advisor |
| 让 Fable 拆任务、分给便宜模型干活 | 把 Fable 当 orchestrator |

## Anthropic 原生协议调用

```bash
curl https://token173.net/v1/messages \
  -H "x-api-key: sk-你的令牌" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-fable-5-1",
    "max_tokens": 2048,
    "messages": [
      {"role": "user", "content": "给这个遗留 Flask 项目出一份迁移到 FastAPI 的分阶段方案，先别写代码"}
    ]
  }'
```

注意 Anthropic 协议走 `x-api-key` 头，不是 `Authorization: Bearer`；`max_tokens` 必填，不填直接报错。

## OpenAI 兼容协议调用

不想改代码的话走这条，SDK 不用换：

```python
from openai import OpenAI

client = OpenAI(api_key="sk-你的令牌", base_url="https://token173.net/v1")

r = client.chat.completions.create(
    model="claude-fable-5-1",
    messages=[{"role": "user", "content": "把这个模块的竞态条件找出来，并给出最小改动方案"}],
    max_tokens=4096,
)
print(r.choices[0].message.content)
```

5.2 上架后，把 `model` 换成 `claude-fable-5-2` 就行，其余不动。切换前先跑一次 `GET /v1/models` 确认名字拼写和单价，别照抄文档。

## Claude Code 里用

两个环境变量，和接 Opus 一样：

```bash
export ANTHROPIC_BASE_URL="https://token173.net"
export ANTHROPIC_AUTH_TOKEN="sk-你的令牌"
claude
```

进去之后切模型：

```
/model claude-fable-5-1
```

`ANTHROPIC_BASE_URL` 这里不带 `/v1`，和写代码时填的 base_url 不是同一个东西，这点最容易接错。

## 关于 effort 档位

Fable 的分数和成本都跟 effort 档绑定。官方数据里，同一个模型从 Low 拉到 Xhigh，分数能差十几个点，成本差好几倍。所以别一上来就开最高档：

- 先 Medium 跑一遍，看结果够不够
- 不够再升 High
- Xhigh 留给真的卡住的情况，一次调用可能比跑十次 Medium 还贵

## 计费

账户 2 元 = 1 美元额度，注册送 0.2 美元，请求失败不计费。Fable 属于高价档，先拿小任务试单价再上量。中文长文、便宜跑量的活交给 `deepseek-v4-pro`、`glm-5.3-flash` 那一档，没必要全用前沿模型。

## 常见问题

**Q：5.2 现在能调吗？**
站点清单里暂时只有 `claude-fable-5-1` 和 `claude-fable-5`。上架后按 `claude-fable-5-2` 调用，名字以 `/v1/models` 返回的为准。

**Q：Fable 和 Mythos 有什么区别？**
同一个底层模型，护栏不同。Fable 带安全护栏，所有用户可用；Mythos 部分能力更强，但只对通过验证计划的机构开放，中转站一般拿不到。

**Q：返回 401 / 403 是什么原因？**
401 是 key 错或没带对请求头（Anthropic 协议要 `x-api-key`）；403 多半是分组权限没开这个模型，去后台看令牌的分组设置。

**Q：流式输出支持吗？**
支持，OpenAI 协议加 `stream: true`，Anthropic 协议加 `"stream": true`，两边事件格式不一样，客户端要分开处理。

**Q：能传图吗？**
能，Anthropic 协议用 `image` 块，base64 或 URL 都行；OpenAI 协议用 `image_url`。图表、截图这类识别是 5.2 明显强于 5.1 的地方。

## 相关

- 官方发布说明：https://www.anthropic.com/claude-fable-and-mythos-5-2
- 文档站：https://docs.token173.net
- Claude 全系接入：[claude-api](https://github.com/Token173/claude-api)
- 多模型聚合：[2026-Token173-Claude-ChatGPT-Grok-Gemini-AI-API](https://github.com/Token173/2026-Token173-Claude-ChatGPT-Grok-Gemini-AI-API)
