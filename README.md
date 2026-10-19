# Loom

对话即生成单页应用的智能体平台：对话是线，应用是布。支持需求澄清与工单拆解、带改动范围核验的增量迭代、版本回滚、预览、发布稳定链接与应用世界画廊克隆。

- `backend/` — FastAPI + 智能体（工具调用循环）+ RAG（Milvus Lite + MiniMax/OpenAI 兼容 embedding）
- `frontend/` — Vue 3 + Vite + Pinia 单页应用
- `docs/` — ADR、工单规格（`docs/tracker/`）与[部署文档](docs/deployment.md)

## 本地开发

```bash
# 后端（先复制 backend/.env 模板并填写 API Key）
cd backend
uv pip install -r requirements.txt   # 或 pip install -r requirements.txt
uvicorn app.main:create_app --factory --host 0.0.0.0 --port 8000

# 前端
cd frontend
npm install
npm run dev        # http://localhost:5173，/api 与 /p/ 代理到后端
```

测试：`cd backend && pytest`；类型检查：`cd frontend && npm run typecheck`。

## 部署（工单 0013 / 0014）

两条路径见 [docs/deployment.md](docs/deployment.md)：

```bash
# 免 Docker 裸机（演示推荐）：代码到机器后跑一键脚本（自动装依赖/建服务/建站点）
bash deploy/aliyun/deploy-bare.sh

# Docker Compose：nginx 直出 /p/ 静态 + 反代 /api，数据落 atoms-data 卷
cp .env.example .env   # 填写 ATOMS_JWT_SECRET 与 ATOMS_LLM_API_KEY
docker compose up -d --build
```

线上验收冒烟（两条路径通用）：`python deploy/smoke/smoke_online.py http://<地址>`。
