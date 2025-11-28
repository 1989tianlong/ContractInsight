# 数据/AI基座详细设计

## 1. 概述

数据/AI基座是统一合同平台的智能底座，为上层业务提供数据存储、处理、分析和AI能力支撑。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           业务服务层                                      │
│         (合同提取 / 合同存储 / 合同查询 / Sidecar服务)                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         数据/AI基座                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  数据湖仓   │  │  向量数据库  │  │  LLM服务    │  │  模型平台   │     │
│  │  (Lakehouse)│  │  (Vector DB)│  │  (LLM Svc)  │  │  (ML Platform)│   │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  特征工程   │  │  数据治理   │  │  MLOps      │  │  实时计算   │     │
│  │  (Feature)  │  │  (Govern)   │  │  (MLOps)    │  │  (Streaming)│     │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 数据湖仓架构 (Lakehouse)

### 2.1 分层设计

```
┌────────────────────────────────────────────────────────────┐
│                      应用层 (Application)                   │
│              BI报表 / 数据API / 自助分析                     │
└────────────────────────────────────────────────────────────┘
                              │
┌────────────────────────────────────────────────────────────┐
│                      服务层 (Serving)                       │
│         ADS (应用数据层) - 面向主题的宽表/指标                │
└────────────────────────────────────────────────────────────┘
                              │
┌────────────────────────────────────────────────────────────┐
│                      公共层 (Common)                        │
│    DWS (汇总层)          │        DWD (明细层)              │
│    聚合指标/轻度汇总      │        清洗后的明细数据           │
└────────────────────────────────────────────────────────────┘
                              │
┌────────────────────────────────────────────────────────────┐
│                      贴源层 (Source)                        │
│                  ODS (原始数据层)                           │
│              原始合同数据 / 业务系统数据                      │
└────────────────────────────────────────────────────────────┘
                              │
┌────────────────────────────────────────────────────────────┐
│                      存储层 (Storage)                       │
│     MinIO/S3 (对象存储)  +  Apache Iceberg (表格式)          │
└────────────────────────────────────────────────────────────┘
```

### 2.2 核心表设计

```sql
-- ODS层：原始合同数据
CREATE TABLE ods.contract_raw (
    id              STRING,
    tenant_id       STRING,
    domain_code     STRING,
    file_path       STRING,
    ocr_result      STRING,       -- JSON格式OCR结果
    raw_metadata    STRING,       -- JSON格式原始元数据
    source_system   STRING,
    ingest_time     TIMESTAMP,
    dt              STRING        -- 分区字段
) USING iceberg
PARTITIONED BY (dt, tenant_id);

-- DWD层：合同明细表
CREATE TABLE dwd.contract_detail (
    contract_id         STRING,
    tenant_id           STRING,
    domain_code         STRING,
    contract_no         STRING,
    contract_name       STRING,
    contract_type       STRING,
    party_a             STRING,
    party_b             STRING,
    amount              DECIMAL(18,2),
    currency            STRING,
    sign_date           DATE,
    effective_date      DATE,
    expiry_date         DATE,
    status              STRING,
    risk_level          STRING,
    key_clauses         ARRAY<STRING>,
    extracted_entities  MAP<STRING, STRING>,
    create_time         TIMESTAMP,
    update_time         TIMESTAMP,
    dt                  STRING
) USING iceberg
PARTITIONED BY (dt, tenant_id, domain_code);

-- DWS层：合同统计汇总
CREATE TABLE dws.contract_statistics (
    stat_date           DATE,
    tenant_id           STRING,
    domain_code         STRING,
    contract_type       STRING,
    total_count         BIGINT,
    total_amount        DECIMAL(20,2),
    avg_amount          DECIMAL(18,2),
    risk_high_count     BIGINT,
    risk_medium_count   BIGINT,
    risk_low_count      BIGINT,
    new_count           BIGINT,
    expired_count       BIGINT
) USING iceberg
PARTITIONED BY (stat_date, tenant_id);

-- ADS层：合同风险看板
CREATE TABLE ads.contract_risk_dashboard (
    dashboard_date      DATE,
    tenant_id           STRING,
    total_contracts     BIGINT,
    high_risk_contracts BIGINT,
    high_risk_amount    DECIMAL(20,2),
    expiring_30days     BIGINT,
    expiring_90days     BIGINT,
    top_risk_types      STRING,      -- JSON数组
    trend_7days         STRING       -- JSON数组
) USING iceberg;
```

### 2.3 数据处理引擎

```python
# Spark批处理作业示例
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

class ContractETLJob:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("ContractETL") \
            .config("spark.sql.catalog.lakehouse", "org.apache.iceberg.spark.SparkCatalog") \
            .config("spark.sql.catalog.lakehouse.type", "hadoop") \
            .config("spark.sql.catalog.lakehouse.warehouse", "s3://lakehouse/warehouse") \
            .getOrCreate()
    
    def ods_to_dwd(self, process_date: str):
        """ODS到DWD的ETL处理"""
        ods_df = self.spark.read.table("lakehouse.ods.contract_raw") \
            .filter(f"dt = '{process_date}'")
        
        dwd_df = ods_df.select(
            col("id").alias("contract_id"),
            col("tenant_id"),
            col("domain_code"),
            get_json_object("raw_metadata", "$.contract_no").alias("contract_no"),
            get_json_object("raw_metadata", "$.contract_name").alias("contract_name"),
            get_json_object("raw_metadata", "$.contract_type").alias("contract_type"),
            get_json_object("raw_metadata", "$.party_a").alias("party_a"),
            get_json_object("raw_metadata", "$.party_b").alias("party_b"),
            get_json_object("raw_metadata", "$.amount").cast("decimal(18,2)").alias("amount"),
            get_json_object("raw_metadata", "$.sign_date").cast("date").alias("sign_date"),
            current_timestamp().alias("create_time"),
            lit(process_date).alias("dt")
        )
        
        dwd_df.writeTo("lakehouse.dwd.contract_detail") \
            .option("merge-schema", "true") \
            .append()
    
    def build_statistics(self, stat_date: str):
        """构建统计汇总表"""
        dwd_df = self.spark.read.table("lakehouse.dwd.contract_detail")
        
        stats_df = dwd_df.groupBy("tenant_id", "domain_code", "contract_type") \
            .agg(
                count("*").alias("total_count"),
                sum("amount").alias("total_amount"),
                avg("amount").alias("avg_amount"),
                sum(when(col("risk_level") == "HIGH", 1).otherwise(0)).alias("risk_high_count"),
                sum(when(col("risk_level") == "MEDIUM", 1).otherwise(0)).alias("risk_medium_count"),
                sum(when(col("risk_level") == "LOW", 1).otherwise(0)).alias("risk_low_count")
            ) \
            .withColumn("stat_date", lit(stat_date).cast("date"))
        
        stats_df.writeTo("lakehouse.dws.contract_statistics").overwritePartitions()
```

---

## 3. 向量数据库

### 3.1 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                    向量检索服务层                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ 语义搜索API │  │ 相似合同API │  │ 智能推荐API │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                    Milvus 集群                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Proxy Layer                        │    │
│  │         (负载均衡 / 请求路由 / 认证鉴权)              │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Coord Layer                        │    │
│  │    RootCoord │ DataCoord │ QueryCoord │ IndexCoord  │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Worker Layer                       │    │
│  │         DataNode │ QueryNode │ IndexNode            │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Storage Layer                      │    │
│  │              MinIO (S3) + etcd                       │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Collection设计

```python
from pymilvus import Collection, FieldSchema, CollectionSchema, DataType, utility

class VectorDBManager:
    """向量数据库管理"""
    
    # 合同向量集合Schema
    CONTRACT_SCHEMA = CollectionSchema(
        fields=[
            FieldSchema(name="id", dtype=DataType.VARCHAR, max_length=64, is_primary=True),
            FieldSchema(name="tenant_id", dtype=DataType.VARCHAR, max_length=32),
            FieldSchema(name="domain_code", dtype=DataType.VARCHAR, max_length=32),
            FieldSchema(name="contract_id", dtype=DataType.VARCHAR, max_length=64),
            FieldSchema(name="chunk_index", dtype=DataType.INT32),
            FieldSchema(name="content", dtype=DataType.VARCHAR, max_length=8192),
            FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1024),
            FieldSchema(name="contract_type", dtype=DataType.VARCHAR, max_length=32),
            FieldSchema(name="create_time", dtype=DataType.INT64),
        ],
        description="Contract text embeddings"
    )
    
    # 条款向量集合Schema
    CLAUSE_SCHEMA = CollectionSchema(
        fields=[
            FieldSchema(name="id", dtype=DataType.VARCHAR, max_length=64, is_primary=True),
            FieldSchema(name="tenant_id", dtype=DataType.VARCHAR, max_length=32),
            FieldSchema(name="clause_type", dtype=DataType.VARCHAR, max_length=32),
            FieldSchema(name="clause_content", dtype=DataType.VARCHAR, max_length=4096),
            FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1024),
            FieldSchema(name="risk_level", dtype=DataType.VARCHAR, max_length=16),
            FieldSchema(name="source_contract_id", dtype=DataType.VARCHAR, max_length=64),
        ],
        description="Contract clause embeddings"
    )
    
    def __init__(self, host: str = "milvus", port: int = 19530):
        from pymilvus import connections
        connections.connect(host=host, port=port)
    
    def create_collections(self):
        """创建向量集合"""
        # 合同向量集合
        contract_col = Collection("contract_vectors", self.CONTRACT_SCHEMA)
        contract_col.create_index(
            field_name="embedding",
            index_params={
                "metric_type": "COSINE",
                "index_type": "IVF_PQ",
                "params": {"nlist": 2048, "m": 16, "nbits": 8}
            }
        )
        
        # 条款向量集合
        clause_col = Collection("clause_vectors", self.CLAUSE_SCHEMA)
        clause_col.create_index(
            field_name="embedding",
            index_params={
                "metric_type": "COSINE",
                "index_type": "HNSW",
                "params": {"M": 32, "efConstruction": 256}
            }
        )
    
    def semantic_search(self, query_embedding: list, tenant_id: str, 
                        top_k: int = 10, filters: dict = None) -> list:
        """语义搜索"""
        collection = Collection("contract_vectors")
        collection.load()
        
        # 构建过滤表达式
        filter_expr = f'tenant_id == "{tenant_id}"'
        if filters:
            if filters.get("domain_code"):
                filter_expr += f' and domain_code == "{filters["domain_code"]}"'
            if filters.get("contract_type"):
                filter_expr += f' and contract_type == "{filters["contract_type"]}"'
        
        results = collection.search(
            data=[query_embedding],
            anns_field="embedding",
            param={"metric_type": "COSINE", "params": {"nprobe": 64}},
            limit=top_k,
            expr=filter_expr,
            output_fields=["contract_id", "content", "chunk_index"]
        )
        
        return [
            {
                "contract_id": hit.entity.get("contract_id"),
                "content": hit.entity.get("content"),
                "score": hit.score
            }
            for hit in results[0]
        ]
```

---

## 4. LLM服务层

### 4.1 服务架构

```
┌────────────────────────────────────────────────────────────────┐
│                       LLM Gateway                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  请求路由 │ 负载均衡 │ 限流熔断 │ Token计量 │ 缓存       │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  OpenAI API  │      │  私有化部署   │      │  国产大模型   │
│  GPT-4/3.5   │      │  Qwen/GLM    │      │ 文心/通义/智谱 │
└──────────────┘      └──────────────┘      └──────────────┘
```

### 4.2 核心实现

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Dict, Optional, AsyncIterator
import asyncio
import hashlib
import json

@dataclass
class LLMRequest:
    prompt: str
    system_prompt: Optional[str] = None
    temperature: float = 0.7
    max_tokens: int = 2048
    model: str = "default"
    stream: bool = False

@dataclass
class LLMResponse:
    content: str
    usage: Dict[str, int]
    model: str
    finish_reason: str

class LLMProvider(ABC):
    @abstractmethod
    async def complete(self, request: LLMRequest) -> LLMResponse:
        pass
    
    @abstractmethod
    async def stream_complete(self, request: LLMRequest) -> AsyncIterator[str]:
        pass

class QwenProvider(LLMProvider):
    """通义千问私有化部署"""
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.api_key = api_key
    
    async def complete(self, request: LLMRequest) -> LLMResponse:
        import aiohttp
        async with aiohttp.ClientSession() as session:
            payload = {
                "model": "qwen-72b-chat",
                "messages": [
                    {"role": "system", "content": request.system_prompt or "你是合同分析助手"},
                    {"role": "user", "content": request.prompt}
                ],
                "temperature": request.temperature,
                "max_tokens": request.max_tokens
            }
            async with session.post(
                f"{self.base_url}/v1/chat/completions",
                json=payload,
                headers={"Authorization": f"Bearer {self.api_key}"}
            ) as resp:
                result = await resp.json()
                return LLMResponse(
                    content=result["choices"][0]["message"]["content"],
                    usage=result["usage"],
                    model=result["model"],
                    finish_reason=result["choices"][0]["finish_reason"]
                )

class LLMService:
    """LLM服务统一入口"""
    
    def __init__(self, redis_client, config: dict):
        self.redis = redis_client
        self.providers = self._init_providers(config)
        self.default_provider = config.get("default_provider", "qwen")
    
    def _init_providers(self, config: dict) -> Dict[str, LLMProvider]:
        providers = {}
        if "qwen" in config:
            providers["qwen"] = QwenProvider(config["qwen"]["url"], config["qwen"]["key"])
        # 可扩展其他provider
        return providers
    
    def _cache_key(self, request: LLMRequest) -> str:
        content = f"{request.model}:{request.prompt}:{request.system_prompt}:{request.temperature}"
        return f"llm:cache:{hashlib.md5(content.encode()).hexdigest()}"
    
    async def complete(self, request: LLMRequest, use_cache: bool = True) -> LLMResponse:
        # 检查缓存
        if use_cache:
            cache_key = self._cache_key(request)
            cached = await self.redis.get(cache_key)
            if cached:
                return LLMResponse(**json.loads(cached))
        
        # 选择provider
        provider_name = request.model if request.model in self.providers else self.default_provider
        provider = self.providers[provider_name]
        
        # 执行请求
        response = await provider.complete(request)
        
        # 写入缓存
        if use_cache:
            await self.redis.setex(cache_key, 3600, json.dumps(response.__dict__))
        
        return response
    
    # === 合同专用方法 ===
    
    async def extract_entities(self, contract_text: str) -> Dict:
        """实体抽取"""
        prompt = f"""请从以下合同文本中提取关键实体信息，以JSON格式返回：
- 甲方、乙方
- 合同金额、币种
- 签署日期、生效日期、到期日期
- 合同标的

合同文本：
{contract_text[:4000]}
"""
        response = await self.complete(LLMRequest(prompt=prompt, temperature=0.1))
        return json.loads(response.content)
    
    async def analyze_risk(self, clause_text: str) -> Dict:
        """条款风险分析"""
        prompt = f"""分析以下合同条款的潜在风险，返回JSON格式：
{{"risk_level": "HIGH/MEDIUM/LOW", "risk_points": [...], "suggestions": [...]}}

条款内容：
{clause_text}
"""
        response = await self.complete(LLMRequest(prompt=prompt, temperature=0.2))
        return json.loads(response.content)
    
    async def generate_summary(self, contract_text: str) -> str:
        """生成合同摘要"""
        prompt = f"""请用200字以内概括以下合同的主要内容：

{contract_text[:6000]}
"""
        response = await self.complete(LLMRequest(prompt=prompt, temperature=0.5))
        return response.content
```

---

## 5. 模型训练平台

### 5.1 平台架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        模型训练平台                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Web Console                           │    │
│  │     实验管理 │ 模型管理 │ 数据集管理 │ 资源监控           │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    任务调度层                             │    │
│  │              Kubernetes + Volcano调度器                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │   训练引擎    │  │   推理引擎    │  │   评估引擎    │       │
│  │  PyTorch/TF   │  │ Triton/vLLM  │  │   Metrics     │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    存储层                                │    │
│  │   模型仓库(MinIO) │ 数据集(HDFS) │ 实验追踪(MLflow)      │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 训练任务定义

```python
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from enum import Enum

class TaskType(Enum):
    NER = "ner"                    # 命名实体识别
    CLASSIFICATION = "classification"  # 合同分类
    EXTRACTION = "extraction"      # 信息抽取
    EMBEDDING = "embedding"        # 向量化模型

@dataclass
class TrainingConfig:
    task_type: TaskType
    model_name: str
    base_model: str               # 基座模型
    dataset_path: str
    output_path: str
    
    # 训练参数
    epochs: int = 10
    batch_size: int = 32
    learning_rate: float = 2e-5
    warmup_ratio: float = 0.1
    weight_decay: float = 0.01
    
    # 资源配置
    gpu_count: int = 1
    gpu_memory: str = "24Gi"
    
    # 分布式配置
    distributed: bool = False
    world_size: int = 1
    
    extra_params: Dict = field(default_factory=dict)

@dataclass
class TrainingJob:
    job_id: str
    config: TrainingConfig
    status: str = "pending"
    metrics: Dict = field(default_factory=dict)
    created_at: str = ""
    started_at: Optional[str] = None
    finished_at: Optional[str] = None

class ModelTrainer:
    """模型训练器"""
    
    def __init__(self, mlflow_uri: str, model_registry: str):
        import mlflow
        mlflow.set_tracking_uri(mlflow_uri)
        self.model_registry = model_registry
    
    def train_ner_model(self, config: TrainingConfig) -> Dict:
        """训练NER模型"""
        import mlflow
        from transformers import (
            AutoTokenizer, AutoModelForTokenClassification,
            TrainingArguments, Trainer
        )
        
        with mlflow.start_run(run_name=config.model_name):
            # 记录参数
            mlflow.log_params({
                "base_model": config.base_model,
                "epochs": config.epochs,
                "batch_size": config.batch_size,
                "learning_rate": config.learning_rate
            })
            
            # 加载模型和分词器
            tokenizer = AutoTokenizer.from_pretrained(config.base_model)
            model = AutoModelForTokenClassification.from_pretrained(
                config.base_model,
                num_labels=len(self._get_ner_labels())
            )
            
            # 准备数据集
            train_dataset, eval_dataset = self._prepare_ner_dataset(
                config.dataset_path, tokenizer
            )
            
            # 训练参数
            training_args = TrainingArguments(
                output_dir=config.output_path,
                num_train_epochs=config.epochs,
                per_device_train_batch_size=config.batch_size,
                learning_rate=config.learning_rate,
                warmup_ratio=config.warmup_ratio,
                weight_decay=config.weight_decay,
                evaluation_strategy="epoch",
                save_strategy="epoch",
                load_best_model_at_end=True,
                fp16=True
            )
            
            # 开始训练
            trainer = Trainer(
                model=model,
                args=training_args,
                train_dataset=train_dataset,
                eval_dataset=eval_dataset,
                compute_metrics=self._compute_ner_metrics
            )
            
            train_result = trainer.train()
            
            # 记录指标
            mlflow.log_metrics({
                "train_loss": train_result.training_loss,
                "eval_f1": trainer.evaluate()["eval_f1"]
            })
            
            # 保存模型
            trainer.save_model(config.output_path)
            mlflow.transformers.log_model(
                transformers_model={"model": model, "tokenizer": tokenizer},
                artifact_path="model",
                registered_model_name=config.model_name
            )
            
            return {"status": "success", "run_id": mlflow.active_run().info.run_id}
    
    def _get_ner_labels(self) -> List[str]:
        return [
            "O", "B-PARTY", "I-PARTY", "B-AMOUNT", "I-AMOUNT",
            "B-DATE", "I-DATE", "B-TERM", "I-TERM", "B-CLAUSE", "I-CLAUSE"
        ]
```

---

## 6. 特征工程

### 6.1 特征存储设计

```python
from dataclasses import dataclass
from typing import Dict, List, Any, Optional
from datetime import datetime
import pandas as pd

@dataclass
class FeatureDefinition:
    name: str
    dtype: str
    description: str
    source: str              # 来源表/计算逻辑
    aggregation: Optional[str] = None  # 聚合方式
    window: Optional[str] = None       # 时间窗口

class FeatureStore:
    """特征存储"""
    
    # 合同特征定义
    CONTRACT_FEATURES = [
        FeatureDefinition("contract_amount", "float", "合同金额", "dwd.contract_detail"),
        FeatureDefinition("contract_duration_days", "int", "合同期限(天)", "computed"),
        FeatureDefinition("party_contract_count_30d", "int", "交易方近30天合同数", "dws", "count", "30d"),
        FeatureDefinition("party_total_amount_90d", "float", "交易方近90天总金额", "dws", "sum", "90d"),
        FeatureDefinition("contract_type_onehot", "array", "合同类型独热编码", "computed"),
        FeatureDefinition("text_embedding", "vector", "合同文本向量", "model"),
        FeatureDefinition("risk_score", "float", "风险评分", "model"),
        FeatureDefinition("clause_count", "int", "条款数量", "dwd.contract_detail"),
        FeatureDefinition("has_penalty_clause", "bool", "是否含违约条款", "dwd.contract_detail"),
    ]
    
    def __init__(self, spark_session, redis_client, feature_table: str):
        self.spark = spark_session
        self.redis = redis_client
        self.feature_table = feature_table
    
    def compute_features(self, contract_id: str) -> Dict[str, Any]:
        """计算合同特征"""
        # 基础特征
        base_features = self._get_base_features(contract_id)
        
        # 聚合特征
        agg_features = self._get_aggregated_features(contract_id, base_features.get("party_b"))
        
        # 向量特征
        embedding = self._get_embedding(contract_id)
        
        features = {**base_features, **agg_features, "text_embedding": embedding}
        
        # 缓存到Redis
        self.redis.hset(f"features:{contract_id}", mapping=features)
        
        return features
    
    def _get_base_features(self, contract_id: str) -> Dict:
        df = self.spark.sql(f"""
            SELECT 
                amount as contract_amount,
                DATEDIFF(expiry_date, effective_date) as contract_duration_days,
                party_b,
                contract_type,
                size(key_clauses) as clause_count,
                array_contains(key_clauses, '违约责任') as has_penalty_clause
            FROM dwd.contract_detail 
            WHERE contract_id = '{contract_id}'
        """)
        return df.first().asDict() if df.count() > 0 else {}
    
    def _get_aggregated_features(self, contract_id: str, party: str) -> Dict:
        df = self.spark.sql(f"""
            SELECT 
                COUNT(*) as party_contract_count_30d,
                SUM(amount) as party_total_amount_90d
            FROM dwd.contract_detail
            WHERE party_b = '{party}'
              AND sign_date >= date_sub(current_date(), 90)
        """)
        return df.first().asDict() if df.count() > 0 else {}
    
    def get_features_batch(self, contract_ids: List[str]) -> pd.DataFrame:
        """批量获取特征"""
        features_list = []
        for cid in contract_ids:
            cached = self.redis.hgetall(f"features:{cid}")
            if cached:
                features_list.append(cached)
            else:
                features_list.append(self.compute_features(cid))
        return pd.DataFrame(features_list)
```

---

## 7. 数据治理

### 7.1 治理体系

```
┌────────────────────────────────────────────────────────────────┐
│                        数据治理体系                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     元数据管理                            │  │
│  │    技术元数据 │ 业务元数据 │ 操作元数据 │ 数据血缘         │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     数据质量                              │  │
│  │    完整性 │ 准确性 │ 一致性 │ 时效性 │ 唯一性              │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     数据安全                              │  │
│  │    分类分级 │ 脱敏规则 │ 访问控制 │ 审计日志              │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     数据标准                              │  │
│  │    命名规范 │ 编码标准 │ 指标定义 │ 术语表                │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 7.2 数据质量规则

```python
from dataclasses import dataclass
from typing import Callable, List, Dict, Any
from enum import Enum

class QualityDimension(Enum):
    COMPLETENESS = "completeness"   # 完整性
    ACCURACY = "accuracy"           # 准确性
    CONSISTENCY = "consistency"     # 一致性
    TIMELINESS = "timeliness"       # 时效性
    UNIQUENESS = "uniqueness"       # 唯一性

@dataclass
class QualityRule:
    rule_id: str
    rule_name: str
    dimension: QualityDimension
    table_name: str
    column_name: str
    rule_expr: str              # SQL表达式
    threshold: float = 0.95     # 通过阈值
    severity: str = "warning"   # error/warning/info

class DataQualityEngine:
    """数据质量引擎"""
    
    # 预定义规则
    CONTRACT_RULES = [
        QualityRule("R001", "合同编号非空", QualityDimension.COMPLETENESS,
                   "dwd.contract_detail", "contract_no",
                   "contract_no IS NOT NULL AND contract_no != ''"),
        QualityRule("R002", "金额非负", QualityDimension.ACCURACY,
                   "dwd.contract_detail", "amount",
                   "amount >= 0"),
        QualityRule("R003", "日期合理性", QualityDimension.ACCURACY,
                   "dwd.contract_detail", "sign_date",
                   "sign_date <= current_date() AND sign_date >= '2000-01-01'"),
        QualityRule("R004", "合同编号唯一", QualityDimension.UNIQUENESS,
                   "dwd.contract_detail", "contract_no",
                   "COUNT(*) = COUNT(DISTINCT contract_no)"),
        QualityRule("R005", "生效日期<=到期日期", QualityDimension.CONSISTENCY,
                   "dwd.contract_detail", "effective_date",
                   "effective_date <= expiry_date"),
    ]
    
    def __init__(self, spark_session):
        self.spark = spark_session
    
    def run_quality_check(self, rules: List[QualityRule] = None) -> List[Dict]:
        """执行质量检查"""
        rules = rules or self.CONTRACT_RULES
        results = []
        
        for rule in rules:
            result = self._check_rule(rule)
            results.append(result)
            
            # 记录到质量日志表
            self._log_result(result)
        
        return results
    
    def _check_rule(self, rule: QualityRule) -> Dict:
        total_sql = f"SELECT COUNT(*) as total FROM {rule.table_name}"
        pass_sql = f"SELECT COUNT(*) as passed FROM {rule.table_name} WHERE {rule.rule_expr}"
        
        total = self.spark.sql(total_sql).first()["total"]
        passed = self.spark.sql(pass_sql).first()["passed"]
        
        pass_rate = passed / total if total > 0 else 1.0
        
        return {
            "rule_id": rule.rule_id,
            "rule_name": rule.rule_name,
            "dimension": rule.dimension.value,
            "total_records": total,
            "passed_records": passed,
            "pass_rate": pass_rate,
            "threshold": rule.threshold,
            "status": "PASS" if pass_rate >= rule.threshold else "FAIL",
            "severity": rule.severity
        }

class DataMasking:
    """数据脱敏"""
    
    MASKING_RULES = {
        "phone": lambda x: x[:3] + "****" + x[-4:] if x and len(x) >= 11 else x,
        "id_card": lambda x: x[:6] + "********" + x[-4:] if x and len(x) >= 18 else x,
        "bank_account": lambda x: x[:4] + "****" + x[-4:] if x and len(x) >= 12 else x,
        "name": lambda x: x[0] + "*" * (len(x) - 1) if x else x,
        "amount": lambda x: round(x, -3) if x else x,  # 金额取整到千位
    }
    
    @classmethod
    def mask(cls, value: Any, mask_type: str) -> Any:
        if mask_type in cls.MASKING_RULES:
            return cls.MASKING_RULES[mask_type](value)
        return value
```

---

## 8. MLOps流水线

### 8.1 流水线架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        MLOps Pipeline                           │
│                                                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ 数据准备 │ -> │ 模型训练 │ -> │ 模型评估 │ -> │ 模型注册 │      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
│       │              │              │              │            │
│       ▼              ▼              ▼              ▼            │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ 数据版本 │    │ 实验追踪 │    │ 指标对比 │    │ 版本管理 │      │
│  │  (DVC)  │    │ (MLflow)│    │         │    │         │      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
│                                                     │           │
│                                                     ▼           │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ 模型监控 │ <- │ AB测试  │ <- │ 灰度发布 │ <- │ 模型部署 │      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 流水线实现

```python
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from enum import Enum
from datetime import datetime

class PipelineStage(Enum):
    DATA_PREP = "data_preparation"
    TRAINING = "training"
    EVALUATION = "evaluation"
    REGISTRATION = "registration"
    DEPLOYMENT = "deployment"
    MONITORING = "monitoring"

@dataclass
class PipelineConfig:
    pipeline_id: str
    model_name: str
    stages: List[PipelineStage]
    schedule: Optional[str] = None  # cron表达式
    trigger: str = "manual"         # manual/schedule/data_change
    params: Dict = field(default_factory=dict)

@dataclass
class PipelineRun:
    run_id: str
    pipeline_id: str
    status: str
    current_stage: PipelineStage
    metrics: Dict = field(default_factory=dict)
    artifacts: Dict = field(default_factory=dict)
    started_at: datetime = None
    finished_at: datetime = None

class MLOpsPipeline:
    """MLOps流水线"""
    
    def __init__(self, mlflow_client, k8s_client, model_registry):
        self.mlflow = mlflow_client
        self.k8s = k8s_client
        self.registry = model_registry
    
    async def run_pipeline(self, config: PipelineConfig) -> PipelineRun:
        """执行流水线"""
        run = PipelineRun(
            run_id=self._generate_run_id(),
            pipeline_id=config.pipeline_id,
            status="running",
            current_stage=config.stages[0],
            started_at=datetime.now()
        )
        
        try:
            for stage in config.stages:
                run.current_stage = stage
                await self._execute_stage(stage, config, run)
            
            run.status = "completed"
        except Exception as e:
            run.status = "failed"
            run.metrics["error"] = str(e)
        finally:
            run.finished_at = datetime.now()
        
        return run
    
    async def _execute_stage(self, stage: PipelineStage, config: PipelineConfig, run: PipelineRun):
        """执行单个阶段"""
        handlers = {
            PipelineStage.DATA_PREP: self._data_preparation,
            PipelineStage.TRAINING: self._training,
            PipelineStage.EVALUATION: self._evaluation,
            PipelineStage.REGISTRATION: self._registration,
            PipelineStage.DEPLOYMENT: self._deployment,
        }
        
        handler = handlers.get(stage)
        if handler:
            await handler(config, run)
    
    async def _data_preparation(self, config: PipelineConfig, run: PipelineRun):
        """数据准备阶段"""
        # 数据版本化
        data_version = await self._version_dataset(config.params.get("dataset_path"))
        run.artifacts["data_version"] = data_version
    
    async def _training(self, config: PipelineConfig, run: PipelineRun):
        """训练阶段"""
        # 提交K8s训练任务
        job_manifest = self._create_training_job(config)
        await self.k8s.create_job(job_manifest)
        
        # 等待训练完成并获取指标
        metrics = await self._wait_for_training(job_manifest["metadata"]["name"])
        run.metrics.update(metrics)
    
    async def _evaluation(self, config: PipelineConfig, run: PipelineRun):
        """评估阶段"""
        # 模型评估
        eval_metrics = await self._evaluate_model(
            run.artifacts.get("model_path"),
            config.params.get("eval_dataset")
        )
        run.metrics["evaluation"] = eval_metrics
        
        # 与基线模型对比
        baseline = await self._get_baseline_metrics(config.model_name)
        run.metrics["improvement"] = self._compare_metrics(eval_metrics, baseline)
    
    async def _registration(self, config: PipelineConfig, run: PipelineRun):
        """注册阶段"""
        # 检查是否达到上线标准
        if run.metrics.get("improvement", {}).get("f1", 0) > 0:
            model_version = await self.registry.register_model(
                name=config.model_name,
                path=run.artifacts.get("model_path"),
                metrics=run.metrics
            )
            run.artifacts["model_version"] = model_version
    
    async def _deployment(self, config: PipelineConfig, run: PipelineRun):
        """部署阶段"""
        model_version = run.artifacts.get("model_version")
        if not model_version:
            return
        
        # 灰度部署
        await self._canary_deploy(
            model_name=config.model_name,
            model_version=model_version,
            traffic_percent=10  # 初始10%流量
        )
    
    async def _canary_deploy(self, model_name: str, model_version: str, traffic_percent: int):
        """金丝雀部署"""
        deployment = {
            "apiVersion": "serving.kubeflow.org/v1beta1",
            "kind": "InferenceService",
            "metadata": {"name": model_name},
            "spec": {
                "predictor": {
                    "canaryTrafficPercent": traffic_percent,
                    "model": {
                        "modelFormat": {"name": "pytorch"},
                        "storageUri": f"s3://models/{model_name}/{model_version}"
                    }
                }
            }
        }
        await self.k8s.apply(deployment)

class ModelMonitor:
    """模型监控"""
    
    def __init__(self, metrics_client, alert_client):
        self.metrics = metrics_client
        self.alerts = alert_client
    
    async def check_model_health(self, model_name: str) -> Dict:
        """检查模型健康状态"""
        metrics = {
            "latency_p99": await self._get_latency_p99(model_name),
            "error_rate": await self._get_error_rate(model_name),
            "qps": await self._get_qps(model_name),
            "data_drift": await self._detect_data_drift(model_name),
            "prediction_drift": await self._detect_prediction_drift(model_name)
        }
        
        # 告警检查
        if metrics["error_rate"] > 0.01:
            await self.alerts.send("模型错误率超过1%", model_name, metrics)
        
        if metrics["data_drift"] > 0.2:
            await self.alerts.send("检测到数据漂移", model_name, metrics)
        
        return metrics
```

---

## 9. 技术选型总结

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 数据湖存储 | MinIO + Apache Iceberg | 开源、兼容S3、支持ACID |
| 计算引擎 | Apache Spark 3.x | 批处理、SQL支持 |
| 实时计算 | Apache Flink | 流处理、CDC |
| 向量数据库 | Milvus | 高性能向量检索 |
| LLM推理 | vLLM + Triton | 高吞吐推理服务 |
| 实验追踪 | MLflow | 实验管理、模型注册 |
| 特征存储 | Feast | 离线/在线特征服务 |
| 任务调度 | Apache Airflow | DAG调度、监控 |
| 模型部署 | KServe | K8s原生推理服务 |
| 数据治理 | Apache Atlas | 元数据管理、血缘追踪 |

---

## 10. 部署架构

```yaml
# Kubernetes部署示例
apiVersion: v1
kind: Namespace
metadata:
  name: data-ai-platform
---
# Milvus向量数据库
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: contract-milvus
  namespace: data-ai-platform
spec:
  mode: cluster
  components:
    queryNode:
      replicas: 3
      resources:
        limits:
          memory: 16Gi
          cpu: 4
    dataNode:
      replicas: 2
  dependencies:
    etcd:
      inCluster:
        values:
          replicaCount: 3
    storage:
      inCluster:
        values:
          mode: distributed
          persistence:
            size: 500Gi
---
# LLM推理服务
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: contract-llm
  namespace: data-ai-platform
spec:
  predictor:
    minReplicas: 2
    maxReplicas: 10
    model:
      modelFormat:
        name: pytorch
      runtime: kserve-vllm
      storageUri: s3://models/qwen-72b
      resources:
        limits:
          nvidia.com/gpu: 2
          memory: 80Gi
---
# Spark Operator
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: contract-etl
  namespace: data-ai-platform
spec:
  type: Python
  mode: cluster
  image: contract-platform/spark:3.5
  mainApplicationFile: s3://jobs/contract_etl.py
  sparkVersion: "3.5.0"
  driver:
    cores: 2
    memory: 4g
  executor:
    cores: 4
    instances: 5
    memory: 8g
```

---

本文档完整描述了数据/AI基座的核心组件设计，包括数据湖仓、向量数据库、LLM服务、模型训练平台、特征工程、数据治理和MLOps流水线，为统一合同平台提供强大的数据和智能能力支撑。
