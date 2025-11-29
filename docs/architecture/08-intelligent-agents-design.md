# 智能合同管理平台智能体设计方案

## 1. 概述

本设计方案旨在为智能合同管理平台设计并实现三个核心智能体：**合同结构化存储智能体**、**合同使用查询智能体**和**审核合同智能体**。这些智能体将利用平台现有的架构、数据模型和AI能力，提供自动化、智能化的合同管理功能。

---

## 2. 合同结构化存储智能体 (Contract Structured Storage Agent)

### 2.1 目标与职责

合同结构化存储智能体负责接收合同文档（原始文件或已提取文本），将其内容进行标准化、结构化处理，并持久化存储。

**核心功能：**
- 合同类型配置（系统级/上下文级）
- 合同字段配置与管理
- 数据映射与转换
- 数据质量校验

### 2.2 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          合同结构化存储智能体架构                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────┐     ┌─────────────────┐     ┌──────────────────────┐          │
│  │ 合同文档上传  │────▶│  Ingestion      │────▶│  合同提取服务         │          │
│  │  /API       │     │  Gateway        │     │  (OCR/NLP/LLM)       │          │
│  └─────────────┘     └─────────────────┘     └──────────┬───────────┘          │
│                                                         │                       │
│                                                         ▼                       │
│                              ┌───────────────────────────────────────────┐      │
│                              │     合同结构化存储智能体 (Agent Core)       │      │
│                              ├───────────────────────────────────────────┤      │
│                              │                                           │      │
│  ┌─────────────────┐        │  ┌─────────────┐   ┌─────────────────┐   │      │
│  │ 合同类型配置服务  │◀───────│  │ 类型识别器   │   │  字段映射引擎    │   │      │
│  │                 │        │  └─────────────┘   └─────────────────┘   │      │
│  │ - 系统预定义类型 │        │                                           │      │
│  │ - 租户自定义类型 │        │  ┌─────────────┐   ┌─────────────────┐   │      │
│  │ - 类型继承关系   │        │  │ 数据转换器   │   │  质量校验器      │   │      │
│  └─────────────────┘        │  └─────────────┘   └─────────────────┘   │      │
│                              │                                           │      │
│  ┌─────────────────┐        └───────────────────────────────────────────┘      │
│  │ 合同字段配置服务  │                          │                                │
│  │                 │                          ▼                                │
│  │ - 标准字段定义   │        ┌───────────────────────────────────────────┐      │
│  │ - 扩展字段定义   │        │            合同存储服务                      │      │
│  │ - 字段校验规则   │        ├─────────────┬─────────────┬───────────────┤      │
│  │ - 字段数据类型   │        │ PostgreSQL  │Elasticsearch│   MinIO/OSS   │      │
│  └─────────────────┘        │ (元数据)    │ (全文索引)  │  (原始文件)   │      │
│                              └─────────────┴─────────────┴───────────────┘      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        合同结构化存储处理流程                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────┐                                                                   │
│   │  开始   │                                                                   │
│   └────┬────┘                                                                   │
│        │                                                                        │
│        ▼                                                                        │
│   ┌─────────────────┐                                                          │
│   │ 接收合同提取结果 │ ◀──── Kafka消息/API调用                                  │
│   └────────┬────────┘                                                          │
│            │                                                                    │
│            ▼                                                                    │
│   ┌─────────────────┐      ┌──────────────────┐                                │
│   │ 解析提取结果     │─────▶│ 识别合同类型      │                                │
│   └─────────────────┘      └────────┬─────────┘                                │
│                                     │                                          │
│            ┌────────────────────────┼────────────────────────┐                 │
│            │                        │                        │                 │
│            ▼                        ▼                        ▼                 │
│   ┌───────────────┐      ┌───────────────┐      ┌───────────────┐             │
│   │ 系统预定义类型 │      │ 租户自定义类型 │      │ 默认/未知类型  │             │
│   └───────┬───────┘      └───────┬───────┘      └───────┬───────┘             │
│           │                      │                      │                      │
│           └──────────────────────┼──────────────────────┘                      │
│                                  │                                             │
│                                  ▼                                             │
│                      ┌─────────────────────┐                                   │
│                      │ 加载合同类型配置     │                                   │
│                      └──────────┬──────────┘                                   │
│                                 │                                              │
│                                 ▼                                              │
│                      ┌─────────────────────┐                                   │
│                      │ 加载合同字段配置     │                                   │
│                      └──────────┬──────────┘                                   │
│                                 │                                              │
│                                 ▼                                              │
│                      ┌─────────────────────┐                                   │
│                      │ 数据映射与类型转换   │                                   │
│                      │                     │                                   │
│                      │ - 标准字段映射      │                                   │
│                      │ - 自定义字段处理    │                                   │
│                      │ - 数据类型转换      │                                   │
│                      └──────────┬──────────┘                                   │
│                                 │                                              │
│                                 ▼                                              │
│                      ┌─────────────────────┐                                   │
│                      │ 执行数据质量校验     │                                   │
│                      └──────────┬──────────┘                                   │
│                                 │                                              │
│              ┌──────────────────┴──────────────────┐                           │
│              │                                     │                           │
│              ▼                                     ▼                           │
│     ┌────────────────┐                   ┌────────────────┐                    │
│     │  校验通过      │                   │  校验失败      │                    │
│     └───────┬────────┘                   └───────┬────────┘                    │
│             │                                    │                             │
│             ▼                                    ▼                             │
│     ┌────────────────┐                   ┌────────────────┐                    │
│     │ 写入PostgreSQL │                   │ 记录错误日志   │                    │
│     │ (合同元数据)   │                   │ 发送告警通知   │                    │
│     └───────┬────────┘                   └────────────────┘                    │
│             │                                                                  │
│             ▼                                                                  │
│     ┌────────────────┐                                                         │
│     │ 更新ES全文索引 │                                                         │
│     └───────┬────────┘                                                         │
│             │                                                                  │
│             ▼                                                                  │
│     ┌────────────────┐                                                         │
│     │ 存储文件引用   │                                                         │
│     └───────┬────────┘                                                         │
│             │                                                                  │
│             ▼                                                                  │
│        ┌─────────┐                                                             │
│        │  结束   │                                                             │
│        └─────────┘                                                             │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 数据模型

#### 2.4.1 合同类型配置表 (contract_types)

```sql
-- 合同类型配置表
CREATE TABLE contract_types (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    type_code VARCHAR(64) NOT NULL,
    type_name VARCHAR(128) NOT NULL,
    parent_type_code VARCHAR(64),              -- 父类型编码，支持类型继承
    description TEXT,
    is_system_defined BOOLEAN DEFAULT FALSE,   -- 是否系统预定义
    is_active BOOLEAN DEFAULT TRUE,
    config JSONB DEFAULT '{}',                 -- 类型级别的配置
    -- 配置示例: {
    --   "required_fields": ["contract_no", "amount"],
    --   "default_workflow": "standard_approval",
    --   "auto_extract_enabled": true
    -- }
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, type_code)
);

-- 索引
CREATE INDEX idx_contract_types_tenant ON contract_types(tenant_id);
CREATE INDEX idx_contract_types_parent ON contract_types(parent_type_code);

COMMENT ON TABLE contract_types IS '合同类型配置表，支持系统预定义和租户自定义';
COMMENT ON COLUMN contract_types.is_system_defined IS '系统预定义类型（如：采购合同、销售合同）';
COMMENT ON COLUMN contract_types.parent_type_code IS '父类型编码，支持类型继承';
```

#### 2.4.2 合同字段配置表 (contract_fields)

```sql
-- 合同字段配置表
CREATE TABLE contract_fields (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    contract_type_code VARCHAR(64) NOT NULL,
    field_code VARCHAR(64) NOT NULL,
    field_name VARCHAR(128) NOT NULL,
    field_group VARCHAR(64),                   -- 字段分组（如：基本信息、财务信息）
    data_type VARCHAR(32) NOT NULL,            -- STRING, NUMBER, DATE, BOOLEAN, JSON, ARRAY
    is_required BOOLEAN DEFAULT FALSE,
    is_searchable BOOLEAN DEFAULT TRUE,
    is_filterable BOOLEAN DEFAULT TRUE,
    is_sortable BOOLEAN DEFAULT FALSE,
    is_system_field BOOLEAN DEFAULT FALSE,     -- 是否系统字段
    default_value JSONB,
    validation_rules JSONB,                    -- 校验规则
    -- 校验规则示例: {
    --   "minLength": 2,
    --   "maxLength": 100,
    --   "pattern": "^C[0-9]{8}$",
    --   "min": 0,
    --   "max": 999999999,
    --   "enum": ["DRAFT", "ACTIVE", "EXPIRED"]
    -- }
    display_config JSONB,                      -- 显示配置
    -- 显示配置示例: {
    --   "label": "合同编号",
    --   "placeholder": "请输入合同编号",
    --   "tooltip": "格式：C+8位数字",
    --   "width": 200,
    --   "hidden": false
    -- }
    extraction_config JSONB,                   -- 提取配置
    -- 提取配置示例: {
    --   "source_field": "contract_number",
    --   "transform": "UPPERCASE",
    --   "fallback_fields": ["ref_no", "document_id"]
    -- }
    display_order INTEGER DEFAULT 0,
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, contract_type_code, field_code),
    FOREIGN KEY (tenant_id, contract_type_code) REFERENCES contract_types(tenant_id, type_code)
);

-- 索引
CREATE INDEX idx_contract_fields_tenant_type ON contract_fields(tenant_id, contract_type_code);
CREATE INDEX idx_contract_fields_searchable ON contract_fields(tenant_id, is_searchable) WHERE is_searchable = TRUE;

COMMENT ON TABLE contract_fields IS '合同字段配置表，定义特定合同类型下的字段属性';
COMMENT ON COLUMN contract_fields.field_group IS '字段分组，用于界面展示分组';
COMMENT ON COLUMN contract_fields.extraction_config IS '字段提取配置，指导智能体如何从原始数据提取';
```

#### 2.4.3 字段映射规则表 (field_mapping_rules)

```sql
-- 字段映射规则表
CREATE TABLE field_mapping_rules (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    contract_type_code VARCHAR(64) NOT NULL,
    source_provider VARCHAR(64),               -- 数据来源提供商（如：OCR_SERVICE, LLM_EXTRACT）
    source_field_path VARCHAR(256) NOT NULL,   -- 源字段路径（支持JSONPath）
    target_field_code VARCHAR(64) NOT NULL,    -- 目标字段编码
    transform_type VARCHAR(32),                -- 转换类型
    transform_config JSONB,                    -- 转换配置
    priority INTEGER DEFAULT 0,                -- 优先级，数值越大优先级越高
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, contract_type_code, source_provider, source_field_path)
);

-- 转换类型枚举说明：
-- DIRECT: 直接映射
-- UPPERCASE/LOWERCASE: 大小写转换
-- DATE_FORMAT: 日期格式转换
-- NUMBER_FORMAT: 数值格式转换
-- LOOKUP: 查表转换
-- SCRIPT: 脚本转换

COMMENT ON TABLE field_mapping_rules IS '字段映射规则表，定义从提取结果到结构化存储的映射关系';
```

### 2.5 数据处理流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          数据处理流程详解                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 数据接收层                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  Kafka Topic: contract.extraction.completed                              │   │
│  │  消息格式:                                                                │   │
│  │  {                                                                       │   │
│  │    "contract_id": "C20241129001",                                        │   │
│  │    "tenant_id": "TENANT_001",                                            │   │
│  │    "extraction_result": {                                                │   │
│  │      "basic_info": {...},                                                │   │
│  │      "parties": [...],                                                   │   │
│  │      "financial": {...},                                                 │   │
│  │      "clauses": [...],                                                   │   │
│  │      "confidence_score": 0.95                                            │   │
│  │    },                                                                    │   │
│  │    "detected_type": "SALES_CONTRACT"                                     │   │
│  │  }                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  2. 类型识别与配置加载                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  SELECT * FROM contract_types                                            │   │
│  │  WHERE tenant_id = :tenant_id AND type_code = :detected_type;            │   │
│  │                                                                          │   │
│  │  SELECT * FROM contract_fields                                           │   │
│  │  WHERE tenant_id = :tenant_id AND contract_type_code = :type_code        │   │
│  │  ORDER BY display_order;                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  3. 数据映射引擎                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  for each field in contract_fields:                                      │   │
│  │    1. 查找映射规则 (field_mapping_rules)                                  │   │
│  │    2. 从提取结果中获取源值 (JSONPath解析)                                  │   │
│  │    3. 执行数据转换 (根据transform_type)                                   │   │
│  │    4. 写入目标字段                                                        │   │
│  │                                                                          │   │
│  │  自定义字段 → contracts.custom_fields (JSONB)                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  4. 数据质量校验                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  校验规则执行:                                                            │   │
│  │  - 必填字段检查: is_required = TRUE 的字段不能为空                        │   │
│  │  - 格式校验: 根据 validation_rules 中的 pattern 进行正则匹配              │   │
│  │  - 范围校验: min/max 数值范围检查                                         │   │
│  │  - 枚举校验: enum 值列表校验                                              │   │
│  │  - 关联校验: 生效日期 <= 到期日期                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  5. 持久化存储                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  PostgreSQL:                                                             │   │
│  │    INSERT INTO contracts (...) VALUES (...);                             │   │
│  │    INSERT INTO contract_parties (...) VALUES (...);                      │   │
│  │    INSERT INTO contract_clauses (...) VALUES (...);                      │   │
│  │                                                                          │   │
│  │  Elasticsearch:                                                          │   │
│  │    PUT /contracts/_doc/:contract_id { ... }                              │   │
│  │                                                                          │   │
│  │  Kafka (事件发布):                                                        │   │
│  │    Topic: contract.storage.completed                                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 合同使用查询 Agent (Contract Query Agent)

### 3.1 目标与职责

合同使用查询Agent负责提供灵活、高效的合同查询能力，支持多种查询维度（全文、结构化、语义），并严格控制数据访问权限。

**核心功能：**
- 数据集配置与管理
- 用户/角色权限配置
- 多引擎智能查询路由
- 数据脱敏与过滤

### 3.2 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          合同使用查询Agent架构                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────┐     ┌─────────────────┐                                       │
│  │ 用户/系统   │────▶│  API Gateway    │                                       │
│  └─────────────┘     └────────┬────────┘                                       │
│                               │                                                 │
│                               ▼                                                 │
│              ┌────────────────────────────────────────────────┐                │
│              │         合同使用查询Agent (Agent Core)          │                │
│              ├────────────────────────────────────────────────┤                │
│              │                                                │                │
│              │  ┌──────────────┐      ┌──────────────────┐   │                │
│              │  │ 查询解析器   │      │ 权限上下文管理器  │   │                │
│              │  └──────────────┘      └──────────────────┘   │                │
│              │                                                │                │
│              │  ┌──────────────┐      ┌──────────────────┐   │                │
│              │  │ 查询路由器   │      │ 结果处理器       │   │                │
│              │  │              │      │ - 字段过滤       │   │                │
│              │  │ - 全文检索   │      │ - 数据脱敏       │   │                │
│              │  │ - 结构化查询 │      │ - 结果聚合       │   │                │
│              │  │ - 语义搜索   │      └──────────────────┘   │                │
│              │  └──────────────┘                              │                │
│              │                                                │                │
│              └────────────────────────────────────────────────┘                │
│                        │              │              │                         │
│          ┌─────────────┼──────────────┼──────────────┼─────────────┐          │
│          │             │              │              │             │          │
│          ▼             ▼              ▼              ▼             ▼          │
│  ┌─────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐      │
│  │ 数据集      │ │权限服务   │ │ES        │ │PostgreSQL│ │ Milvus      │      │
│  │ 配置服务    │ │(RBAC/    │ │全文检索   │ │结构化    │ │ 向量搜索    │      │
│  │             │ │ ABAC)    │ │          │ │ 查询     │ │             │      │
│  └─────────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────────┘      │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          合同查询处理流程                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────┐                                                                   │
│   │  开始   │                                                                   │
│   └────┬────┘                                                                   │
│        │                                                                        │
│        ▼                                                                        │
│   ┌─────────────────┐                                                          │
│   │ 接收查询请求    │                                                          │
│   │ - 查询条件      │                                                          │
│   │ - 数据集标识    │                                                          │
│   │ - 分页/排序     │                                                          │
│   └────────┬────────┘                                                          │
│            │                                                                    │
│            ▼                                                                    │
│   ┌─────────────────┐                                                          │
│   │ 用户身份验证    │                                                          │
│   │ 获取权限上下文  │                                                          │
│   └────────┬────────┘                                                          │
│            │                                                                    │
│            ▼                                                                    │
│   ┌─────────────────┐      ┌──────────────────────────────┐                    │
│   │ 加载数据集配置  │─────▶│ datasets表                    │                    │
│   │                 │      │ - 预设查询条件                │                    │
│   │                 │      │ - 允许访问字段                │                    │
│   │                 │      │ - 数据脱敏规则                │                    │
│   └────────┬────────┘      └──────────────────────────────┘                    │
│            │                                                                    │
│            ▼                                                                    │
│   ┌─────────────────┐                                                          │
│   │ 权限校验        │                                                          │
│   │ - 数据集访问权限│                                                          │
│   │ - 租户隔离检查  │                                                          │
│   │ - 行级权限过滤  │                                                          │
│   └────────┬────────┘                                                          │
│            │                                                                    │
│     ┌──────┴──────┐                                                            │
│     │             │                                                            │
│     ▼             ▼                                                            │
│  ┌──────┐    ┌──────────────────┐                                              │
│  │ 拒绝 │    │ 通过             │                                              │
│  │ 访问 │    └────────┬─────────┘                                              │
│  └──────┘             │                                                        │
│                       ▼                                                        │
│            ┌─────────────────────┐                                             │
│            │ 查询条件重构        │                                             │
│            │ - 合并数据集条件    │                                             │
│            │ - 添加权限过滤条件  │                                             │
│            │ - 字段投影过滤      │                                             │
│            └──────────┬──────────┘                                             │
│                       │                                                        │
│                       ▼                                                        │
│            ┌─────────────────────┐                                             │
│            │ 智能路由选择        │                                             │
│            └──────────┬──────────┘                                             │
│                       │                                                        │
│      ┌────────────────┼────────────────┐                                       │
│      │                │                │                                       │
│      ▼                ▼                ▼                                       │
│  ┌────────┐     ┌──────────┐    ┌───────────┐                                 │
│  │ ES     │     │PostgreSQL│    │  Milvus   │                                 │
│  │ 全文   │     │ 结构化   │    │  语义     │                                 │
│  │ 检索   │     │ 查询     │    │  搜索     │                                 │
│  └───┬────┘     └────┬─────┘    └─────┬─────┘                                 │
│      │               │                │                                        │
│      └───────────────┼────────────────┘                                        │
│                      │                                                         │
│                      ▼                                                         │
│            ┌─────────────────────┐                                             │
│            │ 结果合并与排序      │                                             │
│            └──────────┬──────────┘                                             │
│                       │                                                        │
│                       ▼                                                        │
│            ┌─────────────────────┐                                             │
│            │ 数据脱敏处理        │                                             │
│            │ - 敏感字段掩码      │                                             │
│            │ - 金额取整          │                                             │
│            │ - 身份信息脱敏      │                                             │
│            └──────────┬──────────┘                                             │
│                       │                                                        │
│                       ▼                                                        │
│            ┌─────────────────────┐                                             │
│            │ 审计日志记录        │                                             │
│            └──────────┬──────────┘                                             │
│                       │                                                        │
│                       ▼                                                        │
│                  ┌─────────┐                                                   │
│                  │  结束   │                                                   │
│                  └─────────┘                                                   │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 数据模型

#### 3.4.1 数据集配置表 (datasets)

```sql
-- 数据集配置表
CREATE TABLE datasets (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    dataset_code VARCHAR(64) NOT NULL,
    dataset_name VARCHAR(128) NOT NULL,
    description TEXT,
    dataset_type VARCHAR(32) DEFAULT 'CUSTOM',  -- SYSTEM, CUSTOM
    query_template JSONB,                       -- 预设查询条件
    -- 查询模板示例: {
    --   "must": [
    --     {"term": {"status": "APPROVED"}},
    --     {"range": {"amount": {"gte": 10000}}}
    --   ],
    --   "filter": [
    --     {"term": {"contract_type": "SALES"}}
    --   ]
    -- }
    allowed_fields JSONB,                       -- 允许返回的字段列表
    -- 示例: ["contract_no", "contract_name", "amount", "status", "sign_date"]
    excluded_fields JSONB,                      -- 排除的字段列表
    -- 示例: ["internal_notes", "approval_comments"]
    field_aliases JSONB,                        -- 字段别名映射
    -- 示例: {"contract_no": "合同编号", "amount": "金额"}
    data_masking_rules JSONB,                   -- 数据脱敏规则
    -- 示例: {
    --   "party_contact_phone": "PHONE_MASK",
    --   "party_id_card": "ID_CARD_MASK",
    --   "amount": "ROUND_THOUSAND"
    -- }
    row_filter_expression TEXT,                 -- 行级过滤表达式
    -- 示例: "created_by = :current_user OR department_id IN (:user_departments)"
    max_results INTEGER DEFAULT 1000,           -- 最大返回记录数
    cache_ttl INTEGER DEFAULT 300,              -- 缓存有效期(秒)
    is_active BOOLEAN DEFAULT TRUE,
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, dataset_code)
);

-- 索引
CREATE INDEX idx_datasets_tenant ON datasets(tenant_id);
CREATE INDEX idx_datasets_active ON datasets(tenant_id, is_active) WHERE is_active = TRUE;

COMMENT ON TABLE datasets IS '数据集配置表，定义不同查询场景的数据范围和访问规则';
COMMENT ON COLUMN datasets.query_template IS 'ES DSL格式的预设查询条件';
COMMENT ON COLUMN datasets.row_filter_expression IS '行级过滤SQL表达式，支持变量替换';
```

#### 3.4.2 数据集权限表 (dataset_permissions)

```sql
-- 数据集权限表
CREATE TABLE dataset_permissions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    dataset_id BIGINT NOT NULL REFERENCES datasets(id),
    subject_type VARCHAR(32) NOT NULL,          -- USER, ROLE, DEPARTMENT
    subject_id VARCHAR(64) NOT NULL,            -- 用户ID/角色ID/部门ID
    permission_level VARCHAR(32) NOT NULL,      -- FULL, READ_ONLY, MASKED, NONE
    custom_masking_rules JSONB,                 -- 覆盖数据集默认脱敏规则
    effective_start_date DATE,
    effective_end_date DATE,
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, dataset_id, subject_type, subject_id)
);

-- 权限级别说明：
-- FULL: 完全访问，无脱敏
-- READ_ONLY: 只读访问，应用数据集脱敏规则
-- MASKED: 脱敏访问，应用更严格的脱敏规则
-- NONE: 无访问权限

-- 索引
CREATE INDEX idx_dataset_permissions_dataset ON dataset_permissions(dataset_id);
CREATE INDEX idx_dataset_permissions_subject ON dataset_permissions(subject_type, subject_id);

COMMENT ON TABLE dataset_permissions IS '数据集访问权限表';
```

#### 3.4.3 查询审计日志表 (query_audit_logs)

```sql
-- 查询审计日志表
CREATE TABLE query_audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    dataset_code VARCHAR(64),
    query_type VARCHAR(32),                     -- FULLTEXT, STRUCTURED, SEMANTIC
    query_params JSONB,                         -- 原始查询参数
    applied_filters JSONB,                      -- 实际应用的过滤条件
    result_count INTEGER,
    execution_time_ms INTEGER,
    client_ip VARCHAR(64),
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- 按月分区
CREATE TABLE query_audit_logs_2024_11 PARTITION OF query_audit_logs
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');

-- 索引
CREATE INDEX idx_query_audit_tenant_user ON query_audit_logs(tenant_id, user_id);
CREATE INDEX idx_query_audit_time ON query_audit_logs(created_at);

COMMENT ON TABLE query_audit_logs IS '查询操作审计日志';
```

### 3.5 数据处理流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          查询数据处理流程详解                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 请求解析与上下文构建                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  请求示例:                                                                │   │
│  │  POST /api/v1/contracts/search                                           │   │
│  │  {                                                                       │   │
│  │    "dataset": "my_sales_contracts",                                      │   │
│  │    "query": {                                                            │   │
│  │      "keyword": "采购设备",                                               │   │
│  │      "filters": {                                                        │   │
│  │        "amount_min": 100000,                                             │   │
│  │        "status": ["APPROVED", "ACTIVE"]                                  │   │
│  │      }                                                                   │   │
│  │    },                                                                    │   │
│  │    "page": 1,                                                            │   │
│  │    "size": 20                                                            │   │
│  │  }                                                                       │   │
│  │                                                                          │   │
│  │  权限上下文:                                                              │   │
│  │  {                                                                       │   │
│  │    "user_id": "U001",                                                    │   │
│  │    "tenant_id": "TENANT_001",                                            │   │
│  │    "roles": ["SALES_MANAGER"],                                           │   │
│  │    "departments": ["SALES_DEPT_001"]                                     │   │
│  │  }                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  2. 数据集配置加载与权限校验                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  -- 加载数据集配置                                                        │   │
│  │  SELECT d.*, dp.permission_level, dp.custom_masking_rules                │   │
│  │  FROM datasets d                                                         │   │
│  │  LEFT JOIN dataset_permissions dp ON d.id = dp.dataset_id                │   │
│  │  WHERE d.tenant_id = :tenant_id                                          │   │
│  │    AND d.dataset_code = :dataset_code                                    │   │
│  │    AND d.is_active = TRUE                                                │   │
│  │    AND (                                                                 │   │
│  │      (dp.subject_type = 'USER' AND dp.subject_id = :user_id) OR          │   │
│  │      (dp.subject_type = 'ROLE' AND dp.subject_id IN (:roles)) OR         │   │
│  │      (dp.subject_type = 'DEPARTMENT' AND dp.subject_id IN (:depts))      │   │
│  │    )                                                                     │   │
│  │    AND (dp.effective_start_date IS NULL OR dp.effective_start_date <= NOW()) │
│  │    AND (dp.effective_end_date IS NULL OR dp.effective_end_date >= NOW()) │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  3. 查询条件重构                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  最终查询条件 = 用户查询条件 ∩ 数据集预设条件 ∩ 权限过滤条件               │   │
│  │                                                                          │   │
│  │  ES DSL 示例:                                                             │   │
│  │  {                                                                       │   │
│  │    "bool": {                                                             │   │
│  │      "must": [                                                           │   │
│  │        {"match": {"content": "采购设备"}},      // 用户条件               │   │
│  │        {"term": {"contract_type": "SALES"}}     // 数据集预设             │   │
│  │      ],                                                                  │   │
│  │      "filter": [                                                         │   │
│  │        {"term": {"tenant_id": "TENANT_001"}},   // 租户隔离               │   │
│  │        {"range": {"amount": {"gte": 100000}}},  // 用户条件               │   │
│  │        {"terms": {"status": ["APPROVED", "ACTIVE"]}}, // 用户条件        │   │
│  │        {"terms": {"created_by": ["U001", ...]}} // 权限过滤               │   │
│  │      ]                                                                   │   │
│  │    }                                                                     │   │
│  │  }                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  4. 多引擎查询执行                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  路由策略:                                                                │   │
│  │  - 包含keyword → Elasticsearch 全文检索                                   │   │
│  │  - 纯结构化条件 → PostgreSQL 直接查询                                     │   │
│  │  - 语义相似查询 → Milvus 向量检索 + ES/PG 补充                            │   │
│  │  - 混合查询 → 多引擎并行 + 结果融合                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  5. 结果处理与脱敏                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  脱敏规则应用:                                                            │   │
│  │                                                                          │   │
│  │  PHONE_MASK:    138****1234                                              │   │
│  │  ID_CARD_MASK:  110101********1234                                       │   │
│  │  BANK_MASK:     6222****1234                                             │   │
│  │  NAME_MASK:     张**                                                      │   │
│  │  ROUND_THOUSAND: 125000 → 125000 (取整到千位)                            │   │
│  │                                                                          │   │
│  │  字段过滤:                                                                │   │
│  │  原始结果字段: [contract_no, name, amount, internal_notes, ...]          │   │
│  │  allowed_fields: [contract_no, name, amount]                             │   │
│  │  返回字段: [contract_no, name, amount]                                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 审核合同 Agent (Contract Audit Agent)

### 4.1 目标与职责

审核合同Agent负责自动化合同的审核流程，根据预定义的法务和财务规则对合同进行合规性、风险性及财务合理性评估。

**核心功能：**
- 法务审核规则配置与执行
- 财务审核规则配置与执行
- LLM智能风险分析
- 工作流集成与流程推动

### 4.2 架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           审核合同Agent架构                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌───────────────────┐     ┌──────────────────┐                                │
│  │ 合同存储完成事件   │     │ 工作流审核任务    │                                │
│  │ (Kafka)           │     │ 分配              │                                │
│  └─────────┬─────────┘     └────────┬─────────┘                                │
│            │                        │                                          │
│            └────────────┬───────────┘                                          │
│                         │                                                      │
│                         ▼                                                      │
│         ┌───────────────────────────────────────────────┐                      │
│         │           审核合同Agent (Agent Core)           │                      │
│         ├───────────────────────────────────────────────┤                      │
│         │                                               │                      │
│         │  ┌─────────────────┐  ┌─────────────────────┐ │                      │
│         │  │ 合同数据获取器   │  │ 审核任务调度器      │ │                      │
│         │  └─────────────────┘  └─────────────────────┘ │                      │
│         │                                               │                      │
│         │  ┌─────────────────┐  ┌─────────────────────┐ │                      │
│         │  │ LLM智能分析模块 │  │ 规则引擎执行器      │ │                      │
│         │  │                 │  │                     │ │                      │
│         │  │ - 风险识别      │  │ - 法务规则库        │ │                      │
│         │  │ - 条款分析      │  │ - 财务规则库        │ │                      │
│         │  │ - 摘要生成      │  │ - 自定义规则        │ │                      │
│         │  └─────────────────┘  └─────────────────────┘ │                      │
│         │                                               │                      │
│         │  ┌─────────────────┐  ┌─────────────────────┐ │                      │
│         │  │ 审核报告生成器  │  │ 工作流通知器        │ │                      │
│         │  └─────────────────┘  └─────────────────────┘ │                      │
│         │                                               │                      │
│         └───────────────────────────────────────────────┘                      │
│                         │                                                      │
│      ┌──────────────────┼─────────────────────────────────┐                   │
│      │                  │                  │              │                   │
│      ▼                  ▼                  ▼              ▼                   │
│ ┌──────────┐     ┌───────────┐     ┌───────────┐  ┌────────────┐             │
│ │ 合同存储  │     │ LLM服务   │     │ 工作流    │  │ 审计服务   │             │
│ │ 服务      │     │ (AI平台)  │     │ 引擎      │  │            │             │
│ └──────────┘     └───────────┘     └───────────┘  └────────────┘             │
│                                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐  │
│ │                         规则配置存储 (PostgreSQL)                        │  │
│ │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │  │
│ │  │ 法务规则库  │  │ 财务规则库  │  │ 风险评估库  │  │ 审核记录表      │ │  │
│ │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘ │  │
│ └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           合同审核处理流程                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────┐                                                                   │
│   │  开始   │                                                                   │
│   └────┬────┘                                                                   │
│        │                                                                        │
│        ▼                                                                        │
│   ┌─────────────────────────┐                                                  │
│   │ 接收审核触发事件/任务    │                                                  │
│   │ - Kafka: contract.stored│                                                  │
│   │ - 工作流任务分配        │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ 获取合同完整数据        │                                                  │
│   │ - 结构化元数据          │                                                  │
│   │ - 合同全文内容          │                                                  │
│   │ - 关联附件              │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ LLM智能分析             │                                                  │
│   │                         │                                                  │
│   │ ┌─────────────────────┐ │                                                  │
│   │ │ analyze_risk()      │ │ ──▶ 识别风险条款                                │
│   │ └─────────────────────┘ │                                                  │
│   │ ┌─────────────────────┐ │                                                  │
│   │ │ extract_entities()  │ │ ──▶ 提取关键实体                                │
│   │ └─────────────────────┘ │                                                  │
│   │ ┌─────────────────────┐ │                                                  │
│   │ │ generate_summary()  │ │ ──▶ 生成合同摘要                                │
│   │ └─────────────────────┘ │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐      │
│   │                        规则审核执行                                   │      │
│   │  ┌─────────────────────────┐    ┌─────────────────────────────┐    │      │
│   │  │      法务规则审核        │    │       财务规则审核           │    │      │
│   │  │                         │    │                             │    │      │
│   │  │  ├─ 合同主体资质检查    │    │  ├─ 金额阈值检查            │    │      │
│   │  │  ├─ 必要条款完整性      │    │  ├─ 付款条件合理性          │    │      │
│   │  │  ├─ 违约责任对等性      │    │  ├─ 预算匹配检查            │    │      │
│   │  │  ├─ 争议解决条款        │    │  ├─ 税务合规检查            │    │      │
│   │  │  ├─ 知识产权条款        │    │  ├─ 账期风险评估            │    │      │
│   │  │  └─ 保密条款检查        │    │  └─ 汇率风险检查            │    │      │
│   │  └─────────────────────────┘    └─────────────────────────────┘    │      │
│   └───────────────────────────────────────────────────────────────────────┘      │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ 汇总审核结果            │                                                  │
│   │                         │                                                  │
│   │ - 计算综合风险等级      │                                                  │
│   │ - 汇总所有风险点        │                                                  │
│   │ - 生成处理建议          │                                                  │
│   │ - 生成审核报告          │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ 持久化审核结果          │                                                  │
│   │                         │                                                  │
│   │ INSERT INTO             │                                                  │
│   │ contract_audit_records  │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ 更新合同风险状态        │                                                  │
│   │                         │                                                  │
│   │ UPDATE contracts SET    │                                                  │
│   │ risk_level = :level     │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│   ┌─────────────────────────┐                                                  │
│   │ 通知工作流引擎          │                                                  │
│   │                         │                                                  │
│   │ 根据审核结果:           │                                                  │
│   │ - 通过 → 进入下一环节   │                                                  │
│   │ - 需复审 → 转人工审核   │                                                  │
│   │ - 驳回 → 退回修改       │                                                  │
│   └───────────┬─────────────┘                                                  │
│               │                                                                 │
│               ▼                                                                 │
│          ┌─────────┐                                                           │
│          │  结束   │                                                           │
│          └─────────┘                                                           │
│                                                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 数据模型

#### 4.4.1 审核规则表 (audit_rules)

```sql
-- 审核规则表
CREATE TABLE audit_rules (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    rule_code VARCHAR(64) NOT NULL,
    rule_name VARCHAR(128) NOT NULL,
    rule_type VARCHAR(32) NOT NULL,             -- LEGAL(法务), FINANCIAL(财务), COMPLIANCE(合规)
    rule_category VARCHAR(64),                  -- 规则分类（如：主体资质、条款完整性）
    contract_type_codes JSONB,                  -- 适用的合同类型列表，空表示适用所有类型
    priority INTEGER DEFAULT 0,                 -- 执行优先级
    rule_expression TEXT NOT NULL,              -- 规则表达式
    rule_params JSONB,                          -- 规则参数
    -- 规则参数示例: {
    --   "amount_threshold": 1000000,
    --   "required_clauses": ["违约责任", "争议解决"],
    --   "forbidden_terms": ["无条件免责"]
    -- }
    risk_level VARCHAR(32) DEFAULT 'MEDIUM',    -- HIGH, MEDIUM, LOW
    risk_score INTEGER DEFAULT 50,              -- 风险评分 (0-100)
    suggestion TEXT,                            -- 触发时的处理建议
    auto_action VARCHAR(32),                    -- 自动动作: PASS, REJECT, MANUAL_REVIEW, WARN
    is_blocking BOOLEAN DEFAULT FALSE,          -- 是否为阻断性规则
    is_active BOOLEAN DEFAULT TRUE,
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(64),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE (tenant_id, rule_code)
);

-- 索引
CREATE INDEX idx_audit_rules_tenant_type ON audit_rules(tenant_id, rule_type);
CREATE INDEX idx_audit_rules_active ON audit_rules(tenant_id, is_active) WHERE is_active = TRUE;

COMMENT ON TABLE audit_rules IS '合同审核规则配置表';
COMMENT ON COLUMN audit_rules.rule_expression IS '规则判断表达式，支持SQL/JSONPath/Python表达式';
COMMENT ON COLUMN audit_rules.is_blocking IS '阻断性规则触发时将阻止流程继续';
```

#### 4.4.2 法务规则示例数据

```sql
-- 法务规则示例
INSERT INTO audit_rules (tenant_id, rule_code, rule_name, rule_type, rule_category, 
    rule_expression, rule_params, risk_level, suggestion, auto_action, is_blocking) 
VALUES
-- 主体资质检查
('TENANT_001', 'LEGAL_001', '甲方签章完整性检查', 'LEGAL', '主体资质',
 'contract.party_a_seal IS NOT NULL AND contract.party_a_signature IS NOT NULL',
 '{}', 'HIGH', '请确保甲方签章和签字完整', 'MANUAL_REVIEW', TRUE),

-- 必要条款检查
('TENANT_001', 'LEGAL_002', '争议解决条款检查', 'LEGAL', '条款完整性',
 'contract.clauses EXISTS (clause_type = ''DISPUTE_RESOLUTION'')',
 '{"required_content": ["仲裁", "诉讼"]}', 'MEDIUM', '建议添加争议解决条款', 'WARN', FALSE),

-- 违约责任对等性
('TENANT_001', 'LEGAL_003', '违约责任对等性检查', 'LEGAL', '条款公平性',
 'LLM_ANALYSIS.risk_points NOT CONTAINS ''违约责任不对等''',
 '{}', 'HIGH', '发现违约责任条款可能存在不对等风险，请人工复核', 'MANUAL_REVIEW', FALSE),

-- 保密条款
('TENANT_001', 'LEGAL_004', '保密条款检查', 'LEGAL', '条款完整性',
 'contract.amount > :threshold OR contract.clauses EXISTS (clause_type = ''CONFIDENTIALITY'')',
 '{"threshold": 100000}', 'LOW', '金额超过阈值的合同建议添加保密条款', 'WARN', FALSE);
```

#### 4.4.3 财务规则示例数据

```sql
-- 财务规则示例
INSERT INTO audit_rules (tenant_id, rule_code, rule_name, rule_type, rule_category,
    rule_expression, rule_params, risk_level, suggestion, auto_action, is_blocking)
VALUES
-- 金额阈值检查
('TENANT_001', 'FIN_001', '大额合同审批', 'FINANCIAL', '金额控制',
 'contract.amount <= :max_amount',
 '{"max_amount": 10000000}', 'HIGH', '超过1000万元的合同需要CEO审批', 'MANUAL_REVIEW', TRUE),

-- 付款条件检查
('TENANT_001', 'FIN_002', '预付款比例检查', 'FINANCIAL', '付款条件',
 'contract.prepayment_ratio IS NULL OR contract.prepayment_ratio <= :max_ratio',
 '{"max_ratio": 0.3}', 'MEDIUM', '预付款比例超过30%，请评估资金风险', 'WARN', FALSE),

-- 账期风险
('TENANT_001', 'FIN_003', '收款账期检查', 'FINANCIAL', '账期风险',
 'contract.payment_days <= :max_days',
 '{"max_days": 90}', 'MEDIUM', '收款账期超过90天，建议缩短账期或增加担保', 'WARN', FALSE),

-- 预算匹配
('TENANT_001', 'FIN_004', '预算匹配检查', 'FINANCIAL', '预算控制',
 'budget.remaining >= contract.amount',
 '{}', 'HIGH', '合同金额超出剩余预算，需申请追加预算', 'MANUAL_REVIEW', TRUE);
```

#### 4.4.4 合同审核记录表 (contract_audit_records)

```sql
-- 合同审核记录表
CREATE TABLE contract_audit_records (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    contract_id VARCHAR(32) NOT NULL,
    audit_batch_id VARCHAR(64) NOT NULL,        -- 审核批次ID
    audit_type VARCHAR(32) NOT NULL,            -- AUTO(自动), MANUAL(人工)
    audit_stage VARCHAR(64),                    -- 审核阶段（如：初审、复审）
    auditor VARCHAR(64),                        -- 审核人（智能体ID或用户ID）
    audit_status VARCHAR(32) NOT NULL,          -- PASSED, FAILED, PENDING_REVIEW, REJECTED
    overall_risk_level VARCHAR(32),             -- HIGH, MEDIUM, LOW, NONE
    overall_risk_score INTEGER,                 -- 综合风险评分 (0-100)
    llm_analysis_result JSONB,                  -- LLM分析结果
    -- 示例: {
    --   "risk_points": ["违约责任不明确", "付款条件模糊"],
    --   "summary": "本合同为标准采购合同...",
    --   "key_entities": {"party_a": "xxx公司", "amount": 500000}
    -- }
    rule_execution_results JSONB,               -- 规则执行结果详情
    -- 示例: [
    --   {"rule_code": "LEGAL_001", "passed": true, "message": null},
    --   {"rule_code": "FIN_001", "passed": false, "message": "金额超过阈值", "risk_level": "HIGH"}
    -- ]
    audit_opinion TEXT,                         -- 审核意见
    suggestions JSONB,                          -- 处理建议列表
    workflow_instance_id VARCHAR(64),           -- 关联的工作流实例ID
    workflow_task_id VARCHAR(64),               -- 关联的工作流任务ID
    audit_start_time TIMESTAMP WITH TIME ZONE,
    audit_end_time TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_audit_records_contract ON contract_audit_records(tenant_id, contract_id);
CREATE INDEX idx_audit_records_batch ON contract_audit_records(audit_batch_id);
CREATE INDEX idx_audit_records_status ON contract_audit_records(audit_status);
CREATE INDEX idx_audit_records_workflow ON contract_audit_records(workflow_instance_id);

COMMENT ON TABLE contract_audit_records IS '合同审核记录表，记录每次自动化或人工审核的详细结果';
```

### 4.5 数据处理流程

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          审核数据处理流程详解                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 审核触发                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  触发来源:                                                                │   │
│  │  A. Kafka事件: contract.storage.completed                                │   │
│  │     {"contract_id": "C001", "tenant_id": "T001", "event_type": "CREATED"}│   │
│  │                                                                          │   │
│  │  B. 工作流任务: 审批流程中的"自动审核"节点                                 │   │
│  │     {"task_id": "TASK_001", "contract_id": "C001", "stage": "LEGAL_REVIEW"}│  │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  2. 合同数据获取                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  -- 获取合同主表信息                                                      │   │
│  │  SELECT * FROM contracts WHERE contract_id = :contract_id;               │   │
│  │                                                                          │   │
│  │  -- 获取合同当事人                                                        │   │
│  │  SELECT * FROM contract_parties WHERE contract_id = :contract_id;        │   │
│  │                                                                          │   │
│  │  -- 获取合同条款                                                          │   │
│  │  SELECT * FROM contract_clauses WHERE contract_id = :contract_id;        │   │
│  │                                                                          │   │
│  │  -- 获取合同全文 (从ES或文件存储)                                         │   │
│  │  GET /contracts/_doc/:contract_id?_source=full_text                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  3. LLM智能分析                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  async def perform_llm_analysis(contract_text: str) -> dict:             │   │
│  │      # 风险分析                                                           │   │
│  │      risk_result = await llm_service.analyze_risk(contract_text)         │   │
│  │      # 返回: {"risk_points": [...], "risk_level": "MEDIUM"}              │   │
│  │                                                                          │   │
│  │      # 实体提取                                                           │   │
│  │      entities = await llm_service.extract_entities(contract_text)        │   │
│  │      # 返回: {"party_a": "xxx", "amount": 500000, ...}                   │   │
│  │                                                                          │   │
│  │      # 摘要生成                                                           │   │
│  │      summary = await llm_service.generate_summary(contract_text)         │   │
│  │      # 返回: "本合同为采购合同，甲方为...，金额为..."                      │   │
│  │                                                                          │   │
│  │      return {                                                            │   │
│  │          "risk_analysis": risk_result,                                   │   │
│  │          "entities": entities,                                           │   │
│  │          "summary": summary                                              │   │
│  │      }                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  4. 规则引擎执行                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  -- 加载适用规则                                                          │   │
│  │  SELECT * FROM audit_rules                                               │   │
│  │  WHERE tenant_id = :tenant_id                                            │   │
│  │    AND is_active = TRUE                                                  │   │
│  │    AND (contract_type_codes IS NULL                                      │   │
│  │         OR contract_type_codes @> :contract_type::jsonb)                 │   │
│  │  ORDER BY priority DESC;                                                 │   │
│  │                                                                          │   │
│  │  规则执行伪代码:                                                          │   │
│  │  results = []                                                            │   │
│  │  for rule in rules:                                                      │   │
│  │      context = {                                                         │   │
│  │          "contract": contract_data,                                      │   │
│  │          "LLM_ANALYSIS": llm_result,                                     │   │
│  │          "budget": budget_data,                                          │   │
│  │          **rule.rule_params                                              │   │
│  │      }                                                                   │   │
│  │      passed = evaluate_expression(rule.rule_expression, context)         │   │
│  │      results.append({                                                    │   │
│  │          "rule_code": rule.rule_code,                                    │   │
│  │          "rule_name": rule.rule_name,                                    │   │
│  │          "rule_type": rule.rule_type,                                    │   │
│  │          "passed": passed,                                               │   │
│  │          "risk_level": rule.risk_level if not passed else None,          │   │
│  │          "suggestion": rule.suggestion if not passed else None,          │   │
│  │          "auto_action": rule.auto_action if not passed else None         │   │
│  │      })                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  5. 结果汇总与决策                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  风险等级计算逻辑:                                                        │   │
│  │                                                                          │   │
│  │  if any(r.is_blocking and not r.passed for r in results):               │   │
│  │      overall_status = "REJECTED"                                         │   │
│  │      overall_risk_level = "HIGH"                                         │   │
│  │  elif any(r.auto_action == "MANUAL_REVIEW" and not r.passed):           │   │
│  │      overall_status = "PENDING_REVIEW"                                   │   │
│  │      overall_risk_level = max(failed_rules.risk_level)                   │   │
│  │  elif all(r.passed for r in results):                                   │   │
│  │      overall_status = "PASSED"                                           │   │
│  │      overall_risk_level = "NONE"                                         │   │
│  │  else:                                                                   │   │
│  │      overall_status = "PASSED"  # 带警告通过                              │   │
│  │      overall_risk_level = max(failed_rules.risk_level)                   │   │
│  │                                                                          │   │
│  │  风险评分计算:                                                            │   │
│  │  overall_risk_score = sum(r.risk_score for r in failed_rules) / count    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│  6. 结果持久化与流程推进                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  -- 保存审核记录                                                          │   │
│  │  INSERT INTO contract_audit_records (                                    │   │
│  │      tenant_id, contract_id, audit_batch_id, audit_type,                 │   │
│  │      auditor, audit_status, overall_risk_level, overall_risk_score,      │   │
│  │      llm_analysis_result, rule_execution_results, audit_opinion,         │   │
│  │      suggestions, workflow_instance_id, ...                              │   │
│  │  ) VALUES (...);                                                         │   │
│  │                                                                          │   │
│  │  -- 更新合同风险状态                                                      │   │
│  │  UPDATE contracts SET                                                    │   │
│  │      risk_level = :overall_risk_level,                                   │   │
│  │      last_audit_time = NOW(),                                            │   │
│  │      last_audit_status = :overall_status                                 │   │
│  │  WHERE contract_id = :contract_id;                                       │   │
│  │                                                                          │   │
│  │  -- 通知工作流引擎                                                        │   │
│  │  workflow_service.complete_task(                                         │   │
│  │      task_id = :workflow_task_id,                                        │   │
│  │      result = overall_status,                                            │   │
│  │      variables = {                                                       │   │
│  │          "risk_level": overall_risk_level,                               │   │
│  │          "audit_record_id": audit_record_id,                             │   │
│  │          "next_action": determine_next_action(overall_status)            │   │
│  │      }                                                                   │   │
│  │  )                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 智能体协作关系图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          三大智能体协作关系                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                           ┌─────────────────┐                                  │
│                           │   合同文档上传   │                                  │
│                           └────────┬────────┘                                  │
│                                    │                                           │
│                                    ▼                                           │
│                           ┌─────────────────┐                                  │
│                           │   合同提取服务   │                                  │
│                           └────────┬────────┘                                  │
│                                    │                                           │
│                                    ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    合同结构化存储智能体                                    │   │
│  │                                                                         │   │
│  │   输入: 提取结果 (JSON)                                                  │   │
│  │   处理: 类型识别 → 字段映射 → 数据校验 → 持久化存储                       │   │
│  │   输出: 结构化合同数据 (PostgreSQL + ES + MinIO)                         │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                          ┌─────────┴─────────┐                                │
│                          │                   │                                │
│            ┌─────────────▼──────┐   ┌───────▼────────────────┐               │
│            │  Kafka事件发布      │   │  工作流触发            │               │
│            │  contract.stored    │   │  合同审批流程启动       │               │
│            └─────────────┬──────┘   └───────┬────────────────┘               │
│                          │                   │                                │
│                          └─────────┬─────────┘                                │
│                                    │                                           │
│                                    ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                       审核合同智能体                                      │   │
│  │                                                                         │   │
│  │   输入: contract_id, 工作流任务                                          │   │
│  │   处理: 数据获取 → LLM分析 → 规则执行 → 结果汇总                         │   │
│  │   输出: 审核记录, 风险等级, 工作流推进                                    │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                           │
│                                    ▼                                           │
│                           ┌─────────────────┐                                  │
│                           │  合同状态更新    │                                  │
│                           │  (审批通过/驳回) │                                  │
│                           └────────┬────────┘                                  │
│                                    │                                           │
│                                    ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    合同使用查询智能体                                      │   │
│  │                                                                         │   │
│  │   输入: 用户查询请求 + 权限上下文                                         │   │
│  │   处理: 权限校验 → 数据集过滤 → 多引擎查询 → 结果脱敏                     │   │
│  │   输出: 脱敏后的查询结果                                                  │   │
│  │                                                                         │   │
│  │   ┌───────────────────────────────────────────────────────────────────┐ │   │
│  │   │  支持的查询场景:                                                   │ │   │
│  │   │  • 业务人员: 查询自己创建/参与的合同                               │ │   │
│  │   │  • 法务人员: 查询待审核/高风险合同                                 │ │   │
│  │   │  • 财务人员: 查询待付款/已到期合同                                 │ │   │
│  │   │  • 管理层: 查询各部门合同统计/趋势分析                             │ │   │
│  │   └───────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 总结

本设计方案详细阐述了智能合同管理平台的三个核心智能体：

| 智能体 | 核心功能 | 关键配置 | 集成服务 |
|--------|----------|----------|----------|
| **合同结构化存储智能体** | 合同数据标准化与持久化 | 合同类型配置、字段配置、映射规则 | 提取服务、存储服务、ES、MinIO |
| **合同使用查询智能体** | 多维度智能查询与权限控制 | 数据集配置、权限配置、脱敏规则 | 权限服务、ES、PostgreSQL、Milvus |
| **审核合同智能体** | 自动化合规审核与风险评估 | 法务规则、财务规则、LLM分析 | LLM服务、工作流引擎、审计服务 |

三个智能体通过事件驱动机制紧密协作，形成完整的合同处理闭环：**存储** → **审核** → **查询**，共同构建高效、智能、安全的合同管理系统。
