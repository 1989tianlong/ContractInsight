# 合同存储服务 (Contract Store Service)

## 1. 服务概述

合同存储服务是平台的数据持久化核心，负责：
- 合同文件存储（原件、转换件）
- 结构化元数据存储
- 多租户数据隔离
- 数据版本管理与审计

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          Contract Store Service                                  │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                         接入层 (API Layer)                                 │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │ │
│  │  │  REST    │ │  gRPC    │ │ GraphQL  │ │  Kafka   │ │ Internal │        │ │
│  │  │  API     │ │  API     │ │  API     │ │ Consumer │ │  SDK     │        │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                                      ▼                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                      业务逻辑层 (Business Logic)                           │ │
│  │                                                                            │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐           │ │
│  │  │  合同生命周期   │  │  版本管理       │  │  数据完整性     │           │ │
│  │  │  Lifecycle Mgr  │  │  Version Ctrl   │  │  Integrity Chk  │           │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘           │ │
│  │                                                                            │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐           │ │
│  │  │  租户隔离       │  │  访问控制       │  │  加密服务       │           │ │
│  │  │  Tenant Isolate │  │  Access Control │  │  Encryption Svc │           │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘           │ │
│  │                                                                            │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                          │
│                    ┌─────────────────┼─────────────────┐                       │
│                    ▼                 ▼                 ▼                       │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐   │
│  │   元数据存储          │ │   文件存储           │ │   缓存层             │   │
│  │   Metadata Store     │ │   File Store         │ │   Cache Layer        │   │
│  │                      │ │                      │ │                      │   │
│  │  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │   │
│  │  │  PostgreSQL    │  │ │  │  MinIO/OSS     │  │ │  │  Redis Cluster │  │   │
│  │  │  (主库)        │  │ │  │  (原件存储)    │  │ │  │  (热点数据)    │  │   │
│  │  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │   │
│  │                      │ │                      │ │                      │   │
│  │  ┌────────────────┐  │ │  ┌────────────────┐  │ │  ┌────────────────┐  │   │
│  │  │  Elasticsearch │  │ │  │  CDN           │  │ │  │  Local Cache   │  │   │
│  │  │  (全文索引)    │  │ │  │  (预览加速)    │  │ │  │  (Caffeine)    │  │   │
│  │  └────────────────┘  │ │  └────────────────┘  │ │  └────────────────┘  │   │
│  │                      │ │                      │ │                      │   │
│  └──────────────────────┘ └──────────────────────┘ └──────────────────────┘   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 数据模型设计

### 2.1 核心实体模型

```sql
-- ===================================================================
-- 合同主表 (分区表，按租户+时间分区)
-- ===================================================================
CREATE TABLE contracts (
    id BIGSERIAL,
    contract_id VARCHAR(32) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    domain_code VARCHAR(32) NOT NULL,
    
    -- 基础信息
    contract_no VARCHAR(128),
    contract_name VARCHAR(512) NOT NULL,
    contract_type VARCHAR(64),
    contract_subtype VARCHAR(64),
    
    -- 状态
    status VARCHAR(32) NOT NULL DEFAULT 'DRAFT',
    lifecycle_status VARCHAR(32) DEFAULT 'ACTIVE',
    
    -- 金额信息
    currency VARCHAR(8) DEFAULT 'CNY',
    amount DECIMAL(18, 2),
    amount_text VARCHAR(256),
    
    -- 日期信息
    sign_date DATE,
    effective_date DATE,
    expiry_date DATE,
    
    -- 摘要与标签
    summary TEXT,
    tags JSONB DEFAULT '[]',
    custom_fields JSONB DEFAULT '{}',
    
    -- 来源信息
    source_channel VARCHAR(32),
    source_provider VARCHAR(64),
    original_contract_id VARCHAR(128),
    
    -- 质量评估
    ocr_quality_score DECIMAL(5, 4),
    extraction_confidence DECIMAL(5, 4),
    
    -- 文件信息
    file_count INTEGER DEFAULT 0,
    total_pages INTEGER DEFAULT 0,
    
    -- 版本控制
    version INTEGER DEFAULT 1,
    is_latest BOOLEAN DEFAULT TRUE,
    parent_version_id BIGINT,
    
    -- 审计字段
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    PRIMARY KEY (id, tenant_id, created_at)
) PARTITION BY LIST (tenant_id);

-- 默认分区
CREATE TABLE contracts_default PARTITION OF contracts DEFAULT;

-- 大租户独立分区示例
CREATE TABLE contracts_tenant_001 PARTITION OF contracts 
    FOR VALUES IN ('TENANT_001');

-- ===================================================================
-- 合同当事人表
-- ===================================================================
CREATE TABLE contract_parties (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    party_role VARCHAR(16) NOT NULL,  -- PARTY_A, PARTY_B, PARTY_C, GUARANTOR
    party_name VARCHAR(256) NOT NULL,
    party_short_name VARCHAR(128),
    
    -- 证件信息
    id_type VARCHAR(32),  -- USCC(统一社会信用代码), ID_CARD, PASSPORT
    id_number VARCHAR(64),
    
    -- 联系信息
    legal_representative VARCHAR(64),
    contact_person VARCHAR(64),
    contact_phone VARCHAR(32),
    contact_email VARCHAR(128),
    address TEXT,
    
    -- 银行信息
    bank_name VARCHAR(128),
    bank_account VARCHAR(64),
    
    -- 主体识别
    entity_id VARCHAR(64),  -- 关联到企业主数据
    is_internal BOOLEAN DEFAULT FALSE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ===================================================================
-- 合同文件表
-- ===================================================================
CREATE TABLE contract_files (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    -- 文件信息
    file_id VARCHAR(64) NOT NULL UNIQUE,
    file_name VARCHAR(256) NOT NULL,
    file_type VARCHAR(16) NOT NULL,  -- ORIGINAL, CONVERTED, ATTACHMENT
    file_format VARCHAR(16) NOT NULL,  -- PDF, PNG, DOCX
    file_size BIGINT NOT NULL,
    page_count INTEGER,
    
    -- 存储信息
    storage_bucket VARCHAR(128) NOT NULL,
    storage_path VARCHAR(512) NOT NULL,
    storage_provider VARCHAR(32) DEFAULT 'MINIO',
    
    -- 加密信息
    is_encrypted BOOLEAN DEFAULT FALSE,
    encryption_key_id VARCHAR(64),
    
    -- 校验信息
    md5_hash VARCHAR(32),
    sha256_hash VARCHAR(64),
    
    -- 预览信息
    thumbnail_path VARCHAR(512),
    preview_url VARCHAR(1024),
    
    -- OCR状态
    ocr_status VARCHAR(32),
    ocr_result_path VARCHAR(512),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ===================================================================
-- 合同条款表
-- ===================================================================
CREATE TABLE contract_clauses (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    clause_type VARCHAR(64) NOT NULL,
    clause_code VARCHAR(32),
    clause_title VARCHAR(256),
    clause_content TEXT NOT NULL,
    clause_summary TEXT,
    
    -- 位置信息
    section_path VARCHAR(128),  -- 如 "1.2.3"
    start_position INTEGER,
    end_position INTEGER,
    page_numbers INTEGER[],
    
    -- 标准化
    is_standard BOOLEAN DEFAULT FALSE,
    deviation_from_standard TEXT,
    
    -- 风险标记
    risk_level VARCHAR(16),
    risk_notes TEXT,
    
    -- 向量存储
    embedding_id VARCHAR(64),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ===================================================================
-- 合同变更历史表
-- ===================================================================
CREATE TABLE contract_change_history (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    change_type VARCHAR(32) NOT NULL,  -- CREATE, UPDATE, STATUS_CHANGE, APPROVE, etc.
    change_description TEXT,
    
    -- 变更前后数据
    before_data JSONB,
    after_data JSONB,
    changed_fields TEXT[],
    
    -- 操作信息
    operator_id VARCHAR(64),
    operator_name VARCHAR(128),
    operator_ip VARCHAR(64),
    operation_time TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- 关联工作流
    workflow_instance_id VARCHAR(64),
    workflow_node_id VARCHAR(64)
);

-- ===================================================================
-- 索引设计
-- ===================================================================
-- 主表索引
CREATE INDEX idx_contracts_tenant_domain ON contracts(tenant_id, domain_code);
CREATE INDEX idx_contracts_status ON contracts(tenant_id, status);
CREATE INDEX idx_contracts_dates ON contracts(tenant_id, effective_date, expiry_date);
CREATE INDEX idx_contracts_type ON contracts(tenant_id, contract_type, contract_subtype);
CREATE INDEX idx_contracts_no ON contracts(tenant_id, contract_no);
CREATE INDEX idx_contracts_amount ON contracts(tenant_id, amount);
CREATE INDEX idx_contracts_created ON contracts(tenant_id, created_at DESC);

-- GIN索引 (JSONB字段)
CREATE INDEX idx_contracts_tags ON contracts USING GIN (tags);
CREATE INDEX idx_contracts_custom_fields ON contracts USING GIN (custom_fields);

-- 当事人索引
CREATE INDEX idx_parties_contract ON contract_parties(contract_id);
CREATE INDEX idx_parties_name ON contract_parties(tenant_id, party_name);
CREATE INDEX idx_parties_id ON contract_parties(tenant_id, id_type, id_number);
CREATE INDEX idx_parties_entity ON contract_parties(entity_id);

-- 文件索引
CREATE INDEX idx_files_contract ON contract_files(contract_id);
CREATE INDEX idx_files_hash ON contract_files(md5_hash);

-- 条款索引
CREATE INDEX idx_clauses_contract ON contract_clauses(contract_id);
CREATE INDEX idx_clauses_type ON contract_clauses(tenant_id, clause_type);

-- 变更历史索引
CREATE INDEX idx_history_contract ON contract_change_history(contract_id, operation_time DESC);
CREATE INDEX idx_history_operator ON contract_change_history(tenant_id, operator_id, operation_time DESC);
```

### 2.2 状态机设计

```
                          合同生命周期状态机
                          
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│    ┌────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐  │
│    │ DRAFT  │────▶│ PENDING  │────▶│ APPROVED │────▶│ EFFECTIVE│  │
│    │ 草稿   │     │ APPROVAL │     │ 已审批   │     │ 生效中   │  │
│    └────────┘     │ 待审批   │     └──────────┘     └──────────┘  │
│         │         └──────────┘           │               │        │
│         │              │                 │               │        │
│         │              ▼                 │               ▼        │
│         │         ┌──────────┐          │          ┌──────────┐  │
│         │         │ REJECTED │          │          │ EXPIRING │  │
│         │         │ 已驳回   │──────────┘          │ 即将到期 │  │
│         │         └──────────┘                     └──────────┘  │
│         │              │                                 │        │
│         ▼              ▼                                 ▼        │
│    ┌────────┐     ┌──────────┐                    ┌──────────┐  │
│    │CANCELLED│    │ REVOKED  │                    │ EXPIRED  │  │
│    │ 已取消  │    │ 已撤销   │                    │ 已到期   │  │
│    └────────┘     └──────────┘                    └──────────┘  │
│                                                         │        │
│                                                         ▼        │
│                        ┌──────────┐              ┌──────────┐   │
│                        │TERMINATED│◀─────────────│ RENEWED  │   │
│                        │ 已终止   │              │ 已续签   │   │
│                        └──────────┘              └──────────┘   │
│                              │                                   │
│                              ▼                                   │
│                        ┌──────────┐                             │
│                        │ ARCHIVED │                             │
│                        │ 已归档   │                             │
│                        └──────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

状态转换事件:
- DRAFT → PENDING_APPROVAL: submit_for_approval
- PENDING_APPROVAL → APPROVED: approve
- PENDING_APPROVAL → REJECTED: reject
- REJECTED → DRAFT: revise
- APPROVED → EFFECTIVE: activate (签约日期到达)
- EFFECTIVE → EXPIRING: approaching_expiry (到期前30天)
- EXPIRING → EXPIRED: expire
- EXPIRING → RENEWED: renew
- EFFECTIVE → TERMINATED: terminate
- * → CANCELLED: cancel
- * → REVOKED: revoke
- EXPIRED/TERMINATED → ARCHIVED: archive
```

---

## 3. 文件存储设计

### 3.1 存储架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         文件存储架构                                     │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                        存储路由层                                  │ │
│  │                                                                   │ │
│  │  根据租户配置选择存储策略:                                          │ │
│  │  ├─ 标准租户 → 共享 MinIO 集群                                     │ │
│  │  ├─ VIP租户 → 独立 MinIO 集群                                      │ │
│  │  └─ 合规租户 → 指定区域云存储                                       │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                │                                        │
│                                ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                        存储分层策略                                 │ │
│  │                                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                     热数据层 (Hot)                          │ │ │
│  │  │  存储: SSD / 高性能对象存储                                  │ │ │
│  │  │  保留: 最近6个月的活跃合同                                   │ │ │
│  │  │  访问: 高频读写，支持在线预览                                 │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                     温数据层 (Warm)                         │ │ │
│  │  │  存储: HDD / 标准对象存储                                    │ │ │
│  │  │  保留: 6个月-3年的合同                                       │ │ │
│  │  │  访问: 中频读取，按需加载                                     │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                    │ │
│  │                              ▼                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │                     冷数据层 (Cold)                         │ │ │
│  │  │  存储: 归档存储 / 冷存储                                     │ │ │
│  │  │  保留: 3年以上的已归档合同                                   │ │ │
│  │  │  访问: 低频读取，取回有延迟                                   │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  │                                                                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 文件路径设计

```
存储路径规范:
/{bucket}/{tenant_id}/{domain_code}/{year}/{month}/{contract_id}/{file_type}/{filename}

示例:
/contracts/TENANT_001/SALES/2024/11/CT202411260001234567/ORIGINAL/contract.pdf
/contracts/TENANT_001/SALES/2024/11/CT202411260001234567/CONVERTED/contract_optimized.pdf
/contracts/TENANT_001/SALES/2024/11/CT202411260001234567/THUMBNAIL/page_001.jpg
/contracts/TENANT_001/SALES/2024/11/CT202411260001234567/OCR/ocr_result.json
/contracts/TENANT_001/SALES/2024/11/CT202411260001234567/ATTACHMENT/appendix_01.pdf
```

### 3.3 文件处理流程

```python
class ContractFileService:
    def __init__(self, 
                 storage_client: ObjectStorageClient,
                 encryption_service: EncryptionService,
                 virus_scanner: VirusScanService):
        self.storage = storage_client
        self.encryption = encryption_service
        self.virus_scanner = virus_scanner
        
    async def store_contract_file(self, 
                                   contract_id: str,
                                   tenant_id: str,
                                   domain_code: str,
                                   file_content: bytes,
                                   file_name: str,
                                   file_type: FileType) -> ContractFile:
        # 1. 病毒扫描
        scan_result = await self.virus_scanner.scan(file_content)
        if not scan_result.is_clean:
            raise VirusDetectedException(scan_result.threat_name)
        
        # 2. 计算哈希
        md5_hash = hashlib.md5(file_content).hexdigest()
        sha256_hash = hashlib.sha256(file_content).hexdigest()
        
        # 3. 重复检测
        existing = await self.check_duplicate(tenant_id, md5_hash)
        if existing:
            return self._handle_duplicate(existing, contract_id)
        
        # 4. 加密处理 (根据租户配置)
        tenant_config = await self.get_tenant_config(tenant_id)
        if tenant_config.encryption_enabled:
            file_content, key_id = await self.encryption.encrypt(
                file_content, tenant_id
            )
            is_encrypted = True
        else:
            key_id = None
            is_encrypted = False
        
        # 5. 生成存储路径
        storage_path = self._generate_storage_path(
            tenant_id, domain_code, contract_id, file_type, file_name
        )
        
        # 6. 确定存储桶
        bucket = self._select_bucket(tenant_id, file_type)
        
        # 7. 上传文件
        await self.storage.put_object(
            bucket=bucket,
            key=storage_path,
            body=file_content,
            metadata={
                'contract_id': contract_id,
                'tenant_id': tenant_id,
                'md5': md5_hash,
                'encrypted': str(is_encrypted)
            }
        )
        
        # 8. 生成预览 (异步)
        if file_type == FileType.ORIGINAL:
            await self._schedule_preview_generation(
                contract_id, bucket, storage_path
            )
        
        # 9. 创建记录
        contract_file = ContractFile(
            file_id=generate_file_id(),
            contract_id=contract_id,
            tenant_id=tenant_id,
            file_name=file_name,
            file_type=file_type.value,
            file_format=self._detect_format(file_name),
            file_size=len(file_content),
            storage_bucket=bucket,
            storage_path=storage_path,
            is_encrypted=is_encrypted,
            encryption_key_id=key_id,
            md5_hash=md5_hash,
            sha256_hash=sha256_hash
        )
        
        await self.file_repository.save(contract_file)
        
        return contract_file
    
    async def get_file_download_url(self,
                                     file_id: str,
                                     tenant_id: str,
                                     expiry_seconds: int = 3600) -> str:
        """生成预签名下载URL"""
        file = await self.file_repository.get(file_id, tenant_id)
        if not file:
            raise FileNotFoundException(file_id)
        
        # 生成预签名URL
        url = await self.storage.presign_get_object(
            bucket=file.storage_bucket,
            key=file.storage_path,
            expires=expiry_seconds
        )
        
        return url
```

---

## 4. 缓存策略

### 4.1 多级缓存设计

```yaml
# 缓存配置
cache_config:
  # L1: 本地缓存 (Caffeine)
  local_cache:
    contract_metadata:
      max_size: 10000
      expire_after_write: 5m
      expire_after_access: 10m
    
    file_metadata:
      max_size: 50000
      expire_after_write: 10m
      
  # L2: 分布式缓存 (Redis)
  redis_cache:
    contract_detail:
      prefix: "contract:detail:"
      ttl: 30m
      serializer: json
      
    contract_list:
      prefix: "contract:list:"
      ttl: 5m
      
    file_presign_url:
      prefix: "file:url:"
      ttl: 55m  # 略小于URL过期时间
      
    hot_contracts:
      prefix: "contract:hot:"
      ttl: 1h
      type: sorted_set
      
  # 缓存穿透防护
  bloom_filter:
    contract_exists:
      expected_insertions: 10000000
      false_positive_rate: 0.001
```

### 4.2 缓存更新策略

```python
class ContractCacheService:
    def __init__(self, 
                 local_cache: LocalCache,
                 redis_cache: RedisCache,
                 bloom_filter: BloomFilter):
        self.local = local_cache
        self.redis = redis_cache
        self.bloom = bloom_filter
        
    async def get_contract(self, 
                           contract_id: str, 
                           tenant_id: str) -> Optional[Contract]:
        cache_key = f"{tenant_id}:{contract_id}"
        
        # 1. 布隆过滤器快速判断
        if not self.bloom.might_contain(cache_key):
            return None
            
        # 2. L1本地缓存
        contract = self.local.get(cache_key)
        if contract:
            return contract
            
        # 3. L2 Redis缓存
        contract = await self.redis.get(f"contract:detail:{cache_key}")
        if contract:
            self.local.put(cache_key, contract)
            return contract
            
        # 4. 数据库查询
        contract = await self.contract_repository.get(contract_id, tenant_id)
        if contract:
            # 回填缓存
            await self.redis.set(
                f"contract:detail:{cache_key}", 
                contract, 
                ttl=1800
            )
            self.local.put(cache_key, contract)
            
        return contract
    
    async def invalidate_contract(self, 
                                   contract_id: str, 
                                   tenant_id: str):
        """合同更新时清除缓存"""
        cache_key = f"{tenant_id}:{contract_id}"
        
        # 清除本地缓存
        self.local.invalidate(cache_key)
        
        # 清除Redis缓存
        await self.redis.delete(f"contract:detail:{cache_key}")
        
        # 清除相关列表缓存 (通过发布订阅通知其他节点)
        await self.redis.publish(
            "cache:invalidate",
            json.dumps({
                "type": "contract",
                "tenant_id": tenant_id,
                "contract_id": contract_id
            })
        )
```

---

## 5. 数据同步与一致性

### 5.1 事件发布

```python
class ContractEventPublisher:
    def __init__(self, kafka_producer: KafkaProducer):
        self.producer = kafka_producer
        
    async def publish_contract_created(self, contract: Contract):
        event = ContractEvent(
            event_id=generate_event_id(),
            event_type="contract.created",
            contract_id=contract.contract_id,
            tenant_id=contract.tenant_id,
            domain_code=contract.domain_code,
            timestamp=datetime.utcnow(),
            payload={
                "contract_no": contract.contract_no,
                "contract_name": contract.contract_name,
                "contract_type": contract.contract_type,
                "amount": str(contract.amount),
                "status": contract.status
            }
        )
        
        await self.producer.send(
            topic="contract.lifecycle",
            key=contract.contract_id,
            value=event.to_json(),
            headers={
                "tenant_id": contract.tenant_id,
                "event_type": "contract.created"
            }
        )
        
    async def publish_contract_updated(self, 
                                        contract: Contract, 
                                        changed_fields: List[str]):
        # 类似实现...
        pass
```

### 5.2 数据同步到ES

```python
class ContractSearchSyncService:
    """合同数据同步到Elasticsearch"""
    
    def __init__(self, 
                 es_client: AsyncElasticsearch,
                 contract_repo: ContractRepository):
        self.es = es_client
        self.repo = contract_repo
        
    async def sync_contract(self, contract_id: str, tenant_id: str):
        # 获取完整合同数据
        contract = await self.repo.get_full_contract(contract_id, tenant_id)
        if not contract:
            # 合同已删除，从ES中移除
            await self.es.delete(
                index=f"contracts_{tenant_id}",
                id=contract_id,
                ignore=[404]
            )
            return
            
        # 构建ES文档
        es_doc = self._build_es_document(contract)
        
        # 写入ES
        await self.es.index(
            index=f"contracts_{tenant_id}",
            id=contract_id,
            document=es_doc,
            refresh="wait_for"
        )
        
    def _build_es_document(self, contract: Contract) -> Dict:
        return {
            "contract_id": contract.contract_id,
            "contract_no": contract.contract_no,
            "contract_name": contract.contract_name,
            "contract_type": contract.contract_type,
            "domain_code": contract.domain_code,
            "status": contract.status,
            "amount": float(contract.amount) if contract.amount else None,
            "currency": contract.currency,
            "sign_date": contract.sign_date.isoformat() if contract.sign_date else None,
            "effective_date": contract.effective_date.isoformat() if contract.effective_date else None,
            "expiry_date": contract.expiry_date.isoformat() if contract.expiry_date else None,
            "parties": [
                {
                    "role": p.party_role,
                    "name": p.party_name,
                    "id_number": p.id_number
                }
                for p in contract.parties
            ],
            "tags": contract.tags,
            "summary": contract.summary,
            "full_text": contract.full_text,  # 全文检索字段
            "created_at": contract.created_at.isoformat(),
            "updated_at": contract.updated_at.isoformat()
        }
```

---

## 6. API设计

### 6.1 合同CRUD接口

```yaml
# OpenAPI 3.0
paths:
  /api/v1/contracts:
    post:
      summary: 创建合同
      operationId: createContract
      tags: [Contract Management]
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateContractRequest'
      responses:
        '201':
          description: 创建成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Contract'
                
    get:
      summary: 查询合同列表
      operationId: listContracts
      tags: [Contract Management]
      parameters:
        - name: domain_code
          in: query
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
        - name: contract_type
          in: query
          schema:
            type: string
        - name: party_name
          in: query
          schema:
            type: string
        - name: sign_date_from
          in: query
          schema:
            type: string
            format: date
        - name: sign_date_to
          in: query
          schema:
            type: string
            format: date
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: page_size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: 查询成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ContractListResponse'

  /api/v1/contracts/{contract_id}:
    get:
      summary: 获取合同详情
      operationId: getContract
      tags: [Contract Management]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: 获取成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ContractDetail'
                
    put:
      summary: 更新合同
      operationId: updateContract
      tags: [Contract Management]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateContractRequest'
      responses:
        '200':
          description: 更新成功
          
    delete:
      summary: 删除合同
      operationId: deleteContract
      tags: [Contract Management]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      responses:
        '204':
          description: 删除成功

  /api/v1/contracts/{contract_id}/files:
    post:
      summary: 上传合同文件
      operationId: uploadContractFile
      tags: [Contract Files]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                file:
                  type: string
                  format: binary
                file_type:
                  type: string
                  enum: [ORIGINAL, ATTACHMENT]
      responses:
        '201':
          description: 上传成功
          
    get:
      summary: 获取合同文件列表
      operationId: listContractFiles
      tags: [Contract Files]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: 获取成功

  /api/v1/contracts/{contract_id}/files/{file_id}/download:
    get:
      summary: 下载合同文件
      operationId: downloadContractFile
      tags: [Contract Files]
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
        - name: file_id
          in: path
          required: true
          schema:
            type: string
      responses:
        '302':
          description: 重定向到预签名URL
```
