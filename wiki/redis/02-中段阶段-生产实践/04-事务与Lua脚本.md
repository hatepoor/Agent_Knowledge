---
tags: [redis, 中段, 事务, lua, 生产]
type: guide
created: 2026-10-01
updated: 2026-10-01
---

# 事务与 Lua 脚本

## 学习目标

读完本篇，你应当能够：

1. 准确说出 Redis 事务保证什么、**不**保证什么（不回滚是设计而非缺陷）
2. 用 `WATCH` + `MULTI`/`EXEC` 实现乐观锁（CAS），并解释它与数据库事务的本质差异
3. 用 Lua 脚本把"检查 + 操作"组合成原子单元（以释放分布式锁为例）
4. 在 pipeline / 事务 / Lua 三者间正确选型

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[redis/01-入门阶段-基础教程/第01章-键与五种核心数据类型/1.3-String字符串]] | 单命令原子性（`INCR`、`SET NX`）的已有认知 |
| [[redis/01-入门阶段-基础教程/第03章-Python客户端redis-py/3.2-操作五种数据类型]] | pipeline 的基础用法与"默认非事务"提示 |

**门槛声明**：不要求读过其他中段专题。

---

## 一、为什么单命令原子还不够（生产视角）

单条命令是原子的（单线程保证）。但一个业务动作往往需要**多条命令**，命令之间就可能被别的客户端插队：

```
客户端 A：GET stock        → 得到 1
客户端 B：GET stock        → 得到 1
客户端 B：SET stock 0      # B 抢走了最后一份
客户端 A：SET stock 0      # A 也卖了最后一份 → 超卖
```

`MULTI`/`EXEC` 与 Lua 脚本就是两种把**多条命令打包成不可分割单元**的机制。

---

## 二、MULTI / EXEC：先入队，后执行

```
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET stock 0
QUEUED
127.0.0.1:6379(TX)> INCR sales
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) (integer) 1
```

机制：`MULTI` 后所有命令**只入队不执行**（返回 `QUEUED`），`EXEC` 一次性串行执行——期间**任何其他客户端的命令都无法插入**。

### 事务保证什么、不保证什么（必背）

| 保证 | 不保证 |
|------|--------|
| ✅ **串行**：EXEC 执行期间不被插队 | ❌ **不回滚**：执行阶段某条命令失败（如对 List 做 `INCR`），其余命令照常执行 |
| ✅ **顺序**：按入队顺序执行 | ❌ **不隔离读**：事务执行中其他客户端看不到中间态（这点其实是保证），但**入队时读不到未来值** |
| ✅ 入队阶段语法错误 → 整个事务放弃（EXEC 报 EXECABORT） | |

**不回滚是官方设计**：执行阶段的失败被视为程序 bug，回滚救不了 bug，只会掩盖它。这层认知直接影响你的代码——**不能**像用 SQL 事务那样"失败了指望回滚"。

### WATCH：乐观锁（CAS）

```
WATCH stock          # 盯住 stock
GET stock            # 读当前值，业务上判断
MULTI
SET stock 0
EXEC                 # 若 stock 在 WATCH 之后被任何人改过 → EXEC 返回 nil，事务不执行
```

`EXEC` 返回 `(nil)` 表示"盯着的东西变了，这次白忙"——正确姿势是**重试整个循环**（这就是乐观锁：不改阻塞，冲突了重来）。

---

## 三、Lua 脚本：更强的原子组合

脚本在服务端**作为一个命令**执行，期间不可插队——原子性比事务更强（事务只是"打包"，脚本还能**根据中间结果分支**）。

```lua
-- 释放锁：只有锁的持有者才能删（GET 判断 + DEL 两步合一）
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

调用：

```
EVAL "脚本内容" 1 lock:job1 owner-abc123
      ↑      ↑       ↑
   脚本   key 个数   KEYS/ARGV 参数
```

| 命令 | 作用 | 版本 |
|------|------|------|
| `EVAL 脚本 numkeys key... arg...` | 执行脚本 | 古老 |
| `EVALSHA sha1 numkeys key... arg...` | 按缓存摘要执行（省网络传输） | 古老 |
| `SCRIPT LOAD` / `SCRIPT EXISTS` | 预载脚本 / 查摘要是否存在 | |
| `FUNCTION LOAD` / `FCALL` | 持久化函数库（可带元数据），脚本管理的新形态 | `Redis 7.0+` |

### 为什么"判断 + 删除"必须用脚本

`GET lock` 看是自己的 → 期间过期被别人抢走 → `DEL` 删掉了**别人的锁**。这个"检查与动作之间被插队"的竞态，用任何两条命令都修不掉，只有脚本（或事务 + WATCH）能闭合。Lua 是**所有分布式锁实现的地基**（见 [[redis/02-中段阶段-生产实践/06-分布式锁]]）。

### redis-py 中的用法

```python
# 1) pipeline 以事务形态执行（MULTI/EXEC 打包）
pipe = r.pipeline(transaction=True)      # transaction=True 是默认值！
pipe.set("stock", 0)
pipe.incr("sales")
pipe.execute()                            # [True, 1] ← 打包 + 不被插队

# 2) WATCH 乐观锁
with r.pipeline() as pipe:
    while True:
        try:
            pipe.watch("stock")           # WATCH 之后 pipeline 进入立即执行模式
            current = pipe.get("stock")
            if int(current or 0) <= 0:
                pipe.unwatch()
                break
            pipe.multi()                  # 切回事务模式
            pipe.set("stock", int(current) - 1)
            pipe.execute()                # 冲突则抛 WatchError
            break
        except redis.WatchError:
            continue                      # 别人改了 → 重读重试

# 3) Lua 脚本（register_script 自动处理 EVALSHA 回退 EVAL）
UNLOCK = r.register_script("""
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
""")
UNLOCK(keys=["lock:job1"], args=["owner-abc123"])
```

---

## 四、三者选型决策表

| 需求 | 用什么 | 理由 |
|------|--------|------|
| 只是省网络往返，命令间被插队无所谓 | **pipeline（transaction=False）** | 最快最简单 |
| 多条命令必须连续执行，但都是"固定命令" | **pipeline（transaction=True）**（MULTI/EXEC） | 打包 + 不插队 |
| 需要**看中间结果再决定下一步**（读改写竞态） | **Lua 脚本** | 脚本内可分支 |
| 读改写但想用"读到的旧客户端代码"做复杂判断 | **WATCH + 事务** | 乐观锁，判断逻辑留在 Python（更好调试），代价是冲突重试 |

---

## 五、生产实践要点

1. **脚本必须快**：脚本执行期间主线程停摆，等价于一条慢命令（复习入门的 `KEYS` 教训）。循环百万次的脚本 = 全服务卡死。脚本超时有保护（默认 5 秒后开始拒绝新命令并允许 `SCRIPT KILL`，但已发起写操作的脚本**杀不掉**，只能等它跑完或 `SHUTDOWN NOSAVE`）。
2. **EVALSHA 的 NOSCRIPT 处理**：重启用后脚本缓存清空，`EVALSHA` 报 NOSCRIPT——redis-py 的 `register_script` 已自动处理（先 EVALSHA，NOSCRIPT 时回退 EVAL），直接用它，不要手写 EVALSHA。
3. **脚本复制方式**：`Redis 5.0+` 默认**复制脚本执行效果**（effect replication）而非脚本本身，解决了"脚本内含随机值/时间导致主从不一致"的老问题。但脚本仍应写成**确定性**的（同样的 KEYS/ARGV 出同样的写效果）。
4. **事务/脚本里的键要显式列在 KEYS**：这是集群模式的硬要求（键必须同 slot 才能打包，见 [[redis/02-中段阶段-生产实践/08-高可用-主从哨兵集群]]），也是静态分析工具的约定。多 key 事务在集群上要用 hash tag `{user1000}` 强制同 slot。
5. **ACL 收敛**（`Redis 6.0+`）：可对用户禁 `@scripting` 或限制 EVAL，防止应用侧滥用长脚本。

---

## 六、陷阱

1. **把 MULTI/EXEC 当 SQL 事务**：指望回滚、指望"全有或全无"的**执行失败**保护——执行阶段失败不回滚。能入队期发现的语法错误才会整体放弃。
2. **`WATCH` 后忘了 `UNWATCH`**：放弃路径不释放 watch，连接的 watch 状态会残留（`EXEC`/`DISCARD` 会自动清；单独放弃用 `UNWATCH`）。
3. **在 pipeline（立即执行模式）里混用阻塞命令**：`BLPOP` 等阻塞命令进 pipeline 行为不直观，不要混。
4. **脚本里写死逻辑忘记 KEYS/ARGV 分离**：键名硬编码进脚本体，集群模式直接报错（跨 slot），应用侧也无法复用。
5. **用事务/Lua 做"长流程编排"**：Redis 的原子单元是毫秒级的，几十条命令的"大事务"风险（超时、半途崩溃）高。长流程用应用层状态机 + 幂等，Redis 只存状态。
6. **`transaction=True` 的 pipeline 与真事务混淆**：它确实是 MULTI/EXEC；但 `transaction=False` 只是网络打包——两种语义差一个原子性，调用点必须注释清楚。

---

## 小结

- 事务 = 打包 + 串行 + **不回滚**；入队错整弃，执行错各自活着
- `WATCH` = 乐观锁：变了就重试整个循环
- Lua = 服务端原子函数：能看中间结果分支，是一切"检查+动作"竞态的解药
- 选型：省 RTT 用 pipeline，固定打包用事务，带判断用脚本，判断复杂留应用层用 WATCH

## 练习

1. 用 WATCH + 事务实现"库存扣减"：两个终端并发抢，观察 `WatchError` 重试路径。
2. 用 `register_script` 写"仅当分数高于现有值才更新 ZSet"的原子脚本，并测试并发写。
3. 故意构造执行阶段失败（`MULTI` 里 `LPUSH` 后 `INCR` 同一个 key），验证"前条成功、后条失败、无回滚"。

## 相关专题

- 脚本的原语应用：[[redis/02-中段阶段-生产实践/06-分布式锁]]
- 打包的性能视角（RTT 数学）：[[redis/02-中段阶段-生产实践/05-管道与性能]]
