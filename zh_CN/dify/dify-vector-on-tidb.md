# 私有化部署 Dify + TiDB Cloud Serverless
## 概述
Dify 是一个开源的大语言模型（LLM）应用开发平台，旨在帮助开发者、企业甚至非技术人员快速构建、部署和管理基于 AI 的应用。支持 RAG（检索增强生成），结合外部知识库提升回答准确性。在 Pingcap 官方 blog 中，之前 [Dify.AI x TiDB](https://www.pingcap.com/blog/dify-tidb-build-scalable-ai-agent-with-knowledge-base/) 的这篇文章中有介绍如何在本地部署 Dify + TiDB Serverless 构建 Dify 的知识库。本篇稍有不同，将介绍如何配置基于 qdrant 协议，同时支持 vector + **FTS(Fulltext search)** 的知识库构建。

## 前置准备
_以下为 Dify 官方的推荐环境。_

**硬件环境：**
- CPU >= 2 Core
- 显存/RAM ≥ 16 GiB（推荐）

**软件环境：**
- [Docker](https://www.docker.com/)
- Docker Compose
- [Dify 社区版](https://github.com/langgenius/dify)

## TiDB 准备工作
[TiDB Vector Search (beta)](https://docs.pingcap.com/tidbcloud/vector-search-overview/) 提供了一种高级搜索解决方案，用于对各种数据类型（包括文档、图像、音频和视频）执行语义相似性搜索。此功能使开发人员能够使用熟悉的 MySQL 技能轻松构建具有生成人工智能 (AI) 功能的可扩展应用程序。[TiDB Cloud Serverless](https://docs.pingcap.com/tidbcloud/select-cluster-tier/#tidb-cloud-serverless) 同时还提供了 Qdrant 大部分常用功能的兼容层，方便用户快速验证和迁移基于 Qdrant 构建的应用。

### 创建 TiDB Cloud Serverless 集群
1. 登录 [TiDB Cloud](https://tidbcloud.com/) 控制台；
2. 进入 Cluster 页面，点击 [Create Cluster](https://tidbcloud.com/console/clusters/create-cluster)；
    - 选择 "Serverless" 类型 Tier
    - 选择 "us-east-1" Region (目前 qdrant 兼容层只提供 "us-east-1" 区域的实例)
    - 选择 "Free Cluster" Plan (Free 和 Scalable 两种 Plan 提供相同的功能，可按需选择)
3. 点击 "Create" 按钮，等待 Cluster 创建完成

![](imgs/create-tidb-cloud-serverless-cluster.png)

### 获取 Cluster 信息
Cluster 创建完成后，进入刚刚创建完成的 Cluster 详情页。
1. 点击右上角 "Connect"
2. 在弹出的页面中点击 "Generate Password"
3. 保存对应的 USERNAME 和 PASSWORD

<img src="imgs/connect-tidb.png" width="50%" />

Qdrant 兼容层的接入方式和 TiDB Cloud Serverless 本身的接入方式有一些差别，除了需要记住 USERNAME 和 PASSWORD 之外，这里需要记住 qdrant 接入点。
- qdrant-gateway01.us-east-1.prod.aws.tidbcloud.com

## 部署 Dify
访问 Dify GitHub 项目地址，运行以下命令完成拉取代码仓库和安装流程。

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
vi .env
```

修改 VECTOR_STORE 为 qdrant。

```bash
# 请注意，由于 serverless 提供了 qdrant 的兼容层，这里直接配置成 "qdrant" 即可
VECTOR_STORE=qdrant

# The Qdrant endpoint URL. Only available when VECTOR_STORE is `qdrant`.
QDRANT_URL=qdrant-gateway01.us-east-1.prod.aws.tidbcloud.com
# TiDB Serverless 中的 USERNAME 和 PASSWORD 映射成 QDRANT_API_KEY 的格式为 USERNAME:PASSWORD
QDRANT_API_KEY=USERNAME:PASSWORD
QDRANT_CLIENT_TIMEOUT=20
QDRANT_GRPC_ENABLED=false
QDRANT_GRPC_PORT=443
```

由于 Dify 默认使用的 VECTOR_STORE 是 weaviate, 并且在 docker compose 中启动了一个本地 weaviate 容器，这里可以省去该容器。_这里也可以什么也不做，冗余的运行一个 weaviate 容器。_

```bash
vi docker-compse.yaml # 注释掉 weaviate: 及相关的配置

docker compose up -d
```

等待运行命令后，你应该会看到所有容器的状态和端口映射。
- 默认使用的 80 端口，访问登录地址 [http://localhost/install]
- 设置管理员用户名和密码，进入系统

详细说明请参考 [Docker Compose 部署](https://docs.dify.ai/zh-hans/getting-started/install-self-hosted/docker-compose)，也可以参考 [源码部署](https://docs.dify.ai/getting-started/install-self-hosted/local-source-code) 方式。

## 开始使用 Dify

和其它常见 ai agent 不同，Dify 支持多模型 + 知识库工作流编排设计。Dify 安装完毕后，需要先准备模型和知识库。

### 安装模型

点击 Dify 平台右上角**头像 → 设置 → 模型供应商**，选择模型供应商，轻点对应模型的"安装"，并且设置相关模型的 API-KEY。

<img src="imgs/models.png" width="50%" />

_如果你不是很了解各类模型怎么使用，推荐安装 **Cohere** 一个模型即可，该模型包含了 Dify 的所需的所有功能。Dify 官方也提供了纯本地化 [Ollama + Deepseek + Dify](https://github.com/langgenius/dify-docs/blob/main/zh_CN/learn-more/use-cases/private-ai-ollama-deepseek-dify.md) 部署方案。_

### 准备基于 TiDB Cloud Serverless 的知识库
以上我们已经配置好 TiDB Serverless 的 qdrant endpoint，这里我们直接使用 Dify 的"创建知识库"功能。

1. 点击[知识库](http://localhost/datasets)菜单
2. 点击[创建知识库](http://localhost/datasets/create)按钮
3. 上传文件, 下一步
4. 设置索引参数
    - 通用里的参数可以选择默认
    - 索引方式选择：**高质量**
    - 检索设置选择：**混合检索**

<img src="imgs/datasets-step2.png" width="100%" />

索引参数说明：
- **向量索引**：纯向量的召回策略
- **全文检索**：基于倒排索引的全文检索
- **混合索引**：基于向量+倒排索引的两路召回 Rerank

_注意，如果你使用的是模型的免费 API-KEY，可能面临 call limited 问题，构建知识库的时候可以使用小文档试试_

**知识库完成后是这样子的。**

![](imgs/datasets-done.png)

### 创建 AI Chatflow
Dify 支持各种复杂的流程，这里只介绍较为简单的 chatflow 流程。

1. 点击 Dify 平台首页左侧的"创建空白应用"，选择"Chatflow"应用并进行简单的命名
2. 右键添加"添加节点"，选择"知识检索"

<img src="imgs/chatflow-km.png" width="30%" />

3. 点击"+"，添加"知识库"

<img src="imgs/chatflow-km-model.png" width="50%" />

4. 设置"查询变量" `{{#sys.query#}}`

<img src="imgs/chatflow-km-query.png" width="30%" />
    
5. 点击"召回设置"，设置模型参数"模型

<img src="imgs/chatflow-km-rerank-model.png" width="30%" />

6. 选择"LLM"节点
    - 选择模型
    - 设置"上下文"为`知识检索 - result`
    - 添加系统提示词"SYSTEM"，`{{#sys.query#}}` + `上下文`

<img src="imgs/chatflow-km-llm.png" width="30%" />

7. 链接所有节点

<img src="imgs/chatflow-join.png" width="60%" />

_过程中，可以使用"预览"进行调试。_

8. 发布

### 试试刚刚创建 AI Chatflow
点击顶部菜单中的"探索"，选择刚刚创建的 chatflow，就可以开始使用了。

<img src="imgs/chat.png" width="60%" />

## Why Choose TiDB Cloud Serverless

## Why Choose Dify?

## Learn More
