# 密钥管理方案选型（Vault 与替代方案）

> 背景：quanttide-platform 系统级 IaC（阿里云）需要统一密钥管理服务，2026-08 完成首轮方案调研。
> 状态：**方向待定**——候选方案已收敛至 4 个，待决策后落地。

## 1. 需求与约束

### 1.1 需求

- 为 quanttide 体系（应用、CI、本地工具）提供统一密钥服务，替代散落的 GitHub org secrets + FC 环境变量
- 应用直接从密钥服务获取密钥，不经过 shell 环境变量或中间层（`docs/dev-guide/iac/secrets.md` 既定架构）
- 当前已知消费方：qtcloud-knowl（LLM API key）、共享 RDS 数据库密码、JWT 私钥等

### 1.2 约束（决策标准）

| 约束 | 说明 | 来源 |
|------|------|------|
| **FaaS 为主** | 大型系统以 FaaS（函数计算）为主，k8s 与服务器为辅；服务器仅承载 FaaS 无法承载的有状态/常驻组件 | docs/dev-guide/iac/index.md（2026-08-17 更新） |
| **价格敏感** | 排除阿里云 KMS（密钥费 + 凭据费 + 调用费，决策者认为价格太高） | 决策会议 2026-08 |
| **国内可用** | 排除境外 SaaS（数据出境合规、访问稳定性）；自托管在国内服务器可满足 | 决策会议 2026-08 |
| **供应商可互换** | 密钥服务不应绑定单一云厂商（与「供应商可互换」设计原则一致） | docs/dev-guide/iac/index.md |
| **探索期阶段** | 当前应用多为探索期，应控制运维成本，避免过早引入重组件 | ROADMAP 现状 |

## 2. 候选方案全景

| 方案 | 部署形态 | 成本量级 | 国内 | 结论 |
|------|---------|---------|------|------|
| **自托管 Infisical** ⭐ | 国内 ECS + Docker Compose（Postgres 可复用共享 RDS） | 1 台最低配 ECS（软件免费，MIT） | ✅ 数据自控 | 候选（推荐） |
| **SOPS + age** | 无服务，密钥加密文件入 git，CI/本地解密 | **0 元** | ✅ | 候选（最轻） |
| **Vault 自托管**（去 KMS） | 国内 ECS，OSS 存储 + Shamir 解封 | 1 台最低配 ECS（社区版免费） | ✅ | 候选（能力最强，运维最重） |
| **现状**（GitHub org secrets + FC env） | 无 | **0 元** | ✅ | 基线（探索期可用） |
| 阿里云 KMS 凭据管理 | 托管 | 密钥/凭据/调用三项计费 | ✅ | ❌ 已排除（价格太高） |
| Infisical 云版 | 境外 SaaS | 免费额度起步 | ❌ | ❌ 已排除（境外） |
| 腾讯云 SSM / 华为云 DEW 凭据管理 | 国内托管 | 按量计费（量级与阿里云 KMS 类似） | ✅ | 低优先级（绑定第二家云厂商，价格量级类似 KMS） |
| FC 运行 Vault | — | — | — | ❌ 技术不可行（见 §3.1） |

## 3. 已排除方案与论证

### 3.1 FC 运行 Vault — 技术不可行

| 要求 | FC 能力 | 结论 |
|------|---------|------|
| 监听 8200 TCP 端口 | 只支持 HTTP/WebSocket 触发器，不支持自定义 TCP 端口监听 | ❌ 硬伤 |
| 稳定私网访问地址 | 函数对外只有公网调用地址，无法分配 VPC 内网固定 IP | ❌ 密钥服务暴露公网 |
| 单活跃节点（OSS 后端只允许一个 active writer） | FC 实例随请求伸缩/空闲回收/冷启动重建，无法保证单实例 | ❌ 状态错乱风险 |
| 常驻可用 | 实例重建需重新走 KMS auto-unseal，期间不可用 | ❌ 抖动 |

结论：Vault 属于「有状态、需常驻、需固定地址」组件，必须由服务器承载（符合 FaaS 为主原则中「服务器为辅」的职责定位）。

### 3.2 阿里云 KMS 凭据管理 — 价格

托管凭据 + RDS 自动轮转 + Terraform 原生支持（`alicloud_kms_secret`），能力上最贴合 FaaS 为主；但密钥费 + 凭据托管费 + API 调用费叠加后决策者判定价格太高，排除。

**连带影响**：Vault 原设计的 `alicloudkms` seal（Auto-unseal）同样依赖 KMS，KMS 排除后 Vault 云端方案改为 **OSS 存储 + Shamir 解封**（unseal keys 备份 Nutstore，与本地环境约定一致）。`docs/dev-guide/iac/secrets.md` 中「云端对接云 KMS 实现 Auto-unseal」的描述待随决策更新。

### 3.3 Infisical 云版 — 境外

功能与体验最佳（UI/CLI/SDK/审计），免费额度对探索期够用；但数据在境外，不符合「国内可用」约束。自托管版不受此限制（见 §4.1）。

### 3.4 腾讯云 SSM / 华为云 DEW

国内托管，但价格量级与阿里云 KMS 类似（密钥/凭据/调用计费），且引入第二家云厂商绑定——当前主栈为阿里云，跨云给 FC 取密钥别扭。保留为低优先级选项，未正式排除。

## 4. 候选方案详述

### 4.1 自托管 Infisical（国内 ECS）⭐ 推荐

- **开源免费**（MIT），自托管无功能限制；数据完全自控，满足「国内可用」
- **部署**：1 台最低配 ECS + Docker Compose；Postgres 可复用已有共享 RDS（白名单仅放行 VPC 网段，天然契合）
- **不绑定云厂商**：可部署于阿里云/腾讯云/本地 NAS，符合「供应商可互换」
- **能力**：项目/环境/密钥模型、端到端加密、Web UI + CLI + SDK、CI 集成、审计日志、部分自动轮转
- **代价**：1 台 ECS 月费（~50-100 元，可抢占式）；多一个常驻组件（落在「服务器为辅」职责内）
- **演进**：云版（免费额度）→ 自托管，API 兼容，应用接入方式不变

### 4.2 SOPS + age（最轻）

- **原理**：密钥写 YAML/JSON → `age` 公钥加密 → 密文文件进 git → CI/本地用私钥解密注入环境变量
- **零服务零成本**：无常驻组件，与 FaaS 为主契合度满分；私钥本地保管（备份 Nutstore，与 unseal keys 备份约定一致）
- **审计**：靠 git 历史；吊销粒度差（密钥改版需提交代码）；多人协作靠私钥分发（小团队可行）
- **变体**：git-crypt（透明解密整仓，更简单但粒度粗）
- **适配现状**：GitHub org secrets 已是「半 SOPS」，SOPS 只是把密钥从控制台挪到加密文件，获得版本管理 + 统一解密入口

### 4.3 Vault 自托管（去 KMS）

- 保留原架构方向：OSS 存储 + Shamir 解封（unseal keys 备份 Nutstone）
- 能力最强（policy/token、动态凭据、审计），运维最重（初始化、解封流程、token 生命周期）
- 何时选它：需要动态凭据（数据库临时密码）、细粒度 policy、把加密当服务时

### 4.4 现状（GitHub org secrets + FC 环境变量）

- 探索期基线，零成本；`deployment-standards.md` 已有 secrets 清单规范
- 文档已约定「Vault 不可用时应用静默回退 .env / 环境变量」，现状是被接受的基线

## 5. 关联文档与既有约定

| 文档/约定 | 说明 |
|-----------|------|
| `docs/dev-guide/iac/secrets.md` | 密钥管理架构方向（现为 Vault 架构，待随决策更新；KMS Auto-unseal 描述已过时） |
| AGENTS.md「Vault 密钥命名风格」 | Vault 路径标识提供商/范围（如 `secret/deepseek`），key 简短自描述（如 `api_key`），不与应用字段名强行一致 |
| `docs/dev-guide/iac/deployment-standards.md` | 现有 secrets 清单（GitHub org secrets / FC 环境变量）与账号归属 |
| `docs/dev-guide/iac/index.md` | 计算原则（FaaS 为主，2026-08-17 更新）、供应商可互换原则 |
| qtcloud-knowl | LLM API key 消费方：环境变量优先，Vault 路径 `quanttide/deepseek` 密钥 `api_key` 兜底 |

## 6. 决策状态与下一步

- **2026-08-17**：首轮调研完成，方向待定。已排除：阿里云 KMS（价格）、Infisical 云版（境外）、FC（不可行）。
- **候选收敛**：自托管 Infisical（推荐）/ SOPS + age / Vault 自托管（去 KMS）/ 现状。
- **建议下一步**：若痛点仅为「密钥散落、统一管理」→ SOPS + age（一天落地，永久免费）；若需「密钥服务 API、多环境隔离、审计」→ 自托管 Infisical。决策后同步更新 `secrets.md` 方向与 IaC 代码。
