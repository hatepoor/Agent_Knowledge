---
tags: [meta, docker, 术语, 命令, 索引, 零基础]
type: meta
created: 2026-09-22
updated: 2026-09-22
---

# Docker 命令与术语手册

> **这份文件存在的唯一目的：您永远不会因为"不认识某个命令或名词"而卡住。**
>
> 这里收录：Docker/Compose 子命令、`docker run` 运行参数、Dockerfile 指令、compose.yaml 字段、容器内核术语、常见报错用词。
>
> 使用方式：按 `Ctrl+F` / `Cmd+F` 搜索那个词。**不需要理解，先查到再说。**

## 怎么读

| 列 | 含义 |
|----|------|
| **一句话解释** | 这个词是干什么的 |
| **出现在哪** | 首次被讲解的页面 |

**标注**：⭐ = 入门阶段就会用到，必须认识；⚠️ = 易混/有坑，务必看详解；🐳 = Docker 生态专有名词；🧬 = 来自 Linux 内核的底层机制（中段才会展开讲）。

**缩写约定**：`<...>` 表示需要你替换的占位内容，例如 `docker run <镜像名>` 中的 `<镜像名>` 要换成 `nginx`、`redis` 之类。

**查不到怎么办**：见本页最后一节「查不到某个词时」。

---

## 一、核心概念名词 🐳

| 名词 | 一句话解释 | 出现在哪 |
|------|-----------|----------|
| **镜像 (image)** | 一个**只读**的模板，包含程序 + 依赖 + 配置。不能直接运行，是"菜谱" | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.1-容器与Docker是什么]] |
| **容器 (container)** | 镜像的**运行实例**，可读写、可启停。是"照菜谱做出的菜" | 同上 |
| **仓库 (registry)** | 集中存放镜像的服务。Docker Hub 是默认的公共仓库 | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.1-镜像的获取与管理]] |
| **Docker Hub** | Docker 官方公共镜像仓库，默认拉取来源。地址 <https://hub.docker.com/> | 同上 |
| **tag（标签）** | 镜像的版本标记，写在镜像名冒号后面：`nginx:1.27`。不写 tag 默认 `latest` | 同上 |
| **`latest`** ⚠️ | **不是"最新版"的保证**，只是"没指定 tag 时的默认值"，可能指向很老的版本。**生产禁用** | 同上 |
| **层 (layer)** | 镜像由多层叠加而成，每条构建指令产生一层。层可被多个镜像共享，是缓存与瘦身的基础 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| **digest（摘要）** | 镜像内容的哈希值，形如 `sha256:abc...`。**比 tag 可靠**，因为 tag 可以被覆盖而 digest 不会 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| **manifest（清单）** | 描述"这个镜像有哪些层、什么架构"的元数据文件 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| **Dockerfile** | 描述"如何构建镜像"的文本文件，由一组指令构成 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] |
| **compose.yaml** | 描述"一套多容器服务如何编排"的配置文件（旧名 `docker-compose.yml` 仍可用） | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.1-Compose是什么与文件结构]] |
| **构建上下文 (build context)** | `docker build` 时**发送给守护进程**的目录内容。**不是整个磁盘**，就是命令行末尾那个 `.` 指向的目录 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| **`.dockerignore`** | 排除文件，告诉构建过程"哪些文件不要进上下文"。**不做会有严重后果** | 同上 |
| **数据卷 (volume)** | 由 Docker 管理的数据存储位置，独立于容器生命周期。容器删了数据还在 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] |
| **绑定挂载 (bind mount)** | 把**宿主机的一个具体路径**直接挂进容器。路径由你指定，Docker 不管理 | 同上 |
| **容器网络 (network)** | 容器之间、容器与外界通信的虚拟网络。默认有 `bridge` / `host` / `none` 三种 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.2-容器网络模型]] |
| **命名空间 (namespace)** 🧬 | Linux 内核机制，让进程"看不见"别人的进程、网络、文件系统。**容器隔离的本质来源** | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| **控制组 (cgroup)** 🧬 | Linux 内核机制，限制一组进程能用多少 CPU / 内存 / IO。**资源限制的本质来源** | [[docker/02-中段阶段-专题精讲/资源限制与性能/专题概览]] |
| **联合文件系统 / overlayfs** 🧬 | 把多个只读层叠加成一个可写视图的文件系统技术。镜像分层就靠它 | [[docker/02-中段阶段-专题精讲/镜像优化与瘦身/专题概览]] |
| **PID 1** ⚠️ | 容器内**第一个进程**，是容器的主进程。这个进程退出，容器就结束 | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.2-容器的创建与运行]] |
| **rootfs** | 容器内看到的根文件系统，来自镜像各层的叠加 | [[docker/02-中段阶段-专题精讲/镜像优化与瘦身/专题概览]] |
| **OCI** 🐳 | Open Container Initiative，制定容器镜像与运行时标准的组织。**"符合 OCI 标准"意味着可移植** | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |

---

## 二、Docker 架构相关名词 🐳

| 名词 | 一句话解释 | 出现在哪 |
|------|-----------|----------|
| **Docker CLI（客户端）** | 你敲的 `docker` 命令本体。它**自己不干活**，只是把请求发给守护进程 | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.2-Docker的组成与运行架构]] |
| **Docker daemon（守护进程）** | 后台服务 `dockerd`，真正执行拉取、构建、运行等操作 | 同上 |
| **`docker context`** | 客户端连的是哪个守护进程的配置。"本地"还是"远程主机" | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |
| **containerd** ⚠️ | 被 daemon 调用的容器运行时，负责容器的实际启停。**很多"容器管理"实际由它完成** | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.2-Docker的组成与运行架构]] |
| **runc** 🧬 | 最底层的运行时，负责按 OCI 规范真正创建进程与命名空间 | 同上 |
| **BuildKit** 🐳 | 现代镜像构建引擎（`Dockerfile` 的构建后端）。支持并行构建、缓存挂载、密钥挂载 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| **buildx** 🐳 | BuildKit 的命令行入口，支持多平台构建（如同时构建 amd64 与 arm64） | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| **镜像存储驱动 (storage driver)** 🧬 | 决定镜像层在磁盘上怎么组织。当前常见 `overlay2`；新版默认切到 containerd 镜像存储 | [[docker/02-中段阶段-专题精讲/镜像优化与瘦身/专题概览]] |
| **Docker Desktop** | Windows / macOS 上的桌面发行包，内含 Engine + Compose + Kubernetes。**Windows 必须用它** | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.3-跑通第一个容器]] |
| **WSL 2** ⚠️ | Windows Subsystem for Linux 2。**Windows 上跑 Linux 容器的前提** | 同上 |
| **镜像加速器 / registry mirror** | 第三方镜像源，解决国内拉取镜像慢的问题。配置在 daemon 配置里 | 同上 |
| **Prune（清理）** | 删除未使用的容器/镜像/网络/卷，回收磁盘。"未使用"的定义因对象而异，**删前务必确认** | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] |

---

## 三、容器生命周期命令 ⭐

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker run <镜像>` | **创建并启动**一个新容器（最常用） | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.3-跑通第一个容器]] |
| `docker create <镜像>` | 只创建，不启动。之后用 `docker start` | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.2-容器的创建与运行]] |
| `docker start <容器>` | 启动**已存在**的容器（与 `run` 的区别：不新建） | 同上 |
| `docker stop <容器>` | 优雅停止：先发 `SIGTERM`，超时后 `SIGKILL`（默认等 10 秒） | 同上 |
| `docker kill <容器>` | 立即强杀，发 `SIGKILL`。**不给程序清理机会** | 同上 |
| `docker restart <容器>` | 重启 | 同上 |
| `docker rm <容器>` | 删除容器。运行中的必须先停，或加 `-f` 强删 | 同上 |
| `docker ps` | 列出**运行中**的容器 | 同上 |
| `docker ps -a` | 列出**全部**容器（含已退出）。**排错第一步** | 同上 |
| `docker pause` / `docker unpause` | 冻结 / 解冻容器内所有进程 | [[docker/速查索引]] |
| `docker wait <容器>` | 阻塞直到容器退出，返回退出码 | [[docker/速查索引]] |
| `docker rename <旧> <新>` | 重命名容器 | [[docker/速查索引]] |
| `docker update <容器> <参数>` | **运行中**修改资源限制、重启策略 | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.1-资源限制]] |
| `docker commit <容器> <镜像名>` | 把容器当前状态**保存成新镜像**。仅用于临时调试，**不要用于生产** | [[docker/速查索引]] |

---

## 四、观察与进入容器 ⭐

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker logs <容器>` | 查看容器主进程的输出（stdout/stderr） | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.3-进入容器与查看日志]] |
| `docker logs -f <容器>` | 实时跟踪日志（`f` = follow），`Ctrl+C` 退出查看但不停止容器 | 同上 |
| `docker logs --tail 50 <容器>` | 只看最后 50 行 | 同上 |
| `docker exec -it <容器> <命令>` | 在**运行中**的容器里执行命令，通常用于进 shell | 同上 |
| `docker attach <容器>` | 连接到容器主进程的输入输出。**退出时可能连带停止容器**，一般优先用 `exec` | 同上 |
| `docker cp <容器>:<路径> <宿主路径>` | 容器与宿主机之间复制文件 | 同上 |
| `docker inspect <容器>` | 输出容器的**全部**元数据（JSON）：IP、挂载、环境变量等 | 同上 |
| `docker port <容器>` | 显示端口映射关系 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.3-端口映射与访问]] |
| `docker stats` | 实时显示各容器 CPU / 内存 / 网络占用 | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.1-资源限制]] |
| `docker top <容器>` | 显示容器内运行的进程（在宿主视角看） | [[docker/速查索引]] |
| `docker diff <容器>` | 列出容器相对镜像改了哪些文件 | [[docker/速查索引]] |
| `docker events` | 实时输出 Docker 事件流（启停、创建等） | [[docker/02-中段阶段-专题精讲/日志与监控/专题概览]] |

---

## 五、`docker run` 常用参数 ⭐

| 参数 | 作用 | 出现在哪 |
|------|------|----------|
| `-d` | 后台运行（detached）。不写则占住当前终端 | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.2-容器的创建与运行]] |
| `-it` | `-i` 保持输入流 + `-t` 分配终端。**进交互 shell 的标配** | 同上 |
| `--name <名字>` | 给容器命名。不写会随机分配 | 同上 |
| `--rm` | 容器退出后**自动删除**。**临时任务必加**，否则残留一堆死容器 | 同上 |
| `-p <宿主端口>:<容器端口>` | 端口映射，让外部能访问容器内的服务 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.3-端口映射与访问]] |
| `-P` | 把镜像里 `EXPOSE` 的端口随机映射到宿主高端口 | [[docker/速查索引]] |
| `-v <宿主路径>:<容器路径>` | 挂载数据（绑定挂载，或 `卷名:路径` 用具名卷） | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] |
| `--mount` | 挂载的**推荐新写法**，语义更明确、出错时提示更清楚 | 同上 |
| `-e <变量>=<值>` / `--env` | 设置容器内环境变量 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.3-环境变量与多环境配置]] |
| `--env-file <文件>` | 从文件批量读环境变量 | 同上 |
| `--network <网络名>` | 指定加入哪个容器网络 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.2-容器网络模型]] |
| `--hostname <名字>` | 设置容器主机名 | [[docker/速查索引]] |
| `--restart <策略>` | 退出后是否自动重启：`no` / `on-failure` / `always` / `unless-stopped` | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.1-资源限制]] |
| `-m` / `--memory` | 内存上限，如 `-m 512m` | 同上 |
| `--memory-swap` | 内存 + swap 总量上限，需与 `-m` 配合理解 | 同上 |
| `--cpus` | CPU 核数上限，如 `--cpus=1.5` | 同上 |
| `--cpuset-cpus` | 绑定到具体哪几个核，如 `--cpuset-cpus="0,1"` | 同上 |
| `--pids-limit` | 进程数上限。**防 fork 炸弹** | 同上 |
| `--ulimit` | 设置资源限制，如 `--ulimit nofile=1024:1024` | 同上 |
| `--user` / `-u` | 指定容器内以哪个用户运行。**安全加固关键项** | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `--workdir` / `-w` | 指定容器内工作目录 | [[docker/速查索引]] |
| `--entrypoint` | 覆盖镜像的 `ENTRYPOINT` | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] |
| `--read-only` | 根文件系统只读，**只能写挂载的卷** | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `--tmpfs <路径>` | 挂载内存文件系统，不落盘 | 同上 |
| `--cap-add` / `--cap-drop` | 增加 / 移除 Linux 能力（capability）。**默认就该 drop 掉不需要的** | 同上 |
| `--security-opt` | 安全选项，如 `no-new-privileges` | 同上 |
| `--privileged` ⚠️ | **完全特权模式，等于把宿主交给容器。生产严禁使用** | 同上 |
| `--log-driver` / `--log-opt` | 日志驱动与选项，如限制日志大小 | [[docker/02-中段阶段-专题精讲/日志与监控/专题概览]] |
| `--health-cmd` 等 | 定义健康检查 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.2-用Compose编排多容器服务]] |
| `--add-host` | 向容器内 `/etc/hosts` 追加映射 | [[docker/速查索引]] |
| `--dns` | 指定容器使用的 DNS | [[docker/速查索引]] |
| `--label` | 给容器打标签，便于批量筛选管理 | [[docker/速查索引]] |

**短参数与长形式对照**（索引里两种写法都会出现，认识即可）

| 短形式 | 长形式 | 含义 |
|--------|--------|------|
| `-d` | `--detach` | 后台运行 |
| `-i` | `--interactive` | 保持标准输入 |
| `-t` | `--tty` | 分配伪终端 |
| `-p` | `--publish` | 发布端口 |
| `-v` | `--volume` | 挂载卷/目录 |
| `-e` | `--env` | 设置环境变量 |
| `-m` | `--memory` | 内存上限 |
| `-u` | `--user` | 指定用户 |
| `-w` | `--workdir` | 指定工作目录 |
| `-f` | `--force` | 强制执行（含义随命令而变） |

**资源限制的进阶参数**（入门不常用，中段展开）

| 参数 | 作用 | 出现在哪 |
|------|------|----------|
| `--memory-reservation` | 内存**软限制**（压力大时才生效） | [[docker/02-中段阶段-专题精讲/资源限制与性能/专题概览]] |
| `--oom-kill-disable` | ⚠️ 禁止 OOM 杀容器。**极危险** | 同上 |
| `--memory-swappiness` | swap 使用倾向（0-100） | 同上 |
| `--cpu-shares` | CPU **相对权重**（默认 1024）。**不是上限** | 同上 |
| `--cpu-period` / `--cpu-quota` | CFS 周期与配额的底层写法（`--cpus` 是其封装） | 同上 |
| `--blkio-weight` | 块设备 IO 相对权重 | 同上 |
| `--device-read-bps` / `--device-write-bps` | 块设备读写**速率**上限 | 同上 |
| `--device-read-iops` / `--device-write-iops` | 块设备读写 **IOPS** 上限 | 同上 |
| `--ipc` | IPC 命名空间模式（`host` 会削弱隔离） | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `--pid` | PID 命名空间模式（`host` 会削弱隔离） | 同上 |
| `--userns` | 用户命名空间模式 | 同上 |
| `--security-opt seccomp=` / `apparmor=` | 指定安全策略 profile | 同上 |
| `--device` | 把宿主设备暴露给容器 | [[docker/速查索引]] |
| `--sysctl` | 设置内核参数 | [[docker/速查索引]] |
| `--link` ⚠️ | **已废弃**的容器互联方式（用自定义网络替代） | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.2-容器网络模型]] |
| `--network-alias` | 在网络上给容器增加可用别名 | 同上 |
| `--add-host` | 向容器 `/etc/hosts` 追加映射 | [[docker/速查索引]] |

---

## 六、镜像命令 ⭐

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker pull <镜像>` | 从仓库拉取镜像 | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.1-镜像的获取与管理]] |
| `docker push <镜像>` | 推送镜像到仓库 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `docker images` / `docker image ls` | 列出本地镜像 | [[docker/01-入门阶段-基础教程/第01章-镜像与容器基础操作/1.1-镜像的获取与管理]] |
| `docker rmi <镜像>` | 删除镜像。若有容器（含停止的）在用会失败 | 同上 |
| `docker tag <源> <新名:tag>` | 给镜像打别名/版本号。**推送到仓库前必须先 tag 成仓库格式** | 同上 |
| `docker build -t <名字> <上下文>` | 按 Dockerfile 构建镜像 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| `docker history <镜像>` | 逐层显示镜像是怎么构建出来的、每层多大 | 同上 |
| `docker inspect <镜像>` | 查看镜像元数据（入口命令、环境变量、层信息） | [[docker/速查索引]] |
| `docker save -o <文件> <镜像>` | 把镜像导出为 tar 文件（离线拷贝） | [[docker/02-中段阶段-专题精讲/数据持久化与备份/专题概览]] |
| `docker load -i <文件>` | 从 tar 文件导入镜像 | 同上 |
| `docker search <关键词>` | 在 Docker Hub 搜索镜像 | [[docker/速查索引]] |
| `docker login` / `docker logout` | 登录 / 登出镜像仓库 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `docker image prune` | 清理**悬空**镜像（无 tag 且无容器引用） | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] |
| `docker image prune -a` ⚠️ | 清理**所有**无容器引用的镜像（**会删掉你手动拉的基础镜像**） | 同上 |
| `docker buildx build --platform ...` | 多平台构建，一次产出 amd64/arm64 多个架构 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |

**对象化命令形式**（`docker <对象> <动作>`，与简写等价）

| 简写 | 对象化形式 | 说明 |
|------|-----------|------|
| `docker ps` | `docker container ls` | 列容器 |
| `docker rmi` | `docker image rm` | 删镜像 |
| `docker images` | `docker image ls` | 列镜像 |
| `docker port` | `docker container port` | 看端口 |
| `docker inspect` | `docker container inspect` / `docker image inspect` | 看详情（对象不同） |

> 两种写法**完全等价**，对象化形式在脚本里更明确，简写在日常输入更快。

**镜像导出/导入的两种方式（不要混用）**

| 命令 | 方向 | 保留什么 |
|------|------|----------|
| `docker save` / `docker load` | **镜像** ↔ tar | ✅ 保留全部层、tag、元数据 |
| `docker export` / `docker import` | **容器** ↔ tar | ⚠️ 只导出文件系统，**丢失层结构与元数据** |

**已废弃 / 更名命令**

| 旧命令 | 现状 |
|--------|------|
| `docker scan` ⚠️ | **已被 `docker scout` 取代**。老教程里常见，照抄会提示已废弃 |
| `docker-compose`（连字符） ⚠️ | 已被 `docker compose`（子命令形式）取代，见 [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.1-Compose是什么与文件结构]] |

---

## 七、Dockerfile 指令 ⭐

| 指令 | 作用 | 出现在哪 |
|------|------|----------|
| `FROM <基础镜像>` | **每条 Dockerfile 的第一条有效指令**，指定基于什么镜像构建 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] |
| `FROM ... AS <阶段名>` | 多阶段构建：给这个阶段起名，供后续 `COPY --from=` 引用 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.3-多阶段构建与最佳实践]] |
| `RUN <命令>` | **构建时**执行命令，结果固化进镜像层 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] |
| `CMD ["可执行","参数"]` | **容器启动时**的默认命令。可被 `docker run` 末尾参数覆盖 | 同上 |
| `ENTRYPOINT ["可执行","参数"]` | 容器的**固定入口**，不易被覆盖，`docker run` 末尾参数会作为它的额外参数 | 同上 |
| `COPY <源> <目标>` | 从构建上下文**复制文件**进镜像。**首选** | 同上 |
| `ADD <源> <目标>` ⚠️ | 类似 `COPY`，但会自动解压 tar 且支持 URL。**除非需要解压，否则用 `COPY`** | 同上 |
| `WORKDIR <路径>` | 设置后续指令的工作目录（不存在会创建）。**优于用 `RUN cd`** | 同上 |
| `ENV <键>=<值>` | 设置**构建期与运行期都生效**的环境变量 | 同上 |
| `ARG <名>[=<默认值>]` | **仅构建期**有效的变量，构建后可被 `--build-arg` 覆盖。**不能用来传密钥** | 同上 |
| `EXPOSE <端口>` | 声明镜像打算对外提供哪个端口。**只是文档性质，不真正发布端口** | 同上 |
| `VOLUME <路径>` | 声明该路径应作为数据卷。**可能造成意外的匿名卷** | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] |
| `USER <用户>` | 指定后续指令与容器启动时以哪个用户运行。**安全加固关键项** | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `LABEL <键>=<值>` | 给镜像加元数据（维护者、版本、源码地址） | [[docker/速查索引]] |
| `HEALTHCHECK <选项> CMD <命令>` | 定义健康检查命令，让 Docker 能判断容器"是否真的可用" | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.2-用Compose编排多容器服务]] |
| `SHELL ["..."]` | 指定后续 `RUN` 用哪个 shell（默认 Linux 是 `/bin/sh -c`） | [[docker/速查索引]] |
| `STOPSIGNAL <信号>` | 指定停止容器时发什么信号 | [[docker/速查索引]] |
| `ONBUILD <指令>` | 延迟到"别人以本镜像为基础时"才执行的指令 | [[docker/速查索引]] |
| `MAINTAINER` ⚠️ | **已废弃**，用 `LABEL maintainer=` 替代 | [[docker/速查索引]] |
| `COPY --from=<阶段名或镜像> <源> <目标>` | 从别的构建阶段或镜像里复制产物（多阶段构建的核心） | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.3-多阶段构建与最佳实践]] |
| `RUN --mount=type=cache ...` | 构建期缓存挂载，加速依赖安装（BuildKit 特性） | 同上 |

---

## 八、卷与网络命令

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker volume create <名>` | 创建具名卷 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.1-数据卷与绑定挂载]] |
| `docker volume ls` | 列出所有卷 | 同上 |
| `docker volume inspect <名>` | 查看卷在宿主机上的真实路径 | 同上 |
| `docker volume rm <名>` | 删除卷。**卷内数据一并销毁** | 同上 |
| `docker volume prune` ⚠️ | 删除**所有无容器使用**的卷。**这是最容易误删数据的命令** | 同上 |
| `docker network create <名>` | 创建自定义网络（**同一自定义网络内可用容器名互相访问**） | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.2-容器网络模型]] |
| `docker network ls` | 列出网络 | 同上 |
| `docker network inspect <名>` | 查看网络内有哪些容器、各自 IP | 同上 |
| `docker network connect` / `disconnect` | 把运行中的容器接入 / 断开某网络 | [[docker/02-中段阶段-专题精讲/网络进阶与多容器互联/专题概览]] |
| `docker network rm <名>` | 删除网络（有容器连着会失败） | [[docker/速查索引]] |
| `docker port <容器>` | 查看容器的端口映射 | [[docker/01-入门阶段-基础教程/第03章-数据卷与容器网络/3.3-端口映射与访问]] |

---

## 九、系统与维护命令

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker system df` | 统计镜像/容器/卷/缓存各占多少磁盘。**磁盘告急第一条命令** | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] |
| `docker system prune` | 清理停止的容器、悬空镜像、未用网络 | 同上 |
| `docker system prune -a --volumes` ⚠️ | **连所有无引用镜像和卷一起删。极度危险，务必先确认** | 同上 |
| `docker info` | 显示 Docker 环境信息：版本、存储驱动、内核、资源总量 | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] |
| `docker version` | 分别显示客户端与服务端版本。**两端版本不一致是常见故障源** | 同上 |
| `docker builder prune` | 清理构建缓存 | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] |
| `docker context ls` / `use` | 查看 / 切换客户端连接的守护进程 | [[docker/速查索引]] |
| `docker scout` | 镜像漏洞扫描（Docker Desktop 集成） | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |

| `docker scout cves <镜像>` | 列出镜像中的 CVE 漏洞 |
| `docker scout recommendations <镜像>` | 给出基础镜像升级建议 |

**prune 类命令的语义差异**（最容易误操作，执行前务必区分对象范围）

| 命令 | 清理对象 | 风险 |
|------|----------|------|
| `docker container prune` | 所有**已停止**的容器 | ⚠️ 停止的容器里可能还有待排查的信息 |
| `docker image prune` | **悬空**镜像（无 tag 无引用） | ✅ 低 |
| `docker image prune -a` | 所有**无容器引用**的镜像 | ⚠️ 会删掉手动拉的基础镜像 |
| `docker network prune` | 未被使用的网络 | ✅ 低 |
| `docker volume prune` | 无容器使用的**卷** | ❌ **删数据**，先 `docker volume ls` 确认 |
| `docker builder prune` | 未使用的构建缓存 | ✅ 低（代价是下次构建变慢） |
| `docker system prune` | 停止的容器 + 悬空镜像 + 未用网络 + **构建缓存** | ⚠️ 中 |
| `docker system prune -a --volumes` | 以上全部 **+ 所有无引用镜像 + 所有无容器使用的卷** | ❌ **最高，可能删掉数据库数据** |

> 清理的分档与完整处置流程见 [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]]。

**通用过滤与格式化参数**（多数 `ls` / `ps` / `logs` / `stats` 类命令都支持）

| 参数 | 作用 |
|------|------|
| `--filter` / `-f` | 按条件过滤，如 `--filter "until=24h"`、`--filter "label=app=myapp"` |
| `--format` | 用 Go 模板自定义输出，如 `--format "{{.Names}}\t{{.Status}}"` |
| `--no-trunc` | 不截断输出（看完整 ID、完整命令） |
| `--quiet` / `-q` | 只输出 ID（适合脚本） |
| `--all` / `-a` | 包含已停止 / 未使用的对象 |
| `--since` | 只看某时间之后的内容（`docker logs`、`docker events`） |
| `--until` | 只看某时间之前的内容 |
| `--tail` | `docker logs` 只看最后 N 行 |
| `--no-stream` | `docker stats` 只输出一次（不加会持续刷新，见 [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.1-资源限制]]） |
| `--force` / `-f` | 强制执行（含义随命令而变，**注意与 `--filter` 的 `-f` 缩写冲突**） |

**卷与网络的创建参数**

| 参数 | 作用 |
|------|------|
| `--driver` | 指定卷 / 网络的驱动，如 `--driver local`、`--driver bridge` |
| `--opt` | 给驱动传选项，如 `--driver local --opt type=nfs --opt o=addr=...` |
| `--subnet` | 创建网络时指定子网，如 `--subnet 172.20.0.0/16` |
| `--gateway` | 指定网络网关 |
| `--internal` | 创建**内部网络**（无对外网关，容器无法访问互联网）。Compose 中对应 `internal: true` |
| `--attachable` | 允许普通容器接入（overlay 网络相关） |

**Docker 守护进程参数**（写在 `daemon.json` 中，或作为 `dockerd` 启动参数）

| 参数（daemon.json 键名） | 作用 | 出现在哪 |
|--------------------------|------|----------|
| `data-root` | 数据目录位置（磁盘盘位规划） | [[docker/01-入门阶段-基础教程/第05章-资源限制与日常维护/5.2-磁盘占用与日志清理]] |
| `log-driver` / `log-opts` | 全局日志驱动与轮转选项 | [[docker/02-中段阶段-专题精讲/日志与监控/专题概览]] |
| `registry-mirrors` | 镜像加速器 | [[docker/01-入门阶段-基础教程/第00章-学前准备/0.3-跑通第一个容器]] |
| `userns-remap` | 用户命名空间映射（安全加固） | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `userland-proxy`（命令行形式 `--userland-proxy=false`）⚠️ | 是否启用用户态端口转发（`false` 可少一次拷贝，**会失去部分绑定能力**） | [[docker/02-中段阶段-专题精讲/网络进阶与多容器互联/专题概览]] |
| `insecure-registries` | 允许使用 HTTP 的私有仓库地址 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `default-address-pools` | 自定义容器网段（避免与内网地址冲突） | [[docker/02-中段阶段-专题精讲/网络进阶与多容器互联/专题概览]] |

---

## 十、Compose 命令与字段

### 10.1 命令（注意：`compose` 与 `docker` 之间**有空格**）

| 命令 | 作用 | 出现在哪 |
|------|------|----------|
| `docker compose up -d` | 创建并启动配置里的全部服务，`-d` 后台 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.2-用Compose编排多容器服务]] |
| `docker compose down` | 停止并删除容器与网络（**默认不删卷**） | 同上 |
| `docker compose down -v` ⚠️ | 连**卷一起删**（数据会丢） | 同上 |
| `docker compose ps` | 查看本项目的容器状态 | 同上 |
| `docker compose logs -f <服务>` | 实时查看某服务日志 | 同上 |
| `docker compose exec <服务> <命令>` | 在服务容器里执行命令 | 同上 |
| `docker compose run --rm <服务> <命令>` | 一次性运行（**不映射端口**，用于执行脚本） | 同上 |
| `docker compose build` | 构建（或重建）配置里的镜像 | 同上 |
| `docker compose pull` | 拉取配置里引用的镜像 | 同上 |
| `docker compose restart` | 重启服务（**不应用配置变更**，见 4.2 节） | 同上 |
| `docker compose stop` | 停止服务（容器保留，可再 start） | 同上 |
| `docker compose start` | 启动已停止的服务 | 同上 |
| `docker compose config` | 校验并输出**合并后**的最终配置。**排错利器** | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.3-环境变量与多环境配置]] |
| `docker compose scale <服务>=<数量>` | 扩缩副本数（新版推荐 `up --scale`） | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |
| `docker compose top` / `events` / `images` | 查看进程 / 事件流 / 相关镜像 | [[docker/速查索引]] |
| `docker compose watch` | 监听文件变化，自动同步或重建 | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |

### 10.2 compose.yaml 顶层与常用字段

| 字段 | 作用 | 出现在哪 |
|------|------|----------|
| `services:` | 顶层字段：定义有哪些服务 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.1-Compose是什么与文件结构]] |
| `name:` | 项目名（影响容器名前缀、网络名） | 同上 |
| `include:` | 引用另一个 compose 文件（拆分大型配置） | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |
| `services.<名>.image` | 直接用现成镜像 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.1-Compose是什么与文件结构]] |
| `services.<名>.build` | 从 Dockerfile 构建镜像（`.context` / `.dockerfile` / `.args` / `.target`） | 同上 |
| `services.<名>.ports` | 端口映射，`"8080:80"` 形式（**必须加引号**） | 同上 |
| `services.<名>.volumes` | 挂载卷或宿主路径 | 同上 |
| `services.<名>.environment` | 环境变量（列表或映射两种写法） | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.3-环境变量与多环境配置]] |
| `services.<名>.env_file` | 从文件读环境变量 | 同上 |
| `services.<名>.depends_on` | 启动顺序依赖。**注意：默认只等"启动"，不等"就绪"** | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.2-用Compose编排多容器服务]] |
| `services.<名>.healthcheck` | 健康检查，与 `depends_on: condition: service_healthy` 配合实现"等就绪" | 同上 |
| `services.<名>.restart` | 重启策略 | 同上 |
| `services.<名>.networks` | 加入哪些网络 | 同上 |
| `services.<名>.command` / `.entrypoint` | 覆盖镜像默认命令/入口 | 同上 |
| `services.<名>.container_name` | 固定容器名。**会阻碍扩缩容，一般不用** | 同上 |
| `services.<名>.profiles` | 按 profile 分组，实现"按需启动某组服务" | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |
| `services.<名>.deploy.resources.limits` | 资源限制（`cpus` / `memory`） | [[docker/02-中段阶段-专题精讲/资源限制与性能/专题概览]] |
| `services.<名>.ulimits` | 进程资源限制（nofile / nproc） | 同上 |
| `x-<自定义名>:` | 自定义扩展字段（YAML 锚点复用配置） | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |

### 10.3 `docker compose` 全局参数（写在子命令之前）

| 参数 | 作用 | 出现在哪 |
|------|------|----------|
| `-f <文件>` / `--file` | 指定 compose 文件，可多次出现以合并 | [[docker/01-入门阶段-基础教程/第04章-多容器编排Compose/4.3-环境变量与多环境配置]] |
| `-p <名字>` / `--project-name` | 指定项目名（影响资源名前缀） | 同上 |
| `--profile <名字>` | 启用某个 profile 的服务 | 同上 |
| `--env-file <文件>` | 指定用于**变量替换**的环境文件 | 同上 |
| `--no-deps` | 只操作指定服务，不动它的依赖 | [[docker/02-中段阶段-专题精讲/编排与自动化/专题概览]] |
| `--force-recreate` | 强制重建容器（即使配置未变） | 同上 |
| `--dry-run` | 只显示将要做的变更，不实际执行 | 同上 |
| `--ansi` / `--progress` | 控制输出样式与进度显示 | [[docker/速查索引]] |

### 10.4 构建相关参数（`docker build` 与 `docker buildx build`）

| 参数 | 作用 | 出现在哪 |
|------|------|----------|
| `-t <名字:tag>` / `--tag` | 给构建出的镜像打标签 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| `-f <文件>` | 指定 Dockerfile（默认 `./Dockerfile`） | 同上 |
| `--build-arg <K=V>` | 传构建期变量（对应 `ARG`） | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.1-Dockerfile是什么与指令详解]] |
| `--no-cache` | 完全不用构建缓存 | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| `--pull` | 强制拉取最新的基础镜像（CI 建议加） | 同上 |
| `--target <阶段名>` | 只构建到某个阶段（多阶段构建用） | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.3-多阶段构建与最佳实践]] |
| `--platform <平台>` | 指定目标平台，如 `linux/amd64,linux/arm64` | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `--progress=plain` | 输出完整构建日志（排错用） | [[docker/01-入门阶段-基础教程/第02章-镜像构建与Dockerfile/2.2-构建过程与分层缓存]] |
| `--push` / `--load` | 构建后推到仓库 / 加载到本地镜像库（**多平台构建只能用 `--push`**） | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `--cache-from` / `--cache-to` | 导入/导出构建缓存（CI 中复用），多阶段需 `mode=max` | 同上 |
| `--secret id=<名>,src=<文件>` | 构建期挂载密钥**且不留在镜像里** | [[docker/02-中段阶段-专题精讲/容器安全加固/专题概览]] |
| `--provenance` / `--sbom` | 生成来源证明与软件物料清单 | [[docker/02-中段阶段-专题精讲/CI-CD与镜像分发/专题概览]] |
| `--squash` | 把全部层压成一层（⚠️ 丢失层共享与缓存） | [[docker/02-中段阶段-专题精讲/镜像优化与瘦身/专题概览]] |

---

## 十一、报错与状态用词

| 词 / 现象 | 含义 | 排查方向 |
|-----------|------|----------|
| `Exited (0)` | 容器正常退出，退出码 0 | 命令跑完了。后台服务要检查是不是**前台运行** | 
| `Exited (1)` | 通用错误退出 | 看 `docker logs` |
| `Exited (137)` | **被 `SIGKILL` 杀掉**，常见原因：内存超限（OOM）或被 `docker kill` | 查内存限制；查 `dmesg` 是否有 OOM | 
| `Exited (143)` | 收到 `SIGTERM`（正常 `docker stop` 的结果） | 正常 | 
| `Exited (126)` / `(127)` | 命令不可执行 / 命令不存在 | 检查 `CMD` 路径、基础镜像里有没有这个程序 | 
| `permission denied` | 权限不足 | 检查文件权限、`USER` 指令、卷挂载的权限 | 
| `no space left on device` | 磁盘满 | `docker system df`，清理镜像/卷/日志 | 
| `port is already allocated` / `Address already in use` | 端口被占用 | 换宿主端口，或找出占用者 | 
| `container name is already in use` | 容器名重复 | 改名，或删掉旧容器 | 
| `Cannot connect to the Docker daemon` | 客户端连不上守护进程 | Docker Desktop 没启动；Linux 上证 `dockerd` 服务与用户组权限 | 
| `permission denied while trying to connect to the Docker daemon socket` | 当前用户不在 `docker` 组 | 把用户加入 docker 组并重新登录 | 
| `pull access denied` / `manifest unknown` | 镜像名或 tag 写错，或私有镜像未登录 | 核对镜像名、tag、`docker login` | 
| `docker: command not found` | 没装或 PATH 不对 | 检查安装 | 
| `exec format error` | 镜像架构与当前机器不符 | 如 arm64 镜像跑在 amd64 上，改用正确架构 | 
| `/bin/sh: 1: xxx: not found` | 容器内没有这个命令 | 基础镜像太精简，或漏装依赖 | 
| `OCI runtime create failed` | 底层运行时创建容器失败 | 常见于 `ENTRYPOINT`/`CMD` 写的命令不存在 | 
| `network xx has active endpoints` | 网络里还有容器 | 先断开或删除容器 | 
| `conflict: unable to delete ... (must be forced)` | 镜像被容器引用 | 先删容器，或加 `-f` |
| `WARNING: retrying in ...` | 拉取超时重试 | 网络问题；配置镜像加速器 |

---

## 十二、查不到某个词时

1. **先确认是不是拼写问题。** Docker 子命令都是小写连字符形式（`docker image ls`），参数是 `--` 双横线。
2. **查官方文档**：<https://docs.docker.com/reference/> 是命令与参数的权威来源；<https://docs.docker.com/compose/compose-file/> 是 compose 字段的权威来源。
3. **在容器里敲 `docker <子命令> --help`**，这是最快的第一手帮助。
4. **还是没有？** 说明本手册有缺漏——直接指出，会补齐。

## 相关页面

- [[docker/README]] — Docker 主题总入口
- [[docker/速查索引]] — 命令、参数与配置字段速查（**只给语法，不给解释**）
- [[docker/学习路线图]] — 三阶段学习路径
- [[docker/01-入门阶段-基础教程/第00章-学前准备/0.4-看懂报错信息]] — 报错的系统排查方法
- [[SCHEMA]] — 知识库操作规范
