# LightRAG 前端界面修改指南

## 🎨 前端界面自定义完整指南

### 1. 项目结构详解

```
lightrag_webui/src/
├── components/                 # 通用 UI 组件
│   ├── ui/                    # 基础 UI 组件 (Button, Input, Card 等)
│   ├── documents/             # 文档管理相关组件
│   ├── graph/                 # 图谱可视化组件
│   ├── retrieval/             # 检索测试组件
│   └── status/                # 状态显示组件
├── features/                  # 功能模块
│   ├── DocumentManager.tsx    # 文档管理主界面
│   ├── GraphViewer.tsx        # 知识图谱可视化
│   ├── RetrievalTesting.tsx   # RAG 查询测试
│   ├── ApiSite.tsx           # API 文档展示
│   ├── SiteHeader.tsx        # 顶部导航
│   └── LoginPage.tsx         # 登录页面
├── stores/                    # 状态管理
│   ├── state.ts              # 全局状态
│   ├── settings.ts           # 设置状态
│   └── graph.ts              # 图谱状态
├── api/                       # API 接口
│   └── lightrag.ts           # 后端 API 调用
├── hooks/                     # 自定义 Hooks
├── lib/                       # 工具函数
└── locales/                   # 国际化文件
```

### 2. 主要界面组件分析

#### 主应用组件 (App.tsx)
```typescript
// 主要功能：
// - 标签页切换逻辑
// - 全局状态管理
// - 主题和认证控制
// - 健康检查机制

// 核心结构：
<Tabs defaultValue={currentTab} onValueChange={handleTabChange}>
  <SiteHeader />
  <TabsContent value="documents">
    <DocumentManager />
  </TabsContent>
  <TabsContent value="knowledge-graph">
    <GraphViewer />
  </TabsContent>
  <TabsContent value="retrieval">
    <RetrievalTesting />
  </TabsContent>
  <TabsContent value="api">
    <ApiSite />
  </TabsContent>
</Tabs>
```

#### 文档管理界面 (DocumentManager.tsx)
```typescript
// 主要功能：
// - 文件上传（拖拽支持）
// - 文档状态表格
// - 批量操作
// - 文档扫描

// 关键组件：
// - UploadDocumentsDialog: 上传对话框
// - ClearDocumentsDialog: 清空确认对话框
// - DeleteDocumentsDialog: 删除确认对话框
// - PipelineStatusDialog: 处理状态对话框
```

#### 知识图谱可视化 (GraphViewer.tsx)
```typescript
// 主要功能：
// - 图谱节点和边的可视化
// - 交互式图谱操作
// - 布局算法选择
// - 节点搜索和过滤

// 关键技术：
// - 基于 @react-sigma/core
// - 支持多种布局算法
// - 自定义节点和边样式
```

#### 检索测试界面 (RetrievalTesting.tsx)
```typescript
// 主要功能：
// - 聊天式查询界面
// - 多种查询模式
// - 流式响应支持
// - 历史记录管理

// 查询模式：
// - local: 本地上下文查询
// - global: 全局知识查询
// - hybrid: 混合查询
// - naive: 简单查询
// - mix: 组合查询
```

### 3. 样式和主题系统

#### Tailwind CSS 配置
```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // 自定义颜色变量
        primary: 'hsl(var(--primary))',
        secondary: 'hsl(var(--secondary))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
      },
      animation: {
        // 自定义动画
        'fade-in': 'fadeIn 0.5s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
      }
    }
  }
}
```

#### 主题切换实现
```typescript
// components/ThemeProvider.tsx
const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  useEffect(() => {
    document.documentElement.classList.toggle('dark', theme === 'dark');
  }, [theme]);
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};
```

### 4. 状态管理 (Zustand)

#### 全局状态结构
```typescript
// stores/state.ts
interface BackendState {
  status: 'loading' | 'success' | 'error';
  message: string;
  version: string;
  check: () => Promise<void>;
  clear: () => void;
}

interface AuthState {
  isAuthenticated: boolean;
  token: string | null;
  user: User | null;
  login: (token: string) => void;
  logout: () => void;
}

interface SettingsState {
  currentTab: string;
  theme: 'light' | 'dark';
  language: 'en' | 'zh';
  enableHealthCheck: boolean;
}
```

### 5. API 接口设计

#### 后端 API 调用
```typescript
// api/lightrag.ts
export const queryText = async (params: QueryParams): Promise<string> => {
  const response = await fetch('/query', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${getToken()}`,
    },
    body: JSON.stringify(params),
  });
  
  if (!response.ok) {
    throw new Error(`Query failed: ${response.status}`);
  }
  
  return await response.text();
};

export const uploadDocument = async (file: File): Promise<void> => {
  const formData = new FormData();
  formData.append('file', file);
  
  const response = await fetch('/documents/file', {
    method: 'POST',
    body: formData,
  });
  
  if (!response.ok) {
    throw new Error(`Upload failed: ${response.status}`);
  }
};
```

### 6. 自定义组件开发

#### 创建新的功能模块
```typescript
// features/MyNewFeature.tsx
import { useState, useEffect } from 'react';
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/Card';
import Button from '@/components/ui/Button';

const MyNewFeature = () => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  
  const handleAction = async () => {
    setLoading(true);
    try {
      // 调用 API
      const result = await myCustomAPI();
      setData(result);
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="p-6">
      <Card>
        <CardHeader>
          <CardTitle>我的新功能</CardTitle>
        </CardHeader>
        <CardContent>
          <Button 
            onClick={handleAction}
            disabled={loading}
            className="w-full"
          >
            {loading ? '处理中...' : '执行操作'}
          </Button>
        </CardContent>
      </Card>
    </div>
  );
};

export default MyNewFeature;
```

#### 添加到主应用
```typescript
// App.tsx
import MyNewFeature from '@/features/MyNewFeature';

// 在 Tabs 中添加新的 TabsContent
<TabsContent value="my-feature" className="absolute top-0 right-0 bottom-0 left-0 overflow-auto">
  <MyNewFeature />
</TabsContent>
```

### 7. 国际化支持

#### 添加新的翻译
```json
// locales/zh.json
{
  "common": {
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "edit": "编辑"
  },
  "documents": {
    "title": "文档管理",
    "upload": "上传文档",
    "scan": "扫描文档"
  },
  "graph": {
    "title": "知识图谱",
    "search": "搜索节点"
  }
}
```

#### 使用翻译
```typescript
import { useTranslation } from 'react-i18next';

const MyComponent = () => {
  const { t } = useTranslation();
  
  return (
    <div>
      <h1>{t('documents.title')}</h1>
      <button>{t('common.save')}</button>
    </div>
  );
};
```

### 8. 性能优化技巧

#### 懒加载组件
```typescript
import { lazy, Suspense } from 'react';

const GraphViewer = lazy(() => import('@/features/GraphViewer'));

const App = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <GraphViewer />
    </Suspense>
  );
};
```

#### 虚拟滚动优化
```typescript
import { FixedSizeList as List } from 'react-window';

const VirtualizedList = ({ items }) => {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index]}
    </div>
  );
  
  return (
    <List
      height={400}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </List>
  );
};
```

### 9. 常见自定义需求

#### 1. 修改主题颜色
```css
/* index.css */
:root {
  --primary: 210 40% 50%;        /* 修改主色调 */
  --secondary: 210 40% 90%;      /* 修改次色调 */
  --background: 0 0% 100%;       /* 修改背景色 */
  --foreground: 222.2 84% 4.9%;  /* 修改前景色 */
}

.dark {
  --primary: 210 40% 50%;
  --secondary: 210 40% 10%;
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
}
```

#### 2. 添加新的查询模式
```typescript
// 在 RetrievalTesting.tsx 中添加新模式
const queryModes = [
  'local', 'global', 'hybrid', 'naive', 'mix', 'custom'
];

const handleCustomQuery = async (query: string) => {
  // 实现自定义查询逻辑
  const result = await fetch('/query/custom', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, mode: 'custom' })
  });
  return result.text();
};
```

#### 3. 自定义文档上传界面
```typescript
// components/documents/CustomUploadDialog.tsx
const CustomUploadDialog = ({ onUpload }) => {
  const [files, setFiles] = useState<File[]>([]);
  const [metadata, setMetadata] = useState({});
  
  const handleUpload = async () => {
    for (const file of files) {
      await uploadWithMetadata(file, metadata);
    }
    onUpload();
  };
  
  return (
    <Dialog>
      <DialogContent>
        <div className="space-y-4">
          <FileDropzone onFiles={setFiles} />
          <MetadataForm onChange={setMetadata} />
          <Button onClick={handleUpload}>上传文档</Button>
        </div>
      </DialogContent>
    </Dialog>
  );
};
```

### 10. 开发和调试技巧

#### 开发环境配置
```bash
# 启动开发服务器
npm run dev

# 类型检查
npm run type-check

# 代码格式化
npm run format

# 构建生产版本
npm run build
```

#### 调试工具
- **React Developer Tools**: 组件状态调试
- **Redux DevTools**: 状态管理调试（如果使用）
- **Chrome DevTools**: 网络请求和性能分析

这个指南为您提供了完整的前端界面修改方案，您可以根据具体需求进行相应的定制和扩展。