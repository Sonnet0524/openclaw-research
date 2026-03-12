# 自主开发央国企AI助手：设计参考

---
created: 2026-03-12
phase: Review → Confirm
status: completed
type: design-reference
---

## 📋 文档概述

基于OpenClaw源码深度分析，为自主开发央国企AI助手提供设计参考，重点解决央国企环境的约束问题。

---

## 一、央国企环境约束与设计参考

### 1.1 网络隔离约束

**约束描述**：
- 内外网物理/逻辑隔离
- 云端服务不可用
- 外网访问需审批

**OpenClaw设计参考**：

**参考设计：Local-first架构**
```
~/.openclaw/
├── openclaw.json         # 配置文件
├── credentials/          # 凭证存储
├── agents/               # Agent状态
│   └── <agentId>/sessions/  # 会话记录
└── workspace/            # 工作空间
```

**借鉴要点**：
- ✅ 所有数据本地存储，无需云端依赖
- ✅ 配置、凭证、会话分离管理
- ✅ 支持完全离线运行

**央国企改进方向**：

| 组件 | OpenClaw设计 | 央国企改进 |
|------|------------|----------|
| 数据存储 | 本地文件 | 加密存储 + 分布式存储 |
| 会话管理 | JSONL文件 | 数据库存储 + 审计日志 |
| 配置管理 | JSON文件 | 统一配置中心 |
| 凭证管理 | 文件存储 | 密钥管理系统 |

**具体设计建议**：

```typescript
// 央国企本地化存储设计
type LocalizedStorage = {
  dataStore: {
    type: "encrypted-database";  // 加密数据库
    encryption: "SM4";           // 国密算法
    location: "internal-network"; // 内网存储
  };
  
  sessionManagement: {
    store: "distributed-db";     // 分布式数据库
    audit: "siem-integrated";    // SIEM集成
    retention: "90-days";        // 保留策略
  };
  
  configManagement: {
    type: "centralized";         // 集中管理
    versioning: true;            // 版本控制
    approval: "workflow-enabled"; // 审批流程
  };
  
  credentialsManagement: {
    type: "hsm-integrated";      // HSM集成
    rotation: "90-days";         // 密钥轮换
    backup: "encrypted-vault";   // 加密备份
  };
};
```

---

### 1.2 权限管控约束

**约束描述**：
- ISC统一权限管控
- 权限申请流程复杂
- 动态授权困难

**OpenClaw设计参考**：

**参考设计：多层权限控制**
```typescript
// Gateway认证层
type GatewayAuth = {
  mode: "token" | "password" | "device-auth" | "trusted-proxy";
  token?: string;
};

// DM访问控制层
type DmPolicy = "pairing" | "allowlist" | "open" | "disabled";

// 工具权限层
type ToolPolicy = {
  profile?: ToolProfileId;
  allow?: string[];
  deny?: string[];
};

// 沙箱隔离层
type SandboxPolicy = {
  mode: "off" | "non-main" | "all";
  tools: {
    allow: string[];
    deny: string[];
  };
};
```

**借鉴要点**：
- ✅ 多层次权限设计，逐层收紧
- ✅ DM配对机制防止未授权访问
- ✅ 工具策略灵活可配置
- ✅ 沙箱隔离限制执行权限

**央国企改进方向**：

| 层次 | OpenClaw设计 | 央国企改进 |
|------|------------|----------|
| Gateway认证 | Token/Password | ISC统一认证 + PKI证书 |
| DM访问控制 | 配对/Allowlist | RBAC + 审批流程 |
| 工具权限 | Profile + Allow/Deny | 角色权限 + 审批流程 |
| 沙箱隔离 | Docker隔离 | 容器 + VM双层隔离 |

**具体设计建议**：

```typescript
// 央国企权限体系设计
type PermissionSystem = {
  // 1. 身份认证层 - 对接ISC
  authentication: {
    type: "isc-integrated";      // ISC集成
    method: "PKI + MFA";         // 证书 + 多因素认证
    sessionTimeout: "8-hours";   // 会话超时
  };
  
  // 2. 访问控制层 - RBAC
  authorization: {
    model: "RBAC";               // 基于角色
    roles: {
      admin: {
        permissions: ["*"];
        approval: "none";
      };
      power_user: {
        permissions: ["read", "write", "execute"];
        approval: "simple";
      };
      normal_user: {
        permissions: ["read", "limited_write"];
        approval: "required";
      };
      readonly_user: {
        permissions: ["read"];
        approval: "none";
      };
    };
    
    // 动态权限提升
    privilegeEscalation: {
      enabled: true;
      approvalWorkflow: "oa-integrated";
      timeout: "4-hours";
      audit: "full";
    };
  };
  
  // 3. 工具权限层 - 细粒度控制
  toolPolicy: {
    model: "ABAC";               // 基于属性
    attributes: ["role", "department", "security-level"];
    rules: [
      {
        condition: { role: "normal_user", tool: "exec" },
        action: "deny"
      },
      {
        condition: { role: "power_user", tool: "exec", risk: "high" },
        action: "approve-required"
      }
    ];
  };
  
  // 4. 执行隔离层 - 多级防护
  executionIsolation: {
    levels: [
      { name: "trusted", isolation: "none" },        // 可信用户
      { name: "normal", isolation: "container" },    // 普通用户
      { name: "untrusted", isolation: "vm" }         // 不可信场景
    ];
    
    // 审计追踪
    auditTrail: {
      enabled: true;
      level: "verbose";           // 详细记录
      siem: "integrated";         // SIEM集成
    };
  };
};
```

---

### 1.3 API不足约束

**约束描述**：
- 业务系统API缺乏
- 接口文档不全
- 跨系统调用困难

**OpenClaw设计参考**：

**参考设计：Channels适配器模式**
```typescript
// 统一的渠道接口
interface ChannelAdapter {
  // 消息接收
  onMessage(handler: MessageHandler): void;
  // 消息发送
  send(to: string, message: Message): Promise<void>;
  // DM策略
  dmPolicy: "pairing" | "allowlist" | "open" | "disabled";
  // 群组策略
  groupPolicy: "open" | "allowlist" | "disabled";
}

// 渠道注册机制
type ChannelPlugin = {
  id: ChannelId;
  channel?: {
    send: (params) => Promise<void>;
    receive?: (params) => Promise<void>;
  };
};
```

**借鉴要点**：
- ✅ 统一的抽象接口，降低集成复杂度
- ✅ 插件化架构，易于扩展新渠道
- ✅ 策略在适配层实现，不依赖模型

**央国企改进方向**：

| 设计 | OpenClaw方案 | 央国企改进 |
|------|------------|----------|
| 渠道集成 | 企业微信/钉钉适配器 | 企业内部IM统一适配 |
| 数据获取 | 实时API调用 | ETL + 中间数据库 |
| 系统对接 | 直接API调用 | 中间服务 + 消息队列 |

**具体设计建议**：

```typescript
// 央国企系统集成设计
type SystemIntegration = {
  // 1. 内部IM统一适配
  internalMessaging: {
    adapters: {
      wechat_work: {
        enabled: true;
        apiType: "official";      // 官方API
        features: ["dm", "group", "file"];
      };
      dingtalk: {
        enabled: true;
        apiType: "official";
        features: ["dm", "group", "workflow"];
      };
      feishu: {
        enabled: true;
        apiType: "official";
        features: ["dm", "group", "doc"];
      };
    };
    
    // 统一消息抽象
    messageAbstraction: {
      type: string;              // 统一消息类型
      content: string;           // 统一内容格式
      metadata: Record<string, any>; // 元数据
    };
  };
  
  // 2. 数据获取方案
  dataAccess: {
    // 方案A：中间数据库
    intermediateDatabase: {
      enabled: true;
      type: "data-warehouse";
      syncStrategy: {
        method: "etl";
        schedule: "hourly";       // 每小时同步
        incremental: true;        // 增量同步
      };
      
      // 数据源映射
      sources: [
        { system: "ERP", table: "orders", sync: "realtime" },
        { system: "CRM", table: "customers", sync: "hourly" },
        { system: "OA", table: "approvals", sync: "realtime" }
      ];
    };
    
    // 方案B：消息队列
    messageQueue: {
      enabled: true;
      type: "kafka";
      topics: [
        { name: "business-events", retention: "7-days" },
        { name: "approval-events", retention: "30-days" }
      ];
    };
  };
  
  // 3. 系统对接中间服务
  systemBridge: {
    enabled: true;
    services: [
      {
        name: "approval-bridge";
        source: "OA";
        target: "AI-Assistant";
        transformation: "event-to-message";
      },
      {
        name: "data-bridge";
        source: "DataWarehouse";
        target: "AI-Assistant";
        transformation: "query-to-response";
      }
    ];
  };
};
```

---

### 1.4 系统封闭约束

**约束描述**：
- 数据孤岛现象
- 系统间不互通
- 定制开发困难

**OpenClaw设计参考**：

**参考设计：Skills插件系统**
```typescript
// Skills目录结构
~/.openclaw/workspace/skills/
├── <skill-name>/
│   └── SKILL.md    # 技能定义

// Skills加载机制
type SkillPlugin = {
  name: string;
  description: string;
  tools?: ToolDefinition[];
  handlers?: SkillHandler[];
};
```

**借鉴要点**：
- ✅ 非侵入式扩展，不修改核心
- ✅ 文件驱动，易于管理
- ✅ 按需加载，降低资源消耗

**央国企改进方向**：

| 设计 | OpenClaw方案 | 央国企改进 |
|------|------------|----------|
| 扩展机制 | Skills文件 | 定制化服务模块 |
| 生态依赖 | ClawHub社区 | 内部技能市场 |
| 安全审核 | 社区自审 | 安全审计流程 |

**具体设计建议**：

```typescript
// 央国企定制化扩展设计
type CustomizationSystem = {
  // 1. 内部技能市场
  skillMarketplace: {
    enabled: true;
    visibility: "internal";      // 内部可见
    
    // 技能审核流程
    reviewProcess: {
      security: "required";      // 安全审核
      compliance: "required";    // 合规审核
      testing: "required";       // 测试验证
      approval: "multi-level";   // 多级审批
    };
    
    // 技能分类
    categories: [
      "document-processing",     // 文档处理
      "data-analysis",          // 数据分析
      "workflow-automation",    // 流程自动化
      "knowledge-management"    // 知识管理
    ];
  };
  
  // 2. 定制化服务模块
  serviceModules: {
    // 文档处理模块
    documentProcessing: {
      skills: [
        {
          name: "document-summary";
          capability: "文档摘要";
          securityLevel: "internal";
        },
        {
          name: "document-translation";
          capability: "文档翻译";
          securityLevel: "public";
        }
      ];
    };
    
    // 数据分析模块
    dataAnalysis: {
      skills: [
        {
          name: "report-generation";
          capability: "报告生成";
          dataSource: ["ERP", "CRM"];
          securityLevel: "confidential";
        }
      ];
    };
    
    // 流程自动化模块
    workflowAutomation: {
      skills: [
        {
          name: "approval-assistant";
          capability: "审批助手";
          integration: "OA";
          securityLevel: "confidential";
        }
      ];
    };
  };
  
  // 3. 安全审核机制
  securityReview: {
    codeReview: "required";      // 代码审查
    vulnerabilityScan: "required"; // 漏洞扫描
    penetrationTest: "optional"; // 渗透测试
    complianceCheck: "required"; // 合规检查
  };
};
```

---

### 1.5 安全合规约束

**约束描述**：
- 等保2.0要求
- 行业规范
- 监管要求

**OpenClaw设计参考**：

**参考设计：安全审计工具**
```typescript
// 安全审计检查项
const AUDIT_CHECKS = [
  "gateway.bind_no_auth",
  "fs.state_dir.perms_world_writable",
  "sandbox.dangerous_network_mode",
  "tools.exec.host_sandbox_no_sandbox",
];

// 审计报告结构
type SecurityAuditReport = {
  ts: number;
  summary: { critical: number; warn: number; info: number };
  findings: SecurityAuditFinding[];
};
```

**借鉴要点**：
- ✅ 自动化安全检查
- ✅ 多层次安全审计
- ✅ 提供修复建议

**央国企改进方向**：

| 设计 | OpenClaw方案 | 央国企改进 |
|------|------------|----------|
| 安全审计 | 基础审计 | 等保对标审计 |
| 合规检查 | 无 | 行业规范检查 |
| 审计报告 | 简单格式 | 合规报告生成 |

**具体设计建议**：

```typescript
// 央国企安全合规设计
type ComplianceSystem = {
  // 1. 等保2.0对标
  mlpsCompliance: {
    // 身份鉴别
    authentication: {
      identityVerification: "PKI";    // PKI证书
      mfa: "required";                // 多因素认证
      sessionManagement: "secure";    // 会话管理
    };
    
    // 访问控制
    accessControl: {
      model: "RBAC + ABAC";           // 混合模式
      granularity: "fine-grained";    // 细粒度
      audit: "full";                  // 完整审计
    };
    
    // 安全审计
    securityAudit: {
      logCollection: "centralized";   // 集中收集
      retention: "180-days";          // 保留180天
      analysis: "siem-integrated";    // SIEM集成
    };
    
    // 入侵防范
    intrusionPrevention: {
      waf: "enabled";                 // WAF
      ids: "enabled";                 // 入侵检测
      ips: "enabled";                 // 入侵防御
    };
    
    // 数据完整性
    dataIntegrity: {
      checksum: "required";           // 校验和
      digitalSignature: "required";   // 数字签名
      versionControl: "required";     // 版本控制
    };
    
    // 数据保密性
    dataConfidentiality: {
      encryption: "SM4";              // 国密加密
      keyManagement: "HSM";           // HSM
      secureTransmission: "TLS1.3";   // 安全传输
    };
    
    // 数据备份恢复
    backupRecovery: {
      backup: "daily";                // 每日备份
      retention: "90-days";           // 保留90天
      disasterRecovery: "enabled";    // 灾备
      testRecovery: "monthly";        // 月度恢复测试
    };
  };
  
  // 2. 行业规范检查
  industryStandards: {
    financial: {
      enabled: false;                 // 金融行业标准
    },
    telecom: {
      enabled: false;                 // 电信行业标准
    },
    energy: {
      enabled: true;                  // 能源行业标准
      standards: ["DL/T", "NB/T"];
    }
  };
  
  // 3. 合规报告生成
  complianceReporting: {
    automated: true;                  // 自动生成
    frequency: "quarterly";           // 季度报告
    format: "standardized";           // 标准化格式
    submission: "automated";          // 自动提交
  };
};
```

---

## 二、自主开发风险识别（中等）

### 2.1 风险评估矩阵

**整体风险评级：中等**

| 风险类别 | 风险描述 | 可能性 | 影响 | 综合评级 |
|---------|---------|--------|------|---------|
| **技术风险** | | | | **中等** |
| R-T01 | 架构设计复杂度 | 中 | 中 | 中 |
| R-T02 | 国产模型能力限制 | 中 | 中 | 中 |
| R-T03 | 性能瓶颈 | 低 | 中 | 低 |
| R-T04 | 集成复杂度 | 中 | 中 | 中 |
| **安全风险** | | | | **中等** |
| R-S01 | Prompt注入攻击 | 高 | 高 | 高 |
| R-S02 | 数据泄露 | 低 | 高 | 中 |
| R-S03 | 权限提升 | 低 | 高 | 中 |
| R-S04 | 第三方依赖漏洞 | 中 | 中 | 中 |
| **合规风险** | | | | **低** |
| R-C01 | 等保测评不通过 | 低 | 高 | 中 |
| R-C02 | 审计不满足要求 | 低 | 中 | 低 |
| R-C03 | 数据保护违规 | 低 | 高 | 中 |
| **团队风险** | | | | **中等** |
| R-P01 | 技术能力不足 | 中 | 高 | 中 |
| R-P02 | 人员流动 | 中 | 中 | 中 |
| R-P03 | 培训成本 | 中 | 低 | 低 |
| **运维风险** | | | | **中等** |
| R-O01 | 故障处理能力不足 | 中 | 中 | 中 |
| R-O02 | 监控缺失 | 低 | 中 | 低 |
| R-O03 | 升级维护困难 | 中 | 中 | 中 |

---

### 2.2 重点风险应对

**R-S01: Prompt注入攻击（高）**

**应对措施**：
1. **输入过滤层**
   - 敏感关键词检测
   - 恶意指令识别
   - 输入长度限制

2. **输出校验层**
   - 输出内容检查
   - 敏感信息过滤
   - 合规性验证

3. **隔离防护层**
   - 沙箱隔离执行
   - 最小权限原则
   - 审计日志追踪

---

**R-P01: 技术能力不足（中）**

**应对措施**：
1. **培训体系**
   - AI技术培训
   - 安全开发培训
   - 合规性培训

2. **外部支持**
   - 咨询服务引入
   - 技术合作
   - 开源社区参与

3. **知识沉淀**
   - 文档体系建立
   - 最佳实践积累
   - 技术分享机制

---

## 三、关键技术选型建议

### 3.1 语言和框架

| 组件 | 推荐方案 | 理由 |
|------|---------|------|
| **核心语言** | TypeScript | 开发效率高，生态成熟 |
| **高性能模块** | Rust | 安全性高，性能好 |
| **Web框架** | NestJS | 企业级框架，模块化 |
| **Agent运行时** | 自研（参考pi-mono） | 可控性强，可定制 |

### 3.2 数据存储

| 组件 | 推荐方案 | 理由 |
|------|---------|------|
| **关系数据库** | PostgreSQL / 达梦 | 成熟稳定，支持国产化 |
| **向量数据库** | Milvus | 开源，性能好 |
| **对象存储** | MinIO | 开源，S3兼容 |
| **缓存** | Redis | 成熟稳定 |

### 3.3 安全组件

| 组件 | 推荐方案 | 理由 |
|------|---------|------|
| **身份认证** | ISC对接 + PKI | 满足统一身份管理 |
| **权限控制** | Casbin | 灵活的权限框架 |
| **数据加密** | 国密算法库 | SM4加密 |
| **密钥管理** | HashiCorp Vault | 企业级密钥管理 |

---

## 四、实施路径建议

### 阶段1：架构验证（1-2月）

**目标**：验证技术可行性

**关键活动**：
- OpenClaw源码深入分析
- 核心架构原型开发
- 国产模型集成测试
- 安全机制验证

**交付物**：
- 技术架构文档
- 原型系统
- 可行性报告

---

### 阶段2：核心开发（3-6月）

**目标**：开发核心组件

**关键活动**：
- Gateway核心开发
- Agent运行时开发
- 渠道适配器开发
- 安全机制实现

**交付物**：
- 核心系统代码
- 单元测试
- 技术文档

---

### 阶段3：系统集成（3-4月）

**目标**：对接企业系统

**关键活动**：
- ISC统一认证对接
- 企业IM渠道对接
- 审计日志系统对接
- 数据源集成

**交付物**：
- 集成系统
- 接口文档
- 集成测试报告

---

### 阶段4：合规完善（2-3月）

**目标**：满足合规要求

**关键活动**：
- 等保对标整改
- 安全加固
- 审计功能完善
- 合规报告生成

**交付物**：
- 合规系统
- 测评报告
- 运维文档

---

## 五、总结

### 核心设计参考

1. **架构设计**：借鉴Local-first + Multi-channel架构
2. **安全设计**：借鉴多层次安全防御策略
3. **扩展设计**：借鉴插件化架构
4. **运维设计**：借鉴显式配置 + 自动化审计

### 关键改进方向

1. **多租户支持**：重新设计多租户架构
2. **安全加固**：增加加密存储、完整审计
3. **合规改造**：满足等保2.0要求
4. **国产化适配**：支持国产软硬件

### 实施建议

> 借鉴OpenClaw的优秀设计理念和架构模式，结合央国企实际需求和安全合规要求，自主开发符合央国企环境的AI助手系统。技术难度中等，风险可控，建议采用分阶段迭代开发方式。

---

**维护者**: Research Agent  
**创建时间**: 2026-03-12  
**文档类型**: 设计参考  
**参考状态**: ✅ 已完成
