# 合同接入网关 (Contract Ingestion Gateway)

## 1. 模块概述

合同接入网关是整个平台的入口，负责接收来自不同渠道的合同，进行初步处理后分发到下游服务。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Contract Ingestion Gateway                            │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         接入适配层 (Adapter Layer)                      │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │ │
│  │  │HTTP上传  │ │电子签回调│ │邮件解析  │ │批量导入  │ │SDK推送   │     │ │
│  │  │Multipart │ │Webhook   │ │IMAP/POP3 │ │CSV/Excel │ │gRPC/REST │     │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘     │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         预处理流水线 (Pre-processing Pipeline)          │ │
│  │                                                                        │ │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐            │ │
│  │  │文件校验 │───▶│格式转换 │───▶│病毒扫描 │───▶│去重检测 │            │ │
│  │  │(类型/大小)│    │(统一PDF)│    │(ClamAV) │    │(Hash)   │            │ │
│  │  └─────────┘    └─────────┘    └─────────┘    └─────────┘            │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         OCR & 质检层 (OCR & QC Layer)                   │ │
│  │                                                                        │ │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐              │ │
│  │  │   版面分析    │  │   OCR识别     │  │   质量评分    │              │ │
│  │  │  Layout Det   │  │  Text Recog   │  │  Quality Score│              │ │
│  │  │  ├─表格检测   │  │  ├─印刷体    │  │  ├─清晰度    │              │ │
│  │  │  ├─印章检测   │  │  ├─手写体    │  │  ├─完整性    │              │ │
│  │  │  └─签名检测   │  │  └─混合识别  │  │  └─置信度    │              │ │
│  │  └───────────────┘  └───────────────┘  └───────────────┘              │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      元数据标准化层 (Metadata Normalization)            │ │
│  │                                                                        │ │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐              │ │
│  │  │  字段映射     │  │  数据清洗     │  │  Schema验证   │              │ │
│  │  │  Field Map    │  │  Data Clean   │  │  Validation   │              │ │
│  │  └───────────────┘  └───────────────┘  └───────────────┘              │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      业务域路由层 (Domain Router)                       │ │
│  │                                                                        │ │
│  │  tenant_id + domain_code → 下游服务路由 → Kafka Topic                  │ │
│  │                                                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组件设计

### 2.1 文件接收服务 (File Receiver)

```yaml
# 支持的接入方式
adapters:
  - name: http-upload
    protocol: HTTP/HTTPS
    endpoint: /api/v1/contracts/upload
    max_file_size: 50MB
    allowed_types: [pdf, png, jpg, jpeg, tiff, doc, docx]
    
  - name: e-sign-webhook
    protocol: HTTP Webhook
    providers:
      - e-qianzhang    # e签宝
      - fadada         # 法大大
      - bestsign       # 上上签
    signature_verify: true
    
  - name: email-parser
    protocol: IMAP/POP3
    scan_attachments: true
    subject_pattern: "合同|协议|Contract"
    
  - name: batch-import
    protocol: SFTP/S3
    formats: [zip, tar.gz]
    manifest_file: manifest.json
```

### 2.2 OCR 处理引擎

```python
# OCR Pipeline 伪代码
class OCRPipeline:
    def __init__(self):
        self.layout_detector = LayoutDetector()      # 版面分析
        self.text_recognizer = TextRecognizer()      # 文字识别
        self.table_extractor = TableExtractor()      # 表格提取
        self.seal_detector = SealDetector()          # 印章检测
        self.signature_detector = SignatureDetector() # 签名检测
    
    def process(self, document: Document) -> OCRResult:
        # 1. 版面分析
        layout = self.layout_detector.detect(document)
        
        # 2. 区域分类处理
        results = []
        for region in layout.regions:
            if region.type == RegionType.TEXT:
                text = self.text_recognizer.recognize(region)
                results.append(TextBlock(text, region.bbox, region.confidence))
            elif region.type == RegionType.TABLE:
                table = self.table_extractor.extract(region)
                results.append(TableBlock(table, region.bbox))
            elif region.type == RegionType.SEAL:
                seal = self.seal_detector.detect(region)
                results.append(SealBlock(seal, region.bbox))
            elif region.type == RegionType.SIGNATURE:
                signature = self.signature_detector.detect(region)
                results.append(SignatureBlock(signature, region.bbox))
        
        # 3. 质量评估
        quality_score = self.assess_quality(document, results)
        
        return OCRResult(
            blocks=results,
            quality_score=quality_score,
            raw_text=self.merge_text(results)
        )
```

### 2.3 质检规则引擎

```yaml
# 质检规则配置
quality_rules:
  # 图像质量检测
  image_quality:
    - name: resolution_check
      min_dpi: 150
      action: warn
    - name: clarity_check
      blur_threshold: 100
      action: reject
    - name: contrast_check
      min_contrast: 0.3
      action: warn
      
  # OCR结果质检
  ocr_quality:
    - name: confidence_check
      min_avg_confidence: 0.85
      action: manual_review
    - name: char_error_rate
      max_cer: 0.05
      action: reprocess
      
  # 业务规则检测
  business_rules:
    - name: required_fields
      fields: [contract_no, party_a, party_b, sign_date]
      action: reject
    - name: seal_required
      min_seals: 2
      action: manual_review
    - name: signature_required
      min_signatures: 1
      action: warn
```

### 2.4 元数据标准化

```json
// 统一合同元数据Schema
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ContractMetadata",
  "type": "object",
  "required": ["contract_id", "tenant_id", "domain_code"],
  "properties": {
    "contract_id": {
      "type": "string",
      "description": "全局唯一合同ID",
      "pattern": "^CT[0-9]{18}$"
    },
    "tenant_id": {
      "type": "string",
      "description": "租户ID"
    },
    "domain_code": {
      "type": "string",
      "description": "业务域编码",
      "enum": ["SALES", "PURCHASE", "HR", "LEASE", "SERVICE"]
    },
    "source": {
      "type": "object",
      "properties": {
        "channel": {
          "type": "string",
          "enum": ["UPLOAD", "E_SIGN", "EMAIL", "BATCH", "API"]
        },
        "provider": { "type": "string" },
        "original_id": { "type": "string" },
        "received_at": { "type": "string", "format": "date-time" }
      }
    },
    "basic_info": {
      "type": "object",
      "properties": {
        "contract_no": { "type": "string" },
        "contract_name": { "type": "string" },
        "contract_type": { "type": "string" },
        "currency": { "type": "string" },
        "amount": { "type": "number" }
      }
    },
    "parties": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "role": { "type": "string", "enum": ["PARTY_A", "PARTY_B", "PARTY_C"] },
          "name": { "type": "string" },
          "id_type": { "type": "string" },
          "id_number": { "type": "string" },
          "contact": { "type": "string" }
        }
      }
    },
    "dates": {
      "type": "object",
      "properties": {
        "sign_date": { "type": "string", "format": "date" },
        "effective_date": { "type": "string", "format": "date" },
        "expiry_date": { "type": "string", "format": "date" }
      }
    },
    "ocr_result": {
      "type": "object",
      "properties": {
        "quality_score": { "type": "number", "minimum": 0, "maximum": 1 },
        "page_count": { "type": "integer" },
        "has_seal": { "type": "boolean" },
        "has_signature": { "type": "boolean" }
      }
    }
  }
}
```

---

## 3. 数据流转设计

### 3.1 异步处理流水线

```
┌─────────┐     ┌─────────────────────────────────────────────────────────┐
│  上传   │────▶│                    Kafka Topics                         │
│ Request │     │                                                         │
└─────────┘     │  ┌─────────────────────────────────────────────────┐   │
                │  │ contract.raw.uploaded                            │   │
                │  │   ├── partition by: tenant_id                    │   │
                │  │   └── retention: 7 days                          │   │
                │  └─────────────────────────────────────────────────┘   │
                │                        │                                │
                │                        ▼                                │
                │  ┌─────────────────────────────────────────────────┐   │
                │  │ contract.preprocessed                            │   │
                │  │   └── 格式转换、病毒扫描完成                      │   │
                │  └─────────────────────────────────────────────────┘   │
                │                        │                                │
                │                        ▼                                │
                │  ┌─────────────────────────────────────────────────┐   │
                │  │ contract.ocr.completed                           │   │
                │  │   └── OCR识别完成                                 │   │
                │  └─────────────────────────────────────────────────┘   │
                │                        │                                │
                │                        ▼                                │
                │  ┌─────────────────────────────────────────────────┐   │
                │  │ contract.normalized                              │   │
                │  │   └── 元数据标准化完成，准备进入核心服务          │   │
                │  └─────────────────────────────────────────────────┘   │
                │                                                         │
                └─────────────────────────────────────────────────────────┘
```

### 3.2 状态机设计

```
                    ┌───────────────────────────────────────────┐
                    │            合同接入状态机                   │
                    └───────────────────────────────────────────┘
                                        │
                                        ▼
                    ┌───────────────────────────────────────────┐
                    │               RECEIVED                     │
                    │              (文件已接收)                   │
                    └───────────────────────────────────────────┘
                                        │
                         ┌──────────────┼──────────────┐
                         ▼              ▼              ▼
            ┌─────────────────┐  ┌───────────┐  ┌─────────────────┐
            │   PREPROCESSING │  │  REJECTED │  │   DUPLICATED    │
            │    (预处理中)    │  │  (被拒绝)  │  │    (重复)       │
            └─────────────────┘  └───────────┘  └─────────────────┘
                         │
                         ▼
            ┌─────────────────┐
            │   OCR_PENDING   │
            │   (OCR排队中)    │
            └─────────────────┘
                         │
                         ▼
            ┌─────────────────┐
            │  OCR_PROCESSING │
            │   (OCR处理中)    │
            └─────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
    ┌───────────────┐ ┌─────────┐ ┌───────────────┐
    │ OCR_COMPLETED │ │OCR_FAILED│ │ MANUAL_REVIEW │
    │  (OCR完成)     │ │(OCR失败) │ │  (人工审核)    │
    └───────────────┘ └─────────┘ └───────────────┘
              │                           │
              ▼                           │
    ┌───────────────┐                    │
    │  NORMALIZING  │                    │
    │  (标准化中)    │◀───────────────────┘
    └───────────────┘
              │
              ▼
    ┌───────────────┐
    │   COMPLETED   │
    │   (处理完成)   │───────▶ 发送到下游服务
    └───────────────┘
```

---

## 4. 接口设计

### 4.1 合同上传接口

```yaml
# OpenAPI 3.0 Specification
openapi: 3.0.3
info:
  title: Contract Ingestion Gateway API
  version: 1.0.0

paths:
  /api/v1/contracts/upload:
    post:
      summary: 上传合同文件
      operationId: uploadContract
      tags:
        - Contract Ingestion
      security:
        - BearerAuth: []
      requestBody:
        content:
          multipart/form-data:
            schema:
              type: object
              required:
                - file
                - domain_code
              properties:
                file:
                  type: string
                  format: binary
                  description: 合同文件
                domain_code:
                  type: string
                  enum: [SALES, PURCHASE, HR, LEASE, SERVICE]
                  description: 业务域编码
                contract_type:
                  type: string
                  description: 合同类型
                extra_metadata:
                  type: string
                  format: json
                  description: 额外元数据(JSON格式)
                priority:
                  type: string
                  enum: [LOW, NORMAL, HIGH, URGENT]
                  default: NORMAL
      responses:
        '202':
          description: 上传成功，异步处理中
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UploadResponse'
        '400':
          description: 请求参数错误
        '413':
          description: 文件过大
          
  /api/v1/contracts/{contract_id}/status:
    get:
      summary: 查询处理状态
      operationId: getContractStatus
      parameters:
        - name: contract_id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: 状态查询成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ContractStatus'

components:
  schemas:
    UploadResponse:
      type: object
      properties:
        contract_id:
          type: string
          example: "CT202411260001234567"
        status:
          type: string
          example: "RECEIVED"
        estimated_time:
          type: integer
          description: 预估处理时间(秒)
        tracking_url:
          type: string
          
    ContractStatus:
      type: object
      properties:
        contract_id:
          type: string
        status:
          type: string
          enum: [RECEIVED, PREPROCESSING, OCR_PENDING, OCR_PROCESSING, 
                 OCR_COMPLETED, NORMALIZING, COMPLETED, REJECTED, FAILED]
        progress:
          type: integer
          minimum: 0
          maximum: 100
        quality_score:
          type: number
        issues:
          type: array
          items:
            type: object
            properties:
              code:
                type: string
              message:
                type: string
              severity:
                type: string
                enum: [INFO, WARN, ERROR]
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
```

---

## 5. 部署架构

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contract-ingestion-gateway
  namespace: contract-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ingestion-gateway
  template:
    spec:
      containers:
        - name: gateway
          image: contract-platform/ingestion-gateway:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          env:
            - name: KAFKA_BROKERS
              valueFrom:
                configMapKeyRef:
                  name: kafka-config
                  key: brokers
            - name: OCR_SERVICE_URL
              value: "http://ocr-service:8080"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
# HPA for auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ingestion-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: contract-ingestion-gateway
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: kafka_consumer_lag
        target:
          type: AverageValue
          averageValue: "1000"
```
