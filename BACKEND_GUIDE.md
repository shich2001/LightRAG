# LightRAG 后端 API 修改指南

## 🔧 后端架构和 API 扩展指南

### 1. 后端架构概览

```
lightrag/api/
├── __init__.py
├── lightrag_server.py         # 主服务器文件
├── config.py                  # 配置管理
├── auth.py                    # 认证模块
├── utils_api.py              # API 工具函数
├── routers/                   # 路由模块
│   ├── __init__.py
│   ├── document_routes.py     # 文档管理路由
│   ├── query_routes.py        # 查询路由
│   ├── graph_routes.py        # 图谱路由
│   └── ollama_api.py         # Ollama 兼容接口
└── webui/                     # 静态文件服务
```

### 2. 主要 API 端点详解

#### 文档管理 API (document_routes.py)
```python
# 核心端点：
POST /documents/text           # 插入文本内容
POST /documents/file           # 上传单个文件
POST /documents/batch          # 批量上传文件
POST /documents/scan           # 扫描输入目录
DELETE /documents              # 清空所有文档
GET /documents/status          # 获取文档状态
DELETE /documents/{doc_id}     # 删除特定文档

# 请求/响应格式：
class TextInsertRequest(BaseModel):
    text: str
    description: Optional[str] = None
    ids: Optional[List[str]] = None
    
class DocumentStatusResponse(BaseModel):
    id: str
    status: DocStatus
    file_path: Optional[str] = None
    created_at: datetime
    updated_at: datetime
```

#### 查询 API (query_routes.py)
```python
# 核心端点：
POST /query                    # 标准查询
POST /query/stream             # 流式查询

# 查询参数：
class QueryRequest(BaseModel):
    query: str
    mode: QueryMode = "hybrid"
    stream: bool = False
    top_k: int = 50
    max_token_for_text_unit: int = 4000
    max_token_for_global_context: int = 4000
    max_token_for_local_context: int = 4000
    
# 查询模式：
QueryMode = Literal["local", "global", "hybrid", "naive", "mix"]
```

#### 图谱 API (graph_routes.py)
```python
# 核心端点：
GET /graph/nodes               # 获取图谱节点
GET /graph/edges               # 获取图谱边
POST /graph/search             # 搜索图谱
GET /graph/schema              # 获取图谱模式
POST /graph/subgraph           # 获取子图

# 响应格式：
class NodeResponse(BaseModel):
    id: str
    label: str
    properties: Dict[str, Any]
    
class EdgeResponse(BaseModel):
    id: str
    source: str
    target: str
    label: str
    properties: Dict[str, Any]
```

### 3. 认证和安全系统

#### JWT 认证实现
```python
# auth.py
class AuthHandler:
    def __init__(self):
        self.accounts = self._load_accounts()
        self.secret_key = os.getenv("TOKEN_SECRET", "default-secret")
        self.algorithm = "HS256"
        
    def authenticate_user(self, username: str, password: str) -> Optional[User]:
        """用户认证"""
        if username in self.accounts:
            if bcrypt.checkpw(password.encode(), self.accounts[username].encode()):
                return User(username=username)
        return None
        
    def create_access_token(self, data: dict) -> str:
        """创建访问令牌"""
        expire = datetime.utcnow() + timedelta(hours=4)
        to_encode = data.copy()
        to_encode.update({"exp": expire})
        return jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
```

#### API 密钥保护
```python
# utils_api.py
def get_combined_auth_dependency():
    """组合认证依赖项"""
    async def combined_auth(
        request: Request,
        token: Optional[str] = Depends(get_current_user_optional),
        api_key: Optional[str] = Depends(get_api_key_optional)
    ):
        # API 密钥验证
        if api_key and verify_api_key(api_key):
            return {"type": "api_key", "user": None}
            
        # JWT 令牌验证
        if token:
            return {"type": "jwt", "user": token}
            
        # 检查是否在白名单路径
        if is_whitelisted_path(request.url.path):
            return {"type": "whitelist", "user": None}
            
        raise HTTPException(
            status_code=401,
            detail="Authentication required"
        )
        
    return combined_auth
```

### 4. 自定义路由扩展

#### 创建新的路由模块
```python
# routers/custom_routes.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import List, Dict, Any

router = APIRouter(prefix="/custom", tags=["custom"])

class CustomRequest(BaseModel):
    data: Dict[str, Any]
    parameters: Dict[str, Any]

class CustomResponse(BaseModel):
    result: Any
    status: str
    metadata: Dict[str, Any]

@router.post("/process", response_model=CustomResponse)
async def custom_process(
    request: CustomRequest,
    auth: dict = Depends(get_combined_auth_dependency())
):
    """自定义处理端点"""
    try:
        # 实现自定义逻辑
        result = await process_custom_data(request.data, request.parameters)
        
        return CustomResponse(
            result=result,
            status="success",
            metadata={"processed_at": datetime.utcnow()}
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/status")
async def get_custom_status():
    """获取自定义状态"""
    return {"status": "active", "version": "1.0.0"}
```

#### 将新路由添加到主应用
```python
# lightrag_server.py
from lightrag.api.routers.custom_routes import router as custom_router

def create_app(args):
    app = FastAPI(
        title="LightRAG API",
        description="Advanced RAG System API",
        version=__api_version__
    )
    
    # 添加自定义路由
    app.include_router(custom_router)
    
    return app
```

### 5. 数据库和存储扩展

#### 添加新的存储后端
```python
# lightrag/storage/custom_storage.py
from lightrag.base import BaseKVStorage
from typing import Dict, Any, Optional

class CustomKVStorage(BaseKVStorage):
    def __init__(self, namespace: str, config: Dict[str, Any]):
        super().__init__(namespace)
        self.config = config
        self.client = self._create_client()
        
    def _create_client(self):
        """创建自定义存储客户端"""
        # 实现自定义存储连接逻辑
        pass
        
    async def all_keys(self) -> List[str]:
        """获取所有键"""
        # 实现获取所有键的逻辑
        pass
        
    async def get_by_id(self, id: str) -> Optional[Dict[str, Any]]:
        """根据ID获取数据"""
        # 实现根据ID获取数据的逻辑
        pass
        
    async def set_by_id(self, id: str, data: Dict[str, Any]) -> None:
        """根据ID设置数据"""
        # 实现根据ID设置数据的逻辑
        pass
        
    async def delete_by_id(self, id: str) -> None:
        """根据ID删除数据"""
        # 实现根据ID删除数据的逻辑
        pass
```

### 6. 中间件和钩子

#### 请求日志中间件
```python
# middleware/logging.py
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import time
import logging

logger = logging.getLogger(__name__)

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        
        # 记录请求信息
        logger.info(f"Request: {request.method} {request.url}")
        
        response = await call_next(request)
        
        # 记录响应信息
        process_time = time.time() - start_time
        logger.info(f"Response: {response.status_code} - {process_time:.2f}s")
        
        return response

# 在主应用中添加
app.add_middleware(LoggingMiddleware)
```

#### 错误处理中间件
```python
# middleware/error_handling.py
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

class ErrorHandlingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            response = await call_next(request)
            return response
        except HTTPException as e:
            return JSONResponse(
                status_code=e.status_code,
                content={"error": e.detail, "type": "http_exception"}
            )
        except Exception as e:
            logger.error(f"Unhandled exception: {str(e)}")
            return JSONResponse(
                status_code=500,
                content={"error": "Internal server error", "type": "server_error"}
            )
```

### 7. 异步任务处理

#### 后台任务实现
```python
# tasks/background_tasks.py
from fastapi import BackgroundTasks
import asyncio
from typing import Any, Dict

class TaskManager:
    def __init__(self):
        self.tasks = {}
        
    async def create_task(self, task_id: str, func, *args, **kwargs):
        """创建后台任务"""
        task = asyncio.create_task(func(*args, **kwargs))
        self.tasks[task_id] = {
            "task": task,
            "status": "running",
            "created_at": datetime.utcnow()
        }
        return task_id
        
    async def get_task_status(self, task_id: str) -> Dict[str, Any]:
        """获取任务状态"""
        if task_id not in self.tasks:
            return {"status": "not_found"}
            
        task_info = self.tasks[task_id]
        task = task_info["task"]
        
        if task.done():
            if task.exception():
                return {"status": "failed", "error": str(task.exception())}
            else:
                return {"status": "completed", "result": task.result()}
        else:
            return {"status": "running"}

# 在路由中使用
@router.post("/process-async")
async def process_async(
    data: Dict[str, Any],
    background_tasks: BackgroundTasks
):
    task_id = str(uuid.uuid4())
    
    # 添加后台任务
    background_tasks.add_task(
        task_manager.create_task,
        task_id,
        process_large_data,
        data
    )
    
    return {"task_id": task_id, "status": "started"}
```

### 8. 配置管理

#### 环境变量配置
```python
# config.py
from pydantic import BaseSettings
from typing import List, Optional

class Settings(BaseSettings):
    # 服务器配置
    host: str = "0.0.0.0"
    port: int = 9621
    workers: int = 1
    
    # LLM 配置
    llm_binding: str = "openai"
    llm_model: str = "gpt-4o-mini"
    llm_api_key: Optional[str] = None
    llm_base_url: Optional[str] = None
    
    # 嵌入配置
    embedding_binding: str = "openai"
    embedding_model: str = "text-embedding-3-large"
    embedding_api_key: Optional[str] = None
    
    # 数据库配置
    kv_storage: str = "JsonKVStorage"
    vector_storage: str = "NanoVectorDBStorage"
    graph_storage: str = "NetworkXStorage"
    
    # 安全配置
    api_key: Optional[str] = None
    jwt_secret: str = "default-secret"
    jwt_expire_hours: int = 24
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

### 9. 性能优化

#### 缓存机制
```python
# utils/cache.py
from functools import wraps
from typing import Any, Callable
import asyncio
import hashlib
import json

class AsyncCache:
    def __init__(self, ttl: int = 3600):
        self.cache = {}
        self.ttl = ttl
        
    def _generate_key(self, func_name: str, args: tuple, kwargs: dict) -> str:
        """生成缓存键"""
        key_data = {
            "func": func_name,
            "args": args,
            "kwargs": kwargs
        }
        key_str = json.dumps(key_data, sort_keys=True)
        return hashlib.md5(key_str.encode()).hexdigest()
        
    def cache_async(self, ttl: int = None):
        """异步缓存装饰器"""
        def decorator(func: Callable) -> Callable:
            @wraps(func)
            async def wrapper(*args, **kwargs):
                cache_key = self._generate_key(func.__name__, args, kwargs)
                
                # 检查缓存
                if cache_key in self.cache:
                    cached_data = self.cache[cache_key]
                    if time.time() - cached_data["timestamp"] < (ttl or self.ttl):
                        return cached_data["result"]
                
                # 执行函数并缓存结果
                result = await func(*args, **kwargs)
                self.cache[cache_key] = {
                    "result": result,
                    "timestamp": time.time()
                }
                
                return result
            return wrapper
        return decorator

# 使用缓存
cache = AsyncCache(ttl=1800)  # 30 分钟缓存

@cache.cache_async(ttl=3600)
async def expensive_operation(data: dict) -> dict:
    # 执行耗时操作
    await asyncio.sleep(5)
    return {"processed": data}
```

### 10. 监控和日志

#### 健康检查端点
```python
# routers/health.py
from fastapi import APIRouter
from datetime import datetime
import psutil
import os

router = APIRouter()

@router.get("/health")
async def health_check():
    """系统健康检查"""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow(),
        "version": __version__,
        "system": {
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "disk_percent": psutil.disk_usage('/').percent
        }
    }

@router.get("/metrics")
async def get_metrics():
    """获取系统指标"""
    return {
        "requests_total": request_counter.get(),
        "active_connections": connection_counter.get(),
        "response_times": response_time_histogram.get(),
        "errors_total": error_counter.get()
    }
```

#### 结构化日志
```python
# utils/logging.py
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno
        }
        
        if hasattr(record, 'request_id'):
            log_entry["request_id"] = record.request_id
            
        return json.dumps(log_entry)

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('app.log')
    ]
)
```

### 11. 测试框架

#### API 测试
```python
# tests/test_api.py
import pytest
from fastapi.testclient import TestClient
from lightrag.api.lightrag_server import create_app

@pytest.fixture
def client():
    app = create_app(test_config)
    return TestClient(app)

def test_query_endpoint(client):
    """测试查询端点"""
    response = client.post("/query", json={
        "query": "What is LightRAG?",
        "mode": "hybrid"
    })
    assert response.status_code == 200
    assert "result" in response.json()

def test_document_upload(client):
    """测试文档上传"""
    with open("test_document.txt", "rb") as f:
        response = client.post("/documents/file", files={"file": f})
    assert response.status_code == 200
```

这个指南为您提供了完整的后端 API 修改和扩展方案，您可以根据具体需求进行相应的开发和定制。