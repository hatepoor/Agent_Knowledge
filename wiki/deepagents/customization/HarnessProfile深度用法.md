---
tags: [deepagents, customization, harness-profile, models]
created: 2026-07-29
updated: 2026-07-29
type: concept
deepagents_version: ">=0.7"
---

# HarnessProfile 深度用法

不同 LLM 能力差异大——Claude 擅长长上下文，GPT 擅长工具调用。HarnessProfile 让你**声明式**为不同模型定制行为，无需 `if/else`。

## 一、核心字段

```python
from deepagents import HarnessProfile, register_harness_profile

@register_harness_profile
class MyModelProfile(HarnessProfile):
    # ── Prompt 定制 ──
    base_system_prompt: str = "你是安全助手，始终检查输入。"    # 完全替换默认 prompt
    system_prompt_suffix: str = "注意：你没有执行权限。"         # 追加到末尾

    # ── 工具控制 ──
    excluded_tools: frozenset = frozenset({"execute", "write_file"})

    # ── 中间件注入 ──
    extra_middleware: list = [MyCustomMiddleware()]
```

## 二、字段说明

| 字段 | 作用 | 示例 |
|------|------|------|
| `base_system_prompt` | **完全替换**主 Agent 的默认 prompt | 定制角色和规则 |
| `system_prompt_suffix` | **追加**到已有 prompt 末尾 | 添加模型特定约束 |
| `excluded_tools` | 隐藏指定工具（中间件不删除） | 不给弱模型 `execute` 权限 |
| `extra_middleware` | 注入模型特定的中间件 | Anthropic 用 Prompt Caching |
| `general_purpose_subagent` | 自定义通用子代理的行为 | 不同模型的子代理 prompt |

## 三、注册与匹配

```python
register_harness_profile(
    "claude-sonnet-4-6",           # 模型标识符
    HarnessProfile(
        base_system_prompt="你是 Claude 驱动的技术助手。",
        system_prompt_suffix="始终用中文回答。",
    ),
)

register_harness_profile(
    "deepseek-v4-flash",            # 另一个模型
    HarnessProfile(
        base_system_prompt="你是 DeepSeek 驱动的代码助手。",
        excluded_tools=frozenset({"execute"}),  # DeepSeek 不给执行权限
    ),
)

# 框架自动根据 model 参数匹配对应 Profile
# 不匹配 → 用默认行为
```

## 四、按模型隐藏工具

```python
# ❌ 不要试图移除 FilesystemMiddleware（会被拒绝）
# ✅ 用 excluded_tools 隐藏工具

register_harness_profile(
    "anthropic:claude-sonnet-4-6",
    HarnessProfile(
        excluded_tools=frozenset({
            "ls", "read_file", "write_file",
            "edit_file", "delete", "glob", "grep"
        }),
    ),
)
# 中间件仍在栈中（框架需要），但工具不对模型暴露
```

## 五、子代理的 Profile 覆盖

```python
from deepagents import GeneralPurposeSubagentProfile

register_harness_profile(
    "claude-sonnet-4-6",
    HarnessProfile(
        base_system_prompt="你是主 Agent。",
        general_purpose_subagent=GeneralPurposeSubagentProfile(
            system_prompt="你是子代理，引用来源，返回 JSON。",
        ),
    ),
)
# 主 Agent 用 base_system_prompt
# 通用子代理用 general_purpose_subagent.system_prompt
```

## 六、相关页面

- [[deepagents/customization/自定义概述]] — 其他定制方式
- [[deepagents/production/源码架构]] — profiles/ 目录结构
- [[deepagents/middleware/中间件栈概述]] — excluded_tools 不影响中间件
