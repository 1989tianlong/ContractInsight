# 旁路服务层 (Sidecar Services)

## 1. 概述

旁路服务层提供横切关注点的统一支撑能力，包括：
- 权限服务 (RBAC/ABAC)
- 审计服务 (Audit Logging)
- 工作流引擎 (Workflow Engine)
- 多租户/业务域隔离 (Tenant Isolation)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            旁路服务层架构                                         │
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                          API Gateway / Service Mesh                        │  │
│  │                                                                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │  │
│  │  │ AuthN Filter│  │ AuthZ Filter│  │Audit Filter │  │Tenant Filter│     │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘     │  │
│  │                                                                           │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                          │
│          ┌───────────────────────────┼───────────────────────────┐             │
│          ▼                           ▼                           ▼             │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐       │
│  │    权限服务         │  │    审计服务         │  │    工作流引擎       │       │
│  │  Permission Svc    │  │    Audit Svc       │  │   Workflow Engine  │       │
│  │                    │  │                    │  │                    │       │
│  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │       │
│  │  │ RBAC Engine  │  │  │  │ Event Logger │  │  │  │ BPMN Runtime │  │       │
│  │  │ 角色权限管理  │  │  │  │ 事件日志记录  │  │  │  │ 流程运行时   │  │       │
│  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │       │
│  │                    │  │                    │  │                    │       │
│  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │       │
│  │  │ ABAC Engine  │  │  │  │ Query Service│  │  │  │ State Machine│  │       │
│  │  │ 属性权限控制  │  │  │  │ 审计日志查询  │  │  │  │ 状态机引擎   │  │       │
│  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │       │
│  │                    │  │                    │  │                    │       │
│  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │       │
│  │  │ Data Masking │  │  │  │ Compliance   │  │  │  │ Task Manager │  │       │
│  │  │ 数据脱敏     │  │  │  │ 合规报告     │  │  │  │ 任务管理     │  │       │
│  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │       │
│  │                    │  │                    │  │                    │       │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘       │
│                                      │                                          │
│                                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                        多租户隔离服务 (Tenant Isolation)                   │  │
│  │                                                                           │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐           │  │
│  │  │  租户上下文管理  │  │  数据隔离策略    │  │  资源配额管理    │           │  │
│  │  │ Tenant Context  │  │ Data Isolation  │  │ Resource Quota  │           │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘           │  │
│  │                                                                           │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 权限服务 (Permission Service)

### 2.1 权限模型设计

```
                    RBAC + ABAC 混合权限模型
                    
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│  │    用户      │────▶│    角色      │────▶│    权限      │              │
│  │   User       │     │    Role      │     │ Permission  │              │
│  └─────────────┘     └─────────────┘     └─────────────┘              │
│        │                   │                    │                       │
│        │                   │                    ▼                       │
│        │                   │           ┌─────────────┐                 │
│        │                   │           │   资源       │                 │
│        │                   │           │  Resource   │                 │
│        │                   │           └─────────────┘                 │
│        │                   │                    │                       │
│        ▼                   ▼                    ▼                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                       ABAC 属性策略层                            │  │
│  │                                                                 │  │
│  │  Subject Attributes    Resource Attributes    Context Attributes│  │
│  │  ├─ tenant_id          ├─ domain_code         ├─ time           │  │
│  │  ├─ department         ├─ contract_type       ├─ ip_address     │  │
│  │  ├─ job_level          ├─ amount              ├─ device_type    │  │
│  │  └─ data_scope         ├─ status              └─ location       │  │
│  │                        └─ sensitivity_level                      │  │
│  │                                                                 │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 权限数据模型

```sql
-- ===================================================================
-- 角色表
-- ===================================================================
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    
    role_code VARCHAR(64) NOT NULL,
    role_name VARCHAR(128) NOT NULL,
    description TEXT,
    
    -- 角色类型
    role_type VARCHAR(32) NOT NULL DEFAULT 'CUSTOM',  -- SYSTEM/CUSTOM
    
    -- 层级
    parent_role_id BIGINT,
    role_level INTEGER DEFAULT 0,
    
    -- 状态
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(tenant_id, role_code)
);

-- ===================================================================
-- 权限表
-- ===================================================================
CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY,
    
    permission_code VARCHAR(128) NOT NULL UNIQUE,
    permission_name VARCHAR(256) NOT NULL,
    description TEXT,
    
    -- 资源类型
    resource_type VARCHAR(64) NOT NULL,  -- CONTRACT/FILE/WORKFLOW/REPORT
    
    -- 操作类型
    action VARCHAR(32) NOT NULL,  -- CREATE/READ/UPDATE/DELETE/APPROVE/EXPORT
    
    -- 模块
    module VARCHAR(64),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 预置权限
INSERT INTO permissions (permission_code, permission_name, resource_type, action, module) VALUES
('contract:create', '创建合同', 'CONTRACT', 'CREATE', 'contract'),
('contract:read', '查看合同', 'CONTRACT', 'READ', 'contract'),
('contract:update', '编辑合同', 'CONTRACT', 'UPDATE', 'contract'),
('contract:delete', '删除合同', 'CONTRACT', 'DELETE', 'contract'),
('contract:approve', '审批合同', 'CONTRACT', 'APPROVE', 'contract'),
('contract:export', '导出合同', 'CONTRACT', 'EXPORT', 'contract'),
('contract:download', '下载合同文件', 'FILE', 'DOWNLOAD', 'contract'),
('workflow:manage', '工作流管理', 'WORKFLOW', 'MANAGE', 'workflow'),
('report:view', '查看报表', 'REPORT', 'READ', 'report'),
('report:export', '导出报表', 'REPORT', 'EXPORT', 'report'),
('admin:user', '用户管理', 'ADMIN', 'MANAGE', 'admin'),
('admin:role', '角色管理', 'ADMIN', 'MANAGE', 'admin');

-- ===================================================================
-- 角色-权限关联表
-- ===================================================================
CREATE TABLE role_permissions (
    id BIGSERIAL PRIMARY KEY,
    role_id BIGINT NOT NULL REFERENCES roles(id),
    permission_id BIGINT NOT NULL REFERENCES permissions(id),
    
    -- 数据范围限制 (ABAC扩展)
    data_scope VARCHAR(32) DEFAULT 'ALL',  -- ALL/DOMAIN/DEPARTMENT/SELF
    
    -- 条件限制 (JSON)
    conditions JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(role_id, permission_id)
);

-- ===================================================================
-- 用户-角色关联表
-- ===================================================================
CREATE TABLE user_roles (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    role_id BIGINT NOT NULL REFERENCES roles(id),
    
    -- 生效范围
    domain_code VARCHAR(32),  -- NULL表示所有业务域
    
    -- 有效期
    effective_from TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    effective_to TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(tenant_id, user_id, role_id, domain_code)
);

-- ===================================================================
-- ABAC 策略表
-- ===================================================================
CREATE TABLE abac_policies (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32),  -- NULL表示全局策略
    
    policy_code VARCHAR(64) NOT NULL,
    policy_name VARCHAR(256) NOT NULL,
    description TEXT,
    
    -- 目标资源
    target_resource VARCHAR(64) NOT NULL,
    target_action VARCHAR(32) NOT NULL,
    
    -- 策略规则 (JSON)
    policy_rules JSONB NOT NULL,
    
    -- 效果
    effect VARCHAR(16) NOT NULL DEFAULT 'ALLOW',  -- ALLOW/DENY
    
    -- 优先级
    priority INTEGER DEFAULT 0,
    
    -- 状态
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ABAC策略规则示例
/*
{
  "conditions": {
    "all": [
      {
        "fact": "subject.department",
        "operator": "equal",
        "value": "legal"
      },
      {
        "fact": "resource.contract_type",
        "operator": "in",
        "value": ["SALES", "PURCHASE"]
      },
      {
        "fact": "resource.amount",
        "operator": "lessThanOrEqual",
        "value": 1000000
      }
    ]
  }
}
*/
```

### 2.3 权限服务实现

```python
from dataclasses import dataclass
from typing import Optional, List, Dict, Any
from enum import Enum

class PermissionEffect(Enum):
    ALLOW = "ALLOW"
    DENY = "DENY"

@dataclass
class PermissionContext:
    tenant_id: str
    user_id: str
    user_roles: List[str]
    user_attributes: Dict[str, Any]
    
    resource_type: str
    resource_id: Optional[str]
    resource_attributes: Dict[str, Any]
    
    action: str
    
    context_attributes: Dict[str, Any]  # IP, time, device等


class PermissionService:
    def __init__(self,
                 rbac_store: RBACStore,
                 abac_engine: ABACEngine,
                 cache: PermissionCache):
        self.rbac_store = rbac_store
        self.abac_engine = abac_engine
        self.cache = cache
        
    async def check_permission(self, ctx: PermissionContext) -> PermissionResult:
        """
        权限检查主入口
        1. 检查缓存
        2. RBAC检查
        3. ABAC检查
        4. 综合决策
        """
        
        # 1. 缓存检查
        cache_key = self._build_cache_key(ctx)
        cached_result = await self.cache.get(cache_key)
        if cached_result:
            return cached_result
            
        # 2. RBAC检查
        rbac_result = await self._check_rbac(ctx)
        
        # 3. 如果RBAC允许，继续ABAC检查
        if rbac_result.effect == PermissionEffect.ALLOW:
            abac_result = await self._check_abac(ctx)
            
            # ABAC可以覆盖RBAC的允许决策
            if abac_result.effect == PermissionEffect.DENY:
                final_result = PermissionResult(
                    effect=PermissionEffect.DENY,
                    reason=abac_result.reason,
                    matched_policy=abac_result.matched_policy
                )
            else:
                # 合并数据范围限制
                final_result = PermissionResult(
                    effect=PermissionEffect.ALLOW,
                    data_scope=self._merge_data_scope(rbac_result, abac_result),
                    conditions=abac_result.conditions
                )
        else:
            final_result = rbac_result
            
        # 4. 缓存结果
        await self.cache.set(cache_key, final_result, ttl=300)
        
        return final_result
    
    async def _check_rbac(self, ctx: PermissionContext) -> RBACResult:
        """RBAC权限检查"""
        
        # 获取用户所有角色的权限
        permission_code = f"{ctx.resource_type.lower()}:{ctx.action.lower()}"
        
        for role_code in ctx.user_roles:
            role_perms = await self.rbac_store.get_role_permissions(
                ctx.tenant_id, role_code
            )
            
            for perm in role_perms:
                if perm.permission_code == permission_code:
                    # 检查数据范围
                    if self._check_data_scope(perm.data_scope, ctx):
                        return RBACResult(
                            effect=PermissionEffect.ALLOW,
                            data_scope=perm.data_scope,
                            matched_role=role_code
                        )
                        
        return RBACResult(
            effect=PermissionEffect.DENY,
            reason="No matching role permission"
        )
    
    async def _check_abac(self, ctx: PermissionContext) -> ABACResult:
        """ABAC策略检查"""
        
        # 获取适用的策略
        policies = await self.abac_engine.get_applicable_policies(
            ctx.tenant_id,
            ctx.resource_type,
            ctx.action
        )
        
        # 按优先级排序，DENY策略优先
        policies.sort(key=lambda p: (p.effect != "DENY", -p.priority))
        
        for policy in policies:
            match_result = self.abac_engine.evaluate_policy(policy, ctx)
            if match_result.matched:
                return ABACResult(
                    effect=PermissionEffect[policy.effect],
                    reason=f"Matched policy: {policy.policy_code}",
                    matched_policy=policy.policy_code,
                    conditions=match_result.extracted_conditions
                )
                
        # 没有匹配的策略，默认允许
        return ABACResult(effect=PermissionEffect.ALLOW)


class ABACEngine:
    """ABAC规则引擎"""
    
    def evaluate_policy(self, policy: ABACPolicy, ctx: PermissionContext) -> MatchResult:
        """评估单个策略"""
        
        facts = self._build_facts(ctx)
        rules = policy.policy_rules
        
        return self._evaluate_conditions(rules.get("conditions", {}), facts)
    
    def _build_facts(self, ctx: PermissionContext) -> Dict:
        """构建事实数据"""
        return {
            "subject.tenant_id": ctx.tenant_id,
            "subject.user_id": ctx.user_id,
            "subject.roles": ctx.user_roles,
            **{f"subject.{k}": v for k, v in ctx.user_attributes.items()},
            
            "resource.type": ctx.resource_type,
            "resource.id": ctx.resource_id,
            **{f"resource.{k}": v for k, v in ctx.resource_attributes.items()},
            
            "action": ctx.action,
            
            **{f"context.{k}": v for k, v in ctx.context_attributes.items()}
        }
    
    def _evaluate_conditions(self, conditions: Dict, facts: Dict) -> MatchResult:
        """评估条件表达式"""
        
        if "all" in conditions:
            # AND逻辑
            for cond in conditions["all"]:
                if not self._evaluate_single_condition(cond, facts):
                    return MatchResult(matched=False)
            return MatchResult(matched=True)
            
        elif "any" in conditions:
            # OR逻辑
            for cond in conditions["any"]:
                if self._evaluate_single_condition(cond, facts):
                    return MatchResult(matched=True)
            return MatchResult(matched=False)
            
        elif "fact" in conditions:
            # 单一条件
            return MatchResult(
                matched=self._evaluate_single_condition(conditions, facts)
            )
            
        return MatchResult(matched=True)
    
    def _evaluate_single_condition(self, cond: Dict, facts: Dict) -> bool:
        """评估单个条件"""
        fact_value = facts.get(cond["fact"])
        expected_value = cond["value"]
        operator = cond["operator"]
        
        operators = {
            "equal": lambda a, b: a == b,
            "notEqual": lambda a, b: a != b,
            "in": lambda a, b: a in b,
            "notIn": lambda a, b: a not in b,
            "contains": lambda a, b: b in a,
            "greaterThan": lambda a, b: a > b,
            "greaterThanOrEqual": lambda a, b: a >= b,
            "lessThan": lambda a, b: a < b,
            "lessThanOrEqual": lambda a, b: a <= b,
        }
        
        op_func = operators.get(operator)
        if op_func:
            try:
                return op_func(fact_value, expected_value)
            except:
                return False
        return False
```

### 2.4 数据脱敏服务

```python
class DataMaskingService:
    """敏感数据脱敏服务"""
    
    def __init__(self, masking_rules: Dict[str, MaskingRule]):
        self.rules = masking_rules
        
    def mask_contract(self, 
                      contract: Dict, 
                      user_permissions: List[str]) -> Dict:
        """根据用户权限对合同数据进行脱敏"""
        
        masked = contract.copy()
        
        # 根据字段级权限进行脱敏
        for field, rule in self.rules.items():
            if field in masked:
                if not self._has_field_permission(field, user_permissions):
                    masked[field] = self._apply_mask(masked[field], rule)
                    
        # 嵌套对象处理
        if "parties" in masked:
            masked["parties"] = [
                self._mask_party(p, user_permissions) 
                for p in masked["parties"]
            ]
            
        return masked
    
    def _apply_mask(self, value: Any, rule: MaskingRule) -> Any:
        """应用脱敏规则"""
        if value is None:
            return None
            
        if rule.strategy == "HIDE":
            return "******"
        elif rule.strategy == "PARTIAL":
            return self._partial_mask(str(value), rule.pattern)
        elif rule.strategy == "HASH":
            return hashlib.sha256(str(value).encode()).hexdigest()[:12]
        elif rule.strategy == "NULL":
            return None
        else:
            return value
    
    def _partial_mask(self, value: str, pattern: str) -> str:
        """部分遮蔽"""
        # pattern: "3:*:4" 表示保留前3位和后4位
        parts = pattern.split(":")
        keep_start = int(parts[0])
        keep_end = int(parts[2])
        
        if len(value) <= keep_start + keep_end:
            return "*" * len(value)
            
        return value[:keep_start] + "*" * (len(value) - keep_start - keep_end) + value[-keep_end:]


# 脱敏规则配置
MASKING_RULES = {
    "parties.id_number": MaskingRule(strategy="PARTIAL", pattern="3:*:4"),
    "parties.contact_phone": MaskingRule(strategy="PARTIAL", pattern="3:*:4"),
    "parties.bank_account": MaskingRule(strategy="PARTIAL", pattern="4:*:4"),
    "amount": MaskingRule(strategy="HIDE", required_permission="contract:view_amount"),
    "clauses": MaskingRule(strategy="HIDE", required_permission="contract:view_clauses")
}
```

---

## 3. 审计服务 (Audit Service)

### 3.1 审计事件模型

```sql
-- 审计日志表 (按月分区)
CREATE TABLE audit_logs (
    id BIGSERIAL,
    log_id VARCHAR(64) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    -- 事件信息
    event_type VARCHAR(64) NOT NULL,
    event_category VARCHAR(32) NOT NULL,  -- AUTH/CONTRACT/WORKFLOW/ADMIN
    event_action VARCHAR(32) NOT NULL,
    event_result VARCHAR(16) NOT NULL,  -- SUCCESS/FAILURE
    
    -- 主体信息
    user_id VARCHAR(64),
    user_name VARCHAR(128),
    user_ip VARCHAR(64),
    user_agent TEXT,
    
    -- 资源信息
    resource_type VARCHAR(64),
    resource_id VARCHAR(128),
    resource_name VARCHAR(256),
    
    -- 变更详情
    before_data JSONB,
    after_data JSONB,
    changed_fields TEXT[],
    
    -- 请求信息
    request_id VARCHAR(64),
    request_path VARCHAR(512),
    request_method VARCHAR(16),
    request_params JSONB,
    
    -- 响应信息
    response_code INTEGER,
    error_message TEXT,
    
    -- 业务上下文
    domain_code VARCHAR(32),
    workflow_id VARCHAR(64),
    
    -- 时间戳
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 创建月度分区
CREATE TABLE audit_logs_2024_11 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');
CREATE TABLE audit_logs_2024_12 PARTITION OF audit_logs
    FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

-- 索引
CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_audit_user ON audit_logs(tenant_id, user_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_logs(tenant_id, resource_type, resource_id);
CREATE INDEX idx_audit_event_type ON audit_logs(tenant_id, event_type, created_at DESC);
CREATE INDEX idx_audit_request ON audit_logs(request_id);
```

### 3.2 审计服务实现

```python
from contextlib import contextmanager
from contextvars import ContextVar

# 审计上下文
audit_context: ContextVar[AuditContext] = ContextVar('audit_context')

@dataclass
class AuditContext:
    request_id: str
    tenant_id: str
    user_id: str
    user_name: str
    user_ip: str
    user_agent: str
    domain_code: Optional[str] = None


class AuditService:
    def __init__(self, 
                 log_store: AuditLogStore,
                 event_bus: EventBus):
        self.log_store = log_store
        self.event_bus = event_bus
        
    async def log(self, event: AuditEvent):
        """记录审计事件"""
        
        # 获取上下文
        ctx = audit_context.get(None)
        if ctx:
            event.request_id = ctx.request_id
            event.tenant_id = ctx.tenant_id
            event.user_id = ctx.user_id
            event.user_name = ctx.user_name
            event.user_ip = ctx.user_ip
            event.user_agent = ctx.user_agent
            event.domain_code = ctx.domain_code
            
        # 生成日志ID
        event.log_id = generate_log_id()
        event.created_at = datetime.utcnow()
        
        # 异步写入存储
        await self.log_store.save(event)
        
        # 发布事件 (用于实时监控)
        await self.event_bus.publish("audit.event", event)
        
    async def log_contract_action(self,
                                   action: str,
                                   contract_id: str,
                                   contract_name: str,
                                   result: str,
                                   before_data: Optional[Dict] = None,
                                   after_data: Optional[Dict] = None,
                                   error_message: Optional[str] = None):
        """记录合同操作审计"""
        
        event = AuditEvent(
            event_type=f"contract.{action}",
            event_category="CONTRACT",
            event_action=action,
            event_result=result,
            resource_type="CONTRACT",
            resource_id=contract_id,
            resource_name=contract_name,
            before_data=before_data,
            after_data=after_data,
            changed_fields=self._diff_fields(before_data, after_data),
            error_message=error_message
        )
        
        await self.log(event)
    
    async def query_logs(self,
                         tenant_id: str,
                         filters: AuditQueryFilters,
                         page: int = 1,
                         page_size: int = 50) -> AuditQueryResult:
        """查询审计日志"""
        
        return await self.log_store.query(
            tenant_id=tenant_id,
            filters=filters,
            offset=(page - 1) * page_size,
            limit=page_size
        )
    
    async def generate_compliance_report(self,
                                          tenant_id: str,
                                          report_type: str,
                                          time_range: TimeRange) -> ComplianceReport:
        """生成合规报告"""
        
        # 根据报告类型获取审计数据
        if report_type == "ACCESS_REPORT":
            data = await self._generate_access_report(tenant_id, time_range)
        elif report_type == "CHANGE_REPORT":
            data = await self._generate_change_report(tenant_id, time_range)
        elif report_type == "SECURITY_REPORT":
            data = await self._generate_security_report(tenant_id, time_range)
        else:
            raise ValueError(f"Unknown report type: {report_type}")
            
        return ComplianceReport(
            report_id=generate_report_id(),
            report_type=report_type,
            tenant_id=tenant_id,
            time_range=time_range,
            generated_at=datetime.utcnow(),
            data=data
        )


# 审计装饰器
def audited(event_type: str, resource_type: str):
    """审计装饰器"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            audit_svc = get_audit_service()
            
            # 获取资源信息
            resource_id = kwargs.get('contract_id') or kwargs.get('id')
            
            try:
                result = await func(*args, **kwargs)
                
                # 记录成功
                await audit_svc.log(AuditEvent(
                    event_type=event_type,
                    event_category=resource_type,
                    event_action=event_type.split('.')[-1],
                    event_result="SUCCESS",
                    resource_type=resource_type,
                    resource_id=resource_id
                ))
                
                return result
                
            except Exception as e:
                # 记录失败
                await audit_svc.log(AuditEvent(
                    event_type=event_type,
                    event_category=resource_type,
                    event_action=event_type.split('.')[-1],
                    event_result="FAILURE",
                    resource_type=resource_type,
                    resource_id=resource_id,
                    error_message=str(e)
                ))
                raise
                
        return wrapper
    return decorator


# 使用示例
class ContractService:
    @audited(event_type="contract.create", resource_type="CONTRACT")
    async def create_contract(self, contract_data: Dict) -> Contract:
        # 业务逻辑
        pass
```

---

## 4. 工作流引擎 (Workflow Engine)

### 4.1 工作流模型

```sql
-- 流程定义表
CREATE TABLE workflow_definitions (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    
    workflow_code VARCHAR(64) NOT NULL,
    workflow_name VARCHAR(256) NOT NULL,
    workflow_type VARCHAR(32) NOT NULL,  -- APPROVAL/REVIEW/CUSTOM
    description TEXT,
    
    -- 流程定义 (BPMN JSON)
    definition JSONB NOT NULL,
    
    -- 版本
    version INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- 适用范围
    applicable_domains TEXT[],
    applicable_contract_types TEXT[],
    trigger_conditions JSONB,
    
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(tenant_id, workflow_code, version)
);

-- 流程实例表
CREATE TABLE workflow_instances (
    id BIGSERIAL PRIMARY KEY,
    instance_id VARCHAR(64) NOT NULL UNIQUE,
    tenant_id VARCHAR(32) NOT NULL,
    
    -- 关联定义
    workflow_definition_id BIGINT NOT NULL REFERENCES workflow_definitions(id),
    
    -- 业务关联
    business_type VARCHAR(32) NOT NULL,  -- CONTRACT/TEMPLATE/OTHER
    business_id VARCHAR(64) NOT NULL,
    business_name VARCHAR(256),
    
    -- 状态
    status VARCHAR(32) NOT NULL DEFAULT 'RUNNING',  -- RUNNING/COMPLETED/CANCELLED/SUSPENDED
    current_node_id VARCHAR(64),
    current_node_name VARCHAR(128),
    
    -- 发起信息
    initiator_id VARCHAR(64) NOT NULL,
    initiator_name VARCHAR(128),
    initiated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- 完成信息
    completed_at TIMESTAMP WITH TIME ZONE,
    completion_result VARCHAR(32),  -- APPROVED/REJECTED/CANCELLED
    
    -- 实例数据
    variables JSONB DEFAULT '{}',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 任务表
CREATE TABLE workflow_tasks (
    id BIGSERIAL PRIMARY KEY,
    task_id VARCHAR(64) NOT NULL UNIQUE,
    tenant_id VARCHAR(32) NOT NULL,
    
    -- 关联实例
    instance_id VARCHAR(64) NOT NULL,
    
    -- 节点信息
    node_id VARCHAR(64) NOT NULL,
    node_name VARCHAR(128),
    node_type VARCHAR(32) NOT NULL,  -- APPROVAL/COUNTERSIGN/NOTIFY/CONDITION
    
    -- 任务状态
    status VARCHAR(32) NOT NULL DEFAULT 'PENDING',  -- PENDING/CLAIMED/COMPLETED/CANCELLED/TRANSFERRED
    
    -- 处理人
    assignee_type VARCHAR(32),  -- USER/ROLE/DEPARTMENT
    assignee_id VARCHAR(64),
    assignee_name VARCHAR(128),
    
    -- 候选人 (多人可认领)
    candidate_users TEXT[],
    candidate_roles TEXT[],
    
    -- 处理信息
    claimed_by VARCHAR(64),
    claimed_at TIMESTAMP WITH TIME ZONE,
    completed_by VARCHAR(64),
    completed_at TIMESTAMP WITH TIME ZONE,
    
    -- 处理结果
    action VARCHAR(32),  -- APPROVE/REJECT/TRANSFER/RETURN
    comment TEXT,
    
    -- 时限
    due_date TIMESTAMP WITH TIME ZONE,
    reminder_sent BOOLEAN DEFAULT FALSE,
    
    -- 表单数据
    form_data JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 流程历史表
CREATE TABLE workflow_history (
    id BIGSERIAL PRIMARY KEY,
    instance_id VARCHAR(64) NOT NULL,
    tenant_id VARCHAR(32) NOT NULL,
    
    -- 事件信息
    event_type VARCHAR(32) NOT NULL,  -- NODE_ENTER/NODE_LEAVE/TASK_CREATE/TASK_COMPLETE/VARIABLE_UPDATE
    node_id VARCHAR(64),
    node_name VARCHAR(128),
    task_id VARCHAR(64),
    
    -- 操作信息
    operator_id VARCHAR(64),
    operator_name VARCHAR(128),
    action VARCHAR(32),
    comment TEXT,
    
    -- 数据快照
    data_snapshot JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_wf_instance_tenant ON workflow_instances(tenant_id, status);
CREATE INDEX idx_wf_instance_business ON workflow_instances(business_type, business_id);
CREATE INDEX idx_wf_task_assignee ON workflow_tasks(tenant_id, assignee_id, status);
CREATE INDEX idx_wf_task_instance ON workflow_tasks(instance_id);
CREATE INDEX idx_wf_history_instance ON workflow_history(instance_id, created_at);
```

### 4.2 工作流引擎实现

```python
class WorkflowEngine:
    def __init__(self,
                 definition_store: WorkflowDefinitionStore,
                 instance_store: WorkflowInstanceStore,
                 task_store: TaskStore,
                 event_bus: EventBus):
        self.definitions = definition_store
        self.instances = instance_store
        self.tasks = task_store
        self.event_bus = event_bus
        
    async def start_workflow(self,
                              workflow_code: str,
                              tenant_id: str,
                              business_type: str,
                              business_id: str,
                              initiator: User,
                              variables: Dict = None) -> WorkflowInstance:
        """启动工作流"""
        
        # 1. 获取流程定义
        definition = await self.definitions.get_active(
            tenant_id, workflow_code
        )
        if not definition:
            raise WorkflowNotFoundError(workflow_code)
            
        # 2. 创建流程实例
        instance = WorkflowInstance(
            instance_id=generate_instance_id(),
            tenant_id=tenant_id,
            workflow_definition_id=definition.id,
            business_type=business_type,
            business_id=business_id,
            initiator_id=initiator.user_id,
            initiator_name=initiator.user_name,
            status="RUNNING",
            variables=variables or {}
        )
        
        await self.instances.save(instance)
        
        # 3. 进入开始节点
        start_node = self._find_start_node(definition.definition)
        await self._enter_node(instance, start_node)
        
        # 4. 发布事件
        await self.event_bus.publish("workflow.started", {
            "instance_id": instance.instance_id,
            "workflow_code": workflow_code,
            "business_id": business_id
        })
        
        return instance
    
    async def complete_task(self,
                             task_id: str,
                             operator: User,
                             action: str,
                             comment: str = None,
                             form_data: Dict = None) -> TaskResult:
        """完成任务"""
        
        # 1. 获取任务
        task = await self.tasks.get(task_id)
        if not task:
            raise TaskNotFoundError(task_id)
            
        if task.status != "PENDING" and task.status != "CLAIMED":
            raise TaskAlreadyCompletedError(task_id)
            
        # 2. 验证操作人权限
        if not await self._can_complete_task(task, operator):
            raise PermissionDeniedError("Cannot complete this task")
            
        # 3. 更新任务状态
        task.status = "COMPLETED"
        task.completed_by = operator.user_id
        task.completed_at = datetime.utcnow()
        task.action = action
        task.comment = comment
        task.form_data = form_data
        
        await self.tasks.save(task)
        
        # 4. 记录历史
        await self._record_history(task.instance_id, "TASK_COMPLETE", {
            "task_id": task_id,
            "node_id": task.node_id,
            "action": action,
            "operator_id": operator.user_id
        })
        
        # 5. 推进流程
        await self._advance_workflow(task, action)
        
        return TaskResult(
            task_id=task_id,
            action=action,
            next_tasks=await self._get_pending_tasks(task.instance_id)
        )
    
    async def _advance_workflow(self, completed_task: Task, action: str):
        """推进工作流"""
        
        instance = await self.instances.get(completed_task.instance_id)
        definition = await self.definitions.get_by_id(
            instance.workflow_definition_id
        )
        
        current_node = self._find_node(
            definition.definition, completed_task.node_id
        )
        
        # 查找下一个节点
        next_node = self._find_next_node(
            definition.definition, current_node, action, instance.variables
        )
        
        if next_node:
            if next_node["type"] == "END":
                # 流程结束
                await self._complete_workflow(instance, action)
            else:
                # 进入下一节点
                await self._enter_node(instance, next_node)
    
    async def _enter_node(self, instance: WorkflowInstance, node: Dict):
        """进入节点"""
        
        node_type = node["type"]
        
        if node_type == "APPROVAL":
            # 创建审批任务
            assignees = await self._resolve_assignees(instance, node)
            for assignee in assignees:
                task = await self._create_task(instance, node, assignee)
                
        elif node_type == "COUNTERSIGN":
            # 会签任务 (所有人都需要审批)
            assignees = await self._resolve_assignees(instance, node)
            for assignee in assignees:
                task = await self._create_task(instance, node, assignee)
                
        elif node_type == "CONDITION":
            # 条件网关
            next_node = await self._evaluate_condition(instance, node)
            await self._enter_node(instance, next_node)
            
        elif node_type == "PARALLEL_GATEWAY":
            # 并行网关
            for branch in node["branches"]:
                await self._enter_node(instance, branch["start_node"])
                
        elif node_type == "SERVICE_TASK":
            # 服务任务 (自动执行)
            await self._execute_service_task(instance, node)
            
        # 更新实例状态
        instance.current_node_id = node["id"]
        instance.current_node_name = node["name"]
        await self.instances.save(instance)
    
    async def _resolve_assignees(self, 
                                  instance: WorkflowInstance, 
                                  node: Dict) -> List[Assignee]:
        """解析任务处理人"""
        
        assignee_config = node.get("assignee", {})
        assignee_type = assignee_config.get("type")
        
        if assignee_type == "USER":
            return [Assignee(type="USER", id=assignee_config["user_id"])]
            
        elif assignee_type == "ROLE":
            # 根据角色查找用户
            users = await self.user_service.get_users_by_role(
                instance.tenant_id, assignee_config["role_code"]
            )
            return [Assignee(type="USER", id=u.user_id) for u in users]
            
        elif assignee_type == "DEPARTMENT_HEAD":
            # 获取部门负责人
            dept_head = await self.org_service.get_department_head(
                instance.tenant_id, 
                instance.variables.get("department_id")
            )
            return [Assignee(type="USER", id=dept_head.user_id)]
            
        elif assignee_type == "INITIATOR":
            return [Assignee(type="USER", id=instance.initiator_id)]
            
        elif assignee_type == "VARIABLE":
            # 从变量中获取
            var_name = assignee_config["variable_name"]
            user_id = instance.variables.get(var_name)
            return [Assignee(type="USER", id=user_id)]
            
        elif assignee_type == "RULE":
            # 根据规则动态计算
            return await self._evaluate_assignee_rule(
                instance, assignee_config["rule"]
            )
            
        return []
```

### 4.3 合同审批流程示例

```json
// 合同审批流程定义
{
  "id": "contract_approval",
  "name": "合同审批流程",
  "version": 1,
  "nodes": [
    {
      "id": "start",
      "type": "START",
      "name": "开始",
      "next": "legal_review"
    },
    {
      "id": "legal_review",
      "type": "APPROVAL",
      "name": "法务审核",
      "assignee": {
        "type": "ROLE",
        "role_code": "LEGAL_REVIEWER"
      },
      "form": {
        "fields": [
          {"name": "legal_opinion", "type": "TEXT", "required": true},
          {"name": "risk_level", "type": "SELECT", "options": ["HIGH", "MEDIUM", "LOW"]}
        ]
      },
      "transitions": [
        {"action": "APPROVE", "next": "amount_gateway"},
        {"action": "REJECT", "next": "end_rejected"},
        {"action": "RETURN", "next": "start"}
      ]
    },
    {
      "id": "amount_gateway",
      "type": "CONDITION",
      "name": "金额判断",
      "conditions": [
        {
          "expression": "variables.amount >= 1000000",
          "next": "cfo_approval"
        },
        {
          "expression": "variables.amount >= 100000",
          "next": "finance_director_approval"
        },
        {
          "expression": "true",
          "next": "finance_manager_approval"
        }
      ]
    },
    {
      "id": "finance_manager_approval",
      "type": "APPROVAL",
      "name": "财务经理审批",
      "assignee": {
        "type": "ROLE",
        "role_code": "FINANCE_MANAGER"
      },
      "due_days": 2,
      "transitions": [
        {"action": "APPROVE", "next": "end_approved"},
        {"action": "REJECT", "next": "end_rejected"}
      ]
    },
    {
      "id": "finance_director_approval",
      "type": "APPROVAL",
      "name": "财务总监审批",
      "assignee": {
        "type": "ROLE",
        "role_code": "FINANCE_DIRECTOR"
      },
      "due_days": 3,
      "transitions": [
        {"action": "APPROVE", "next": "end_approved"},
        {"action": "REJECT", "next": "end_rejected"}
      ]
    },
    {
      "id": "cfo_approval",
      "type": "APPROVAL",
      "name": "CFO审批",
      "assignee": {
        "type": "ROLE",
        "role_code": "CFO"
      },
      "due_days": 5,
      "transitions": [
        {"action": "APPROVE", "next": "ceo_approval"},
        {"action": "REJECT", "next": "end_rejected"}
      ]
    },
    {
      "id": "ceo_approval",
      "type": "APPROVAL",
      "name": "CEO审批",
      "assignee": {
        "type": "ROLE",
        "role_code": "CEO"
      },
      "due_days": 7,
      "transitions": [
        {"action": "APPROVE", "next": "end_approved"},
        {"action": "REJECT", "next": "end_rejected"}
      ]
    },
    {
      "id": "end_approved",
      "type": "END",
      "name": "审批通过",
      "result": "APPROVED",
      "callbacks": [
        {
          "type": "UPDATE_CONTRACT_STATUS",
          "params": {"status": "APPROVED"}
        },
        {
          "type": "SEND_NOTIFICATION",
          "params": {"template": "contract_approved"}
        }
      ]
    },
    {
      "id": "end_rejected",
      "type": "END",
      "name": "审批驳回",
      "result": "REJECTED",
      "callbacks": [
        {
          "type": "UPDATE_CONTRACT_STATUS",
          "params": {"status": "REJECTED"}
        },
        {
          "type": "SEND_NOTIFICATION",
          "params": {"template": "contract_rejected"}
        }
      ]
    }
  ]
}
```

---

## 5. 多租户隔离服务

### 5.1 租户配置管理

```sql
-- 租户配置表
CREATE TABLE tenant_configs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL UNIQUE,
    tenant_name VARCHAR(256) NOT NULL,
    tenant_code VARCHAR(64) NOT NULL UNIQUE,
    
    -- 租户类型
    tenant_type VARCHAR(32) DEFAULT 'STANDARD',  -- STANDARD/ENTERPRISE/VIP
    
    -- 隔离策略
    isolation_level VARCHAR(32) DEFAULT 'SHARED',  -- SHARED/SCHEMA/DATABASE
    
    -- 数据库配置 (独立数据库租户)
    db_config JSONB,
    
    -- 存储配置
    storage_bucket VARCHAR(128),
    storage_quota_gb INTEGER DEFAULT 100,
    
    -- 功能配置
    features JSONB DEFAULT '{}',
    
    -- 资源配额
    quotas JSONB DEFAULT '{}',
    
    -- 状态
    status VARCHAR(32) DEFAULT 'ACTIVE',  -- ACTIVE/SUSPENDED/TERMINATED
    
    -- 有效期
    effective_from DATE,
    effective_to DATE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 租户配额示例
/*
{
  "max_contracts": 100000,
  "max_users": 500,
  "max_storage_gb": 500,
  "max_api_calls_per_day": 1000000,
  "max_concurrent_users": 100,
  "max_workflows": 50
}
*/

-- 业务域配置表
CREATE TABLE domain_configs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(32) NOT NULL,
    domain_code VARCHAR(32) NOT NULL,
    domain_name VARCHAR(128) NOT NULL,
    
    -- 域配置
    config JSONB DEFAULT '{}',
    
    -- 域管理员
    admin_user_ids TEXT[],
    
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(tenant_id, domain_code)
);
```

### 5.2 租户上下文管理

```python
from contextvars import ContextVar
from typing import Optional

# 租户上下文
tenant_context: ContextVar[TenantContext] = ContextVar('tenant_context')

@dataclass
class TenantContext:
    tenant_id: str
    tenant_code: str
    tenant_type: str
    isolation_level: str
    domain_code: Optional[str] = None
    features: Dict = None
    quotas: Dict = None


class TenantContextMiddleware:
    """租户上下文中间件"""
    
    def __init__(self, tenant_service: TenantService):
        self.tenant_service = tenant_service
        
    async def __call__(self, request: Request, call_next):
        # 从请求头或JWT中提取租户ID
        tenant_id = self._extract_tenant_id(request)
        
        if tenant_id:
            # 加载租户配置
            tenant_config = await self.tenant_service.get_config(tenant_id)
            
            if not tenant_config or tenant_config.status != "ACTIVE":
                raise TenantNotActiveError(tenant_id)
                
            # 设置上下文
            ctx = TenantContext(
                tenant_id=tenant_id,
                tenant_code=tenant_config.tenant_code,
                tenant_type=tenant_config.tenant_type,
                isolation_level=tenant_config.isolation_level,
                domain_code=request.headers.get("X-Domain-Code"),
                features=tenant_config.features,
                quotas=tenant_config.quotas
            )
            
            token = tenant_context.set(ctx)
            
            try:
                response = await call_next(request)
                return response
            finally:
                tenant_context.reset(token)
        else:
            raise TenantRequiredError()


class TenantAwareRepository:
    """租户感知的数据仓库基类"""
    
    def _get_tenant_id(self) -> str:
        ctx = tenant_context.get(None)
        if not ctx:
            raise TenantContextNotSetError()
        return ctx.tenant_id
    
    def _get_table_name(self, base_table: str) -> str:
        """根据隔离级别获取表名"""
        ctx = tenant_context.get()
        
        if ctx.isolation_level == "SCHEMA":
            return f"{ctx.tenant_code}.{base_table}"
        else:
            return base_table
    
    def _add_tenant_filter(self, query):
        """添加租户过滤条件"""
        ctx = tenant_context.get()
        
        if ctx.isolation_level == "SHARED":
            # 共享模式需要添加tenant_id过滤
            return query.where(table.tenant_id == ctx.tenant_id)
        else:
            # Schema/Database隔离模式不需要额外过滤
            return query


class QuotaEnforcer:
    """配额强制执行器"""
    
    def __init__(self, quota_store: QuotaStore):
        self.quota_store = quota_store
        
    async def check_quota(self, 
                          tenant_id: str, 
                          resource_type: str,
                          increment: int = 1) -> bool:
        """检查配额"""
        
        quota = await self.quota_store.get_quota(tenant_id, resource_type)
        usage = await self.quota_store.get_usage(tenant_id, resource_type)
        
        if usage + increment > quota:
            raise QuotaExceededError(
                tenant_id, resource_type, quota, usage + increment
            )
            
        return True
    
    async def increment_usage(self,
                               tenant_id: str,
                               resource_type: str,
                               amount: int = 1):
        """增加使用量"""
        await self.quota_store.increment_usage(tenant_id, resource_type, amount)
        
    async def decrement_usage(self,
                               tenant_id: str,
                               resource_type: str,
                               amount: int = 1):
        """减少使用量"""
        await self.quota_store.decrement_usage(tenant_id, resource_type, amount)
```
