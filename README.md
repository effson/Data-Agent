## Data Agent
基于 LangGraph + Claude 的自然语言转 SQL（NL2SQL）智能代理系统。用户通过中文自然语言描述业务问题，系统自动完成关键词提取、Schema 检索、SQL 生成与校验、查询执行的全流程，并通过 SSE 流式返回结果。
## 架构概览

```
用户中文提问 → 分词提取关键词 → 向量/全文检索召回相关元数据
    → LLM 过滤候选表/列/指标 → LLM 生成 SQL → MySQL EXPLAIN 校验
    → 失败则自动修复 → 执行查询 → SSE 流式返回结果
```

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.12+ |
| Web 框架 | FastAPI + uvicorn |
| Agent 框架 | LangGraph |
| 大模型 | Anthropic Claude（通过 langchain-anthropic） |
| 向量数据库 | Qdrant |
| 全文检索 | Elasticsearch 8.x（集成 IK 中文分词插件） |
| 元数据库 | MySQL 8.0（SQLAlchemy 2.0 异步驱动） |
| 数据仓库 | MySQL 8.0 |
| 嵌入模型 | BAAI/bge-large-zh-v1.5（通过 HuggingFace TEI） |
| 中文分词 | jieba |
| 配置管理 | OmegaConf |
| 日志 | loguru |
| 可观测性 | LangSmith（追踪、评估、反馈） |
| 容器化 | Docker Compose |
| 包管理 | uv |

## 项目结构

```
data-agent/
├── main.py                          # FastAPI 入口，中间件，uvicorn 启动
├── pyproject.toml                   # 项目元信息与依赖
├── uv.lock                          # 锁定依赖版本
├── .env                             # LangSmith 环境变量
│
├── conf/
│   ├── app_config.yaml              # 主配置：数据库、Qdrant、ES、Embedding、LLM
│   └── meta_config.yaml             # 元数据 Schema 定义（表、列、指标）
│
├── prompts/                         # LLM 提示词模板
│   ├── extend_keywords_for_column_recall.prompt
│   ├── extend_keywords_for_metric_recall.prompt
│   ├── extend_keywords_for_value_recall.prompt
│   ├── filter_metric_info.prompt
│   ├── filter_table_info.prompt
│   ├── generate_sql.prompt
│   └── regulate_sql.prompt
│
├── app/
│   ├── conf/                        # 配置数据类
│   ├── core/                        # 生命周期、请求上下文、日志
│   ├── entities/                    # 领域实体（TableInfo, ColumnInfo, MetricInfo 等）
│   ├── models/                      # SQLAlchemy ORM 模型
│   ├── clients/                     # 外部服务客户端管理器（单例）
│   ├── repositories/                # 数据访问层
│   │   ├── mysql/meta/              # 元数据 MySQL CRUD
│   │   ├── mysql/dw/                # 数据仓库查询/校验/执行
│   │   ├── qdrant/                  # Qdrant 向量检索
│   │   └── es/                      # Elasticsearch 全文检索
│   ├── agent/                       # LangGraph 代理
│   │   ├── graph.py                 # 状态图定义与编译
│   │   ├── state.py                 # 状态类型定义
│   │   └── nodes/                   # 9 个图节点实现
│   ├── api/                         # FastAPI 路由与请求 Schema
│   ├── services/                    # 业务编排层
│   ├── scripts/                     # CLI 脚本（构建元数据仓库）
│   └── prompt/                      # 提示词文件加载器
│
├── docker/
│   ├── docker-compose.yaml          # 基础设施服务编排
│   ├── mysql/                       # meta.sql + dw.sql（含示例零售数据）
│   ├── elasticsearch/               # ES + IK 中文分词插件
│   └── embedding/                   # bge-large-zh-v1.5 模型文件
│
└── logs/                            # 日志输出目录
```

## Agent 流水线

整个 NL2SQL 流程通过 LangGraph StateGraph 编排为 9 个节点：

| 步骤 | 节点 | 说明 |
|------|------|------|
| 1 | `extract_keywords` | 使用 jieba 从用户问题中提取关键词（名词、动词、专有名词等） |
| 2a | `recall_column` | LLM 扩展同义词 → Qdrant 向量检索相关列 |
| 2b | `recall_metric` | LLM 扩展同义词 → Qdrant 向量检索相关指标 |
| 2c | `recall_value` | LLM 扩展同义词 → Qdrant 示例值向量检索 + ES 全文检索列值 |
| 3 | `merge_retrieved` | 合并所有检索结果，补全主键/外键关系，按表分组 |
| 4a | `filter_metric` | LLM 过滤仅与问题相关的指标 |
| 4b | `filter_table` | LLM 过滤仅与问题相关的表和列 |
| 5 | `add_extra_context` | 注入当前日期信息和数据库方言信息 |
| 6 | `generate_sql` | LLM 基于精简后的 Schema 生成 SQL |
| 7 | `validate_sql` | 执行 `EXPLAIN` 校验 SQL 语法 |
| 8 | `regulate_sql` | （条件执行）校验失败时由 LLM 修复 SQL |
| 9 | `execute_sql` | 执行 SQL 并返回查询结果 |

其中步骤 2a/2b/2c 并行执行，步骤 4a/4b 并行执行，步骤 8 仅在步骤 7 失败时触发。

<img width="500" height="600" alt="Mermaid" src="https://github.com/user-attachments/assets/df4438a6-4cda-4202-b032-5332dd9101f9" />

## 快速开始

### 1. 环境要求

- Python 3.12+
- Docker & Docker Compose
- [uv](https://docs.astral.sh/uv/)（推荐）或 pip

### 2. 启动基础设施

```bash
cd docker
docker-compose up -d
```
启动以下服务：

| 服务 | 端口 | 说明 |
|------|------|------|
| MySQL 8.0 | 3306 | 元数据库（`meta`）+ 数据仓库（`dw`），含示例零售数据 |
| Elasticsearch | 9200 | 全文检索，集成 IK 中文分词 |
| Qdrant | 6333 | 向量数据库 |
| TEI Server | 8088 | HuggingFace 文本嵌入推理服务 |
| Kibana | 5601 | ES 可视化管理（可选） |

### 3. 构建元数据仓库

将 `conf/meta_config.yaml` 中定义的表、列、指标信息写入 MySQL，并为其生成向量嵌入存入 Qdrant 和 Elasticsearch：

```bash
python -m app.scripts.build_meta_repository -c conf/meta_config.yaml
```

### 4.项目运行
#### query 1

<img width="686" height="531" alt="image" src="https://github.com/user-attachments/assets/ebf6512a-ef54-4852-a4c3-8a5ebebafcfb" />

#### query 2

<img width="686" height="531" alt="image" src="https://github.com/user-attachments/assets/054a7688-1eaf-479e-8f39-6a26abcef78f" />

#### query 3

<img width="500" height="531" alt="image" src="https://github.com/user-attachments/assets/62becf2e-a6ed-4467-be1c-7b0eefd186d8" />

#### query 4

<img width="656" height="450" alt="image" src="https://github.com/user-attachments/assets/cc2888d7-6cb4-4752-8cd3-671ec38ab8ac" />

### 5. LangSmith

#### 在.env中添加：
```
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=<your_langsmith_key>
LANGSMITH_PROJECT="Data Agent"
```

<img width="1095" height="325" alt="image" src="https://github.com/user-attachments/assets/2b3f9f25-032e-4fb2-9cfc-5c9c8747fec9" />

<img width="1269" height="562" alt="image" src="https://github.com/user-attachments/assets/31fc4814-a964-47bf-8b6d-198810b90aed" />

### 6 用户反馈功能

<img width="736" height="274" alt="image" src="https://github.com/user-attachments/assets/115c6977-3d87-486c-8e6f-e924218b67da" />

<img width="1142" height="187" alt="image" src="https://github.com/user-attachments/assets/c92a24e6-e179-4dc4-9e96-0ececba8e94b" />
