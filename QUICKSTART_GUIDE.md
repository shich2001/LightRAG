# LightRAG 快速开始示例

## 🚀 环境准备和启动

### 1. 安装依赖
```bash
# 安装核心依赖
pip install -e .

# 安装 API 服务依赖
pip install -e ".[api]"

# 或者手动安装 API 依赖
pip install fastapi uvicorn gunicorn cryptography passlib PyJWT
```

### 2. 配置环境变量
```bash
# 复制环境变量模板
cp env.example .env

# 编辑 .env 文件
nano .env
```

#### 基本配置示例 (.env)
```env
# 服务器配置
HOST=0.0.0.0
PORT=9621
WORKERS=2

# LLM 配置 (OpenAI)
LLM_BINDING=openai
LLM_MODEL=gpt-4o-mini
LLM_BINDING_HOST=https://api.openai.com/v1
LLM_BINDING_API_KEY=your-openai-api-key

# 嵌入模型配置
EMBEDDING_BINDING=openai
EMBEDDING_MODEL=text-embedding-3-large
EMBEDDING_BINDING_HOST=https://api.openai.com/v1
EMBEDDING_BINDING_API_KEY=your-openai-api-key

# 可选：认证配置
# LIGHTRAG_API_KEY=your-secure-api-key
# AUTH_ACCOUNTS=admin:admin123
# TOKEN_SECRET=your-jwt-secret
```

### 3. 启动服务

#### 开发模式
```bash
# 启动后端服务
lightrag-server

# 或者使用 Python 直接启动
python -m lightrag.api.lightrag_server
```

#### 生产模式
```bash
# 使用 Gunicorn (推荐)
lightrag-gunicorn --workers 4

# 使用 Docker
docker compose up
```

### 4. 前端开发
```bash
# 进入前端目录
cd lightrag_webui

# 安装依赖
npm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build
```

## 🔧 基本使用示例

### 1. 核心 API 使用
```python
import asyncio
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed
from lightrag.kg.shared_storage import initialize_pipeline_status
from lightrag.utils import setup_logger

# 设置日志
setup_logger("lightrag", level="INFO")

async def main():
    # 初始化 LightRAG
    rag = LightRAG(
        working_dir="./rag_storage",
        llm_model_func=gpt_4o_mini_complete,
        embedding_func=openai_embed,
    )
    
    # 必须进行初始化
    await rag.initialize_storages()
    await initialize_pipeline_status()
    
    # 插入文档
    rag.insert("Your document content here")
    
    # 查询
    result = await rag.query(
        "What is the main topic?",
        param=QueryParam(mode="hybrid")
    )
    print(result)

if __name__ == "__main__":
    asyncio.run(main())
```

### 2. HTTP API 使用
```python
import requests

# 基本查询
response = requests.post("http://localhost:9621/query", json={
    "query": "What is LightRAG?",
    "mode": "hybrid"
})
print(response.json())

# 文档上传
with open("document.txt", "rb") as f:
    response = requests.post(
        "http://localhost:9621/documents/file",
        files={"file": f}
    )
print(response.json())

# 获取图谱节点
response = requests.get("http://localhost:9621/graph/nodes")
print(response.json())
```

### 3. JavaScript 前端集成
```javascript
// 查询示例
async function queryRAG(query, mode = 'hybrid') {
    const response = await fetch('/query', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            query: query,
            mode: mode
        })
    });
    
    if (!response.ok) {
        throw new Error(`Query failed: ${response.status}`);
    }
    
    return await response.text();
}

// 文档上传示例
async function uploadDocument(file) {
    const formData = new FormData();
    formData.append('file', file);
    
    const response = await fetch('/documents/file', {
        method: 'POST',
        body: formData
    });
    
    if (!response.ok) {
        throw new Error(`Upload failed: ${response.status}`);
    }
    
    return await response.json();
}

// 使用示例
queryRAG("What are the main topics in the documents?")
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

## 🎨 界面定制示例

### 1. 添加自定义主题
```css
/* lightrag_webui/src/index.css */
:root {
  /* 自定义主题颜色 */
  --primary: 142 76% 36%;        /* 绿色主题 */
  --secondary: 142 76% 90%;      
  --accent: 142 76% 50%;
  --background: 0 0% 100%;
  --foreground: 142 76% 10%;
}

.dark {
  --primary: 142 76% 50%;
  --secondary: 142 76% 10%;
  --accent: 142 76% 60%;
  --background: 142 76% 4%;
  --foreground: 142 76% 90%;
}

/* 自定义组件样式 */
.custom-card {
  @apply bg-gradient-to-r from-primary/10 to-secondary/10;
  @apply border-primary/20 rounded-lg p-6;
  @apply shadow-lg hover:shadow-xl transition-shadow;
}
```

### 2. 自定义组件示例
```typescript
// lightrag_webui/src/components/CustomCard.tsx
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/Card';

interface CustomCardProps {
  title: string;
  content: string;
  action?: () => void;
  className?: string;
}

export const CustomCard: React.FC<CustomCardProps> = ({
  title,
  content,
  action,
  className = ''
}) => {
  return (
    <Card className={`custom-card ${className}`}>
      <CardHeader>
        <CardTitle className="text-primary">{title}</CardTitle>
      </CardHeader>
      <CardContent>
        <p className="text-foreground/80">{content}</p>
        {action && (
          <button
            onClick={action}
            className="mt-4 px-4 py-2 bg-primary text-primary-foreground rounded hover:bg-primary/90"
          >
            执行操作
          </button>
        )}
      </CardContent>
    </Card>
  );
};
```

### 3. 添加新功能页面
```typescript
// lightrag_webui/src/features/CustomFeature.tsx
import React, { useState } from 'react';
import { useTranslation } from 'react-i18next';
import { CustomCard } from '@/components/CustomCard';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';

export const CustomFeature: React.FC = () => {
  const { t } = useTranslation();
  const [inputValue, setInputValue] = useState('');
  const [result, setResult] = useState('');

  const handleProcess = async () => {
    try {
      const response = await fetch('/custom/process', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ input: inputValue })
      });
      
      const data = await response.json();
      setResult(data.result);
    } catch (error) {
      console.error('处理失败:', error);
    }
  };

  return (
    <div className="p-6 space-y-6">
      <CustomCard
        title="自定义功能"
        content="这是一个自定义功能页面的示例"
      />
      
      <div className="space-y-4">
        <Input
          placeholder="输入内容..."
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
        />
        
        <Button onClick={handleProcess} disabled={!inputValue}>
          处理
        </Button>
        
        {result && (
          <div className="p-4 bg-secondary/10 rounded-lg">
            <p>处理结果: {result}</p>
          </div>
        )}
      </div>
    </div>
  );
};
```

## 🔧 后端扩展示例

### 1. 添加自定义路由
```python
# lightrag/api/routers/custom_routes.py
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import Dict, Any
from datetime import datetime

router = APIRouter(prefix="/custom", tags=["custom"])

class ProcessRequest(BaseModel):
    input: str
    options: Dict[str, Any] = {}

class ProcessResponse(BaseModel):
    result: str
    timestamp: datetime
    metadata: Dict[str, Any]

@router.post("/process", response_model=ProcessResponse)
async def custom_process(request: ProcessRequest):
    """自定义处理端点"""
    try:
        # 实现自定义处理逻辑
        processed_result = f"已处理: {request.input}"
        
        return ProcessResponse(
            result=processed_result,
            timestamp=datetime.now(),
            metadata={"input_length": len(request.input)}
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/status")
async def custom_status():
    """获取自定义功能状态"""
    return {
        "status": "active",
        "version": "1.0.0",
        "features": ["processing", "analytics"]
    }
```

### 2. 集成到主应用
```python
# lightrag/api/lightrag_server.py
from lightrag.api.routers.custom_routes import router as custom_router

def create_app(args):
    app = FastAPI(
        title="LightRAG API",
        description="Advanced RAG System with Custom Features",
        version="1.0.0"
    )
    
    # 添加自定义路由
    app.include_router(custom_router)
    
    return app
```

### 3. 数据库操作示例
```python
# utils/database.py
import asyncio
from lightrag import LightRAG
from lightrag.kg.shared_storage import initialize_pipeline_status

class DatabaseManager:
    def __init__(self, working_dir: str):
        self.working_dir = working_dir
        self.rag = None
        
    async def initialize(self):
        """初始化数据库连接"""
        self.rag = LightRAG(
            working_dir=self.working_dir,
            llm_model_func=your_llm_func,
            embedding_func=your_embedding_func
        )
        await self.rag.initialize_storages()
        await initialize_pipeline_status()
        
    async def add_document(self, content: str, metadata: dict = None):
        """添加文档到知识库"""
        if not self.rag:
            await self.initialize()
        
        # 插入文档
        await self.rag.insert(content)
        
        # 可选：存储额外元数据
        if metadata:
            await self.store_metadata(content, metadata)
    
    async def query_knowledge(self, query: str, mode: str = "hybrid"):
        """查询知识库"""
        if not self.rag:
            await self.initialize()
            
        from lightrag import QueryParam
        result = await self.rag.query(
            query,
            param=QueryParam(mode=mode)
        )
        return result
```

## 🐳 Docker 部署示例

### 1. 自定义 Dockerfile
```dockerfile
FROM python:3.12-slim

# 设置工作目录
WORKDIR /app

# 复制文件
COPY requirements.txt .
COPY lightrag/ lightrag/
COPY lightrag_webui/dist/ lightrag_webui/dist/

# 安装依赖
RUN pip install -r requirements.txt

# 暴露端口
EXPOSE 9621

# 启动命令
CMD ["lightrag-server", "--host", "0.0.0.0", "--port", "9621"]
```

### 2. docker-compose.yml
```yaml
version: '3.8'
services:
  lightrag:
    build: .
    ports:
      - "9621:9621"
    environment:
      - LLM_BINDING=openai
      - LLM_API_KEY=${OPENAI_API_KEY}
      - EMBEDDING_BINDING=openai
      - EMBEDDING_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./data:/app/data
      - ./.env:/app/.env
    restart: unless-stopped
    
  # 可选：添加数据库
  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=lightrag
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    
volumes:
  postgres_data:
```

## 📊 监控和日志

### 1. 日志配置
```python
# config/logging.py
import logging
from logging.handlers import RotatingFileHandler

def setup_logging():
    """设置日志配置"""
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.StreamHandler(),
            RotatingFileHandler(
                'lightrag.log',
                maxBytes=10*1024*1024,  # 10MB
                backupCount=5
            )
        ]
    )
```

### 2. 性能监控
```python
# middleware/metrics.py
from prometheus_client import Counter, Histogram, Gauge
import time

# 定义指标
REQUEST_COUNT = Counter('lightrag_requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('lightrag_request_duration_seconds', 'Request latency')
ACTIVE_CONNECTIONS = Gauge('lightrag_active_connections', 'Active connections')

async def metrics_middleware(request, call_next):
    start_time = time.time()
    
    # 记录请求
    REQUEST_COUNT.labels(method=request.method, endpoint=request.url.path).inc()
    
    response = await call_next(request)
    
    # 记录延迟
    REQUEST_LATENCY.observe(time.time() - start_time)
    
    return response
```

这个快速开始指南为您提供了 LightRAG 的基本使用方法和定制示例，您可以根据这些示例快速上手并进行个性化定制。