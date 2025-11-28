# 合同提取服务 (Contract Extract Service)

## 1. 服务概述

合同提取服务负责从OCR识别结果中智能提取结构化信息，包括：
- 合同要素抽取（甲乙方、金额、日期等）
- 关键条款识别（付款条款、违约条款等）
- 合同分类与模板匹配
- 风险点识别与预警

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Contract Extract Service                              │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         输入层 (Input Layer)                           │ │
│  │                                                                        │ │
│  │  Kafka Consumer: contract.ocr.completed                                │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  OCR Result + Raw Text + Layout Info + Quality Score             │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      文档理解层 (Document Understanding)               │ │
│  │                                                                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │ │
│  │  │  段落分割   │  │  章节识别   │  │  条款定位   │  │  表格解析   │  │ │
│  │  │ Paragraph   │  │  Section    │  │  Clause     │  │  Table      │  │ │
│  │  │ Segmenter   │  │  Detector   │  │  Locator    │  │  Parser     │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      信息抽取层 (Information Extraction)               │ │
│  │                                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │                   NER + RE Pipeline                              │  │ │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │  │ │
│  │  │  │ 实体识别  │  │ 关系抽取  │  │ 事件抽取  │  │ 属性抽取  │    │  │ │
│  │  │  │   NER     │  │    RE     │  │   Event   │  │ Attribute │    │  │ │
│  │  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │ │
│  │  │                   LLM Extraction Engine                          │  │ │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │  │ │
│  │  │  │ 条款理解  │  │ 要素提取  │  │ 智能问答  │  │ 摘要生成  │    │  │ │
│  │  │  │ Clause    │  │ Element   │  │   Q&A     │  │  Summary  │    │  │ │
│  │  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │  │ │
│  │  └─────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      业务处理层 (Business Logic)                       │ │
│  │                                                                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │ │
│  │  │  合同分类   │  │  模板匹配   │  │  风险识别   │  │  合规检查   │  │ │
│  │  │ Classifier  │  │  Template   │  │    Risk     │  │ Compliance  │  │ │
│  │  │             │  │  Matcher    │  │  Detector   │  │  Checker    │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         输出层 (Output Layer)                          │ │
│  │                                                                        │ │
│  │  Kafka Producer: contract.extracted                                    │ │
│  │  ┌──────────────────────────────────────────────────────────────────┐ │ │
│  │  │  Structured Contract Data + Clauses + Risk Alerts                │ │ │
│  │  └──────────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心能力设计

### 2.1 实体抽取 (Named Entity Recognition)

```python
# 合同领域实体定义
class ContractEntityType(Enum):
    # 主体类
    PARTY_A = "甲方"
    PARTY_B = "乙方"
    PARTY_C = "丙方"
    GUARANTOR = "担保方"
    LEGAL_REP = "法定代表人"
    AGENT = "代理人"
    
    # 金额类
    TOTAL_AMOUNT = "合同总金额"
    UNIT_PRICE = "单价"
    DEPOSIT = "定金/押金"
    PENALTY = "违约金"
    
    # 日期类
    SIGN_DATE = "签订日期"
    EFFECTIVE_DATE = "生效日期"
    EXPIRY_DATE = "到期日期"
    DELIVERY_DATE = "交付日期"
    PAYMENT_DATE = "付款日期"
    
    # 标识类
    CONTRACT_NO = "合同编号"
    PROJECT_NAME = "项目名称"
    SUBJECT_MATTER = "标的物"
    
    # 条款类
    PAYMENT_TERM = "付款条款"
    DELIVERY_TERM = "交付条款"
    WARRANTY_TERM = "保修条款"
    PENALTY_TERM = "违约条款"
    DISPUTE_TERM = "争议解决"
    CONFIDENTIAL_TERM = "保密条款"
    TERMINATION_TERM = "解除条款"


# 抽取结果数据结构
@dataclass
class ExtractedEntity:
    entity_type: ContractEntityType
    text: str                      # 原文
    normalized_value: Any          # 标准化值
    confidence: float              # 置信度
    position: TextPosition         # 文档位置
    context: str                   # 上下文
    source: str                    # 抽取来源 (NER/LLM/RULE)


@dataclass
class ExtractionResult:
    contract_id: str
    entities: List[ExtractedEntity]
    clauses: List[ContractClause]
    relations: List[EntityRelation]
    summary: str
    risk_alerts: List[RiskAlert]
    classification: ContractClassification
    extraction_metadata: Dict
```

### 2.2 LLM 抽取引擎

```python
# LLM Prompt Template for Contract Extraction
CONTRACT_EXTRACTION_PROMPT = """
你是一个专业的合同分析助手。请从以下合同文本中提取关键信息。

## 合同文本
{contract_text}

## 提取要求
请严格按照JSON格式输出以下信息：

```json
{
  "basic_info": {
    "contract_no": "合同编号",
    "contract_name": "合同名称",
    "contract_type": "合同类型(买卖/租赁/服务/劳动/借款/其他)"
  },
  "parties": [
    {
      "role": "甲方/乙方/丙方",
      "name": "全称",
      "id_type": "统一社会信用代码/身份证/其他",
      "id_number": "证件号码",
      "legal_representative": "法定代表人",
      "address": "地址",
      "contact": "联系方式"
    }
  ],
  "financial": {
    "total_amount": {
      "value": 数值,
      "currency": "CNY/USD/EUR",
      "text": "原文描述"
    },
    "payment_method": "付款方式",
    "payment_schedule": [
      {
        "phase": "阶段",
        "amount": 数值,
        "condition": "付款条件",
        "due_date": "日期"
      }
    ]
  },
  "dates": {
    "sign_date": "YYYY-MM-DD",
    "effective_date": "YYYY-MM-DD",
    "expiry_date": "YYYY-MM-DD",
    "auto_renewal": true/false
  },
  "key_clauses": {
    "subject_matter": "标的物/服务内容描述",
    "delivery_terms": "交付条款",
    "warranty_terms": "保修/质保条款",
    "penalty_clause": "违约条款",
    "dispute_resolution": "争议解决方式",
    "confidentiality": "保密条款"
  },
  "risk_points": [
    {
      "type": "风险类型",
      "description": "风险描述",
      "clause_reference": "相关条款",
      "severity": "HIGH/MEDIUM/LOW"
    }
  ]
}
```

注意事项：
1. 如果某字段无法从文本中提取，请填写null
2. 金额请统一转换为数值，保留2位小数
3. 日期请统一为YYYY-MM-DD格式
4. 如发现明显风险点，请在risk_points中标注
"""


class LLMExtractionEngine:
    def __init__(self, model_config: ModelConfig):
        self.llm_client = self._init_llm_client(model_config)
        self.prompt_template = CONTRACT_EXTRACTION_PROMPT
        
    async def extract(self, contract_text: str, 
                      contract_type: Optional[str] = None) -> Dict:
        # 文本预处理
        processed_text = self._preprocess(contract_text)
        
        # 长文本分块处理
        if len(processed_text) > self.max_context_length:
            return await self._extract_long_document(processed_text)
        
        # 构建prompt
        prompt = self.prompt_template.format(contract_text=processed_text)
        
        # 调用LLM
        response = await self.llm_client.generate(
            prompt=prompt,
            temperature=0.1,  # 低温度保证输出稳定性
            max_tokens=4096,
            response_format={"type": "json_object"}
        )
        
        # 解析并验证结果
        result = self._parse_response(response)
        validated_result = self._validate_extraction(result)
        
        return validated_result
    
    async def _extract_long_document(self, text: str) -> Dict:
        """长文档分块提取后合并"""
        chunks = self._split_into_chunks(text, overlap=200)
        
        chunk_results = await asyncio.gather(*[
            self._extract_chunk(chunk, i) 
            for i, chunk in enumerate(chunks)
        ])
        
        # 合并多个chunk的提取结果
        merged_result = self._merge_results(chunk_results)
        
        return merged_result
```

### 2.3 合同分类器

```python
# 多级合同分类体系
CONTRACT_TAXONOMY = {
    "SALES": {  # 买卖合同
        "GOODS_SALE": "商品买卖",
        "EQUIPMENT_SALE": "设备采购",
        "RAW_MATERIAL": "原材料采购",
        "IMPORT_EXPORT": "进出口贸易"
    },
    "SERVICE": {  # 服务合同
        "IT_SERVICE": "IT服务",
        "CONSULTING": "咨询服务",
        "OUTSOURCING": "外包服务",
        "MAINTENANCE": "运维服务"
    },
    "LEASE": {  # 租赁合同
        "PROPERTY_LEASE": "房屋租赁",
        "EQUIPMENT_LEASE": "设备租赁",
        "VEHICLE_LEASE": "车辆租赁"
    },
    "HR": {  # 人事合同
        "EMPLOYMENT": "劳动合同",
        "DISPATCH": "劳务派遣",
        "INTERNSHIP": "实习协议",
        "CONFIDENTIALITY": "保密协议",
        "NON_COMPETE": "竞业禁止"
    },
    "FINANCIAL": {  # 金融合同
        "LOAN": "借款合同",
        "GUARANTEE": "担保合同",
        "PLEDGE": "质押合同"
    },
    "IP": {  # 知识产权
        "LICENSE": "许可协议",
        "TECHNOLOGY_TRANSFER": "技术转让",
        "COOPERATION_DEVELOPMENT": "合作开发"
    }
}


class ContractClassifier:
    def __init__(self):
        self.text_classifier = self._load_text_classifier()
        self.keyword_rules = self._load_keyword_rules()
        
    def classify(self, contract_text: str, 
                 extracted_entities: List[ExtractedEntity]) -> ContractClassification:
        # 1. 基于关键词规则的分类
        keyword_result = self._keyword_classify(contract_text)
        
        # 2. 基于ML模型的分类
        ml_result = self.text_classifier.predict(contract_text)
        
        # 3. 基于提取实体的分类
        entity_result = self._entity_based_classify(extracted_entities)
        
        # 4. 融合多个分类结果
        final_result = self._ensemble_classify(
            keyword_result, ml_result, entity_result
        )
        
        return ContractClassification(
            primary_category=final_result.primary,
            secondary_category=final_result.secondary,
            confidence=final_result.confidence,
            alternative_categories=final_result.alternatives
        )
```

### 2.4 风险识别引擎

```yaml
# 风险规则配置
risk_detection_rules:
  # 金额风险
  financial_risks:
    - name: large_amount_no_guarantee
      condition: "amount > 1000000 AND guarantee IS NULL"
      severity: HIGH
      message: "大额合同缺少担保条款"
      
    - name: unclear_payment_schedule
      condition: "payment_schedule IS NULL OR payment_schedule.length == 0"
      severity: MEDIUM
      message: "付款计划不明确"
      
    - name: no_penalty_clause
      condition: "penalty_clause IS NULL"
      severity: MEDIUM
      message: "缺少违约金条款"
      
  # 期限风险
  term_risks:
    - name: long_term_no_review
      condition: "term_years > 3 AND review_clause IS NULL"
      severity: MEDIUM
      message: "长期合同缺少定期审查条款"
      
    - name: auto_renewal_risk
      condition: "auto_renewal == true AND termination_notice_days < 30"
      severity: HIGH
      message: "自动续约条款通知期过短"
      
  # 法律风险
  legal_risks:
    - name: unclear_dispute_resolution
      condition: "dispute_resolution IS NULL"
      severity: HIGH
      message: "缺少争议解决条款"
      
    - name: unfavorable_jurisdiction
      condition: "jurisdiction NOT IN company_preferred_jurisdictions"
      severity: MEDIUM
      message: "管辖法院不利于我方"
      
    - name: unlimited_liability
      condition: "liability_cap IS NULL"
      severity: HIGH
      message: "缺少责任上限条款"
      
  # 合规风险
  compliance_risks:
    - name: missing_confidentiality
      condition: "confidentiality_clause IS NULL AND contract_type IN ['IT_SERVICE', 'CONSULTING']"
      severity: HIGH
      message: "服务类合同缺少保密条款"
      
    - name: data_protection_missing
      condition: "involves_personal_data AND data_protection_clause IS NULL"
      severity: CRITICAL
      message: "涉及个人数据但缺少数据保护条款"
```

```python
class RiskDetectionEngine:
    def __init__(self, rules_config: str):
        self.rules = self._load_rules(rules_config)
        self.llm_risk_analyzer = LLMRiskAnalyzer()
        
    async def detect_risks(self, 
                           extraction_result: ExtractionResult) -> List[RiskAlert]:
        risks = []
        
        # 1. 规则引擎检测
        rule_risks = self._apply_rules(extraction_result)
        risks.extend(rule_risks)
        
        # 2. LLM 深度分析
        llm_risks = await self.llm_risk_analyzer.analyze(
            extraction_result.clauses,
            extraction_result.entities
        )
        risks.extend(llm_risks)
        
        # 3. 去重和优先级排序
        unique_risks = self._deduplicate_risks(risks)
        sorted_risks = self._sort_by_severity(unique_risks)
        
        return sorted_risks
```

---

## 3. 模板匹配系统

### 3.1 模板库设计

```python
@dataclass
class ContractTemplate:
    template_id: str
    template_name: str
    category: str
    version: str
    
    # 结构定义
    sections: List[TemplateSection]
    required_clauses: List[str]
    optional_clauses: List[str]
    
    # 字段定义
    field_schema: Dict[str, FieldDefinition]
    
    # 匹配特征
    keywords: List[str]
    patterns: List[str]
    embedding: np.ndarray  # 向量表示


class TemplateMatchingService:
    def __init__(self, template_store: TemplateStore):
        self.template_store = template_store
        self.embedding_model = self._load_embedding_model()
        
    async def match_template(self, 
                             contract_text: str,
                             contract_type: Optional[str] = None) -> TemplateMatchResult:
        # 1. 获取候选模板
        candidates = await self.template_store.get_candidates(contract_type)
        
        # 2. 向量相似度匹配
        contract_embedding = self.embedding_model.encode(contract_text)
        similarity_scores = self._calculate_similarities(
            contract_embedding, 
            [t.embedding for t in candidates]
        )
        
        # 3. 结构相似度匹配
        contract_structure = self._extract_structure(contract_text)
        structure_scores = [
            self._structure_similarity(contract_structure, t.sections)
            for t in candidates
        ]
        
        # 4. 综合评分
        final_scores = [
            0.6 * sim + 0.4 * struct 
            for sim, struct in zip(similarity_scores, structure_scores)
        ]
        
        best_idx = np.argmax(final_scores)
        best_template = candidates[best_idx]
        
        # 5. 差异分析
        differences = self._analyze_differences(
            contract_text, contract_structure, best_template
        )
        
        return TemplateMatchResult(
            matched_template=best_template,
            confidence=final_scores[best_idx],
            differences=differences,
            missing_clauses=self._find_missing_clauses(
                contract_structure, best_template
            ),
            extra_clauses=self._find_extra_clauses(
                contract_structure, best_template
            )
        )
```

---

## 4. 数据模型

### 4.1 提取结果存储模型

```sql
-- 合同提取结果主表
CREATE TABLE contract_extraction_results (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL UNIQUE,
    tenant_id VARCHAR(32) NOT NULL,
    domain_code VARCHAR(32) NOT NULL,
    
    -- 分类信息
    primary_category VARCHAR(64),
    secondary_category VARCHAR(64),
    classification_confidence DECIMAL(5,4),
    
    -- 模板匹配
    matched_template_id VARCHAR(64),
    template_match_score DECIMAL(5,4),
    
    -- 基础信息 (JSONB)
    basic_info JSONB,
    parties JSONB,
    financial_info JSONB,
    dates_info JSONB,
    
    -- 提取元数据
    extraction_model VARCHAR(64),
    extraction_version VARCHAR(32),
    extraction_duration_ms INTEGER,
    
    -- 审计字段
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 实体提取结果表
CREATE TABLE extracted_entities (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    
    entity_type VARCHAR(64) NOT NULL,
    entity_text TEXT NOT NULL,
    normalized_value JSONB,
    confidence DECIMAL(5,4),
    
    -- 位置信息
    page_number INTEGER,
    bbox JSONB,  -- {x1, y1, x2, y2}
    context_text TEXT,
    
    -- 来源
    extraction_source VARCHAR(32),  -- NER/LLM/RULE
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    FOREIGN KEY (contract_id) REFERENCES contract_extraction_results(contract_id)
);

-- 条款提取结果表
CREATE TABLE extracted_clauses (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    
    clause_type VARCHAR(64) NOT NULL,
    clause_title VARCHAR(256),
    clause_content TEXT NOT NULL,
    clause_summary TEXT,
    
    -- 位置信息
    section_number VARCHAR(32),
    start_page INTEGER,
    end_page INTEGER,
    
    -- 风险标记
    has_risk BOOLEAN DEFAULT FALSE,
    risk_level VARCHAR(16),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    FOREIGN KEY (contract_id) REFERENCES contract_extraction_results(contract_id)
);

-- 风险预警表
CREATE TABLE contract_risk_alerts (
    id BIGSERIAL PRIMARY KEY,
    contract_id VARCHAR(32) NOT NULL,
    
    risk_type VARCHAR(64) NOT NULL,
    risk_category VARCHAR(64),
    severity VARCHAR(16) NOT NULL,  -- CRITICAL/HIGH/MEDIUM/LOW
    
    description TEXT NOT NULL,
    recommendation TEXT,
    related_clause_id BIGINT,
    
    -- 状态
    status VARCHAR(32) DEFAULT 'OPEN',  -- OPEN/ACKNOWLEDGED/RESOLVED/IGNORED
    resolved_by VARCHAR(64),
    resolved_at TIMESTAMP WITH TIME ZONE,
    resolution_note TEXT,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    FOREIGN KEY (contract_id) REFERENCES contract_extraction_results(contract_id),
    FOREIGN KEY (related_clause_id) REFERENCES extracted_clauses(id)
);

-- 索引
CREATE INDEX idx_extraction_tenant_domain ON contract_extraction_results(tenant_id, domain_code);
CREATE INDEX idx_extraction_category ON contract_extraction_results(primary_category, secondary_category);
CREATE INDEX idx_entities_contract ON extracted_entities(contract_id);
CREATE INDEX idx_entities_type ON extracted_entities(entity_type);
CREATE INDEX idx_clauses_contract ON extracted_clauses(contract_id);
CREATE INDEX idx_clauses_type ON extracted_clauses(clause_type);
CREATE INDEX idx_risks_contract ON contract_risk_alerts(contract_id);
CREATE INDEX idx_risks_severity ON contract_risk_alerts(severity, status);
```

---

## 5. 服务接口

### 5.1 内部 gRPC 接口

```protobuf
syntax = "proto3";

package contract.extract.v1;

service ContractExtractService {
    // 同步提取（小文档）
    rpc Extract(ExtractRequest) returns (ExtractResponse);
    
    // 异步提取（大文档）
    rpc ExtractAsync(ExtractAsyncRequest) returns (ExtractAsyncResponse);
    
    // 查询提取结果
    rpc GetExtractionResult(GetExtractionResultRequest) returns (ExtractionResult);
    
    // 重新提取
    rpc ReExtract(ReExtractRequest) returns (ExtractAsyncResponse);
    
    // 人工修正
    rpc CorrectExtraction(CorrectExtractionRequest) returns (CorrectExtractionResponse);
}

message ExtractRequest {
    string contract_id = 1;
    string tenant_id = 2;
    string domain_code = 3;
    bytes ocr_result = 4;  // OCR结果
    string raw_text = 5;
    ExtractOptions options = 6;
}

message ExtractOptions {
    bool enable_llm = 1;
    bool enable_risk_detection = 2;
    bool enable_template_matching = 3;
    string preferred_template_id = 4;
    repeated string required_fields = 5;
}

message ExtractResponse {
    string contract_id = 1;
    ExtractionResult result = 2;
    ExtractMetadata metadata = 3;
}

message ExtractionResult {
    BasicInfo basic_info = 1;
    repeated Party parties = 2;
    FinancialInfo financial_info = 3;
    DatesInfo dates_info = 4;
    repeated ExtractedEntity entities = 5;
    repeated ExtractedClause clauses = 6;
    repeated RiskAlert risk_alerts = 7;
    Classification classification = 8;
    TemplateMatch template_match = 9;
}

message ExtractedEntity {
    string entity_type = 1;
    string text = 2;
    string normalized_value = 3;
    float confidence = 4;
    Position position = 5;
    string context = 6;
    string source = 7;
}

message RiskAlert {
    string risk_id = 1;
    string risk_type = 2;
    string category = 3;
    string severity = 4;
    string description = 5;
    string recommendation = 6;
    string related_clause_id = 7;
}
```
