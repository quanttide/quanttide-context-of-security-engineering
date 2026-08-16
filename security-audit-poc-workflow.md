# 漏洞管理与安全审计 PoC 工作流总结

> 来源：安全工程实验室 PoC-001（qtcloud-secret @ `cec7701`，15 条发现）与 PoC-002（qtcloud-auth @ `f017ffa`，18 条发现）实战沉淀。
> 状态：**已验证可复用**（同一模板在两个对象上零结构调整跑通）。

## 1. 工作流全景

```
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ P0 规划  │→│ P1 准备  │→│ P2 扫描  │→│ P3 审计  │→│ P4 台账  │→│ P5 复盘  │→│ P6 落地  │
└─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
  定位/范围    基线锁定     工具矩阵    手工取证    登记评级     差距清单     issue/提交
```

| 阶段 | 输入 | 关键动作 | 产出 |
|------|------|----------|------|
| **P0 规划** | 对象仓库、领域定义 | 定位文档：架构速览、攻击面清单、已知风险基线（如 security.md R1–R11）、扫描矩阵、里程碑 | `docs/poc-*.md` |
| **P1 准备** | 对象仓库 | 锁定基线 commit（防漂移）；安装工具链；确认无真实凭证依赖 | 基线记录 |
| **P2 扫描** | 基线代码 | 8 层扫描矩阵全量执行，原始报告归档 | `scans*/0X-*.json` |
| **P3 审计** | 扫描结果 + 源码 | 逐文件代码审计（文件:行号取证）、STRIDE 威胁建模、基础设施/CI 审计；与已知风险基线逐条对照 | 审计报告 v1 |
| **P4 台账** | 全部发现 | 登记 → CVSS 3.1 + EPSS 评级 → 严重级/优先级 → 状态机 | 漏洞台账 v1 |
| **P5 复盘** | 台账/报告 | 目标达成度、工具有效性、误报处置、方法复用性评估、差距清单 | 复盘报告 v1 |
| **P6 落地** | 全部产物 | GitHub issues（标签 + 分配）→ 状态机联动；分层提交（lab → domain → root） | issues + 提交链 |

## 2. 扫描矩阵（8 层，按对象裁剪）

| 层面 | 工具 | 目标 | 安装 | 备注 |
|------|------|------|------|------|
| Go 依赖漏洞 | govulncheck | src/provider | `go install golang.org/x/vuln/cmd/govulncheck@latest` | 符号级可达性；**输出为多文档 JSON 流，需流式解析** |
| 全依赖匹配 | osv-scanner | go.mod / pubspec.lock | `go install github.com/google/osv-scanner/cmd/osv-scanner@latest` | 锁文件全量；stdlib 版本取 go 指令有偏差 |
| Go 静态分析 | gosec | src/provider | `go install github.com/securego/gosec/v2/cmd/gosec@latest` | 与手工审计互证率高 |
| 语义规则 | semgrep | 全仓 | PyPI wheel（Python ≤3.12）或 Docker 镜像 | **本环境受限**（无 wheel、镜像拉取失败），待 CI 补跑 |
| 密钥泄露 | gitleaks | 全历史 | `go install github.com/zricethezav/gitleaks/v8@latest`（**注意模块路径已变更**） | 命中需人工判定一方程 vs 三方夹具 |
| IaC | tfsec / trivy misconfig | manifests/terraform | `go install github.com/aquasecurity/tfsec/cmd/tfsec@latest` | 加固到位的仓库为 0 发现 |
| 镜像/FS | trivy | 仓库全量 | 官方安装脚本 `install.sh -b ~/bin` | 首次运行需下载漏洞库 |
| Dart SAST | dart analyze | Flutter 客户端 | Flutter SDK 自带 | 无 studio 的对象跳过该层 |

**关键命令形态**（注意：输出路径用**绝对路径**，避免写入目标仓库树污染工作区）：

```bash
export PATH=$PATH:/home/iguo/go/bin
cd <目标>/src/provider   # 模块上下文
govulncheck -format json ./... > <lab>/scans/01-govulncheck.json
gosec -fmt json -out <lab>/scans/03-gosec.json ./...
osv-scanner --format json -r . > <lab>/scans/02-osv-scanner.json
gitleaks git --report-format json --report-path <lab>/scans/04-gitleaks.json --log-opts "--all" <目标>
tfsec --format json --out <lab>/scans/05-tfsec.json <目标>/manifests/terraform
trivy fs --scanners vuln,misconfig,secret --format json -o <lab>/scans/06-trivy-fs.json <目标>
```

## 3. 审计方法论

### 3.1 代码审计（手工取证）

- **逐文件审计 + 工具交叉验证**：每条发现须有 `文件:行号` 代码级证据；gosec/G115/G114/G101 等命中与手工发现互证
- **按对象定制重点**（同一模板，不同侧重）：
  - 机密云：加密实现、导出面、密钥可见面
  - 身份云：密码哈希、JWT 签发/验签、OAuth2 流程、SMS 验证码、管理员端点、DB 连接
- **已知风险基线对照法**：对象文档中的风险清单（如 security.md R1–R11）逐条给结论：`已验证修复 ✓ / 已缓解 ✓ / 确认未修复 ✗ / 已接受 / 新增`

### 3.2 威胁建模（STRIDE × 资产）

资产维度（对象相关）：密文/明文数据、令牌/会话、凭证、审计日志、管理端点。每资产一行 STRIDE 六列，输出「主要风险 + 状态」。

### 3.3 基础设施与供应链审计

- tfstate 明文凭据、OSS/RDS 加密与 ACL、FC 环境变量可见面、RAM 权限最小化
- CI：actions 是否锁 SHA、job `permissions` 收敛、`environment: production` 审批、AK 权限宽度

### 3.4 误报处置（审计严谨性）

| 案例 | 结论方法 |
|------|----------|
| gitleaks 命中 `.gocache/mod/**` 第三方测试密钥 | 检查文件路径归属 → 非一方程泄露 → 归类为仓库卫生问题（A-11） |
| OSS 404 类型断言疑云 | 读 SDK 源码（error.go 值类型返回）→ 确认断言正确 → **排除** |

## 4. 评级体系

- **CVSS 3.1**：漏洞库直接取；代码级发现按规范人工评分
- **EPSS**：`https://api.first.org/data/v1/epss?cve=<CVE>`（代表性 CVE 查询即可）
- **严重级**：Critical / High / Medium / Low / Info（对应 issue 标签 `security|*`）
- **优先级**：P0（依赖漏洞、配置守卫、核心爆破面）→ P1（认证/数据面）→ P2（纵深防御）
- **状态机**：`新发现 → 分析中 → 待修复 → 修复中 → 复测中 → 已关闭`（`已接受` 用于残余项）
- 台账字段：`ID / 来源层 / 工具 / CVE / 组件与版本 / CVSS / EPSS / 利用条件 / 严重级 / 状态 / 证据 / 修复建议`

## 5. 模板结构（零调整复用）

- **漏洞台账**：汇总表 + 依赖漏洞（V-xx）+ 代码审计发现（A-xx，每条含证据表）+ 扫描矩阵对照 + 已验证良好面
- **审计报告**：执行摘要（Top 结论）+ 范围方法 + 发现明细表 + STRIDE 结论 + 风险基线对照 + P0–P2 整改清单 + 残余风险
- **复盘报告**：目标达成度表 + 关键发现 + 方法学复盘（工具有效性/环境限制/误报处置）+ 差距清单（→安全云能力输入）+ 后续建议

## 6. issue 落地规范

- **粒度**：每条可独立修复的发现一个 issue（已接受/不适用项不提，避免噪声）
- **标签**：`poc-XXX`（审计批次）+ `security|high/medium/low/info`（严重级）
- **正文**：来源 + 严重级/优先级 + 类型（CWE）+ 问题描述 + 证据（文件:行号）+ 影响 + 修复建议 + 实验室报告链接
- **分配**：处理人统一分配（如 @LiXiang050789）；issue 关闭 = 台账状态机「复测中 → 已关闭」触发点
- **复盘报告**：补记 issue 编号、分配人、未提项说明

## 7. 分层提交规范

```
子模块仓库（lab / data/context）提交推送 → 领域仓库（指针 + CHANGELOG）提交推送 → 根仓库（指针）提交推送
```

- 产物只进实验室仓库（`scans*/`、`findings*/`、`docs/`）；**目标对象仓库工作区不得留任何产物**
- 领域 CHANGELOG 记录子模块变更（新增/执行完成）

## 8. 经验教训（实战坑位）

1. **输出路径必须绝对路径**：扫描命令若 `cd` 失败，产物会写入目标仓库树，污染对象子模块工作区（发生过，已归位）
2. **govulncheck JSON 是多文档流**：`json.load` 单文档解析会报 `Extra data`；用 `JSONDecoder().raw_decode` 循环解析
3. **gitleaks 模块路径已变**：`github.com/gitleaks/gitleaks/v8` → `github.com/zricethezav/gitleaks/v8`（go install 报 module path mismatch）
4. **semgrep 环境依赖重**：Python 3.14 无 wheel；Docker 镜像 ~1.5GB 拉取易断——SAST 层由 gosec + dart analyze 兜底，semgrep 记录待 CI 补跑
5. **工具版本与对象偏差**：govulncheck 的 stdlib 发现基于**扫描用工具链**版本，与对象 `go 1.25.0` 指令/构建镜像有偏差——结论按「升级工具链 + 固定镜像 digest」聚合处理
6. **stdout/stderr 分离**：工具进度输出与 JSON 混流时，重定向 stderr 或使用 `-o` 输出文件
7. **标签名含冒号/管道符**：gh label 命令行解析易错位，用 `cut -d'|'` 等分隔符规避
8. **瞬时失败复核**：批量分配/创建时个别报 FAILED，需用 `gh issue view` 单条复核（出现过误报）

## 9. 复用指引（第三个对象怎么跑）

1. 复制 `docs/poc-*.md` 规划模板 → 改对象架构/攻击面/审计重点
2. 按对象裁剪扫描矩阵（无 studio 跳 dart 层；无 IaC 跳 tfsec）
3. 锁定基线 commit → P2 扫描（绝对路径输出）→ P3 手工审计（对象定制重点 + 风险基线对照）
4. P4 台账 → P5 复盘（重点评估**模板复用性**与新增经验）→ 更新本文档
5. P6 issue 落地 + 分层提交

## 10. 现状与后续

| 项 | 状态 |
|----|------|
| PoC-001（qtcloud-secret） | ✅ 完成：15 条发现，issues `#1–#15` |
| PoC-002（qtcloud-auth） | ✅ 完成：18 条发现，issues `#1–#16`（2 项已接受未提） |
| 方法复用性 | ✅ 验证通过（零结构调整） |
| 待办 | semgrep CI 补跑；扫描矩阵固化 `run-all.sh`；季度工具链巡检；对象 P0–P2 整改复测闭环 |
