# usage-agent

AI 编程工具（Claude Code / Codex / OpenCode）的每日 token 用量与花费统计。

数据由各机器**本地解析自身日志**后发布，网页只读聚合结果。

**站点**：https://functionhx.github.io/usage-agent/

## 这里有什么

```
data/
├── usage.schema.json   JSON Schema（draft 2020-12），数据的契约
├── hosts.json          由 CI 从目录推导的机器清单，不要手工编辑
└── <host>.json         每台机器只写自己这一个文件
scripts/
└── validate_usage.py   零依赖校验：schema + 语义 + 隐私守卫
index.html              零依赖图表页
```

## 只发布聚合数字

数据里**没有项目名、没有文件路径、没有对话内容**。

这不是"过滤掉"的结果——`data/*.json` 的**结构里根本没有这些字段**。schema 用
`additionalProperties: false` 固化了这一点，多写一个字段就会校验失败。

`scripts/validate_usage.py` 里另有一段 `walk_public_values()`，递归遍历所有公开值并拒绝：

- `null`（未知就省略，不要写 null）
- 敏感键名（`project` / `path` / `cwd` / `session` / `prompt` / …）
- **字符串里出现的绝对路径**

为什么要把隐私检查做成断言而不是靠人工审阅：数据是**自动发布**的，没人会逐次检查。
把约束固化进 CI，一旦哪天产出结构变化带出了路径，构建会直接失败而不是静默公开。

## 时区口径

**一律 UTC+8**。`date` 字段是 UTC+8 的日历日，日界落在 **UTC 16:00**。

`updatedAt` 则是 UTC 时刻（带 `Z`）——两者含义不同，不要混。

各机器的系统时区可能不是 UTC+8（比如 `America/Los_Angeles`），所以生成端必须用
固定偏移做日期运算，**不能依赖系统时区**，否则同一份日志在不同环境下会算出不同的日键。

## 各部分怎么工作

**每台机器**跑 `vibe-local publish`，产出 `data/<host>.json`（整份重写，不做增量）。
每台机器只碰自己那个文件，所以多机器同时更新也**不会产生 git 冲突**。

**CI**（`manifest.yml`）在 `data/` 变化时从目录推导出 `hosts.json`，并跑一次校验。
清单由 CI 生成而非各机器写，避免了共享文件的写冲突。提交时带 `[skip ci]` 且先做
`git diff --quiet` 判断，两道防线防止自触发循环。

**页面**读取 `hosts.json` 得到机器列表，再逐个 fetch，合并成一张总表。

## 校验

```bash
python3 scripts/validate_usage.py            # 校验 data/ 下全部
python3 scripts/validate_usage.py 某个文件    # 只校验指定文件
```

零依赖，只用 Python 标准库。

## 相关

- 产数工具：[vibe-local](https://github.com/Functionhx/vibe-local)
- 主站：https://functionhx.github.io/

## License

MIT
