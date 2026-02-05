# GitHub Actions Workflow 重用机制深度架构剖析

## 📋 目录

1. [架构概览](#1-架构概览)
2. [核心概念](#2-核心概念)
3. [代码化定义详解](#3-代码化定义详解)
4. [调用机制与引用模式](#4-调用机制与引用模式)
5. [数据传递架构](#5-数据传递架构)
6. [嵌套与组合设计](#6-嵌套与组合设计)
7. [权限与安全模型](#7-权限与安全模型)
8. [Composite Actions 对比](#8-composite-actions-对比)
9. [最佳实践与设计模式](#9-最佳实践与设计模式)
10. [实际案例分析](#10-实际案例分析)

---

## 1. 架构概览

### 1.1 系统分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         调用层 (Caller Layer)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Workflow A   │  │ Workflow B   │  │ Workflow C   │              │
│  │ (uses: ...)  │  │ (uses: ...)  │  │ (uses: ...)  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
└─────────┼──────────────────┼──────────────────┼─────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       重用层 (Reusable Layer)                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Reusable Workflow Repository                     │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐                │  │
│  │  │ ci.yml     │ │ deploy.yml │ │ test.yml   │                │  │
│  │  │ on:        │ │ on:        │ │ on:        │                │  │
│  │  │ workflow_  │ │ workflow_  │ │ workflow_  │                │  │
│  │  │   call     │ │   call     │ │   call     │                │  │
│  │  └────────────┘ └────────────┘ └────────────┘                │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        执行层 (Execution Layer)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ GitHub-Hosted│  │ Self-Hosted  │  │   Actions    │              │
│  │   Runner     │  │   Runner     │  │   Runtime    │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心组件关系图

```
                    ┌──────────────────┐
                    │  Caller Workflow │
                    │  (调用方工作流)   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │   Job 1    │ │   Job 2    │ │   Job 3    │
     │ (独立执行)  │ │ (uses: ...)│ │ (独立执行)  │
     └────────────┘ └─────┬──────┘ └────────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Reusable Workflow│
                 │  (被调用工作流)   │
                 │  on:workflow_call │
                 └────────┬────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌────────┐  ┌────────┐  ┌────────┐
        │ Job A  │  │ Job B  │  │ Job C  │
        │(子任务)│  │(子任务)│  │(子任务)│
        └────────┘  └────────┘  └────────┘
```

---

## 2. 核心概念

### 2.1 关键术语定义

| 术语 | 英文 | 定义 |
|------|------|------|
| **可重用工作流** | Reusable Workflow | 通过 `on: workflow_call` 事件触发器定义的 YAML 文件，可被其他工作流调用 |
| **调用方工作流** | Caller Workflow | 使用 `uses` 关键字调用其他工作流的工作流 |
| **被调用工作流** | Called Workflow | 被调用执行的可重用工作流 |
| **复合动作** | Composite Action | 将多个步骤打包为单个动作的机制 |
| **工作流调用事件** | workflow_call | 使工作流可被其他工作流调用的事件类型 |

### 2.2 设计哲学

GitHub Actions 的工作流重用机制遵循以下设计原则：

1. **声明式配置**：通过 YAML 声明式定义，而非编程式调用
2. **版本控制**：通过 Git 引用（SHA、Tag、Branch）实现不可变版本
3. **接口契约**：通过 `inputs` 和 `outputs` 定义清晰的接口边界
4. **权限继承**：遵循最小权限原则，支持权限向下传递

---

## 3. 代码化定义详解

### 3.1 基础结构定义

```yaml
# ============================================
# 可重用工作流定义模板 (reusable-workflow.yml)
# ============================================

name: Reusable Workflow Template

# 核心：workflow_call 事件触发器
on:
  workflow_call:
    # 输入参数定义
    inputs:
      environment:
        description: '部署环境'
        required: true
        type: string
        default: 'staging'
      
      version:
        description: '应用版本'
        required: false
        type: string
      
      dry_run:
        description: '是否干运行'
        required: false
        type: boolean
        default: false
    
    # 密钥定义
    secrets:
      api_token:
        description: 'API 访问令牌'
        required: true
      
      deploy_key:
        description: '部署密钥'
        required: false
    
    # 输出定义
    outputs:
      deployment_id:
        description: '部署ID'
        value: ${{ jobs.deploy.outputs.id }}
      
      status:
        description: '部署状态'
        value: ${{ jobs.deploy.outputs.status }}

# 环境变量
env:
  GLOBAL_ENV: "shared"

jobs:
  # 作业定义
  deploy:
    runs-on: ubuntu-latest
    
    outputs:
      id: ${{ steps.deploy.outputs.id }}
      status: ${{ steps.deploy.outputs.status }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Deploy Application
        id: deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          echo "id=deploy-123" >> $GITHUB_OUTPUT
          echo "status=success" >> $GITHUB_OUTPUT
        env:
          API_TOKEN: ${{ secrets.api_token }}
```

### 3.2 输入参数类型系统

```yaml
on:
  workflow_call:
    inputs:
      # 字符串类型
      string_input:
        type: string
        required: true
      
      # 数字类型
      number_input:
        type: number
        required: false
        default: 42
      
      # 布尔类型
      boolean_input:
        type: boolean
        required: false
        default: true
      
      # 枚举类型（通过描述约定）
      environment:
        type: string
        description: '环境选择: dev, staging, production'
        required: true
```

### 3.3 高级配置模式

```yaml
on:
  workflow_call:
    # 输入参数分组
    inputs:
      # 基础配置组
      app_name:
        type: string
        description: '应用名称'
        required: true
      
      app_version:
        type: string
        description: '应用版本'
        required: true
      
      # 部署配置组
      deploy_region:
        type: string
        description: '部署区域'
        required: false
        default: 'us-east-1'
      
      deploy_strategy:
        type: string
        description: '部署策略: blue-green, canary, rolling'
        required: false
        default: 'rolling'
      
      # 功能开关组
      enable_monitoring:
        type: boolean
        description: '启用监控'
        required: false
        default: true
      
      enable_notifications:
        type: boolean
        description: '启用通知'
        required: false
        default: false

# 条件执行
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Environment
        run: echo "Setting up ${{ inputs.environment }}"

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    # 条件判断
    if: ${{ inputs.deploy_strategy != 'manual' }}
    steps:
      - name: Execute Deployment
        run: ./scripts/deploy.sh --strategy ${{ inputs.deploy_strategy }}

  notify:
    needs: deploy
    runs-on: ubuntu-latest
    # 条件执行
    if: ${{ inputs.enable_notifications && success() }}
    steps:
      - name: Send Notification
        run: echo "Deployment completed"
```

---

## 4. 调用机制与引用模式

### 4.1 引用语法规范

```yaml
# ============================================
# 调用方工作流 (caller-workflow.yml)
# ============================================

name: Caller Workflow

on:
  push:
    branches: [main]

jobs:
  # 模式1：同一仓库引用
  local-reusable:
    uses: ./.github/workflows/reusable-workflow.yml
  
  # 模式2：其他仓库引用 - 分支
  external-branch:
    uses: org/shared-workflows/.github/workflows/ci.yml@main
  
  # 模式3：其他仓库引用 - 标签
  external-tag:
    uses: org/shared-workflows/.github/workflows/ci.yml@v1.0.0
  
  # 模式4：其他仓库引用 - SHA（推荐）
  external-sha:
    uses: org/shared-workflows/.github/workflows/ci.yml@a1b2c3d4e5f6
  
  # 模式5：传递参数
  with-params:
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
    with:
      environment: 'production'
      version: '1.2.3'
      dry_run: false
    secrets:
      api_token: ${{ secrets.API_TOKEN }}
      deploy_key: ${{ secrets.DEPLOY_KEY }}
  
  # 模式6：继承所有密钥
  inherit-secrets:
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
    secrets: inherit
```

### 4.2 版本控制策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    版本引用策略矩阵                              │
├───────────────┬────────────────────┬────────────────────────────┤
│   引用类型     │       示例         │        适用场景            │
├───────────────┼────────────────────┼────────────────────────────┤
│   Branch      │ @main              │ 开发环境、快速迭代          │
│   Tag         │ @v1.0.0            │ 生产环境、稳定发布          │
│   SHA         │ @abc123            │ 安全敏感、审计要求          │
│   Floating Tag│ @v1                │ 向后兼容、自动更新          │
└───────────────┴────────────────────┴────────────────────────────┘
```

### 4.3 调用时元数据传递

```yaml
jobs:
  # 权限控制
  deploy-with-permissions:
    permissions:
      contents: read
      packages: write
      id-token: write
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
    with:
      environment: 'production'
  
  # 环境配置
  deploy-to-environment:
    environment:
      name: production
      url: https://app.example.com
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
  
  # 并发控制
  deploy-with-concurrency:
    concurrency:
      group: deploy-${{ inputs.environment }}
      cancel-in-progress: true
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
```

---

## 5. 数据传递架构

### 5.1 输入输出流

```
┌─────────────────────────────────────────────────────────────────────┐
│                        数据流架构图                                  │
└─────────────────────────────────────────────────────────────────────┘

 Caller Workflow                            Reusable Workflow
┌──────────────────────┐                  ┌──────────────────────┐
│                      │                  │                      │
│  ┌────────────────┐  │   inputs         │  ┌────────────────┐  │
│  │ with:          │──────────────────────▶│  │ on:            │  │
│  │   env: prod    │  │                  │  │   workflow_call│  │
│  │   version: 1.0 │  │                  │  │   inputs:      │  │
│  └────────────────┘  │                  │  │     env:       │  │
│                      │                  │  │     version:   │  │
│  ┌────────────────┐  │   outputs        │  └────────────────┘  │
│  │ needs:         │◀──────────────────────│  ┌────────────────┐  │
│  │   job.outputs  │  │                  │  │ outputs:       │  │
│  │     result     │  │                  │  │   result:      │  │
│  └────────────────┘  │                  │  │     value:     │  │
│                      │                  │  └────────────────┘  │
└──────────────────────┘                  └──────────────────────┘
```

### 5.2 完整数据传递示例

```yaml
# ========== 被调用工作流 (reusable.yml) ==========
name: Reusable Workflow with I/O

on:
  workflow_call:
    inputs:
      build_config:
        type: string
        required: true
    outputs:
      artifact_name:
        description: '生成的制品名称'
        value: ${{ jobs.build.outputs.name }}
      artifact_path:
        description: '制品路径'
        value: ${{ jobs.build.outputs.path }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      name: ${{ steps.build.outputs.name }}
      path: ${{ steps.build.outputs.path }}
    steps:
      - id: build
        run: |
          echo "name=my-app-${{ inputs.build_config }}" >> $GITHUB_OUTPUT
          echo "path=dist/my-app" >> $GITHUB_OUTPUT

# ========== 调用方工作流 (caller.yml) ==========
name: Caller with I/O

on:
  push:
    branches: [main]

jobs:
  call-reusable:
    uses: ./.github/workflows/reusable.yml
    with:
      build_config: 'production'

  use-outputs:
    needs: call-reusable
    runs-on: ubuntu-latest
    steps:
      - name: Use Outputs
        run: |
          echo "Artifact: ${{ needs.call-reusable.outputs.artifact_name }}"
          echo "Path: ${{ needs.call-reusable.outputs.artifact_path }}"
```

### 5.3 密钥传递机制

```yaml
# 被调用工作流定义
on:
  workflow_call:
    secrets:
      # 显式定义需要的密钥
      api_key:
        description: 'API 密钥'
        required: true
      db_password:
        description: '数据库密码'
        required: false

# 调用方式1：显式传递
jobs:
  explicit-secrets:
    uses: ./.github/workflows/reusable.yml
    secrets:
      api_key: ${{ secrets.API_KEY }}
      db_password: ${{ secrets.DB_PASSWORD }}

# 调用方式2：继承所有密钥（简化模式）
jobs:
  inherit-all-secrets:
    uses: ./.github/workflows/reusable.yml
    secrets: inherit
```

---

## 6. 嵌套与组合设计

### 6.1 嵌套架构限制

```
┌─────────────────────────────────────────────────────────────────────┐
│                     嵌套层级限制（最大10层）                          │
└─────────────────────────────────────────────────────────────────────┘

Level 0: caller-workflow.yml
    │
    ├── Level 1: called-workflow-1.yml
    │       │
    │       └── Level 2: called-workflow-2.yml
    │               │
    │               └── Level 3: called-workflow-3.yml
    │                       │
    │                       └── ... (最多到 Level 9)
    │
    └── Level 1: another-workflow.yml
            │
            └── Level 2: nested-workflow.yml

限制规则：
- 最大嵌套深度：10层（调用方 + 9层被调用方）
- 不允许循环依赖
- 每个工作流文件最多调用50个不同的可重用工作流
```

### 6.2 嵌套工作流示例

```yaml
# ========== Level 0: 顶层调用 ==========
name: Top Level Workflow

on:
  push:
    branches: [main]

jobs:
  level-1-call:
    uses: ./.github/workflows/level-1.yml
    with:
      param: "from-top"

# ========== Level 1: 第一层被调用 ==========
name: Level 1 Workflow

on:
  workflow_call:
    inputs:
      param:
        type: string

jobs:
  process:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Level 1: ${{ inputs.param }}"
  
  # 嵌套调用 Level 2
  level-2-call:
    needs: process
    uses: ./.github/workflows/level-2.yml
    with:
      param: "from-level-1"

# ========== Level 2: 第二层被调用 ==========
name: Level 2 Workflow

on:
  workflow_call:
    inputs:
      param:
        type: string

jobs:
  deep-process:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Level 2: ${{ inputs.param }}"
```

### 6.3 矩阵与可重用工作流组合

```yaml
name: Matrix with Reusable Workflows

on:
  push:
    branches: [main]

jobs:
  # 矩阵策略调用可重用工作流
  matrix-deploy:
    strategy:
      matrix:
        environment: [dev, staging, production]
        region: [us-east-1, eu-west-1, ap-southeast-1]
        include:
          - environment: production
            region: us-west-2
    uses: ./.github/workflows/deploy.yml
    with:
      environment: ${{ matrix.environment }}
      region: ${{ matrix.region }}
```

---

## 7. 权限与安全模型

### 7.1 权限继承架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                      权限传播模型                                    │
└─────────────────────────────────────────────────────────────────────┘

Caller Workflow                          Called Workflow
┌─────────────────────┐                 ┌─────────────────────┐
│ permissions:        │                 │ permissions:        │
│   contents: write   │────────────────▶│   contents: write   │
│   packages: read    │                 │   packages: read    │
│                     │                 │                     │
│   (可缩小权限)       │                 │   (不可扩大权限)     │
│                     │                 │                     │
│   例如：             │                 │   例如：            │
│   contents: read    │────────────────▶│   contents: read    │
│                     │  (缩小为只读)    │   packages: read    │
└─────────────────────┘                 └─────────────────────┘

规则：
1. 权限只能缩小，不能扩大
2. 未明确设置的权限默认为无权限
3. GITHUB_TOKEN 权限在嵌套中传递
```

### 7.2 访问控制矩阵

| 调用方仓库类型 | 被调用方仓库类型 | 访问权限 |
|----------------|------------------|----------|
| Private | Private（同仓库） | ✅ 允许 |
| Private | Private（同组织） | ✅ 允许（需配置访问策略） |
| Private | Public | ✅ 允许 |
| Public | Private | ❌ 不允许 |
| Public | Public | ✅ 允许 |

### 7.3 安全最佳实践

```yaml
# 安全的工作流调用模式

name: Secure Caller Workflow

on:
  push:
    branches: [main]

permissions:
  contents: read  # 最小权限原则

jobs:
  # 实践1：使用 SHA 而非标签（防止标签篡改）
  secure-by-sha:
    uses: org/shared-workflows/.github/workflows/ci.yml@a1b2c3d4e5f6...
  
  # 实践2：显式声明权限
  explicit-permissions:
    permissions:
      contents: read
      packages: write
      id-token: write  # OIDC 令牌
    uses: org/shared-workflows/.github/workflows/deploy.yml@v1
    secrets: inherit
  
  # 实践3：使用 OpenID Connect (OIDC)
  oidc-auth:
    permissions:
      id-token: write
      contents: read
    uses: org/shared-workflows/.github/workflows/aws-deploy.yml@v1
    with:
      role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
```

---

## 8. Composite Actions 对比

### 8.1 特性对比矩阵

| 特性 | Reusable Workflows | Composite Actions |
|------|---------------------|-------------------|
| **文件位置** | `.github/workflows/` | `.github/actions/` 或独立仓库 |
| **定义文件** | `*.yml` | `action.yml` |
| **包含内容** | 完整的 jobs、steps | 仅 steps（无 jobs） |
| **调用方式** | `jobs.<job_id>.uses` | `steps.<step_id>.uses` |
| **运行器指定** | ✅ 可指定不同 runner | ❌ 使用调用方的 runner |
| **日志粒度** | ✅ 每个 step 独立显示 | ❌ 整体显示为一个 step |
| **密钥访问** | ✅ 支持 secrets | ❌ 不支持 secrets |
| **市场发布** | ❌ 不支持 | ✅ 支持 GitHub Marketplace |
| **嵌套深度** | 10层工作流 | 10层 composite actions |
| **适用场景** | 完整 CI/CD 流程 | 步骤组合、工具封装 |

### 8.2 选择决策树

```
                    ┌─────────────────────┐
                    │  需要复用代码？      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ 需要多 job 编排？    │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │ Yes            │                │ No
              ▼                │                ▼
    ┌──────────────────┐       │       ┌──────────────────┐
    │ Reusable Workflow│       │       │ Composite Action │
    └──────────────────┘       │       └──────────────────┘
                               │
              ┌────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ 需要跨 runner 执行？ │
    └──────────┬──────────┘
               │
     ┌─────────┴─────────┐
     │ Yes               │ No
     ▼                   ▼
┌──────────────┐  ┌──────────────────┐
│ Reusable WF  │  │ Composite Action │
│ (不同 runner) │  │ 或 Reusable WF   │
└──────────────┘  └──────────────────┘
```

### 8.3 混合使用模式

```yaml
# ============================================
# 混合模式：Reusable Workflow + Composite Action
# ============================================

name: Hybrid Workflow Pattern

on:
  push:
    branches: [main]

jobs:
  # 使用 Reusable Workflow 进行整体编排
  ci-pipeline:
    uses: org/shared-workflows/.github/workflows/ci-pipeline.yml@v1
    with:
      node_version: '18'
    secrets: inherit

  # 在 job 中使用 Composite Action
  custom-job:
    runs-on: ubuntu-latest
    steps:
      # Composite Action：环境设置
      - name: Setup Development Environment
        uses: ./.github/actions/setup-dev-env
        with:
          node-version: '18'
          install-deps: true
      
      # Composite Action：代码质量检查
      - name: Quality Checks
        uses: ./.github/actions/quality-checks
        with:
          run-lint: true
          run-test: true
      
      # 普通 step
      - name: Build
        run: npm run build
```

---

## 9. 最佳实践与设计模式

### 9.1 模块化设计模式

```yaml
# ============================================
# 模式1：分层架构设计
# ============================================

# .github/workflows/
# ├── foundation/
# │   ├── setup.yml          # 基础环境设置
# │   ├── checkout.yml       # 代码检出
# │   └── cache.yml          # 缓存管理
# ├── quality/
# │   ├── lint.yml           # 代码检查
# │   ├── test.yml           # 测试执行
# │   └── security-scan.yml  # 安全扫描
# ├── build/
# │   ├── docker-build.yml   # Docker 构建
# │   ├── package.yml        # 打包
# │   └── sign.yml           # 签名
# └── deploy/
    ├── deploy-k8s.yml       # K8s 部署
    ├── deploy-vm.yml        # VM 部署
    └── deploy-lambda.yml    # Lambda 部署

# ============================================
# 基础工作流：setup.yml
# ============================================
name: Foundation - Setup

on:
  workflow_call:
    inputs:
      node_version:
        type: string
        default: '18'
      
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}
      - uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

# ============================================
# 复合工作流：完整 CI/CD 流程
# ============================================
name: Complete CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # 基础设置
  setup:
    uses: ./.github/workflows/foundation/setup.yml
    with:
      node_version: '18'
  
  # 质量检查
  quality:
    needs: setup
    uses: ./.github/workflows/quality/lint.yml
  
  test:
    needs: setup
    uses: ./.github/workflows/quality/test.yml
  
  security:
    needs: setup
    uses: ./.github/workflows/quality/security-scan.yml
  
  # 构建
  build:
    needs: [quality, test, security]
    uses: ./.github/workflows/build/docker-build.yml
    with:
      push_image: ${{ github.ref == 'refs/heads/main' }}
  
  # 部署
  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/deploy/deploy-k8s.yml
    with:
      environment: 'staging'
    secrets: inherit
  
  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/deploy/deploy-k8s.yml
    with:
      environment: 'production'
    secrets: inherit
```

### 9.2 配置驱动设计

```yaml
# ============================================
# 模式2：配置驱动的可重用工作流
# ============================================

name: Configurable Workflow

on:
  workflow_call:
    inputs:
      # 通过配置对象驱动行为
      config:
        type: string
        description: 'JSON 格式的配置'
        required: true

jobs:
  execute:
    runs-on: ubuntu-latest
    steps:
      - name: Parse Configuration
        id: config
        run: |
          echo "config=${{ inputs.config }}" | jq -r 'to_entries[] | "\(.key)=\(.value)"' >> $GITHUB_OUTPUT
      
      - name: Execute Based on Config
        run: |
          if [ "${{ fromJSON(inputs.config).skip_tests }}" != "true" ]; then
            npm test
          fi
          
          if [ "${{ fromJSON(inputs.config).build_target }}" = "production" ]; then
            npm run build:prod
          else
            npm run build:dev
          fi

# 调用示例
jobs:
  deploy:
    uses: ./.github/workflows/configurable.yml
    with:
      config: |
        {
          "environment": "production",
          "skip_tests": false,
          "build_target": "production",
          "notify_team": true
        }
```

### 9.3 版本管理策略

```
┌─────────────────────────────────────────────────────────────────────┐
│                     语义化版本管理策略                               │
└─────────────────────────────────────────────────────────────────────┘

仓库结构：
.github-workflows/
├── ci/
│   ├── v1/
│   │   └── workflow.yml      # 稳定版本 v1.x
│   ├── v2/
│   │   └── workflow.yml      # 稳定版本 v2.x
│   └── beta/
│       └── workflow.yml      # 测试版本
└── deploy/
    ├── v1/
    │   └── workflow.yml
    └── v2/
        └── workflow.yml

引用方式：
- 生产环境：uses: org/workflows/.github/workflows/ci/v1/workflow.yml@v1
- 开发测试：uses: org/workflows/.github/workflows/ci/beta/workflow.yml@main
```

### 9.4 错误处理与重试机制

```yaml
name: Resilient Workflow

on:
  workflow_call:
    inputs:
      max_retries:
        type: number
        default: 3

jobs:
  resilient-job:
    runs-on: ubuntu-latest
    steps:
      - name: Attempt with Retry
        uses: nick-fields/retry@v3
        with:
          timeout_minutes: 10
          max_attempts: ${{ inputs.max_retries }}
          command: npm run deploy
          retry_wait_seconds: 30
      
      - name: Failure Handler
        if: failure()
        run: |
          echo "::error::Deployment failed after ${{ inputs.max_retries }} attempts"
          # 发送告警通知
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{"text":"Deployment failed: ${{ github.repository }}"}'
```

---

## 10. 实际案例分析

### 10.1 企业级 CI/CD 平台设计

```yaml
# ============================================
# 场景：多团队共享 CI/CD 平台
# ============================================

# 中央仓库：org/shared-ci-platform
# ├── .github/
# │   └── workflows/
# │       ├── ci-node.yml
# │       ├── ci-python.yml
# │       ├── ci-go.yml
# │       ├── security-scan.yml
# │       ├── deploy-aws.yml
# │       └── notify.yml
# └── README.md

# ============================================
# Node.js CI 模板
# ============================================
name: '[Shared] Node.js CI'

on:
  workflow_call:
    inputs:
      node_versions:
        type: string
        default: '["18", "20"]'
      run_e2e:
        type: boolean
        default: false
    outputs:
      test_results:
        value: ${{ jobs.test.outputs.results }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: ${{ fromJSON(inputs.node_versions) }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
    outputs:
      results: ${{ steps.test.outputs.results }}

  e2e:
    if: ${{ inputs.run_e2e }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:e2e

# ============================================
# 应用仓库调用示例
# ============================================
name: Application CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # 使用共享 CI 工作流
  ci:
    uses: org/shared-ci-platform/.github/workflows/ci-node.yml@v2
    with:
      node_versions: '["18", "20", "22"]'
      run_e2e: true
  
  # 使用共享安全扫描
  security:
    needs: ci
    uses: org/shared-ci-platform/.github/workflows/security-scan.yml@v2
    secrets: inherit
  
  # 使用共享部署
  deploy:
    needs: [ci, security]
    if: github.ref == 'refs/heads/main'
    uses: org/shared-ci-platform/.github/workflows/deploy-aws.yml@v2
    with:
      environment: production
    secrets: inherit
  
  # 使用共享通知
  notify:
    needs: deploy
    if: always()
    uses: org/shared-ci-platform/.github/workflows/notify.yml@v2
    with:
      status: ${{ needs.deploy.result }}
    secrets: inherit
```

### 10.2 多环境部署流水线

```yaml
# ============================================
# 场景：多环境渐进式部署
# ============================================

name: Progressive Deployment Pipeline

on:
  push:
    tags: ['v*']

jobs:
  # 构建阶段
  build:
    uses: ./.github/workflows/build.yml
    with:
      version: ${{ github.ref_name }}
  
  # 开发环境 - 自动部署
  deploy-dev:
    needs: build
    uses: ./.github/workflows/deploy.yml
    with:
      environment: development
      auto_approve: true
    secrets: inherit
  
  # 冒烟测试
  smoke-test-dev:
    needs: deploy-dev
    uses: ./.github/workflows/smoke-test.yml
    with:
      environment: development
      endpoint: https://dev.example.com
  
  # 测试环境 - 自动部署
  deploy-staging:
    needs: smoke-test-dev
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
      auto_approve: true
    secrets: inherit
  
  # 集成测试
  integration-test:
    needs: deploy-staging
    uses: ./.github/workflows/integration-test.yml
    with:
      environment: staging
  
  # 预生产环境 - 需要审批
  deploy-preprod:
    needs: integration-test
    uses: ./.github/workflows/deploy.yml
    with:
      environment: pre-production
      auto_approve: false
    secrets: inherit
  
  # 生产环境 - 需要审批 +  Canary 发布
  deploy-production:
    needs: deploy-preprod
    uses: ./.github/workflows/deploy-canary.yml
    with:
      environment: production
      canary_percentage: 10
    secrets: inherit
  
  # 渐进式提升流量
  promote-canary:
    needs: deploy-production
    strategy:
      matrix:
        percentage: [25, 50, 100]
    uses: ./.github/workflows/promote-canary.yml
    with:
      percentage: ${{ matrix.percentage }}
    secrets: inherit
```

---

## 附录

### A. 完整语法速查表

```yaml
on:
  workflow_call:
    inputs:                          # 输入参数
      <input_id>:
        description: <字符串>
        required: <true|false>
        type: <string|number|boolean>
        default: <默认值>
    
    secrets:                         # 密钥定义
      <secret_id>:
        description: <字符串>
        required: <true|false>
    
    outputs:                         # 输出定义
      <output_id>:
        description: <字符串>
        value: <表达式>

jobs:
  <job_id>:
    uses: <workflow_ref>            # 工作流引用
    with:                           # 传递输入
      <input_id>: <值>
    secrets:                        # 传递密钥
      <secret_id>: ${{ secrets.<name> }}
    secrets: inherit               # 继承所有密钥
```

### B. 限制与约束

| 限制项 | 限制值 |
|--------|--------|
| 最大嵌套深度 | 10 层 |
| 单个文件可调用工作流数 | 50 个 |
| 输入参数数量 | 无明确限制（建议 < 20）|
| 输出参数数量 | 无明确限制（建议 < 10）|
| 密钥名称长度 | 100 字符 |
| 工作流文件大小 | 无明确限制 |

### C. 参考链接

- [GitHub Docs - Reusing Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Reusable Workflows Reference](https://docs.github.com/en/actions/reference/reusable-workflows-reference)

---

*文档版本: 1.0.0*
*最后更新: 2026-02-05*
