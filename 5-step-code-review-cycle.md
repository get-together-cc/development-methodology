# 5步法（改→测→审→改→绿）——全员通用代码流程规范

> 用户拍板（2026-08-03）：**所有写代码任务**（不限项目/语言/Agent）必须走此循环。
> 适用范围：本机所有 Hermes 会话 + 共享团队所有 Agent（小赫/龙虾/安安/雨）
> 最后更新: 2026-09-04

## 核心定义

```
① 改 → ② 测 → ③ 审 → ④ 改 → ⑤ 绿    （③④ 循环直到无新问题才进 ⑤）
```

## ⚠️ 最重要规则：循环次数不限（但有收敛规则）

- **没有"固定 5 轮"**——轮数由问题数量决定，不设上限
- **终止条件**：某轮只剩误报/无新问题 → 才进 ⑤ 绿
- **禁止提前停**：不得因"跑了几轮差不多了"就宣布全绿
- **每轮结束主动问自己**："这轮还有 critical/major 吗？有就继续，没有才停"

### 🔀 收敛规则（2026-08-04 龙虾建议，用户采纳）

"次数不限" ≠ "无限打磨"。加两条收敛规则平衡：

1. **连续 2~3 轮无实质改动（或只剩风格/优化类意见）→ 停**
   - 每轮记录改动量，连续 2-3 轮无实质改动即视为收敛完成
2. **续循环入场券：只接受正确性 / 安全 / 性能 三类硬问题**
   - 风格类、优化类意见**不构成再开一轮的理由**（记入待办即可，不阻塞交付）

**与"不限次数"的关系**：次数不限但**有实质进展才算数**——只要每轮都在修硬问题就继续；只剩风格意见就停，不无限打磨。

## 五步详解

### ① 改
- 遵守代码同步铁律：先拉最新 → 本地改 → 语法检查 → diff 确认 → 上传
- 一次只改一个根因，不做 drive-by 重构

### ② 测
- **测试标准不确定时必须先问用户确认**（测什么范围/对比什么/什么算通过）
- 实际调用/运行验证：全端点 curl，确认 200 + 产物完整可读
- **与旧版输出对比**（有原版必对比）：Excel 逐单元格、PDF 双标签正则、排序按原版逻辑
- 回归：已验证的基线必须重跑确认没改坏

### ③ 审（审核视角——以 `multi-model-code-review` 技能 09-06 版为唯一权威）

**当前生效矩阵（2026-09-04/06 用户拍板，与 multi-model-code-review 技能一致）**：

| 路 | 模型 | 状态 |
|----|------|------|
| A | DeepSeek Flash `--variant max` | ✅ 必跑（逻辑链 bug + 主动实测）|
| B | FreeToken 35B（Qwen3.6 本地 100.64.0.6:1918）| ✅ 必跑（深度细节/TDZ/命名空间）|
| C | Flash max 测试视角（prompt 模板见 multi-model-code-review references/test-perspective-prompt.md）| ✅ 必跑（数组注入/并发/状态组合，独有发现 ~70%）|
| E | FT 测试视角（FreeToken 35B 版，prompt 模板同 references）| ✅ **必跑第 4 路**（2026-09-06 用户拍板默认启用；抓集成/运维类：API 限流/honeypot/脚本硬编码）|
| D | DeepSeek Pro max | ⏸️ **暂停中**（2026-09-04 用户指令；恢复时取消注释）|

**命令**（cd 代码目录后并行跑，各写独立 log）：
```bash
opencode run -m deepseek/deepseek-v4-flash --variant max '审核prompt' > /tmp/review-flash.log 2>&1 &
opencode run -m freetoken/Qwen3.6-35B-A3B-NVFP4 '审核prompt' > /tmp/review-ft.log 2>&1 &
opencode run -m deepseek/deepseek-v4-flash --variant max "$(cat <测试视角prompt>)" > /tmp/review-test.log 2>&1 &
opencode run -m freetoken/Qwen3.6-35B-A3B-NVFP4 "$(cat <FT测试视角prompt>)" > /tmp/review-ft-test.log 2>&1 &
# Pro max 暂停中：opencode run -m deepseek/deepseek-v4-pro --variant max '...' &
```

- **按模块分批并行审**（1-2 文件/批），一次全量会挂死
- prompt 必须带上一轮已修清单（"第一轮已修复: X/Y/Z。请验证修复是否正确"）
- 要求：中文输出、Severity critical/major/minor + 行号 + 修复建议、只审不改
- **FT 测试视角（黑盒实测）**：不做代码阅读，重点测数组/批量参数注入、越权、限流伪造（X-Forwarded-For）、状态标记绕过、边界值；发现必须带复现证据（请求+响应）；修复后必须重测确认
- ⚠️ 版本归一说明：本文档与 `code-change-review-cycle` 技能 + GitHub `code-review-cycle` 仓库均以 `multi-model-code-review` 技能（09-06）为唯一权威同步。变更史：08-21 3模型 → 08-30 +测试视角 → 09-04 Pro max 暂停(3路) → 09-06 FT测试视角默认启用(4路必跑)

### ④ 改
- 修 critical/major；minor 评估后决定
- **误报判定三件套**：grep 符号（0 匹配）+ 编译通过 + git diff 确认——都过就是误报，不盲改
- **理论建议 vs 实证冲突**：AI 报理论问题时，若与旧版导出已 0 差异 = 行为符合原版，以实证为准不改，加注释说明

### ⑤ 绿（终止条件）
- 某轮只剩误报/无新问题 → 终止
- 全部修复部署后跑最终回归（全端点 + 数据对比基线）→ **全绿才交付**

## 全绿的完整含义（双重）

1. **功能测试全绿**：全端点 200 + 与旧版数据逐单元格 0 差异 + 批量 0 错误 + 服务 0 崩溃
2. **复审全绿**：多轮交叉复审无 critical/major 遗留（某系统 7 轮复审经验）

## 配套纪律

- **新问题点必须固化**：每次遇到新问题 → 修复 → 验证 → 立即加入 TESTING.md 回归清单
- **review 运行期间不要改文件**（AI 读启动时快照，中途修改导致误报）
- **修复后全局 grep 同类模式**（修 1 处还要找漏网）
- **测试文档随流程更新**：每条测试 = 方法 + 预期 + 状态，执行后回填

## 各 Agent 执行要求

| Agent | 要求 |
|------|------|
| 小赫（Hermes mbp） | 技能 `code-change-review-cycle` + 记忆已固化，所有写代码任务自动执行 |
| 龙虾（QClaw air） | 按此文档执行，写代码必须走 5步法，审用 OpenCode |
| 安安 | 涉及代码改动时按此流程，安全审查结果也按 severity 分级 |
| 雨 | 按此文档执行 |

## 验证标准（怎么算执行正确）

- ✅ 循环次数 ≥ 问题所需轮数（不限，直到无新问题）
- ✅ 每轮审核都带上一轮已修清单
- ✅ 误报有判定证据（grep/build/diff 三件套）
- ✅ 最终回归全绿才交付
- ❌ "做了 3 轮差不多了"提前停 = 违规

## 相关文件

- 本机技能: `~/.hermes/skills/software-development/code-change-review-cycle/SKILL.md`
- 审核编排细节: `go-php-hybrid-service` 技能 references/opencode-review-orchestration.md
