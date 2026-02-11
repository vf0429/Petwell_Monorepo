# 🐾 Petwell Monorepo 开发指南 (Antigravity版)

## 📡 架构与端口定义 (Architecture & Ports)

本项目包含三个核心服务，各服务端口定义严格如下，**请勿随意更改，否则会导致连接失败**。

| 服务 (Service) | 运行环境 (Runtime) | 端口 (Port) | 说明 (Description) |
| --- | --- | --- | --- |
| **Frontend** | **Sweetpad / Antigravity** | `(动态/3000)` | 用户界面，由 Sweetpad 管理启动 |
| **Backend** | **Go (Gin)** | **`8000`** | 核心 API 网关，处理业务逻辑 |
| **RAG Engine** | **Python (FastAPI)** | **`8001`** | AI 检索引擎，仅供 Backend 调用 |

---

## ⚙️ 关键配置 (Environment Variables)

为了让服务跑通，请确保你的本地配置文件 (`.env`) 内容如下：

### 1. Frontend (Antigravity/Sweetpad)

在前端项目的环境变量设置中（或 `.env.local`）：

```bash
# ⚠️ 注意：后端现在运行在 8000 端口
NEXT_PUBLIC_API_URL=http://localhost:8000

```

### 2. Backend (Go) - `backend/.env`

后端服务监听 8000，并需要知道 RAG 在 8001。

```bash
# 本服务端口
PORT=8000

# 下游 RAG 服务地址
# 如果是 Docker 内部通信: http://rag:8001
# 如果是 本地混合开发: http://localhost:8001
RAG_SERVICE_URL=http://localhost:8001

```

### 3. RAG Service (Python) - `rag/.env`

AI 服务必须监听 8001。

```bash
# ⚠️ 注意：这里必须是 8001，不要用默认的 8000
PORT=8001

OPENAI_API_KEY=sk-xxxxxx
CHROMA_DB_PATH=./data/chroma

```

---

## 🚀 启动流程 (Startup Workflow)

由于前端由 Sweetpad 管理，推荐的开发启动方式如下：

### 方式 A: 混合开发 (推荐)

前端用工具跑，后端和 AI 用 Docker 跑。

1. **启动后端和 AI**:
在 Monorepo 根目录运行：
```bash
# 注意：docker-compose.yml 需映射 8000 和 8001 端口
docker-compose up backend rag

```


2. **启动前端**:
在 **Sweetpad** 中点击 "Run" (Antigravity Frontend)。

### 方式 B: 纯本地开发 (全手动)

如果你不使用 Docker，手动启动命令如下：

1. **Terminal 1 (RAG)**:
```bash
cd rag
# 强制指定端口 8001
uvicorn main:app --reload --port 8001

```


2. **Terminal 2 (Backend)**:
```bash
cd backend
# 确保 main.go 里读取了 PORT=8000
go run cmd/server/main.go

```


3. **Terminal 3 / Sweetpad (Frontend)**:
启动你的前端环境。

---

## 🐳 Docker Compose 配置更新

*(请确保根目录的 `docker-compose.yml` 已同步更新端口)*

```yaml
version: '3.8'
services:
  rag:
    build: ./rag
    ports:
      - "8001:8001"  # <--- 暴露 8001
    environment:
      - PORT=8001

  backend:
    build: ./backend
    ports:
      - "8000:8000"  # <--- 暴露 8000
    environment:
      - RAG_SERVICE_URL=http://rag:8001 # 指向容器内的 8001
    depends_on:
      - rag

```

---
## 🤝 提交规范 (Git Flow)

由于是 Monorepo，提交代码时请遵循 **原子提交 (Atomic Commits)** 原则。

* **❌ 错误做法**: 修改了前端和后端，用一个 commit `update code` 提交。
* **✅ 正确做法**:
1. `git add backend/` -> `git commit -m "feat(backend): add user validation logic"`
2. `git add frontend/` -> `git commit -m "style(frontend): update chat bubble color"`



**Commit Message 前缀建议**:

* `feat(scope)`: 新功能 (scope 可以是 frontend, backend, rag)
* `fix(scope)`: 修 Bug
* `docs`: 仅文档修改
* `chore`: 配置变动 (如 docker-compose, .gitignore)

---

## 🚨 常见排错 (Troubleshooting)

* **Q: 前端请求报错 404 或 Connection Refused?**
* **检查**: 你的浏览器 Network 面板，请求的地址是不是 `http://localhost:8000/api/chat`？
* **错误原因**: 以前是 8080，可能你的 `.env.local` 没更新，或者浏览器缓存了旧地址。


* **Q: 后端报错 "RAG connection failed"?**
* **检查**: 后端日志显示它在连哪里？
* **修正**: 确保后端连的是 `8001`。如果以前是 8000，现在后端自己占用了 8000，**绝对不能**让后端连自己（会死循环或报错）。


* **Q: 端口冲突 (Port already in use)?**
* **检查**: 8000 端口非常热门。如果你电脑上跑了其他 Python 服务或 Django，可能会占用 8000。
* **解决**: `lsof -i :8000` 查一下谁在用。




