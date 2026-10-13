# frappe_docker 崩溃循环诊断报告

> 2026-10-13 ｜ 现象：frontend / websocket / queue-long 三个容器持续 Restarting
> 结论：**依赖链断裂** —— 根因明确，修复动作简单（重启依赖服务或整栈重启）

---

## 一、现象（实测）

| 容器 | 状态 | 退出码 | 重启策略 | 重启次数 |
|---|---|---|---|---|
| frappe_docker-frontend-1 | **Restarting** | 1 | on-failure | 27+ |
| frappe_docker-websocket-1 | **Restarting** | 1 | on-failure | 20+ |
| frappe_docker-queue-long-1 | **Restarting** | 1 | on-failure | 31+ |
| frappe_docker-scheduler-1 | Up（正常） | 0 | on-failure | 0 |
| frappe_docker-backend-1 | Exited | 0 | on-failure | 1 |
| frappe_docker-db-1 | **Exited** | 0 | on-failure | 0 |
| frappe_docker-redis-cache-1 | **Exited** | 0 | on-failure | 0 |
| frappe_docker-redis-queue-1 | **Exited** | 0 | on-failure | 0 |
| frappe_docker-queue-short-1 | Exited | 0 | on-failure | 0 |
| frappe_docker-create-site-1 | Exited | 143 | no | 0 |
| frappe_docker-configurator-1 | Exited | 0 | no | 0 |

**实测重启频率**：约每 3-30 秒一次（30 秒采样内重启计数 +1）。

---

## 二、根因

### ⚠️ 三个容器失败原因**不同**（别一概而论）

逐个核实后的真实错误：

| 容器 | 失败的直接原因 | 缺失的依赖 |
|---|---|---|
| **websocket** | `Error: getaddrinfo ENOTFOUND redis-queue` | `redis-queue` |
| **queue-long** | `redis.exceptions.ConnectionError: Error -2 connecting to redis-queue:6379. Name or service not known.` | `redis-queue` |
| **frontend** | `[emerg] host not found in upstream "backend:8000" in /etc/nginx/conf.d/frappe.conf` | **`backend`（不是 redis）** |

**原始日志**：

```
# websocket（node）
Error: getaddrinfo ENOTFOUND redis-queue
    at RedisSocket.connect ... (realtime/index.js:76)
  errno: -3008, code: 'ENOTFOUND', hostname: 'redis-queue'

# queue-long（python）
redis.exceptions.ConnectionError: Error -2 connecting to redis-queue:6379.
Name or service not known.

# frontend（nginx）
[emerg] 12#12: host not found in upstream "backend:8000" in /etc/nginx/conf.d/frappe.conf:2
```

> 📌 **修正说明**：本节初版把三个容器都归因为"连不上 redis"，**不准确**。
> frontend 是 nginx，它的上游是 `backend:8000` —— 失败原因是 **backend 容器不在**，与 redis 无关。

### 上游原因：依赖容器全部已退出，DNS 名解析不到

```
frappe_docker-db-1           exited (exit 0)  started 2026-09-16
frappe_docker-redis-cache-1  exited (exit 0)  started 2026-09-15
frappe_docker-redis-queue-1  exited (exit 0)  started 2026-09-11
frappe_docker-backend-1      exited (exit 0)  started 2026-09-15   ← frontend 依赖它
```

**统一机制**：Docker 里**每个容器的服务名就是 DNS 名**。容器一退出，内嵌 DNS 里就没有这个条目 →

- 依赖它的容器启动时解析失败 → **exit 1**
- 退出码 1 + `restart: on-failure` → **无限重启循环**

```
依赖链断裂图
  frontend  ──需要──> backend:8000     ❌ 已退出 → nginx [emerg] host not found
  websocket ──需要──> redis-queue:6379  ❌ 已退出 → ENOTFOUND
  queue-long──需要──> redis-queue:6379  ❌ 已退出 → ConnectionError
  scheduler ──需要──> ?                 ✅ 还活着（restarts=0）
```

> 关键对照：`scheduler-1` 是唯一还活着的 frappe 容器（它不依赖上面这些就能起），
> 恰好反证了"**不是镜像坏了，是依赖没了**"。

### 时间线（推测，证据支持）

1. 09-11 / 09-15 / 09-16 三个依赖容器先后退出（`exit 0` = 正常停止，可能是手动 `docker compose stop` 或系统重启）
2. 但 on-failure 策略**不会**在 exit 0 时自动重启它们 → 依赖永久缺失
3. 而 frontend/websocket/queue-long 以 **exit 1** 失败 → 触发 on-failure → 循环至今
4. 10-13 18:58 docker daemon 重启后，循环容器被重新拉起，继续失败（scheduler 也重启了，但依赖仍缺）

---

## 三、影响

| 项 | 评估 |
|---|---|
| 对增长工具栈 | ✅ **无影响**（完全独立的 compose 项目） |
| 对 bench-* 栈 | ✅ 无影响（另一套，`bench-redis` / `bench-mariadb` 正常运行） |
| 资源消耗 | ⚠️ 循环容器 CPU 占用 0%（秒退），但**日志持续产出** + 容器反复创建销毁 |
| 数据风险 | ✅ **无** —— 数据在 named volume（`frappe_docker_db-data` / `sites` / `logs` / `redis-queue-data`），容器重启不丢 |

---

## 四、修复方案（三选一）

### 方案 A｜只重启依赖（最小动作）⭐

```bash
cd /home/l/.openclaw/workspace/projects/factory-erp/local-ops/frappe_docker

# 注意：compose 记录的 working_dir 是已失效的旧路径
#   /home/l/.openclaw/workspace/projects/factory-erp/frappe_docker（不存在）
#   实际文件在 local-ops/frappe_docker/
# 用 -f 显式指定可绕开

# ⚠️ 四个依赖都要起：frontend 需要 backend（不是 redis）
docker compose -f pwd.yml start db redis-cache redis-queue backend
# → 依赖起来后，循环容器应能连上（它们仍在重试）
#
# 只起 db/redis 三个的话，frontend 仍会因 "backend:8000 host not found" 继续循环
```

### 方案 B｜整栈重启（推荐，一次到位）

```bash
cd /home/l/.openclaw/workspace/projects/factory-erp/local-ops/frappe_docker
docker compose -f pwd.yml up -d
# 或先 down（不删卷）再 up
docker compose -f pwd.yml down && docker compose -f pwd.yml up -d
```

### 方案 C｜不打算用了 → 停掉（止损）

```bash
cd /home/l/.openclaw/workspace/projects/factory-erp/local-ops/frappe_docker
docker compose -f pwd.yml down
# 数据卷保留；要彻底清理加 -v（⚠️ 会删数据）
```

---

## 五、⚠️ 附带发现：compose working_dir 指向失效路径

**问题**：容器 label 记录的 compose 文件路径是
```
/home/l/.openclaw/workspace/projects/factory-erp/frappe_docker/pwd.yml   ← 已不存在
```
**实际文件在**：
```
/home/l/.openclaw/workspace/projects/factory-erp/local-ops/frappe_docker/pwd.yml
```

→ 意味着**在旧路径下跑 `docker compose` 会全部失败**（找不到文件）。
→ 这也是为什么"看起来在跑，但没人管"的原因之一：**没有目录能自然落脚去执行 compose 命令**。

**建议**：修复后在正确目录重跑一次 `up -d`，让 label 更新为新路径。

---

## 六、为什么"跑了几周没人发现"

1. **循环是静默的** —— 容器秒退秒起，CPU 0%，监控（如果有）多半只看"容器是否存在"，不看"重启次数"
2. **没有告警** —— 这类失败不会推送通知
3. **compose 路径失效** → 没有自然的入口去检查它

> 👉 这正是 SOP 该管的场景：**周期性巡检要检查"重启次数"和"退出码"，而不只是"容器在不在"**。
> 建议纳入 T05（监控与告警维护）。

---

## 七、待用户决策

- [ ] frappe_docker 这套还要吗？
  - **要** → 跑方案 A 或 B
  - **不要** → 跑方案 C 停掉（省资源、去噪音）
- [ ] 是否需要我把"容器重启次数巡检"写进监控 SOP
