# 合同查询服务 (Contract Query Service)

## 1. 服务概述

合同查询服务提供高性能、多维度的合同检索能力，支持：
- 全文检索
- 结构化条件查询
- 语义相似检索
- 聚合统计分析

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          Contract Query Service                                  │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                         API Gateway Layer                                  │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                      │ │
│  │  │  REST    │ │  GraphQL │ │  gRPC    │ │ WebSocket│                      │ │
│  │  │  API     │ │  API     │ │  API     │ │ (实时)   │                      │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘                      │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                       Query Router & Optimizer                             │ │
│  │                                                                            │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │ │
│  │  │  查询解析 → 查询优化 → 数据源路由 → 结果合并                          │  │ │
│  │  └─────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│          ┌───────────────────────────┼───────────────────────────┐             │
│          ▼                           ▼                           ▼             │
│  ┌────────────────┐        ┌────────────────┐        ┌────────────────┐        │
│  │   全文检索引擎  │        │   结构化查询    │        │   语义检索引擎  │        │
│  │ Full-text Search│        │ Structured Query│        │ Semantic Search│        │
│  │                │        │                │        │                │        │
│  │ ┌────────────┐ │        │ ┌────────────┐ │        │ ┌────────────┐ │        │
│  │ │Elasticsearch│ │        │ │ PostgreSQL │ │        │ │   Milvus   │ │        │
│  │ │            │ │        │ │            │ │        │ │  /pgvector │ │        │
│  │ └────────────┘ │        │ └────────────┘ │        │ └────────────┘ │        │
│  │                │        │                │        │                │        │
│  │ • 关键词检索   │        │ • 精确条件     │        │ • 相似合同检索  │        │
│  │ • 高亮显示    │        │ • 范围查询     │        │ • 条款语义匹配  │        │
│  │ • 模糊匹配    │        │ • 关联查询     │        │ • 智能问答      │        │
│  │ • 分词搜索    │        │ • 聚合统计     │        │ • 推荐关联合同  │        │
│  │                │        │                │        │                │        │
│  └────────────────┘        └────────────────┘        └────────────────┘        │
│                                      │                                          │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Result Processing Layer                            │ │
│  │                                                                            │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │ │
│  │  │ 权限过滤 │  │ 结果排序 │  │ 高亮处理 │  │ 分页处理 │  │ 缓存管理 │    │ │
│  │  │ ACL Filter│  │ Ranking  │  │Highlight │  │Pagination│  │  Cache   │    │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │ │
│  │                                                                            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 全文检索设计

### 2.1 Elasticsearch 索引设计

```json
// 合同索引映射
PUT /contracts_template
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "contract_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "contract_synonym"]
        },
        "contract_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["lowercase", "contract_synonym"]
        }
      },
      "filter": {
        "contract_synonym": {
          "type": "synonym",
          "synonyms_path": "analysis/contract_synonyms.txt"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      // 基础字段
      "contract_id": { "type": "keyword" },
      "contract_no": { 
        "type": "keyword",
        "fields": {
          "text": { "type": "text", "analyzer": "standard" }
        }
      },
      "contract_name": {
        "type": "text",
        "analyzer": "contract_analyzer",
        "search_analyzer": "contract_search_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      
      // 分类字段
      "domain_code": { "type": "keyword" },
      "contract_type": { "type": "keyword" },
      "contract_subtype": { "type": "keyword" },
      "status": { "type": "keyword" },
      
      // 金额字段
      "amount": { "type": "double" },
      "currency": { "type": "keyword" },
      
      // 日期字段
      "sign_date": { "type": "date" },
      "effective_date": { "type": "date" },
      "expiry_date": { "type": "date" },
      "created_at": { "type": "date" },
      
      // 当事人 (嵌套类型)
      "parties": {
        "type": "nested",
        "properties": {
          "role": { "type": "keyword" },
          "name": {
            "type": "text",
            "analyzer": "contract_analyzer",
            "fields": {
              "keyword": { "type": "keyword" }
            }
          },
          "id_number": { "type": "keyword" }
        }
      },
      
      // 标签
      "tags": { "type": "keyword" },
      
      // 全文检索字段
      "summary": {
        "type": "text",
        "analyzer": "contract_analyzer",
        "search_analyzer": "contract_search_analyzer"
      },
      "full_text": {
        "type": "text",
        "analyzer": "contract_analyzer",
        "search_analyzer": "contract_search_analyzer",
        "index_options": "offsets"  // 支持高亮
      },
      
      // 条款内容 (嵌套)
      "clauses": {
        "type": "nested",
        "properties": {
          "clause_type": { "type": "keyword" },
          "clause_title": { "type": "text" },
          "clause_content": {
            "type": "text",
            "analyzer": "contract_analyzer"
          }
        }
      },
      
      // 自动补全
      "suggest": {
        "type": "completion",
        "analyzer": "standard",
        "preserve_separators": true,
        "preserve_position_increments": true,
        "max_input_length": 50
      }
    }
  }
}
```

### 2.2 搜索同义词配置

```
# analysis/contract_synonyms.txt
# 合同类型同义词
买卖合同,销售合同,采购合同
租赁合同,租房合同,房屋租赁
服务合同,服务协议
劳动合同,雇佣合同,用工合同

# 条款同义词
违约金,罚款,赔偿金
保证金,押金,定金
有效期,合同期限,履行期限

# 当事人同义词
甲方,卖方,出租方,委托方
乙方,买方,承租方,受托方

# 金融术语
人民币,CNY,RMB
美元,USD,美金
```

### 2.3 搜索服务实现

```python
class ContractSearchService:
    def __init__(self, es_client: AsyncElasticsearch):
        self.es = es_client
        
    async def search(self, 
                     tenant_id: str,
                     query: SearchQuery) -> SearchResult:
        # 构建ES查询
        es_query = self._build_es_query(tenant_id, query)
        
        # 执行搜索
        response = await self.es.search(
            index=f"contracts_{tenant_id}",
            body=es_query,
            request_timeout=30
        )
        
        # 解析结果
        return self._parse_response(response, query)
    
    def _build_es_query(self, 
                        tenant_id: str, 
                        query: SearchQuery) -> Dict:
        must_clauses = []
        filter_clauses = []
        should_clauses = []
        
        # 1. 关键词搜索
        if query.keyword:
            must_clauses.append({
                "multi_match": {
                    "query": query.keyword,
                    "fields": [
                        "contract_name^3",      # 名称权重最高
                        "contract_no^2",
                        "summary^2",
                        "full_text",
                        "clauses.clause_content"
                    ],
                    "type": "best_fields",
                    "fuzziness": "AUTO",
                    "minimum_should_match": "70%"
                }
            })
            
        # 2. 当事人搜索 (嵌套查询)
        if query.party_name:
            must_clauses.append({
                "nested": {
                    "path": "parties",
                    "query": {
                        "match": {
                            "parties.name": {
                                "query": query.party_name,
                                "fuzziness": "AUTO"
                            }
                        }
                    }
                }
            })
            
        # 3. 过滤条件
        if query.domain_code:
            filter_clauses.append({"term": {"domain_code": query.domain_code}})
            
        if query.contract_type:
            filter_clauses.append({"term": {"contract_type": query.contract_type}})
            
        if query.status:
            filter_clauses.append({"terms": {"status": query.status}})
            
        if query.amount_min or query.amount_max:
            range_query = {"range": {"amount": {}}}
            if query.amount_min:
                range_query["range"]["amount"]["gte"] = query.amount_min
            if query.amount_max:
                range_query["range"]["amount"]["lte"] = query.amount_max
            filter_clauses.append(range_query)
            
        if query.sign_date_from or query.sign_date_to:
            range_query = {"range": {"sign_date": {}}}
            if query.sign_date_from:
                range_query["range"]["sign_date"]["gte"] = query.sign_date_from
            if query.sign_date_to:
                range_query["range"]["sign_date"]["lte"] = query.sign_date_to
            filter_clauses.append(range_query)
            
        if query.tags:
            filter_clauses.append({"terms": {"tags": query.tags}})
        
        # 构建最终查询
        es_query = {
            "query": {
                "bool": {
                    "must": must_clauses if must_clauses else [{"match_all": {}}],
                    "filter": filter_clauses,
                    "should": should_clauses
                }
            },
            "from": (query.page - 1) * query.page_size,
            "size": query.page_size,
            "sort": self._build_sort(query.sort_by, query.sort_order),
            "highlight": {
                "fields": {
                    "contract_name": {},
                    "summary": {"fragment_size": 200},
                    "full_text": {"fragment_size": 200, "number_of_fragments": 3}
                },
                "pre_tags": ["<em>"],
                "post_tags": ["</em>"]
            },
            "aggs": self._build_aggregations(query.enable_aggregations)
        }
        
        return es_query
    
    def _build_aggregations(self, enabled: bool) -> Dict:
        if not enabled:
            return {}
            
        return {
            "by_domain": {
                "terms": {"field": "domain_code", "size": 20}
            },
            "by_type": {
                "terms": {"field": "contract_type", "size": 50}
            },
            "by_status": {
                "terms": {"field": "status", "size": 20}
            },
            "amount_stats": {
                "stats": {"field": "amount"}
            },
            "by_sign_month": {
                "date_histogram": {
                    "field": "sign_date",
                    "calendar_interval": "month",
                    "format": "yyyy-MM"
                }
            },
            "by_party": {
                "nested": {"path": "parties"},
                "aggs": {
                    "party_names": {
                        "terms": {"field": "parties.name.keyword", "size": 100}
                    }
                }
            }
        }
```

---

## 3. 语义检索设计

### 3.1 向量存储架构

```python
# 向量数据库配置 (Milvus)
from pymilvus import Collection, CollectionSchema, FieldSchema, DataType

# 合同向量集合Schema
contract_vector_schema = CollectionSchema(
    fields=[
        FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
        FieldSchema(name="contract_id", dtype=DataType.VARCHAR, max_length=32),
        FieldSchema(name="tenant_id", dtype=DataType.VARCHAR, max_length=32),
        FieldSchema(name="chunk_type", dtype=DataType.VARCHAR, max_length=32),  # SUMMARY/CLAUSE/FULL
        FieldSchema(name="chunk_index", dtype=DataType.INT32),
        FieldSchema(name="text", dtype=DataType.VARCHAR, max_length=8192),
        FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1024),  # BGE-large
    ],
    description="Contract text embeddings for semantic search"
)

# 创建索引
index_params = {
    "index_type": "IVF_PQ",
    "metric_type": "IP",  # Inner Product for normalized vectors
    "params": {
        "nlist": 1024,
        "m": 32,
        "nbits": 8
    }
}
```

### 3.2 语义检索服务

```python
class SemanticSearchService:
    def __init__(self, 
                 embedding_model: EmbeddingModel,
                 milvus_client: MilvusClient):
        self.embedding = embedding_model
        self.milvus = milvus_client
        
    async def search_similar_contracts(self,
                                        query_text: str,
                                        tenant_id: str,
                                        top_k: int = 10,
                                        filters: Optional[Dict] = None) -> List[SimilarContract]:
        """基于文本查找相似合同"""
        
        # 1. 文本向量化
        query_embedding = await self.embedding.encode(query_text)
        
        # 2. 构建过滤表达式
        filter_expr = f'tenant_id == "{tenant_id}"'
        if filters:
            if filters.get("domain_code"):
                filter_expr += f' and domain_code == "{filters["domain_code"]}"'
            if filters.get("contract_type"):
                filter_expr += f' and contract_type == "{filters["contract_type"]}"'
        
        # 3. 向量检索
        results = await self.milvus.search(
            collection_name="contract_vectors",
            data=[query_embedding],
            anns_field="embedding",
            param={"metric_type": "IP", "params": {"nprobe": 64}},
            limit=top_k * 3,  # 多检索一些，后续去重
            expr=filter_expr,
            output_fields=["contract_id", "chunk_type", "text"]
        )
        
        # 4. 结果去重 (同一合同可能有多个chunk命中)
        seen_contracts = set()
        unique_results = []
        for hit in results[0]:
            contract_id = hit.entity.get("contract_id")
            if contract_id not in seen_contracts:
                seen_contracts.add(contract_id)
                unique_results.append(SimilarContract(
                    contract_id=contract_id,
                    similarity_score=hit.score,
                    matched_chunk=hit.entity.get("text"),
                    chunk_type=hit.entity.get("chunk_type")
                ))
            if len(unique_results) >= top_k:
                break
                
        return unique_results
    
    async def search_similar_clauses(self,
                                      clause_text: str,
                                      clause_type: str,
                                      tenant_id: str,
                                      top_k: int = 20) -> List[SimilarClause]:
        """查找相似条款"""
        
        query_embedding = await self.embedding.encode(clause_text)
        
        results = await self.milvus.search(
            collection_name="clause_vectors",
            data=[query_embedding],
            anns_field="embedding",
            param={"metric_type": "IP", "params": {"nprobe": 64}},
            limit=top_k,
            expr=f'tenant_id == "{tenant_id}" and clause_type == "{clause_type}"',
            output_fields=["contract_id", "clause_id", "clause_text", "clause_title"]
        )
        
        return [
            SimilarClause(
                contract_id=hit.entity.get("contract_id"),
                clause_id=hit.entity.get("clause_id"),
                clause_title=hit.entity.get("clause_title"),
                clause_text=hit.entity.get("clause_text"),
                similarity_score=hit.score
            )
            for hit in results[0]
        ]
    
    async def contract_qa(self,
                          question: str,
                          contract_id: str,
                          tenant_id: str) -> ContractQAResponse:
        """合同智能问答"""
        
        # 1. 检索相关片段
        relevant_chunks = await self._retrieve_relevant_chunks(
            question, contract_id, tenant_id, top_k=5
        )
        
        # 2. 构建上下文
        context = "\n\n".join([chunk.text for chunk in relevant_chunks])
        
        # 3. LLM问答
        answer = await self.llm_client.generate(
            prompt=CONTRACT_QA_PROMPT.format(
                context=context,
                question=question
            ),
            temperature=0.3
        )
        
        return ContractQAResponse(
            question=question,
            answer=answer,
            source_chunks=relevant_chunks,
            confidence=self._calculate_confidence(relevant_chunks)
        )
```

---

## 4. 聚合分析设计

### 4.1 统计分析接口

```python
class ContractAnalyticsService:
    def __init__(self, 
                 es_client: AsyncElasticsearch,
                 db_client: AsyncDatabase):
        self.es = es_client
        self.db = db_client
        
    async def get_dashboard_stats(self,
                                   tenant_id: str,
                                   domain_code: Optional[str] = None,
                                   time_range: Optional[TimeRange] = None) -> DashboardStats:
        """获取仪表盘统计数据"""
        
        # 构建基础过滤条件
        filters = [{"term": {"tenant_id": tenant_id}}]
        if domain_code:
            filters.append({"term": {"domain_code": domain_code}})
        if time_range:
            filters.append({
                "range": {
                    "created_at": {
                        "gte": time_range.start.isoformat(),
                        "lte": time_range.end.isoformat()
                    }
                }
            })
        
        # 执行聚合查询
        response = await self.es.search(
            index=f"contracts_{tenant_id}",
            body={
                "size": 0,
                "query": {"bool": {"filter": filters}},
                "aggs": {
                    "total_count": {"value_count": {"field": "contract_id"}},
                    "total_amount": {"sum": {"field": "amount"}},
                    "avg_amount": {"avg": {"field": "amount"}},
                    
                    "by_status": {
                        "terms": {"field": "status"}
                    },
                    
                    "by_type": {
                        "terms": {"field": "contract_type", "size": 10}
                    },
                    
                    "expiring_soon": {
                        "filter": {
                            "range": {
                                "expiry_date": {
                                    "gte": "now",
                                    "lte": "now+30d"
                                }
                            }
                        }
                    },
                    
                    "monthly_trend": {
                        "date_histogram": {
                            "field": "sign_date",
                            "calendar_interval": "month"
                        },
                        "aggs": {
                            "amount": {"sum": {"field": "amount"}},
                            "count": {"value_count": {"field": "contract_id"}}
                        }
                    },
                    
                    "top_parties": {
                        "nested": {"path": "parties"},
                        "aggs": {
                            "party_ranking": {
                                "terms": {
                                    "field": "parties.name.keyword",
                                    "size": 10
                                }
                            }
                        }
                    }
                }
            }
        )
        
        return self._parse_dashboard_stats(response["aggregations"])
    
    async def get_expiry_alerts(self,
                                 tenant_id: str,
                                 days_ahead: int = 30) -> List[ExpiryAlert]:
        """获取即将到期的合同"""
        
        response = await self.es.search(
            index=f"contracts_{tenant_id}",
            body={
                "query": {
                    "bool": {
                        "filter": [
                            {"term": {"status": "EFFECTIVE"}},
                            {
                                "range": {
                                    "expiry_date": {
                                        "gte": "now",
                                        "lte": f"now+{days_ahead}d"
                                    }
                                }
                            }
                        ]
                    }
                },
                "sort": [{"expiry_date": "asc"}],
                "size": 100,
                "_source": [
                    "contract_id", "contract_name", "contract_no",
                    "expiry_date", "amount", "parties"
                ]
            }
        )
        
        return [
            ExpiryAlert(
                contract_id=hit["_source"]["contract_id"],
                contract_name=hit["_source"]["contract_name"],
                expiry_date=hit["_source"]["expiry_date"],
                days_until_expiry=self._calculate_days_until(hit["_source"]["expiry_date"]),
                amount=hit["_source"].get("amount"),
                parties=hit["_source"].get("parties", [])
            )
            for hit in response["hits"]["hits"]
        ]
```

### 4.2 自定义报表

```python
class ContractReportService:
    async def generate_report(self,
                               tenant_id: str,
                               report_config: ReportConfig) -> Report:
        """生成自定义报表"""
        
        # 1. 构建查询条件
        query_builder = ReportQueryBuilder(report_config)
        
        # 2. 执行数据查询
        if report_config.data_source == "es":
            data = await self._query_from_es(tenant_id, query_builder)
        else:
            data = await self._query_from_db(tenant_id, query_builder)
        
        # 3. 数据转换
        transformed_data = self._transform_data(data, report_config.transformations)
        
        # 4. 生成报表
        report = Report(
            report_id=generate_report_id(),
            title=report_config.title,
            created_at=datetime.utcnow(),
            data=transformed_data,
            summary=self._generate_summary(transformed_data),
            charts=self._generate_charts(transformed_data, report_config.charts)
        )
        
        # 5. 导出 (可选)
        if report_config.export_format:
            report.export_url = await self._export_report(
                report, report_config.export_format
            )
        
        return report


# 报表配置示例
report_config_example = {
    "title": "月度合同统计报表",
    "data_source": "es",
    "time_range": {
        "field": "sign_date",
        "start": "2024-01-01",
        "end": "2024-12-31"
    },
    "dimensions": ["contract_type", "domain_code"],
    "metrics": [
        {"name": "count", "type": "count"},
        {"name": "total_amount", "type": "sum", "field": "amount"},
        {"name": "avg_amount", "type": "avg", "field": "amount"}
    ],
    "filters": [
        {"field": "status", "operator": "in", "value": ["EFFECTIVE", "EXPIRED"]}
    ],
    "group_by": [
        {"field": "sign_date", "interval": "month"}
    ],
    "charts": [
        {"type": "line", "x": "sign_date", "y": "count", "title": "签约数量趋势"},
        {"type": "bar", "x": "contract_type", "y": "total_amount", "title": "各类型合同金额"},
        {"type": "pie", "dimension": "domain_code", "metric": "count", "title": "业务域分布"}
    ],
    "export_format": "xlsx"
}
```

---

## 5. GraphQL 查询接口

```graphql
# GraphQL Schema
type Query {
  # 合同查询
  contract(id: ID!): Contract
  contracts(filter: ContractFilter, page: Int, pageSize: Int): ContractConnection!
  
  # 搜索
  searchContracts(keyword: String!, filter: ContractFilter, page: Int, pageSize: Int): SearchResult!
  
  # 相似检索
  similarContracts(contractId: ID!, topK: Int): [SimilarContract!]!
  similarClauses(text: String!, clauseType: String!, topK: Int): [SimilarClause!]!
  
  # 智能问答
  contractQA(contractId: ID!, question: String!): QAResponse!
  
  # 统计
  dashboardStats(domainCode: String, timeRange: TimeRangeInput): DashboardStats!
  expiryAlerts(daysAhead: Int): [ExpiryAlert!]!
}

type Contract {
  id: ID!
  contractId: String!
  contractNo: String
  contractName: String!
  contractType: String
  domainCode: String!
  status: ContractStatus!
  amount: Float
  currency: String
  signDate: Date
  effectiveDate: Date
  expiryDate: Date
  
  # 关联数据
  parties: [Party!]!
  files: [ContractFile!]!
  clauses: [Clause!]!
  riskAlerts: [RiskAlert!]!
  changeHistory: [ChangeRecord!]!
  
  # 计算字段
  daysUntilExpiry: Int
  isExpiringSoon: Boolean!
  
  createdAt: DateTime!
  updatedAt: DateTime!
}

type Party {
  role: PartyRole!
  name: String!
  idType: String
  idNumber: String
  legalRepresentative: String
  contactPerson: String
  contactPhone: String
  address: String
}

type SearchResult {
  total: Int!
  hits: [SearchHit!]!
  aggregations: Aggregations
}

type SearchHit {
  contract: Contract!
  score: Float!
  highlights: [Highlight!]
}

type Highlight {
  field: String!
  fragments: [String!]!
}

type SimilarContract {
  contract: Contract!
  similarityScore: Float!
  matchedChunk: String
}

type QAResponse {
  question: String!
  answer: String!
  sourceChunks: [SourceChunk!]!
  confidence: Float!
}

input ContractFilter {
  domainCode: String
  contractType: String
  status: [ContractStatus!]
  partyName: String
  amountMin: Float
  amountMax: Float
  signDateFrom: Date
  signDateTo: Date
  tags: [String!]
}

input TimeRangeInput {
  start: DateTime!
  end: DateTime!
}
```

---

## 6. 性能优化

### 6.1 查询优化策略

```python
class QueryOptimizer:
    """查询优化器"""
    
    def optimize(self, query: SearchQuery) -> OptimizedQuery:
        optimizations = []
        
        # 1. 索引选择
        best_index = self._select_best_index(query)
        optimizations.append(f"Selected index: {best_index}")
        
        # 2. 查询重写
        rewritten_query = self._rewrite_query(query)
        optimizations.append("Query rewritten for optimal execution")
        
        # 3. 分页优化 (深分页使用 search_after)
        if query.page * query.page_size > 10000:
            rewritten_query.use_search_after = True
            optimizations.append("Using search_after for deep pagination")
        
        # 4. 字段裁剪
        if not query.need_full_text:
            rewritten_query.source_excludes = ["full_text"]
            optimizations.append("Excluded full_text from source")
        
        # 5. 缓存策略
        cache_key = self._generate_cache_key(rewritten_query)
        rewritten_query.cache_key = cache_key
        
        return OptimizedQuery(
            query=rewritten_query,
            optimizations=optimizations,
            estimated_cost=self._estimate_cost(rewritten_query)
        )
    
    def _select_best_index(self, query: SearchQuery) -> str:
        """选择最佳索引"""
        # 如果只查询特定时间范围，使用时间分片索引
        if query.time_range and query.time_range.days <= 30:
            return f"contracts_{query.tenant_id}_recent"
        return f"contracts_{query.tenant_id}"
```

### 6.2 缓存策略

```yaml
# 查询缓存配置
query_cache:
  # 热门查询缓存
  hot_queries:
    enabled: true
    max_size: 10000
    ttl: 5m
    cache_condition: "query.is_common_pattern"
    
  # 聚合结果缓存
  aggregation_cache:
    enabled: true
    ttl: 15m
    refresh_interval: 5m
    
  # 搜索建议缓存
  suggestion_cache:
    enabled: true
    ttl: 1h
    preload: true
    
  # 向量检索缓存
  vector_search_cache:
    enabled: true
    max_size: 5000
    ttl: 30m
```
