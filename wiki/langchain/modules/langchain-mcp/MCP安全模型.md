---
tags: [langchain, mcp, security, authentication, permissions]
type: concept
created: 2026-07-11
updated: 2026-07-20
---

# MCP 安全模型

MCP 集成引入了新的安全考量：Agent 可以调用外部服务器提供的工具，这意味着信任边界扩展到了第三方代码。本文档覆盖威胁模型、认证方式、权限控制和生产部署最佳实践。

## 威胁模型

使用 MCP 时面临的主要安全威胁：

| 威胁 | 描述 | 影响 |
|------|------|------|
| **恶意 MCP 服务器** | 返回有害数据、执行危险操作 | 数据泄露、系统损坏 |
| **中间人攻击** | 远程 MCP 服务器通信被窃听或篡改 | 凭据泄露、数据篡改 |
| **工具滥用** | Agent 被诱导调用危险工具 | 未授权操作、数据删除 |
| **提示词注入** | MCP 服务器返回的内容包含恶意指令 | Agent 行为被劫持 |
| **权限提升** | MCP 服务器利用宿主进程权限 | 系统级破坏 |

## 本地服务器安全（stdio）

### 天然隔离优势

stdio 模式通过进程间通信，不暴露网络端口，具备天然的安全优势：

```python
# stdio 配置 —— 仅本地进程可访问
config = {
    "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": {"GITHUB_TOKEN": os.environ.get("GITHUB_TOKEN", "")},
        "transport": "stdio",
    },
}
```

### 权限风险

**注意：** stdio 模式下，服务器进程拥有与宿主进程**相同的权限**。

```python
# 危险：MCP 服务器可以访问宿主进程能访问的一切
config = {
    "untrusted-server": {
        "command": "python",
        "args": ["some_untrusted_server.py"],
        "transport": "stdio",
        # 该进程可以读取文件系统、网络、环境变量...
    },
}
```

### 沙箱隔离

对于不可信的 MCP 服务器，建议在沙箱或容器中运行：

```python
import os

# Docker 沙箱示例
config = {
    "sandboxed-server": {
        "command": "docker",
        "args": [
            "run", "--rm", "-i",
            "--network=none",           # 禁用网络
            "--read-only",              # 只读文件系统
            "--memory=256m",            # 内存限制
            "--cpus=0.5",              # CPU 限制
            "--security-opt=no-new-privileges",
            "mcp-server-image:latest",
        ],
        "transport": "stdio",
    },
}

# 或使用 nsjail / firejail 等沙箱工具
config = {
    "jailed-server": {
        "command": "firejail",
        "args": [
            "--net=none",
            "--private",
            "--rlimit-as=256000000",
            "python", "untrusted_server.py",
        ],
        "transport": "stdio",
    },
}
```

## 远程服务器安全（HTTP/SSE）

远程 MCP 服务器通过网络通信，需要额外的安全措施。

### OAuth 2.1 认证

MCP 规范推荐使用 OAuth 2.1 进行远程服务器认证：

```python
from mcp.client.auth import OAuthHandler
from langchain-mcp.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "remote-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "auth": OAuthHandler(
            client_id="your-client-id",
            client_secret="your-secret",
            token_url="https://auth.example.com/token",
        ),
    },
})
```

**OAuth 2.1 要点：**

- 必须使用 HTTPS（禁止 HTTP）
- 支持 PKCE（Proof Key for Code Exchange）
- Token 有过期时间，需定期刷新
- 适合多用户、多租户场景

### API Key 认证（简单场景）

对于内部服务或简单场景，可以使用 API Key：

```python
client = MultiServerMCPClient({
    "api-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "headers": {"Authorization": "Bearer your-api-key"},
    },
})
```

**API Key 安全要点：**

- 永远不要硬编码 API Key（使用环境变量）
- 定期轮换 API Key
- 为每个 MCP 服务器使用独立的 Key
- 限制 Key 的权限范围

```python
import os

# 正确：从环境变量读取
client = MultiServerMCPClient({
    "api-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "headers": {
            "Authorization": f"Bearer {os.environ['MCP_API_KEY']}",
        },
    },
})
```

### JWT 认证

JWT（JSON Web Token）是无状态认证方案，token 本身携带用户信息和权限，服务端无需查 session。MCP 中使用 JWT 分客户端和服务端两侧。

#### JWT 结构

```python
import jwt, time

# 签发 JWT（认证服务器完成）
payload = {
    "sub": "user-123",              # 用户标识
    "name": "Alice",
    "scopes": ["read", "write"],    # 权限范围
    "iat": int(time.time()),        # 签发时间
    "exp": int(time.time()) + 3600, # 过期时间
}
token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
# token 是一个长字符串，客户端携带它发起请求
```

#### 客户端：直接使用已有 JWT

```python
import os
from langchain_mcp_adapters.client import MultiServerMCPClient

jwt_token = os.environ.get("MCP_JWT_TOKEN", "")
client = MultiServerMCPClient({
    "my-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "headers": {"Authorization": f"Bearer {jwt_token}"},
    },
})
```

#### 客户端：自动刷新 JWT

JWT 有过期时间，长时间运行的 Agent 需要自动续期：

```python
import httpx, time, os

class JWTAuth(httpx.Auth):
    """自动获取和刷新 JWT 的认证类"""
    def __init__(self, client_id: str, client_secret: str, token_url: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self._token = None
        self._expires_at = 0

    def _get_token(self) -> str:
        if time.time() > self._expires_at - 60:  # 提前 60 秒刷新
            resp = httpx.post(self.token_url, json={
                "client_id": self.client_id,
                "client_secret": self.client_secret,
                "grant_type": "client_credentials",
            })
            data = resp.json()
            self._token = data["access_token"]
            self._expires_at = time.time() + data.get("expires_in", 3600)
        return self._token

    def auth_flow(self, request):
        token = self._get_token()
        request.headers["Authorization"] = f"Bearer {token}"
        yield request

# 使用
client = MultiServerMCPClient({
    "my-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "auth": JWTAuth(
            client_id=os.environ["CLIENT_ID"],
            client_secret=os.environ["CLIENT_SECRET"],
            token_url="https://auth.example.com/token",
        ),
    },
})
```

#### 服务端：验证 JWT

```python
# pip install PyJWT
import jwt, os
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("secure-server")
SECRET_KEY = os.environ["JWT_SECRET"]

def verify_token(token: str) -> dict:
    """验证 JWT 签名和有效期，返回 payload。失败抛异常。"""
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise PermissionError("Token 已过期")
    except jwt.InvalidTokenError:
        raise PermissionError("Token 无效")
```

#### 服务端：权限装饰器

统一处理认证和权限检查，避免每个工具重复写验证逻辑：

```python
def require_auth(scopes: list[str] = None):
    """认证 + 权限检查装饰器"""
    def decorator(func):
        async def wrapper(*args, token: str = "", **kwargs):
            payload = verify_token(token)
            if scopes:
                user_scopes = payload.get("scopes", [])
                missing = set(scopes) - set(user_scopes)
                if missing:
                    return f"权限不足，缺少: {missing}"
            return await func(*args, **kwargs)
        wrapper.__name__ = func.__name__
        wrapper.__doc__ = func.__doc__
        return wrapper
    return decorator

@mcp.tool()
@require_auth(scopes=["read"])
async def list_projects(token: str) -> str:
    """列出项目（需要 read 权限）"""
    return "Project A, Project B"

@mcp.tool()
@require_auth(scopes=["write"])
async def create_project(name: str, token: str) -> str:
    """创建项目（需要 write 权限）"""
    return f"已创建: {name}"
```

#### JWT 认证流程

```
认证服务器                  客户端                      MCP 服务器
    │                        │                            │
    │  1. 登录，返回 JWT      │                            │
    │ ◄────────────────────── │                            │
    │                        │  2. Authorization: Bearer xxx│
    │                        │ ──────────────────────────► │
    │                        │                  3. 验证签名 │
    │                        │                  4. 检查权限 │
    │                        │     5. 返回结果              │
    │                        │ ◄────────────────────────── │
    │                        │                              │
    │                  6. JWT 过期                           │
    │  7. 请求新 token       │                              │
    │ ◄────────────────────── │                              │
```

#### JWT vs API Key 选择

| 维度 | JWT | API Key |
|------|-----|---------|
| 状态 | 无状态，token 自带信息 | 有状态，需查库验证 |
| 权限控制 | token 内嵌 scopes | 服务端额外配置 |
| 过期管理 | token 自带 exp | 需额外实现 |
| 适用场景 | 多用户、分布式系统 | 内部服务、简单场景 |
| 安全性 | 高（签名+过期） | 中（泄露即失控） |

> 如果使用 OAuth 2.1，其 access token 通常就是 JWT，无需单独实现。

### FastMCP 内置认证组件（TokenVerifier + RSA）

MCP Python SDK 提供了内置的认证框架，核心是两个组件：

| 组件 | 作用 |
|------|------|
| `TokenVerifier` | 协议类，实现 `verify_token()` 方法验证 token |
| `AuthSettings` | 配置认证服务器地址、资源 URL、所需 scope |

SDK 自动完成：拦截所有请求 → 提取 `Authorization: Bearer <token>` → 调用你的 `verify_token()` → 无效返回 401 → 有效放行到工具。**工具函数无需感知认证。**

#### 架构流程

```
客户端请求 → Authorization: Bearer <JWT>
    │
    ▼
SDK 自动拦截（BearerAuthBackend + RequireAuthMiddleware）
    │
    ▼
调用 TokenVerifier.verify_token(token)
    │
    ├── 返回 AccessToken → 工具正常执行
    └── 返回 None → 401 Unauthorized
```

#### RSA 公钥验证实现

```python
# pip install "mcp[cli]" PyJWT[crypto]
from mcp.server.fastmcp import FastMCP
from mcp.server.auth.settings import AuthSettings
from mcp.server.auth.provider import TokenVerifier, AccessToken
import jwt

class RSATokenVerifier(TokenVerifier):
    """使用 RSA 公钥验证 JWT 签名"""

    def __init__(self, public_key: str, issuer: str, audience: str):
        self.public_key = public_key  # RSA 公钥（PEM 格式）
        self.issuer = issuer
        self.audience = audience

    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=["RS256"],
                issuer=self.issuer,
                audience=self.audience,
            )
            return AccessToken(
                token=token,
                client_id=payload.get("client_id", "unknown"),
                scopes=payload.get("scope", "").split(),
                expires_at=payload.get("exp"),
                subject=payload.get("sub"),
                claims=payload,
            )
        except jwt.ExpiredSignatureError:
            return None
        except jwt.InvalidTokenError:
            return None
```

#### 创建带认证的 FastMCP 服务器

```python
RSA_PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"""

token_verifier = RSATokenVerifier(
    public_key=RSA_PUBLIC_KEY,
    issuer="https://auth.example.com",
    audience="https://mcp.example.com/mcp",
)

mcp = FastMCP(
    "secure-server",
    token_verifier=token_verifier,
    auth=AuthSettings(
        issuer_url="https://auth.example.com",
        resource_server_url="https://mcp.example.com/mcp",
        required_scopes=["mcp:access"],
    ),
)

# 工具定义 — 不需要任何认证代码，SDK 自动保护
@mcp.tool()
async def get_time() -> dict:
    """获取服务器时间"""
    from datetime import datetime
    return {"time": datetime.now().isoformat()}

@mcp.tool()
async def query_data(query: str) -> str:
    """查询数据"""
    return f"结果: {query}"

if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="0.0.0.0", port=8000)
```

#### RSA 密钥对生成（认证服务器侧）

```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

# 生成 2048 位 RSA 密钥对
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)

# 私钥 PEM（认证服务器持有，用于签发 JWT）
private_pem = private_key.private_bytes(
    serialization.Encoding.PEM,
    serialization.PrivateFormat.PKCS8,
    serialization.NoEncryption(),
).decode()

# 公钥 PEM（MCP 服务器持有，用于验证 JWT）
public_pem = private_key.public_key().public_bytes(
    serialization.Encoding.PEM,
    serialization.PublicFormat.SubjectPublicKeyInfo,
).decode()

# 认证服务器用私钥签发 JWT
import jwt, time

token = jwt.encode(
    {
        "sub": "user-123",
        "client_id": "my-agent",
        "scope": "mcp:access",
        "iss": "https://auth.example.com",
        "aud": "https://mcp.example.com/mcp",
        "iat": int(time.time()),
        "exp": int(time.time()) + 3600,
    },
    private_pem,
    algorithm="RS256",
)
```

#### JWKS 动态获取公钥（生产推荐）

生产环境中公钥可能轮换，使用 JWKS 端点动态获取：

```python
class JWKSTokenVerifier(TokenVerifier):
    """从 JWKS 端点动态获取 RSA 公钥"""

    def __init__(self, jwks_url: str, issuer: str, audience: str):
        self.issuer = issuer
        self.audience = audience
        self._jwks_client = jwt.PyJWKClient(jwks_url)

    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            signing_key = self._jwks_client.get_signing_key_from_jwt(token)
            payload = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                issuer=self.issuer,
                audience=self.audience,
            )
            return AccessToken(
                token=token,
                client_id=payload.get("client_id", "unknown"),
                scopes=payload.get("scope", "").split(),
                expires_at=payload.get("exp"),
                subject=payload.get("sub"),
                claims=payload,
            )
        except jwt.InvalidTokenError:
            return None

# 使用
token_verifier = JWKSTokenVerifier(
    jwks_url="https://auth.example.com/.well-known/jwks.json",
    issuer="https://auth.example.com",
    audience="https://mcp.example.com/mcp",
)
```

#### 客户端连接认证服务器

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "secure-server": {
        "url": "https://mcp.example.com/mcp",
        "transport": "streamable_http",
        "headers": {"Authorization": f"Bearer {jwt_token}"},
    },
})
# 未携带有效 token 时，服务器返回 401
```

#### 认证方式对比

| 方式 | 实现位置 | 适用场景 |
|------|---------|---------|
| `TokenVerifier` + RSA | FastMCP 内置 | 生产环境，多用户 |
| `TokenVerifier` + JWKS | FastMCP 内置 | 公钥会轮换的场景 |
| `TokenVerifier` + Introspection | FastMCP 内置 | 需要实时撤销 token |
| ASGI 中间件 | 手动包装 | 需要更细粒度控制 |
| 工具参数传 token | 工具内部 | 快速原型（不推荐） |

### TLS/HTTPS

远程服务器**必须**使用 HTTPS：

```python
# 错误：明文 HTTP
config = {
    "bad-server": {
        "url": "http://mcp.example.com/mcp",  # 不安全！
        "transport": "streamable_http",
    },
}

# 正确：HTTPS
config = {
    "good-server": {
        "url": "https://mcp.example.com/mcp",  # 加密传输
        "transport": "streamable_http",
    },
}
```

## 工具级权限控制

### 工具白名单

限制 Agent 可以使用的 MCP 工具：

```python
from langchain-mcp.client import MultiServerMCPClient

# 允许的工具白名单
ALLOWED_TOOLS = {
    "github": ["list_repositories", "get_file_contents", "search_code"],
    "database": ["query", "list_tables"],
    # 注意：不包含 delete、drop 等危险操作
}

async def load_filtered_tools():
    client = MultiServerMCPClient(mcp_config)
    all_tools = await client.get_tools()

    # 过滤：只保留白名单中的工具
    filtered = [
        t for t in all_tools
        if any(
            t.name in ALLOWED_TOOLS.get(server, [])
            for server in ALLOWED_TOOLS
        )
    ]
    return filtered
```

### 危险工具拦截

使用中间件在调用前检查危险操作，详见 [[langchain/modules/langchain-mcp/MCP与Agent中间件集成|MCP 与 Agent 中间件集成]]：

```python
from langchain.agents.middleware import wrap_tool_call

@wrap_tool_call
def mcp_tool_gatekeeper(request, handler):
    """MCP 工具权限控制"""
    tool_name = request.tool_call["name"]

    # 定义危险工具列表
    dangerous = ["delete_file", "execute_sql", "send_email", "drop_table"]
    if tool_name in dangerous:
        # 检查是否已获审批
        approved = request.runtime.state.get("approved_tools", set())
        if tool_name not in approved:
            return ToolResponse(output=f"⚠️ {tool_name} 需要审批")

    return handler(request)
```

### 只读模式

对于数据库类 MCP 服务器，可以限制为只读操作：

```python
@wrap_tool_call
def readonly_database_guard(request, handler):
    """数据库只读保护"""
    tool_name = request.tool_call["name"]

    if tool_name == "execute_sql":
        sql = request.tool_call["args"].get("query", "").upper().strip()
        # 禁止写操作
        write_keywords = ["INSERT", "UPDATE", "DELETE", "DROP", "ALTER", "CREATE", "TRUNCATE"]
        if any(sql.startswith(kw) for kw in write_keywords):
            return ToolResponse(output="⚠️ 数据库为只读模式，禁止写操作")

    return handler(request)
```

## 输入验证

### 服务器端验证

MCP 服务器必须验证所有工具参数：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("validated-server")

@mcp.tool()
async def query_user(user_id: str) -> str:
    """查询用户信息"""
    # 类型验证
    if not isinstance(user_id, str):
        return "错误：user_id 必须是字符串"

    # 长度验证
    if len(user_id) > 64:
        return "错误：user_id 过长"

    # 格式验证（防止 SQL 注入等）
    if not user_id.isalnum():
        return "错误：user_id 只能包含字母和数字"

    # 范围验证
    if len(user_id) < 3:
        return "错误：user_id 至少 3 个字符"

    # 安全查询...
    return f"User: {user_id}"
```

### Agent 端验证

使用中间件在调用前校验参数：

```python
@wrap_tool_call
def input_validator(request, handler):
    """MCP 工具输入参数校验"""
    tool_name = request.tool_call["name"]
    args = request.tool_call["args"]

    # 通用校验：参数不能过大
    for key, value in args.items():
        if isinstance(value, str) and len(value) > 10000:
            return ToolResponse(output=f"⚠️ 参数 {key} 过长（>{10000} 字符）")

    # 特定工具校验
    if tool_name == "execute_sql":
        sql = args.get("query", "")
        if ";" in sql:
            return ToolResponse(output="⚠️ 不允许执行多条 SQL 语句")

    return handler(request)
```

### 输出验证（防止提示词注入）

**不要信任 MCP 服务器返回的数据**——可能包含注入内容：

```python
@wrap_tool_call
def output_sanitizer(request, handler):
    """清理 MCP 工具输出，防止提示词注入"""
    result = handler(request)
    output = result.output

    if isinstance(output, str):
        # 检测可疑的提示词注入模式
        injection_patterns = [
            "ignore previous instructions",
            "you are now",
            "system:",
            "assistant:",
        ]
        output_lower = output.lower()
        for pattern in injection_patterns:
            if pattern in output_lower:
                # 标记可疑内容
                output = f"[注意：以下内容可能包含注入尝试]\n{output}"
                break

    return ToolResponse(output=output)
```

## 生产部署清单

部署 MCP 集成到生产环境前，逐项检查：

### 传输安全

- [ ] 远程服务器使用 HTTPS（禁止 HTTP）
- [ ] stdio 服务器在沙箱/容器中运行（不可信场景）
- [ ] 内部 SSE/WebSocket 服务器在内网或 VPN 中

### 认证

- [ ] 远程服务器启用 OAuth 2.1、JWT 或 API Key 认证
- [ ] API Key / JWT Secret 通过环境变量传递（不硬编码）
- [ ] API Key 定期轮换
- [ ] OAuth Token / JWT 有合理的过期时间
- [ ] 长时间运行的 Agent 配置了 JWT 自动刷新

### 权限控制

- [ ] 限制工具权限（最小权限原则）
- [ ] 配置工具白名单（只暴露必要的工具）
- [ ] 危险操作需要人工审批
- [ ] 数据库类工具限制为只读（如适用）

### 监控与日志

- [ ] 记录所有工具调用日志（工具名、参数、结果、耗时）
- [ ] 设置工具调用频率限制
- [ ] 异常调用告警（如频繁调用删除类工具）
- [ ] 定期审查 MCP 服务器列表

### 容错

- [ ] MCP 服务器连接失败时的降级策略
- [ ] 工具调用超时设置
- [ ] 服务器健康检查

## 安全最佳实践

| 场景 | 推荐做法 |
|------|---------|
| 本地开发 | stdio 传输，无需额外认证 |
| 内部服务 | HTTP + API Key + 内网隔离 |
| 外部服务 | HTTP + OAuth 2.1 / JWT + HTTPS |
| 多用户内部系统 | HTTP + JWT（自动刷新）+ 工具白名单 |
| 不可信服务器 | 沙箱隔离 + 工具白名单 + 输出校验 |
| 数据库操作 | 只读模式 + SQL 注入防护 |
| 文件操作 | 路径白名单 + 目录限制 |
| 多用户场景 | OAuth 2.1 / JWT + 每用户独立权限 |

## 安全审计日志

记录所有 MCP 工具调用，便于事后审计：

```python
import logging
import time
from langchain.agents.middleware import wrap_tool_call

logger = logging.getLogger("mcp.audit")

@wrap_tool_call
def audit_logger(request, handler):
    """MCP 工具调用审计日志"""
    tool_name = request.tool_call["name"]
    args = request.tool_call["args"]
    start = time.time()

    # 记录调用开始
    logger.info(f"[MCP-AUDIT] 调用开始: {tool_name}, 参数: {args}")

    try:
        result = handler(request)
        duration = time.time() - start

        # 记录调用结果
        logger.info(
            f"[MCP-AUDIT] 调用完成: {tool_name}, "
            f"耗时: {duration:.2f}s, "
            f"结果长度: {len(str(result.output))}"
        )
        return result

    except Exception as e:
        duration = time.time() - start
        logger.error(
            f"[MCP-AUDIT] 调用失败: {tool_name}, "
            f"耗时: {duration:.2f}s, "
            f"错误: {e}"
        )
        raise
```

## 相关

- [[langchain/modules/langchain-mcp/概述|<- 返回 mcp-adapters 概述]]
- [[langchain/modules/langchain-mcp/MCP服务器开发|MCP 服务器开发]]
- [[langchain/modules/langchain-mcp/MCP与Agent中间件集成|MCP 与 Agent 中间件集成]]
- [[langchain/modules/langchain-mcp/MultiServerMCPClient|MultiServerMCPClient]]
- [[langchain/modules/langchain-mcp/传输方式对比|传输方式对比]]
- [[langchain/modules/langchain-mcp/MCP协议概述|MCP 协议]]
