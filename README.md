# LLM评测系统 (LLM Evaluation System)

企业级AI测试与评测系统MVP版本，支持LLM评测集管理、自动化评测执行、质量门禁等功能。

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                        用户界面层                            │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │   Web UI     │  │   CLI工具    │                         │
│  │  (React)     │  │  (Python)    │                         │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                      API服务层 (FastAPI)                    │
│  - 评测集管理  - 评分规则管理  - 评测任务调度                  │
│  - 报告生成    - 质量门禁                                    │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                    Celery Worker                            │
│  - 批量评测执行  - DeepEval集成  - 结果写入数据库              │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                      数据层                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  PostgreSQL  │  │    Redis     │  │    MinIO     │       │
│  │  (主数据库)   │  │ (Celery队列) │  │ (报告存储)    │       │ 
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

## 功能特性

### 核心功能
- **评测集管理**: 测试用例CRUD、JSON导入导出、批量导入
- **评分规则**: 集成DeepEval指标（Answer Relevancy、Faithfulness、Contextual Relevancy、GEval）
- **评测执行**: 支持同步/异步执行模式，Celery任务队列，任务历史复用
- **报告生成**: 评测结果展示、详细指标分析、优化建议、通过率统计
- **质量门禁**: CI/CD集成、阈值配置、自动判定
- **多LLM支持**: 支持OpenAI、DeepSeek等多种LLM提供商进行评分
- **目标系统适配**: 支持AnythingLLM等OpenAI兼容API的待评测系统

### 技术栈
- **后端**: FastAPI + SQLAlchemy + Celery
- **前端**: React + Ant Design
- **数据库**: PostgreSQL + Redis
- **评测核心**: DeepEval
- **部署**: Docker Compose

## 快速开始

### 环境要求
- Docker & Docker Compose
- OpenAI API Key 或 DeepSeek API Key (用于评测评分)

### 启动服务

```bash
# 1. 克隆项目
cd LLM_evaluation_system

# 2. 配置环境变量
cp backend/.env.example backend/.env
# 编辑 backend/.env，设置以下关键配置：
# - OPENAI_API_KEY: 用于评测评分（OpenAI或DeepSeek）
# - LLM_PROVIDER_TYPE: 评分提供商类型（openai/deepseek）
# - LLM_PROVIDER_MODEL: 评分模型名称（如 deepseek-chat）
# - TARGET_URL: 待评测系统API地址（如 AnythingLLM）
# - TARGET_API_KEY: 待评测系统API密钥

# 3. 启动服务
docker-compose up -d

# 4. 查看服务状态
docker-compose ps
```

服务启动后访问:
- Web界面: http://localhost:3000
- API文档: http://localhost:8000/docs
- Flower监控: http://localhost:5555

### 使用CLI工具

```bash
# 安装CLI
cd cli
pip install -e .

# 配置服务器地址
llm-eval config set-server http://localhost:8000

# 查看帮助
llm-eval --help
```

## 使用指南

### 1. 创建评测集

**通过Web界面:**
1. 访问 http://localhost:3000
2. 点击"评测集" → "新建评测集"
3. 填写名称、描述，添加测试用例

**通过CLI:**
```bash
# 创建评测集
llm-eval dataset create "客服场景测试" --description "测试客服Agent问答能力"

# 导入测试用例
llm-eval dataset import <dataset-id> test_cases.json
```

### 2. 配置评分规则

**预定义指标:**
- Answer Relevancy: 回答相关性
- Faithfulness: 忠实度
- Contextual Relevancy: 上下文相关性

**自定义GEval:**
```bash
llm-eval rule create "回答质量" --type geval \
  --config '{"criteria": "评估回答是否准确且有帮助", "evaluation_steps": ["检查准确性", "检查完整性"]}' \
  --threshold 0.8
```

### 3. 执行评测

**Web界面:**
1. 点击"执行评测"
2. 选择评测集和评分规则
3. 配置目标Agent API地址
4. 提交任务并查看结果

**CLI:**
```bash
llm-eval evaluate run <dataset-id> \
  --target http://your-agent-api.com/chat \
  --rules "rule-id-1,rule-id-2" \
  --wait
```

### 4. 质量门禁 (CI/CD集成)

```bash
# 在CI/CD流水线中执行
llm-eval gate webhook <gate-id> \
  --target http://your-agent-api.com/chat \
  --rules "rule-id-1,rule-id-2"

# 失败时返回exit code 1，阻止部署
```

## API接口

### 核心接口

| 接口 | 方法 | 说明 |
|-----|------|------|
| `/api/v1/datasets` | GET/POST | 评测集列表/创建 |
| `/api/v1/datasets/{id}` | GET/PUT/DELETE | 评测集详情/更新/删除 |
| `/api/v1/datasets/{id}/cases` | GET/POST | 测试用例管理 |
| `/api/v1/rules` | GET/POST | 评分规则管理 |
| `/api/v1/evaluate/tasks` | GET/POST | 评测任务管理 |
| `/api/v1/evaluate/quick` | POST | 快速评测(同步) |
| `/api/v1/gates` | GET/POST | 质量门禁管理 |
| `/api/v1/gates/{id}/check` | POST | 执行门禁检查 |

完整API文档: http://localhost:8000/docs

## 项目结构

```
LLM_evaluation_system/
├── backend/                    # 后端服务
│   ├── app/
│   │   ├── api/               # API路由
│   │   ├── core/              # 核心配置(Celery, DeepEval)
│   │   ├── models/            # 数据模型
│   │   ├── services/          # 业务逻辑
│   │   ├── tasks/             # Celery任务
│   │   └── main.py            # FastAPI入口
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/                   # 前端应用
│   ├── src/
│   │   ├── pages/             # 页面组件
│   │   ├── services/          # API服务
│   │   └── App.js             # 应用入口
│   └── Dockerfile
├── cli/                        # CLI工具
│   ├── llm_eval/
│   │   ├── main.py            # CLI入口
│   │   ├── api.py             # API客户端
│   │   └── config.py          # 配置管理
│   └── setup.py
├── docker-compose.yml
└── README.md
```

## 开发计划

### MVP (已完成)
- [x] 评测集管理（CRUD、JSON导入导出）
- [x] 测试用例管理（批量导入、单条编辑）
- [x] 评分规则管理（DeepEval指标、GEval自定义规则）
- [x] 评测执行（同步/异步、Celery任务队列）
- [x] 任务历史与重跑功能
- [x] 详细评测报告（得分、评估说明、优化建议）
- [x] 质量门禁（阈值配置、自动判定）
- [x] Web界面（React + Ant Design）
- [x] 多LLM提供商支持（OpenAI、DeepSeek）
- [x] 目标系统适配（AnythingLLM等OpenAI兼容API）
- [x] CLI工具

### v1.1 (计划)
- [ ] 评测集版本对比
- [ ] 报告导出PDF/Excel
- [ ] 评测集模板
- [ ] 批量重跑失败用例

### v1.2 (计划)
- [ ] 基础自动化测试(PyTest)
- [ ] 简单漂移监控
- [ ] 评测结果趋势分析

### v2.0 (计划)
- [ ] 场景模拟器
- [ ] 可观测性平台
- [ ] 安全测试模块
- [ ] 多租户支持

## 配置说明

### 环境变量

| 变量 | 说明 | 默认值 |
|-----|------|-------|
| `DATABASE_URL` | PostgreSQL连接URL | postgresql+asyncpg://... |
| `REDIS_URL` | Redis连接URL | redis://localhost:6379/0 |
| `OPENAI_API_KEY` | OpenAI/DeepSeek API密钥 | - |
| `OPENAI_BASE_URL` | API基础URL | https://api.openai.com/v1 |
| `LLM_PROVIDER_TYPE` | 评分提供商类型 | openai |
| `LLM_PROVIDER_MODEL` | 评分模型名称 | gpt-3.5-turbo |
| `TARGET_URL` | 待评测系统API地址 | - |
| `TARGET_API_KEY` | 待评测系统API密钥 | - |
| `CELERY_WORKER_CONCURRENCY` | Celery并发数 | 4 |
| `EVAL_TIMEOUT` | 评测超时时间(秒) | 30 |

## 贡献指南

欢迎提交Issue和Pull Request。

## License

MIT License
