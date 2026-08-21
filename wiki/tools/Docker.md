好的，针对你的 PostgreSQL 容器（容器名 `my-postgres`，数据库名 `agent_memory`），我帮你把常用的 Docker 命令整理成了 **5 个实用场景**。你可以把这个当作速查表：

---

### 1️⃣ 查看状态与日志（日常体检）

- **查看所有容器（包括已停止的）**：
  ```bash
  docker ps -a
  ```

- **只看正在运行的容器**：
  ```bash
  docker ps
  ```

- **只看你的 PostgreSQL 容器是否在跑**：
  ```bash
  docker ps | grep my-postgres
  ```

- **查看 PostgreSQL 的启动日志（排错神器，比如连不上时看这里）**：
  ```bash
  docker logs my-postgres
  ```

- **实时滚动查看最新日志（按 Ctrl+C 退出）**：
  ```bash
  docker logs -f my-postgres
  ```

---

### 2️⃣ 启停与重启（日常维护）

- **停止容器**（相当于正常关机）：
  ```bash
  docker stop my-postgres
  ```

- **启动已停止的容器**：
  ```bash
  docker start my-postgres
  ```

- **重启容器**（相当于重启数据库服务，常用于加载配置）：
  ```bash
  docker restart my-postgres
  ```

- **暂停/恢复容器**（暂停时进程冻结，不常用，但偶尔用于紧急资源调度）：
  ```bash
  docker pause my-postgres   # 暂停
  docker unpause my-postgres # 恢复
  ```

---

### 3️⃣ 连接数据库执行 SQL（最常用）

**直接进入 `psql` 命令行交互界面**（连接到你的 `agent_memory` 库）：
```bash
docker exec -it my-postgres psql -U zzzfh -d agent_memory
```
进入后，你可以直接执行 SQL（比如 `\l` 看所有库，`\dt` 看所有表），输入 `\q` 退出。

**不进入交互界面，直接执行单条 SQL**（比如建表、查数据）：
```bash
docker exec -it my-postgres psql -U zzzfh -d agent_memory -c "SELECT version();"
```

**备份数据库（导出为 SQL 文件）**：
```bash
docker exec -it my-postgres pg_dump -U zzzfh agent_memory > backup.sql
```

**恢复数据库（导入 SQL 文件）**：
```bash
cat backup.sql | docker exec -i my-postgres psql -U zzzfh -d agent_memory
```
（注意这里 `-i` 不加 `-t`，因为 `cat` 命令不需要交互终端）

---

### 4️⃣ 资源与磁盘管理（省空间）

- **查看容器占用磁盘大小**：
  ```bash
  docker system df
  ```

- **清理所有停止的容器、悬空的镜像、网络等（小心操作，确认无误再执行）**：
  ```bash
  docker system prune
  ```

- **只想清理日志文件（如果 PostgreSQL 日志把磁盘占满了）**：
  ```bash
  sudo sh -c "truncate -s 0 /var/lib/docker/containers/*/*-json.log"
  ```

---

### 5️⃣ 进阶：查看容器详细信息

- **查看容器的 IP 地址、端口映射、环境变量等（JSON 格式输出）**：
  ```bash
  docker inspect my-postgres
  ```

- **只看端口映射**：
  ```bash
  docker port my-postgres
  ```

- **只看环境变量（确认用户名、密码配置）**：
  ```bash
  docker inspect my-postgres | grep -A 5 "Env"
  ```

---

### 💡 你的专属速查卡

| 你的需求 | 对应的命令 |
| :--- | :--- |
| **想知道数据库有没有在跑** | `docker ps \| grep my-postgres` |
| **连不上时，看报错原因** | `docker logs my-postgres --tail 50` |
| **开机自动启动（已配置过）** | （你启动时带了 `--restart unless-stopped`，不用管） |
| **想进数据库敲 SQL** | `docker exec -it my-postgres psql -U zzzfh -d agent_memory` |
| **关机（省资源）** | `docker stop my-postgres` |
| **重新开机** | `docker start my-postgres` |

---

