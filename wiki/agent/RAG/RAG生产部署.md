---
tags: [concept, RAG, 生产部署, 监控, 缓存, 成本优化]
type: concept
created: 2026-07-08
updated: 2026-07-08
---

# RAG 生产部署

> 从原型到生产，RAG 系统需要考虑缓存、监控、成本控制、扩展性、安全性等多个维度。本文覆盖生产环境的关键实践。

## 生产架构设计

### 典型生产架构

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端层                              │
│  Web UI / API / Chat Interface                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      API 网关层                              │
│  认证/授权、限流、负载均衡、日志                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     RAG 服务层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 查询处理  │→│ 混合检索  │→│ 重排序   │→│ 生成回答  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     基础设施层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 向量数据库│  │ LLM API  │  │ 缓存层   │  │ 监控告警  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 缓存策略

### 查询缓存

```python
from functools import lru_cache
import hashlib

class QueryCache:
    """查询结果缓存"""
    
    def __init__(self, maxsize=10000, ttl=3600):
        self._cache = {}
        self.maxsize = maxsize
        self.ttl = ttl
    
    def _get_key(self, query: str) -> str:
        """生成缓存键（归一化查询）"""
        normalized = query.lower().strip()
        return hashlib.md5(normalized.encode()).hexdigest()
    
    def get(self, query: str):
        key = self._get_key(query)
        if key in self._cache:
            result, timestamp = self._cache[key]
            if time.time() - timestamp < self.ttl:
                return result
            else:
                del self._cache[key]
        return None
    
    def set(self, query: str, result):
        key = self._get_key(query)
        self._cache[key] = (result, time.time())
        
        # LRU 淘汰
        if len(self._cache) > self.maxsize:
            oldest = min(self._cache.items(), key=lambda x: x[1][1][[)
            del self._cache[oldest[0]]

# 使用
cache = QueryCache(maxsize=10000, ttl=3600)

def retrieve_with_cache(query):
    cached = cache.get(query)
    if cached:
        return cached
    
    results = retriever.invoke(query)
    cache.set(query, results)
    return results
```

### 嵌入缓存

```python
class EmbeddingCache:
    """嵌入向量缓存，避免重复计算"""
    
    def __init__(self, embedding_model):
        self.model = embedding_model
        self._cache = {}
    
    def embed_query(self, text: str):
        if text not in self._cache:
            self._cache[text] = self.model.embed_query(text)
        return self._cache[text]
    
    def embed_documents(self, texts: list):
        results = []
        to_embed = []
        
        for text in texts:
            if text in self._cache:
                results.append(self._cache[text])
            else:
                results.append(None)
                to_embed.append(text)
        
        if to_embed:
            embedded = self.model.embed_documents(to_embed)
            for text, vec in zip(to_embed, embedded):
                self._cache[text] = vec
            
            # 填充结果
            idx = 0
            for i, r in enumerate(results):
                if r is None:
                    results[i] = embedded[idx]
                    idx += 1
        
        return results
```

### Redis 分布式缓存

```python
import redis
import json

class RedisRAGCache:
    """Redis 分布式缓存"""
    
    def __init__(self, redis_url="redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1 小时
    
    def get(self, query: str):
        key = f"rag:query:{hashlib.md5(query.encode()).hexdigest()}"
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None
    
    def set(self, query: str, result):
        key = f"rag:query:{hashlib.md5(query.encode()).hexdigest()}"
        self.redis.setex(
            key,
            self.ttl,
            json.dumps(result, ensure_ascii=False)
        )
```

## 监控与告警

### 关键指标监控

```python
import time
from dataclasses import dataclass
from typing import Dict, Any
import logging

logger = logging.getLogger(__name__)

@dataclass
class RAGMetrics:
    """RAG 系统指标"""
    query: str
    latency_ms: float
    retrieval_latency_ms: float
    generation_latency_ms: float
    tokens_used: int
    documents_retrieved: int
    cache_hit: bool
    error: bool = False
    error_message: str = ""

class RAGMonitor:
    """RAG 系统监控"""
    
    def __init__(self):
        self.metrics_history = []
        self.alert_thresholds = {
            "latency_ms": 3000,
            "tokens_used": 4000,
            "error_rate": 0.05,  # 5%
        }
    
    def record(self, metrics: RAGMetrics):
        """记录指标"""
        self.metrics_history.append(metrics)
        
        # 检查告警
        self._check_alerts(metrics)
        
        # 日志记录
        logger.info(
            f"RAG Query: latency={metrics.latency_ms:.0f}ms, "
            f"tokens={metrics.tokens_used}, "
            f"docs={metrics.documents_retrieved}, "
            f"cache={'hit' if metrics.cache_hit else 'miss'}"
        )
    
    def _check_alerts(self, metrics: RAGMetrics):
        """检查告警阈值"""
        if metrics.latency_ms > self.alert_thresholds["latency_ms"]:
            logger.warning(
                f"High latency alert: {metrics.latency_ms:.0f}ms "
                f"(threshold: {self.alert_thresholds['latency_ms']}ms)"
            )
        
        if metrics.tokens_used > self.alert_thresholds["tokens_used"]:
            logger.warning(
                f"High token usage alert: {metrics.tokens_used} tokens "
                f"(threshold: {self.alert_thresholds['tokens_used']})"
            )
    
    def get_stats(self) -> Dict[str, Any]:
        """获取统计信息"""
        if not self.metrics_history:
            return {}
        
        latencies = [m.latency_ms for m in self.metrics_history]
        tokens = [m.tokens_used for m in self.metrics_history]
        errors = [m for m in self.metrics_history if m.error]
        cache_hits = [m for m in self.metrics_history if m.cache_hit]
        
        return {
            "total_queries": len(self.metrics_history),
            "avg_latency_ms": sum(latencies) / len(latencies),
            "p95_latency_ms": sorted(latencies)[int(len(latencies) * 0.95)],
            "avg_tokens": sum(tokens) / len(tokens),
            "error_rate": len(errors) / len(self.metrics_history),
            "cache_hit_rate": len(cache_hits) / len(self.metrics_history),
        }
```

### Prometheus 集成

```python
from prometheus_client import Counter, Histogram, Gauge

# 定义指标
rag_query_total = Counter(
    'rag_queries_total',
    'Total number of RAG queries',
    ['status']  # success, error
)

rag_query_duration = Histogram(
    'rag_query_duration_seconds',
    'RAG query duration in seconds',
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
)

rag_tokens_used = Histogram(
    'rag_tokens_used',
    'Tokens used per RAG query',
    buckets=[100, 500, 1000, 2000, 4000]
)

rag_cache_hit = Counter(
    'rag_cache_hits_total',
    'RAG cache hits'
)

# 使用
def instrumented_rag_query(query: str):
    start_time = time.time()
    
    try:
        result = rag_chain.invoke({"input": query})
        rag_query_total.labels(status='success').inc()
        rag_tokens_used.observe(result.get("usage", {}).get("total_tokens", 0))
        return result
    except Exception as e:
        rag_query_total.labels(status='error').inc()
        raise
    finally:
        rag_query_duration.observe(time.time() - start_time)
```

## 成本控制

### Token 成本优化

```python
class CostOptimizer:
    """成本优化器"""
    
    def __init__(self, llm, max_tokens_per_query=2000):
        self.llm = llm
        self.max_tokens = max_tokens_per_query
        self.total_cost = 0
        self.cost_per_1k_tokens = 0.002  # GPT-4o-mini: $0.002/1K tokens
    
    def estimate_cost(self, text: str) -> float:
        """估算成本"""
        tokens = len(text) // 4  # 粗略估算：4 字符 ≈ 1 token
        return (tokens / 1000) * self.cost_per_1k_tokens
    
    def optimize_context(self, contexts: list, max_tokens: int = None) -> list:
        """优化上下文长度"""
        max_tokens = max_tokens or self.max_tokens
        
        # 按相关性排序（假设已重排序）
        optimized = []
        current_tokens = 0
        
        for doc in contexts:
            doc_tokens = len(doc.page_content) // 4
            if current_tokens + doc_tokens <= max_tokens:
                optimized.append(doc)
                current_tokens += doc_tokens
            else:
                break
        
        return optimized
    
    def track_cost(self, tokens_used: int):
        """跟踪成本"""
        cost = (tokens_used / 1000) * self.cost_per_1k_tokens
        self.total_cost += cost
        return cost
```

### 模型选择策略

```python
class ModelRouter:
    """根据查询复杂度路由到不同模型"""
    
    def __init__(self):
        self.simple_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        self.complex_llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    def estimate_complexity(self, query: str) -> str:
        """估算查询复杂度"""
        # 简单规则：关键词数量、问题类型
        if any(kw in query for kw in ["是什么", "什么是", "定义"]):
            return "simple"
        elif any(kw in query for kw in ["分析", "比较", "为什么", "如何"]):
            return "complex"
        return "simple"
    
    def get_llm(self, query: str):
        complexity = self.estimate_complexity(query)
        return self.complex_llm if complexity == "complex" else self.simple_llm
```

## 扩展性设计

### 水平扩展

```python
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
import uvicorn

app = FastAPI(title="RAG API")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# 依赖注入
def get_rag_chain():
    """每个请求独立的 RAG 链"""
    return create_rag_chain()

@app.post("/query")
async def query(
    request: QueryRequest,
    rag_chain = Depends(get_rag_chain)
):
    """查询接口"""
    result = rag_chain.invoke({"input": request.query})
    return {
        "answer": result["answer"],
        "sources": [
            {
                "content": doc.page_content[:200],
                "metadata": doc.metadata,
            }
            for doc in result["context"]
        ],
    }

# 启动多个 worker
# uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4
```

### 异步处理

```python
import asyncio
from langchain_core.runnables import RunnableLambda

class AsyncRAGChain:
    """异步 RAG 链"""
    
    def __init__(self, retriever, llm):
        self.retriever = retriever
        self.llm = llm
    
    async def ainvoke(self, query: str):
        """异步调用"""
        # 并发执行检索和预处理
        retrieval_task = asyncio.create_task(
            self.retriever.ainvoke(query)
        )
        
        # 等待检索完成
        contexts = await retrieval_task
        
        # 生成回答
        response = await self.llm.ainvoke(
            self._format_prompt(query, contexts)
        )
        
        return {
            "answer": response.content,
            "context": contexts,
        }
```

## 安全性

### 输入验证

```python
from pydantic import BaseModel, Field, validator
import re

class QueryRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=1000)
    
    @validator('query')
    def validate_query(cls, v):
        # 防止注入攻击
        if re.search(r'[<>{}]', v):
            raise ValueError("Invalid characters in query")
        return v.strip()

class QueryResponse(BaseModel):
    answer: str
    sources: list
    confidence: float
```

### 输出过滤

```python
class OutputFilter:
    """输出过滤器"""
    
    def __init__(self, blocked_patterns: list = None):
        self.blocked_patterns = blocked_patterns or [
            r"ignore previous instructions",
            r"system prompt",
            r"you are now",
        ]
    
    def filter(self, text: str) -> str:
        """过滤敏感内容"""
        for pattern in self.blocked_patterns:
            if re.search(pattern, text, re.IGNORECASE):
                return "抱歉，我无法回答这个问题。"
        return text
```

### 访问控制

```python
from functools import wraps
from fastapi import HTTPException, Depends

def require_api_key(api_key: str = Depends(api_key_header)):
    """API Key 验证"""
    if api_key not in VALID_API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return api_key

def rate_limit(max_requests: int = 100, window: int = 3600):
    """限流装饰器"""
    def decorator(func):
        request_counts = {}
        
        @wraps(func)
        async def wrapper(*args, **kwargs):
            api_key = kwargs.get('api_key')
            current_time = time.time()
            
            if api_key not in request_counts:
                request_counts[api_key] = []
            
            # 清理过期记录
            request_counts[api_key] = [
                t for t in request_counts[api_key]
                if current_time - t < window
            ]
            
            if len(request_counts[api_key]) >= max_requests:
                raise HTTPException(
                    status_code=429,
                    detail="Rate limit exceeded"
                )
            
            request_counts[api_key].append(current_time)
            return await func(*args, **kwargs)
        
        return wrapper
    return decorator
```

## 部署检查清单

### 上线前检查

- [ ] **性能测试**：P95 延迟 < 3s，QPS 满足需求
- [ ] **成本估算**：月度 Token 成本在预算内
- [ ] **缓存配置**：查询缓存命中率 > 30%
- [ ] **监控告警**：关键指标接入 Prometheus/Grafana
- [ ] **错误处理**：优雅降级，不暴露内部错误
- [ ] **安全审查**：输入验证、输出过滤、API Key 管理
- [ ] **日志记录**：结构化日志，便于排查问题
- [ ][[ **备份恢复**：向量数据库定期备份

### 运维手册

| 场景 | 处理方案 |
|------|---------|
| LLM API 超时 | 自动重试 3 次，降级到备用模型 |
| 向量库连接失败 | 切换到只读副本，告警通知 |
| 缓存失效 | 自动重建缓存，临时降低 QPS |
| 成本超预算 | 自动切换到更便宜的模型，告警通知 |
| 质量下降 | 触发评估流程，回滚到上一版本 |

## 相关页面

- [[agent/RAG/RAG核心概念与演进]] — RAG 定义与三代演进
- [[agent/RAG/RAG评估与优化]] — 评估指标与优化策略
- [[agent/RAG/RAG参数调优手册]] — 全链路参数速查表
- [[python/advanced/异步编程]] — 异步编程基础

## 来源

- LangChain 生产部署指南
- 2025-2026 RAG 生产最佳实践
- 字节跳动 RAG 实践手册
- FastAPI 官方文档
