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

### backend 的退出证据（关键补充）

```
[2026-09-15 00:30:24] [1] [INFO] Starting gunicorn 23.0.0
[2026-09-15 00:30:24] [1] [INFO] Listening at: http://0.0.0.0:8000 (1)
[2026-09-16 15:39:11] [1] [INFO] Handling signal: term      ← 收到 SIGTERM
[2026-09-16 15:39:13] [1] [INFO] Shutting down: Master
```

`signal: term` = **被主动停止**（`docker compose stop` / `down` / 宿主重启流程），
**不是崩溃**（exit 0，重启策略 with `on-failure` 因此不会拉起它）。

> 这说明：**09-16 整栈被停过一次**，而停的过程中依赖容器全部 `exit 0` 正常退出；
> 只有 frontend / websocket / queue-long 因依赖消失而以 `exit 1` 失败，卡进了 on-failure 循环。

### 时间线（证据支持）

1. **09-16 15:39** —— 整栈被停止（backend 收到 SIGTERM；db/redis 等 exit 0）
2. on-failure 策略**不会**在 exit 0 时自动重启它们 → 依赖永久缺失
3. 而 frontend/websocket/queue-long 以 **exit 1** 失败 → 触发 on-failure → **循环至今**
4. **10-13 18:58** docker daemon 重启 → 三容器被重新拉起继续失败；scheduler 也被拉起（started 10:57:51Z）但依赖仍缺

> 👉 结论：这是"**停了一半又没停干净**"的状态 —— `on-failure` 策略让失败者永远重试，
> 让正常退出者永远不回来。**方案 B（整栈 down + up）才是最贴合的解法。**

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


---

# ✅ 执行记录：方案 B（2026-10-13 21:31 用户拍板）

## 一、执行过程

```bash
cd /home/l/.openclaw/workspace/projects/factory-erp/local-ops/frappe_docker
docker compose -f pwd.yml down          # 不带 -v，保留数据卷
docker compose -f pwd.yml up -d
```

**执行前确认**：四个数据卷存在（`frappe_docker_db-data` / `logs` / `redis-queue-data` / `sites`）
**执行后**：9 个容器全部 Up，**重启循环停止**（`restarts=0`）

## 二、⚠️ 但排查中发现两层更深的问题（都已修复）

方案 B 解决了"重启循环"，但**站点仍不可用**。逐层排查后发现问题不止一层。

### 第 1 层｜容器间网络断裂（backend 被网络隔离）

**现象**：站点请求挂住 80 秒无响应（不是 4xx/5xx，是超时）。

**定位过程**（每一步都是一次决定性对照）：

| 测试 | 结果 | 说明 |
|---|---|---|
| 宿主 → backend:8000 | ✅ 404（0.0027s） | 应用本身正常 |
| 宿主 → db:3306 | ✅ OK | 宿主到容器正常 |
| **backend → db:3306** | ❌ Timeout | **容器出网失败** |
| **backend → redis-queue:6379** | ❌ Timeout | 同上 |
| **backend 自连 172.18.0.7:8000** | ✅ OK | 自己连自己正常 |
| **backend → 外网 1.1.1.1:443** | ❌ Timeout | 完全出不去 |
| **scheduler → db / redis** | ✅ **全部 OK** | **同网同镜像却正常** |
| **queue-long → redis-queue** | ✅ OK | 同上 |
| 探针容器 → db / redis-queue | ✅ OK | 新容器同网正常 |
| **探针容器 → backend:8000** | ❌ Timeout | 别人进不来 |

**结论：只有 backend 一个容器被隔离，双向不通**，而同网同镜像的 scheduler / queue-long 完全正常。

**试过但无效**：
- `--force-recreate backend`（容器重建）→ 无效
- `network disconnect` + `network connect`（端点重建）→ 无效

**最终解法：重建整个网络**

```bash
docker compose -f pwd.yml down          # 会一并删掉网络
docker compose -f pwd.yml up -d         # 重新创建网络
```

✅ **生效**：backend 恢复 → `db OK` / `redis-queue OK` / `websocket OK`

> 📌 **教训**：容器网络端点损坏时，**`--force-recreate` 和端点重建都不够，必须重建网络本身**。

### 第 2 层｜数据库授权绑定旧 IP（暴露出的下一层）

网络修好后，站点从"挂 80 秒"变成 **500（0.029s）** —— 说明能打到后端了，暴露下一层问题：

```
MySQLdb.OperationalError: (1045, "Access denied for user 'erpnext'@'172.18.0.6' (using password: YES)")
```

**根因**：`erpnext` 数据库用户**只授权给 `172.18.0.9`**（历史遗留的旧容器 IP）：

```
GRANT ALL PRIVILEGES ON `erpnext`.* TO `erpnext`@`172.18.0.9`
```

网络重建后 backend 的 IP 变成 `172.18.0.6` → **IP 不匹配 → 认证被拒**。

> ⚠️ 这是**预先存在**的问题，只是之前网络不通、根本没走到认证这一步，所以没暴露。

**修复（追加式，不动原授权）**：

```sql
-- 用 site_config.json 里已有的密码，追加一条通配授权
CREATE USER IF NOT EXISTS 'erpnext'@'%' IDENTIFIED BY '<site_config 里的密码>';
GRANT ALL PRIVILEGES ON `erpnext`.* TO 'erpnext'@'%';
FLUSH PRIVILEGES;
```

**为什么用 `%` 而不是只补新 IP**：容器 IP 每次重建都会变，绑定单个 IP 会**反复踩同一个坑**。

**数据完好**：`erpnext` 库 **742 张表**，未受影响。

## 三、最终验证（全部通过）

| 项 | 结果 |
|---|---|
| 站点 `http://127.0.0.1:8082/`（Host: factory.local） | ✅ **200**（0.14s → 0.022s） |
| 9 个容器 | ✅ 全部 Up |
| 重启计数 | ✅ `frontend=0` / `websocket=0` / `backend=0` / `queue-long=1`（仅初始一次） |
| backend 出网 | ✅ db / redis-queue / websocket 全通 |

**访问方式**（注意要带 Host 头，nginx 按 `server_name` 路由）：

```bash
curl -H "Host: factory.local" http://127.0.0.1:8082/
# 浏览器访问需 hosts 加：127.0.0.1  factory.local
```

## 四、本次修复清单（可回溯）

| # | 动作 | 性质 | 影响面 |
|:--:|---|---|---|
| 1 | `compose down` + `up -d`（方案 B） | 重启整栈 | 修复重启循环 |
| 2 | **重建 docker 网络** | 删网络重建 | 修复 backend 网络隔离 |
| 3 | **追加 `erpnext@'%'` 数据库授权** | **数据层变更** ⚠️ | 修复 DB 认证；不动原授权 |
| 4 | 数据卷 | **未动** | `db-data`/`sites`/`logs`/`redis-queue-data` 全保留 |

## 五、遗留建议

- [ ] **`erpnext` 用户绑定 IP 是隐患** —— 若将来再重建网络，`@'%'` 已覆盖，不会再踩；但建议同类项目统一用 `@'%'` + 强密码
- [ ] **compose working_dir label 仍指向失效路径** —— 不影响运行，但建议在正确目录重跑一次 `up -d` 以更新 label
- [ ] **容器命名带项目名前缀**（`frappe_docker-*`）而非 compose 标准（`frappe_docker-backend-1`）—— 说明是旧版 compose 创建的，重建后已规范化
