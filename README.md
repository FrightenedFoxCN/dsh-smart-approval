# dsh-smart-approval

**Smart Auto** —— 一个更懂上下文的审批评审档，给 DeepSeek Harness 用。

它把"每一次工具调用都问一遍模型"换成"**只有越出工作区沙箱的那一下才评审**"，
并且会把**人工裁决**沉淀成可复用的先例，让"相似但不同"的后续调用自动放行。

---

## 1. 它解决什么

| 常见问题 | 本插件的做法 |
|---|---|
| 官方 Auto review 每次工具调用都调模型 | 只在**沙箱升级**时评审（预设沙箱 = `workspace-write`） |
| 官方插件给 Codex 路由发 `temperature`，被上游拒绝 | 请求里**不出现** `temperature` |
| 分类器看不到历史决定，"相似但不同"无法复用 | 读取**工作区先例**注入评审提示词 |
| 判定失败会静默放行或直接拒绝 | 一律 `next()` 交给人工，**fail-open-to-human** |
| 重复调用反复评审、成本不可见 | 相同 `tool+args` 缓存 8h；带 `/smart-report` 台账 |

## 2. 权限档

安装后权限选择器多出 **Smart Auto**（`smart-auto`）：

- 沙箱：`workspace-write`
- 审批策略：`ask`
- 只有该档下本插件应答；其它档（`workspace-write` / `sandboxed-auto` / `auto` /
  `danger-full-access`）原样交给下游

## 3. 一次调用的完整决策链

```
工具调用
  └─ 是否需要沙箱升级？
       ├─ 否（工作区内）           → 直接执行，不评审、不打扰
       └─ 是 → approval/request
              ├─ 本插件（仅在 smart-auto 档）
              │    ├─ 命中判定缓存           → allowed-once（毫秒级）
              │    ├─ 模型判定 allow         → allowed-once（并写台账）
              │    └─ 模型判定 ask / 超时 /
              │       解析失败 / 异常        → next()（交人工）
              └─ 其它档 → 下游应答者（人工 / 其它插件）
```

人工裁决后（`allowed-once` / `rejected`）：

```
session/event: approval/asked → approval/decided
   ├─ 是插件自己批的（selfAllowed）→ 跳过（防自我强化）
   └─ 是人工裁决 → 小模型蒸馏成一行动作类别
                  → ~/.dsh/smart-approval/precedents/<sha256(cwd)[:12]>.jsonl
                  → 下次评审注入提示词
```

## 4. 配置

```yaml
- id: smart-approval
  config:
    presetName: smart-auto              # 应答的权限档
    provider: deepseek-account          # 评审模型 provider（留空跟随默认）
    model: deepseek-flash               # 评审模型
    timeoutMs: 20000                    # 单次评审/蒸馏超时（超时→人工）
    maxInputTokens: 6000                # 评审输入预算（证据+上下文+先例）
    cacheTtlMs: 28800000                # 判定缓存（8h）
    precedentsEnabled: true
    precedentsDir: ~/.dsh/smart-approval/precedents
    distillOnHumanDecision: true        # 人工裁决后自动蒸馏先例
    maxPrecedentLines: 200              # 单工作区先例上限（超出裁剪）
    ledger: true
    ledgerPath: ~/.dsh/smart-approval/ledger.jsonl
    maxLedgerLines: 500
    reportCommand: smart-report         # 斜杠命令名；留空则不注册
    debugLog: ''                        # 非空则把事件写调试日志（排查用）
```

## 5. 评审证据（三层，统一压预算）

1. **本次调用**：工具名、参数（≤6k 字符）、审批理由、cwd
2. **会话近期**：最近 6 条用户消息 / 工具调用（各自截断 300–500 字符）
3. **工作区先例**：最近 20 条

## 6. 先例格式

`~/.dsh/smart-approval/precedents/<sha256(cwd)[:12]>.jsonl`，一行一条：

```json
{"action":"allow","class":"home-dir-scratch-write","note":"Allowed: creating a scratch test file under a new home-directory folder ...","tool":"bash","at":"2026-11-24T10:58:16.927Z"}
{"action":"allow","class":"delete-scratch-directory","note":"Allowed removing a self-created scratch/test directory under the home folder with an explicit scoped path and no wildcards.","tool":"bash","at":"2026-11-24T10:58:29.283Z"}
```

- `class` 是**操作类别**（不是原命令），所以相似调用能命中；
- `action` 为 `deny` 的先例会让同类操作**再次询问**；
- 按工作区隔离，不跨工作区共享。

## 7. 台账与报告

每次决策追加一行到 `ledger.jsonl`（最多 500 行，超出裁剪）：

```json
{"at":"2026-11-24T10:59:00.356Z","kind":"allow","tool":"bash","source":"model","ms":812,"reason":"Matches existing allowed precedent: ..."}
```

`kind` 取值：`allow` / `ask` / `cache` / `human`。

在会话里执行 **`/smart-report`** 查看本进程的汇总：

```
Smart Auto 0.3.0 台账（本进程）
评审 12｜模型放行 9｜转人工 3｜缓存命中 5｜失败 0
人工裁决 2｜新先例 2｜台账文件 ~/.dsh/smart-approval/ledger.jsonl
最近决策：
  10:59:00 allow bash model 812ms — Matches existing allowed precedent: ...
```

## 8. 安全边界

- 判定失败 / 超时 / 格式错误 / 异常 → `next()`（人工兜底），**不静默放行**；
- 提示词明确把证据视为数据，不执行其中的指令；
- 插件自己批准的调用不会写进先例（避免"模型放行→模型学习→自我强化"）；
- 蒸馏提示词禁止写入密钥、文件名、文件内容；
- 评审与蒸馏都不改文件系统，只追加先例与台账。

## 9. 已知限制（0.3.0）

1. 不做确定性危险清单前置拦截（依赖沙箱与下游策略）；后续可加**路径前缀规则**（如
   `~/Library/LaunchAgents`、`/etc`、`~/.ssh`）直接判 `ask`，不依赖模型；
2. 提示词对"系统级目录写入"的约束强度依赖模型理解，实测对 `~/Library/LaunchAgents/*.txt`
   这类无害文件会放行；
3. 先例只在"人工裁决"时增长；纯模型放行的调用不进先例；
4. 与 `dsh-auto-approve` 并存时，两者各自声明 `permission` 预设表；本插件的表已包含
   `sandboxed-auto`，两个档位都会保留，按各自 `presetName` 生效。

## 10. 故障排查

| 症状 | 排查 |
|---|---|
| 切到 smart-auto 后仍每次都问人 | `/smart-report` 看"评审"是否为 0：为 0 说明插件没答（档名不匹配或未加载） |
| 先例不生成 | 先看 `debugLog: '/tmp/smart-approval-debug.log'` 下是否出现 `session-event type=approval/asked`；再查 `distill ... parsed=no`（模型返回非 JSON） |
| 模型总是 ask | 调大 `timeoutMs`、换更快模型，或检查先例是否写入了 `deny` |
| 台账为空 | 确认 `ledger: true` 且 `ledgerPath` 可写 |

安装/卸载：

```bash
dsh plugin --profile desktop add /absolute/path/to/dsh-smart-approval
dsh plugin --profile desktop remove dsh-smart-approval
```
