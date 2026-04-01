# 从源码部署 Billing MCP Server 到 AgentCore Runtime 并接入 Quick Suite（Service Authentication）

## 1. 方案概述

### 1.1 目标

从源码部署 [awslabs.billing-cost-management-mcp-server](https://github.com/awslabs/mcp/tree/main/src/billing-cost-management-mcp-server) 到 Amazon Bedrock AgentCore Runtime，并通过 Amazon Quick Suite 的 Chat Agent 以 Service Authentication (2LO) 方式调用，使业务用户能在对话界面中直接查询 AWS 账单与成本数据。

从源码部署的优势：
- 直接修改工具实现逻辑（如自定义过滤条件、调整返回格式、修改默认 Metric）
- 添加新工具或移除不需要的工具
- 修改 prompts 模板
- 便于调试和排查问题
- 不受 PyPI 发布周期限制

### 1.2 架构

```
Amazon Quick Suite Chat Agent (MCP 客户端)
        │
        │  HTTPS (streamable-http + OAuth 2.0 / 2LO client_credentials)
        ▼
Amazon Bedrock AgentCore Runtime
        │
        │  容器内部 0.0.0.0:8000/mcp
        ▼
billing-cost-management-mcp-server (ARM64 容器，源码构建)
        │
        │  boto3 (使用执行角色的临时凭证)
        ▼
AWS Cost Explorer / Budgets / Compute Optimizer / ... 等 API
```

### 1.3 关键约束

| 约束项 | 要求 |
|--------|------|
| 传输协议 | streamable-http（stateless 模式） |
| 监听地址 | `0.0.0.0:8000`，路径 `/mcp` |
| 容器架构 | ARM64（AWS Graviton），由 CodeBuild 自动构建 |
| 认证方式 | OAuth 2.0 JWT Bearer Token（Cognito） |
| Quick Suite 认证 | Service authentication (2LO)，需将 Quick Suite 的 M2M Client ID 加入 AgentCore 的 allowedClients |

> 参考文档：
> - [Deploy MCP servers in AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp.html)
> - [MCP protocol contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp-protocol-contract.html)
> - [Amazon Quick Suite MCP integration](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)

---

## 2. 前置条件

- Python 3.10+、git
- AWS CLI v2 已配置凭证
- pip、jq 已安装
- Amazon Quick Suite Enterprise 订阅，用户拥有 Author Pro 角色

```bash
aws sts get-caller-identity
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 3. 获取源码并准备项目

### 步骤 1：克隆源码

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/awslabs/mcp.git /tmp/awslabs-mcp
cd /tmp/awslabs-mcp
git sparse-checkout set src/billing-cost-management-mcp-server

# 将源码复制到工作目录
mkdir -p ~/billing-mcp-source
cp -r src/billing-cost-management-mcp-server/awslabs ~/billing-mcp-source/
cp src/billing-cost-management-mcp-server/pyproject.toml ~/billing-mcp-source/
cp src/billing-cost-management-mcp-server/__init__.py ~/billing-mcp-source/
cp src/billing-cost-management-mcp-server/README.md ~/billing-mcp-source/
cp src/billing-cost-management-mcp-server/LICENSE ~/billing-mcp-source/ 2>/dev/null
cp src/billing-cost-management-mcp-server/NOTICE ~/billing-mcp-source/ 2>/dev/null

cd ~/billing-mcp-source
rm -rf /tmp/awslabs-mcp
```

### 步骤 2：适配 AgentCore Runtime

源码中的 `server.py` 默认使用 stdio 传输，需要修改两处以适配 AgentCore Runtime：

1. 在所有 import 之前设置日志环境变量（避免容器内权限问题）
2. 将 `mcp.run()` 改为 `streamable-http` 传输，监听 `0.0.0.0:8000`

需要修改三个文件：

#### 修改 1：`awslabs/billing_cost_management_mcp_server/utilities/logging_utils.py`

`__init__.py` 在模块加载时会通过 import 链触发 `logging_utils.py` 的 `configure_logging()`，这比 `server.py` 中任何代码都早执行。原始代码中 `get_server_directory()` 尝试在源码目录下创建 `logs/` 文件夹，容器内非 root 用户没有写权限会直接报 `PermissionError`。

修改 `get_server_directory()` 函数，添加 `PermissionError` 的 fallback：

```python
def get_server_directory() -> Path:
    base_dir = Path(os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__)))))
    log_dir = base_dir / 'logs'
    try:
        log_dir.mkdir(exist_ok=True)
    except PermissionError:
        # 容器内非 root 用户可能没有源码目录的写权限，fallback 到 /tmp
        log_dir = Path('/tmp')
    return log_dir
```

#### 修改 2：`awslabs/billing_cost_management_mcp_server/utilities/sql_utils.py`

同样的问题：`get_session_db_path()` 尝试在源码目录下创建 `sessions/` 文件夹存放 SQLite 数据库。

修改 `get_session_db_path()` 函数中的目录创建逻辑：

```python
        # Create sessions directory if it doesn't exist
        try:
            os.makedirs(session_dir, exist_ok=True)
        except PermissionError:
            # 容器内非 root 用户可能没有源码目录的写权限，fallback 到 /tmp
            session_dir = os.path.join('/tmp', 'billing-mcp-sessions')
            os.makedirs(session_dir, exist_ok=True)
```

#### 修改 3：`awslabs/billing_cost_management_mcp_server/server.py`

```python
# ===== 在文件顶部 import asyncio/os/sys 之后、其他 import 之前，添加 =====
import asyncio
import os
import sys

# --- 新增：AgentCore 容器日志适配 ---
os.environ.setdefault('FASTMCP_LOG_LEVEL', 'ERROR')
os.environ.setdefault('FASTMCP_LOG_FILE', '/tmp/billing-mcp-server.log')
# --- 新增结束 ---

if __name__ == '__main__':
    ...
```

```python
# ===== 将 main() 函数中的 mcp.run() 改为 =====
def main():
    """Main entry point for the server."""
    asyncio.run(setup())
    # 原始：mcp.run()
    mcp.run(transport='streamable-http', host='0.0.0.0', port=8000)
```

#### 修改 4（可选但推荐）：`awslabs/billing_cost_management_mcp_server/tools/cost_explorer_tools.py`

原始代码中 `metrics` 参数类型为 `Optional[str]`，需要传 JSON 数组字符串如 `'["UnblendedCost"]'`。Quick Suite 的 Action Review 界面对这种格式做类型校验时会报 "Validation failed for type"。

将 `metrics` 参数类型改为 `Optional[List[str]]`，同时更新 import 和下游函数：

```python
# cost_explorer_tools.py - 修改 import
from typing import Any, Dict, List, Optional

# cost_explorer_tools.py - 修改参数类型
async def cost_explorer(
    ...
    metrics: Optional[List[str]] = None,  # 原始: Optional[str] = None
    ...
```

同时修改 `cost_explorer_operations.py` 中 `get_cost_and_usage` 和 `get_cost_and_usage_with_resources` 的 `metrics` 参数类型为 `Optional[List[str]]`，并将 `parse_json(metrics, 'metrics')` 替换为 `metrics if metrics else ['UnblendedCost']`。

> 如果你使用的是 `billing-mcp-source/` 项目目录中已准备好的源码，以上修改已经完成。

### 步骤 3：创建 requirements.txt

源码部署时，`requirements.txt` 只列出直接依赖，不包含 MCP Server 包本身（源码会作为项目的一部分被 AgentCore 打包进容器）：

```bash
cat > requirements.txt << 'EOF'
mcp[cli]>=1.23.0
fastmcp>=2.14.0
boto3>=1.34.0
pydantic>=2.10.6
loguru>=0.7.0
python-dotenv>=1.0.0
EOF
```

### 步骤 4：安装依赖

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install bedrock-agentcore-starter-toolkit

# 以可编辑模式安装源码包
# 读取 pyproject.toml 中的依赖并安装，将当前目录作为包源
# 之后对 awslabs/ 目录下源码的修改会立即生效，无需重新 pip install
pip install -e .

agentcore --help
```

项目结构：

```
billing-mcp-source/
├── requirements.txt                           # Python 依赖（不含 MCP Server 包本身）
├── pyproject.toml                             # 源码包定义（来自上游仓库）
├── __init__.py
└── awslabs/
    ├── __init__.py
    └── billing_cost_management_mcp_server/
        ├── server.py                          # 主服务器逻辑 + AgentCore 入口
        ├── models.py
        ├── tools/                             # 各工具实现（可定制）
        │   ├── cost_explorer_tools.py
        │   ├── budget_tools.py
        │   ├── compute_optimizer_tools.py
        │   ├── cost_anomaly_tools.py
        │   ├── cost_comparison_tools.py
        │   ├── cost_optimization_hub_tools.py
        │   ├── free_tier_usage_tools.py
        │   ├── ri_performance_tools.py
        │   ├── sp_performance_tools.py
        │   ├── aws_pricing_tools.py
        │   ├── bcm_pricing_calculator_tools.py
        │   ├── billing_conductor_tools.py
        │   ├── recommendation_details_tools.py
        │   ├── storage_lens_tools.py
        │   ├── unified_sql_tools.py
        │   └── ...
        ├── prompts/                           # Prompt 模板（可定制）
        ├── templates/
        └── utilities/
```

### 步骤 5：本地测试（可选）

```bash
python awslabs/billing_cost_management_mcp_server/server.py

# 新开终端，发送 MCP initialize 请求验证
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

预期返回包含 `serverInfo` 和工具列表的 JSON 响应。

> 注意：MCP 协议要求 Accept header 同时包含 `application/json` 和 `text/event-stream`。

---

## 4. 配置 IAM 执行角色

### 步骤 6：创建信任策略和角色

```bash
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AssumeRolePolicy",
    "Effect": "Allow",
    "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"aws:SourceAccount": "${AWS_ACCOUNT_ID}"},
      "ArnLike": {"aws:SourceArn": "arn:aws:bedrock-agentcore:${AWS_REGION}:${AWS_ACCOUNT_ID}:*"}
    }
  }]
}
EOF

aws iam create-role \
  --role-name BillingMCPServerAgentCoreRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "AgentCore Runtime Role - Billing MCP Server"
```

### 步骤 7：添加 AgentCore 基础权限

```bash
cat > agentcore-base-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Sid": "ECRImageAccess", "Effect": "Allow", "Action": ["ecr:BatchGetImage","ecr:GetDownloadUrlForLayer"], "Resource": ["arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/*"]},
    {"Sid": "ECRTokenAccess", "Effect": "Allow", "Action": ["ecr:GetAuthorizationToken"], "Resource": "*"},
    {"Sid": "LogsCreate", "Effect": "Allow", "Action": ["logs:DescribeLogStreams","logs:CreateLogGroup"], "Resource": ["arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/aws/bedrock-agentcore/runtimes/*"]},
    {"Sid": "LogsDescribe", "Effect": "Allow", "Action": ["logs:DescribeLogGroups"], "Resource": ["arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:*"]},
    {"Sid": "LogsWrite", "Effect": "Allow", "Action": ["logs:CreateLogStream","logs:PutLogEvents"], "Resource": ["arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/aws/bedrock-agentcore/runtimes/*:log-stream:*"]},
    {"Sid": "XRay", "Effect": "Allow", "Action": ["xray:PutTraceSegments","xray:PutTelemetryRecords","xray:GetSamplingRules","xray:GetSamplingTargets"], "Resource": "*"},
    {"Sid": "Metrics", "Effect": "Allow", "Action": "cloudwatch:PutMetricData", "Resource": "*", "Condition": {"StringEquals": {"cloudwatch:namespace": "bedrock-agentcore"}}}
  ]
}
EOF

aws iam put-role-policy \
  --role-name BillingMCPServerAgentCoreRole \
  --policy-name AgentCoreBasePolicy \
  --policy-document file://agentcore-base-policy.json
```

### 步骤 8：添加 Billing and Cost Management API 权限

```bash
cat > billing-mcp-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Sid": "CostExplorer", "Effect": "Allow", "Action": ["ce:GetCostAndUsage","ce:GetCostAndUsageWithResources","ce:GetCostAndUsageComparisons","ce:GetCostComparisonDrivers","ce:GetCostForecast","ce:GetUsageForecast","ce:GetDimensionValues","ce:GetTags","ce:GetCostCategories","ce:GetReservationPurchaseRecommendation","ce:GetReservationCoverage","ce:GetReservationUtilization","ce:GetSavingsPlansUtilization","ce:GetSavingsPlansCoverage","ce:GetSavingsPlansUtilizationDetails","ce:GetSavingsPlansPurchaseRecommendation","ce:GetAnomalies"], "Resource": "*"},
    {"Sid": "CostOptimizationHub", "Effect": "Allow", "Action": ["cost-optimization-hub:GetRecommendation","cost-optimization-hub:ListRecommendations","cost-optimization-hub:ListRecommendationSummaries"], "Resource": "*"},
    {"Sid": "ComputeOptimizer", "Effect": "Allow", "Action": ["compute-optimizer:GetAutoScalingGroupRecommendations","compute-optimizer:GetEBSVolumeRecommendations","compute-optimizer:GetEC2InstanceRecommendations","compute-optimizer:GetECSServiceRecommendations","compute-optimizer:GetRDSDatabaseRecommendations","compute-optimizer:GetLambdaFunctionRecommendations","compute-optimizer:GetEnrollmentStatus","compute-optimizer:GetIdleRecommendations"], "Resource": "*"},
    {"Sid": "Budgets", "Effect": "Allow", "Action": ["budgets:ViewBudget"], "Resource": "*"},
    {"Sid": "Pricing", "Effect": "Allow", "Action": ["pricing:DescribeServices","pricing:GetAttributeValues","pricing:GetProducts"], "Resource": "*"},
    {"Sid": "FreeTier", "Effect": "Allow", "Action": ["freetier:GetFreeTierUsage"], "Resource": "*"},
    {"Sid": "BCMPricingCalc", "Effect": "Allow", "Action": ["bcm-pricing-calculator:GetPreferences","bcm-pricing-calculator:GetWorkloadEstimate","bcm-pricing-calculator:ListWorkloadEstimateUsage","bcm-pricing-calculator:ListWorkloadEstimates"], "Resource": "*"},
    {"Sid": "BillingConductor", "Effect": "Allow", "Action": ["billingconductor:ListBillingGroups","billingconductor:ListBillingGroupCostReports","billingconductor:ListAccountAssociations","billingconductor:ListPricingPlans","billingconductor:ListPricingRules","billingconductor:ListPricingPlansAssociatedWithPricingRule","billingconductor:ListPricingRulesAssociatedToPricingPlan","billingconductor:ListCustomLineItems","billingconductor:ListCustomLineItemVersions","billingconductor:ListResourcesAssociatedToCustomLineItem"], "Resource": "*"}
  ]
}
EOF

aws iam put-role-policy \
  --role-name BillingMCPServerAgentCoreRole \
  --policy-name BillingMCPServerPolicy \
  --policy-document file://billing-mcp-policy.json

export EXECUTION_ROLE_ARN=$(aws iam get-role --role-name BillingMCPServerAgentCoreRole --query 'Role.Arn' --output text)
```

---

## 5. 设置 Cognito 认证

### 步骤 9：创建 User Pool 和测试用户

```bash
export POOL_ID=$(aws cognito-idp create-user-pool \
  --pool-name "BillingMCPServerPool" \
  --policies '{"PasswordPolicy":{"MinimumLength":8}}' \
  --region ${AWS_REGION} | jq -r '.UserPool.Id')

export CLIENT_ID=$(aws cognito-idp create-user-pool-client \
  --user-pool-id ${POOL_ID} \
  --client-name "BillingMCPClient" \
  --no-generate-secret \
  --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
  --region ${AWS_REGION} | jq -r '.UserPoolClient.ClientId')

export COGNITO_USERNAME="mcp-test-user"
export COGNITO_PASSWORD="YourSecurePassword123!"

aws cognito-idp admin-create-user \
  --user-pool-id ${POOL_ID} --username ${COGNITO_USERNAME} \
  --region ${AWS_REGION} --message-action SUPPRESS > /dev/null

aws cognito-idp admin-set-user-password \
  --user-pool-id ${POOL_ID} --username ${COGNITO_USERNAME} \
  --password ${COGNITO_PASSWORD} --region ${AWS_REGION} --permanent > /dev/null

export DISCOVERY_URL="https://cognito-idp.${AWS_REGION}.amazonaws.com/${POOL_ID}/.well-known/openid-configuration"
echo "Discovery URL: ${DISCOVERY_URL}"
echo "Client ID: ${CLIENT_ID}"
```

### 步骤 10：创建 Cognito Domain

```bash
export COGNITO_DOMAIN_PREFIX="billing-mcp-$(echo ${AWS_ACCOUNT_ID} | tail -c 9)"

aws cognito-idp create-user-pool-domain \
  --user-pool-id ${POOL_ID} \
  --domain ${COGNITO_DOMAIN_PREFIX} \
  --region ${AWS_REGION}

echo "Cognito Domain: ${COGNITO_DOMAIN_PREFIX}"
```

### 步骤 11：创建 Resource Server

Service authentication (2LO) 使用 `client_credentials` 授权流程，Cognito 要求必须有 Resource Server 定义 scope。

```bash
aws cognito-idp create-resource-server \
  --user-pool-id ${POOL_ID} \
  --identifier "billing-mcp" \
  --name "Billing MCP Server" \
  --scopes '[{"ScopeName":"invoke","ScopeDescription":"调用 Billing MCP Server"}]' \
  --region ${AWS_REGION}
```

### 步骤 12：创建 Machine-to-Machine App Client（Quick Suite 专用）

```bash
QS_M2M_RESULT=$(aws cognito-idp create-user-pool-client \
  --user-pool-id ${POOL_ID} \
  --client-name "QuickSuiteM2MClient" \
  --generate-secret \
  --allowed-o-auth-flows "client_credentials" \
  --allowed-o-auth-scopes "billing-mcp/invoke" \
  --allowed-o-auth-flows-user-pool-client \
  --supported-identity-providers "COGNITO" \
  --region ${AWS_REGION})

export QS_M2M_CLIENT_ID=$(echo ${QS_M2M_RESULT} | jq -r '.UserPoolClient.ClientId')
export QS_M2M_CLIENT_SECRET=$(echo ${QS_M2M_RESULT} | jq -r '.UserPoolClient.ClientSecret')

echo "M2M Client ID: ${QS_M2M_CLIENT_ID}"
echo "M2M Client Secret: ${QS_M2M_CLIENT_SECRET}"
```

### 步骤 13：获取 Bearer Token（用于 CLI 测试）

```bash
export BEARER_TOKEN=$(aws cognito-idp initiate-auth \
  --client-id "${CLIENT_ID}" \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=${COGNITO_USERNAME},PASSWORD=${COGNITO_PASSWORD} \
  --region ${AWS_REGION} | jq -r '.AuthenticationResult.AccessToken')
```

> Token 有效期默认 1 小时，过期后重新执行上面的命令。

---

## 6. 配置并部署到 AgentCore Runtime

### 步骤 14：运行 agentcore configure

```bash
agentcore configure -e awslabs/billing_cost_management_mcp_server/server.py --protocol MCP
```

该命令会启动交互式引导，在项目根目录下生成 `.bedrock_agentcore.yaml` 配置文件。按以下方式回答各提示项：

| 提示项 | 输入内容 | 说明 |
|--------|----------|------|
| Agent name | `billing_mcp_server` | AgentCore 中的 agent 标识名 |
| Execution role ARN | 粘贴 `${EXECUTION_ROLE_ARN}` 的实际值 | 步骤 6 创建的 IAM 角色 ARN |
| ECR repository | 直接回车（留空） | 工具会自动创建 ECR 仓库 |
| Dependency file | 自动检测到 `requirements.txt`，直接回车确认 | 步骤 3 创建的依赖文件 |
| Memory | 选择 `STM_ONLY` | 短期记忆模式，MCP Server 无需持久化 |
| Enable OAuth | 输入 `yes` | 启用 JWT 认证 |
| Discovery URL | 粘贴 `${DISCOVERY_URL}` 的实际值 | 步骤 9 输出的 OpenID Connect 发现端点 |
| Client ID | 粘贴 `${CLIENT_ID}` 的实际值 | 步骤 9 创建的 App Client ID |
| Memory Configuration| 输入 `s` 跳过 | 

命令执行完成后，会在项目根目录生成 `.bedrock_agentcore.yaml` 文件，内容类似：

```yaml
agent_name: billing_mcp_server
entry_point: awslabs/billing_cost_management_mcp_server/server.py
protocol: MCP
execution_role_arn: arn:aws:iam::123456789012:role/BillingMCPServerAgentCoreRole
ecr_repository: ""
dependency_file: requirements.txt
memory_type: STM_ONLY
authorizer_configuration:
  customJWTAuthorizer:
    discoveryUrl: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX/.well-known/openid-configuration
    allowedClients:
    - abcdef1234567890  # 步骤 9 创建的 CLIENT_ID
```

> 如果 `agentcore configure` 因任何原因失败或需要重新配置，可以直接手动创建或编辑该文件。
> 文件路径为项目根目录下的 `.bedrock_agentcore.yaml`。

### 步骤 15：将 M2M Client ID 加入 allowedClients

这是最容易遗漏也最关键的一步。`agentcore configure` 生成的配置文件中 `allowedClients` 只包含步骤 9 创建的测试用 Client ID。Quick Suite 使用的是步骤 12 创建的 M2M Client ID（`QS_M2M_CLIENT_ID`），必须手动加入列表，否则 Quick Suite 连接时会因为 token 的 `client_id` 不在允许列表中而被拒绝。

编辑 `.bedrock_agentcore.yaml`，找到 `authorizer_configuration` 部分，将 `QS_M2M_CLIENT_ID` 添加到 `allowedClients` 列表中：

```yaml
# 修改前（只有测试用的 Client ID）：
authorizer_configuration:
  customJWTAuthorizer:
    discoveryUrl: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX/.well-known/openid-configuration
    allowedClients:
    - abcdef1234567890

# 修改后（加入 Quick Suite 的 M2M Client ID）：
authorizer_configuration:
  customJWTAuthorizer:
    discoveryUrl: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX/.well-known/openid-configuration
    allowedClients:
    - abcdef1234567890
    - 你的QS_M2M_CLIENT_ID实际值
```

也可以用命令行快速追加：

```bash
echo "    - ${QS_M2M_CLIENT_ID}" >> .bedrock_agentcore.yaml
```

> 如果不做这一步，Quick Suite 连接时会被拒绝，表现为 "Creation failed"，只能看到一个 `listTools` Action。
> Cognito `client_credentials` 授权流程生成的 token 中包含 `client_id` claim，AgentCore Runtime 的 JWT authorizer 会验证这个 claim 是否在 `allowedClients` 列表中。

### 步骤 16：部署

```bash
agentcore launch
```

部署完成后记录 Agent ARN：

```bash
export AGENT_ARN="输出的ARN"
```

### 步骤 17：验证部署

```bash
# 刷新 token
export BEARER_TOKEN=$(aws cognito-idp initiate-auth \
  --client-id "${CLIENT_ID}" \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=${COGNITO_USERNAME},PASSWORD=${COGNITO_PASSWORD} \
  --region ${AWS_REGION} | jq -r '.AuthenticationResult.AccessToken')

ENCODED_ARN=$(echo -n ${AGENT_ARN} | jq -sRr '@uri')
MCP_ENDPOINT="https://bedrock-agentcore.${AWS_REGION}.amazonaws.com/runtimes/${ENCODED_ARN}/invocations?qualifier=DEFAULT"

# 发送 MCP initialize 请求
curl -X POST "${MCP_ENDPOINT}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer ${BEARER_TOKEN}" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

预期返回包含 `serverInfo`、25 个工具和 2 个 prompts 的 JSON 响应。

> 注意：
> - 必须先发 `initialize` 请求完成 MCP 协议握手，直接发 `tools/list` 会返回错误
> - `agentcore invoke` 可能因跳过 initialize 握手而报 400，不影响正常 MCP 客户端使用
> - 确认 CloudWatch 日志中没有 `400 Bad Request`：
>   ```bash
>   aws logs tail /aws/bedrock-agentcore/runtimes/你的agent-id-DEFAULT \
>     --log-stream-name-prefix "$(date +%Y/%m/%d)/[runtime-logs]" \
>     --since 2m --region ${AWS_REGION} | grep "400 Bad Request"
>   ```

---

## 7. 接入 Amazon Quick Suite Chat Agent

### 步骤 18：构造端点 URL 和认证信息

```bash
ENCODED_ARN=$(echo -n ${AGENT_ARN} | jq -sRr '@uri')
MCP_SERVER_ENDPOINT="https://bedrock-agentcore.${AWS_REGION}.amazonaws.com/runtimes/${ENCODED_ARN}/invocations"
TOKEN_URL="https://${COGNITO_DOMAIN_PREFIX}.auth.${AWS_REGION}.amazoncognito.com/oauth2/token"

echo "========================================="
echo "Quick Suite Service Auth 所需信息："
echo "========================================="
echo "MCP Server 端点: ${MCP_SERVER_ENDPOINT}"
echo "Client ID:       ${QS_M2M_CLIENT_ID}"
echo "Client Secret:   ${QS_M2M_CLIENT_SECRET}"
echo "Token URL:       ${TOKEN_URL}"
echo "========================================="
```

### 步骤 19：在 Quick Suite 控制台创建 MCP Actions 集成

1. 登录 [Amazon Quick Suite 控制台](https://quicksight.aws.amazon.com/)（需要 Author Pro 角色）
2. 左侧导航栏 → Connections → Integrations → Actions 标签页
3. 在 "Model Context Protocol" 卡片上点击 "+"
4. 填写集成信息：
   - Name: 自定义名称
   - Description: 描述 MCP Server 的能力（Quick Suite 会根据描述决定何时调用）
   - MCP server endpoint: 步骤 18 输出的 `MCP_SERVER_ENDPOINT`
5. 点击 "Next"
6. 认证方式选择 "Service authentication"，填写：
   - Client ID: `${QS_M2M_CLIENT_ID}`
   - Client Secret: `${QS_M2M_CLIENT_SECRET}`
   - Token URL: `${TOKEN_URL}`
7. 点击 "Create and continue"
8. 等待工具发现完成（约 1-2 分钟），确认能看到 25 个 Actions
9. 点击 "Next"，可选共享给其他用户，点击 "Done"

> 注意：Service authentication 不需要 Authorization URL 和 Redirect URL，配置比 User Auth 更简单。

### 步骤 20：在 Chat Agent 中使用

在 Quick Suite 控制台打开 Chat Agents，选择 "My Assistant" 或自定义 Agent，输入自然语言提问：

```
帮我查看上个月的 AWS 总费用，按服务分组显示
我的 AWS 账户有哪些成本异常？
预测下个月的 AWS 费用
查看我的 AWS 预算使用情况
有哪些 EC2 实例可以进行成本优化？
分析我的 Reserved Instance 覆盖率
```

> 注意事项：
> - MCP 操作有 60 秒超时限制
> - 工具列表在首次注册后是静态的，MCP Server 更新工具后需删除集成重新创建
> - 可变操作会弹出 Action Review 确认

---

## 8. 源码定制化

### 8.1 移除不需要的工具

编辑 `awslabs/billing_cost_management_mcp_server/server.py`，需要同时注释掉顶部的 import 和 `setup()` 函数中的 `mcp.import_server()` 调用。

例如，移除 Storage Lens 和 Billing Conductor 相关工具：

```python
# ===== 1. 注释掉顶部的 import =====
# from awslabs.billing_cost_management_mcp_server.tools.storage_lens_tools import storage_lens_server
# from awslabs.billing_cost_management_mcp_server.tools.billing_conductor_tools import (
#     billing_conductor_server,
# )

# ===== 2. 在 setup() 函数中注释掉对应的 import_server 调用 =====
async def setup():
    await mcp.import_server(cost_explorer_server)
    await mcp.import_server(compute_optimizer_server)
    await mcp.import_server(cost_optimization_hub_server)
    # await mcp.import_server(storage_lens_server)        # 已移除
    await mcp.import_server(aws_pricing_server)
    ...
    # await mcp.import_server(billing_conductor_server)   # 已移除
```

### 8.2 修改工具默认行为

例如，将 Cost Explorer 默认 Metric 从 `UnblendedCost` 改为 `AmortizedCost`：

```bash
# 编辑对应的工具文件，找到相关函数的参数定义，修改默认值
vi awslabs/billing_cost_management_mcp_server/tools/cost_explorer_tools.py
```

修改后本地测试验证：

```bash
python awslabs/billing_cost_management_mcp_server/server.py
# 在另一个终端发送测试请求
```

### 8.3 添加自定义工具

1. 在 `awslabs/billing_cost_management_mcp_server/tools/` 下创建新文件，例如 `custom_tools.py`：

```python
"""自定义工具示例。"""

from fastmcp import FastMCP

custom_server = FastMCP(name="custom-tools")

@custom_server.tool()
async def my_custom_tool(param1: str) -> str:
    """自定义工具描述。"""
    # 你的实现逻辑
    return "result"
```

2. 在 `server.py` 中注册：

```python
# 顶部添加 import
from awslabs.billing_cost_management_mcp_server.tools.custom_tools import custom_server

# setup() 函数中添加
await mcp.import_server(custom_server)
```

### 8.4 修改 Server Instructions

`server.py` 中 `FastMCP()` 的 `instructions` 参数定义了 LLM 使用工具时的引导说明。Quick Suite Chat Agent 会参考这些说明决定何时调用哪个工具。你可以根据业务需求修改，例如调整默认的成本分析策略、添加公司特定的分析流程等。

### 8.5 同步上游更新

```bash
# 克隆最新源码到临时目录
git clone --depth 1 --filter=blob:none --sparse https://github.com/awslabs/mcp.git /tmp/awslabs-mcp-update
cd /tmp/awslabs-mcp-update
git sparse-checkout set src/billing-cost-management-mcp-server

# 对比差异
diff -r /tmp/awslabs-mcp-update/src/billing-cost-management-mcp-server/awslabs \
  ~/billing-mcp-source/awslabs

# 根据 diff 结果选择性合并（直接 cp -r 会覆盖本地修改）
rm -rf /tmp/awslabs-mcp-update
```

> 建议对项目目录初始化 git 仓库，方便追踪定制化修改和合并上游更新：
> ```bash
> cd ~/billing-mcp-source
> git init && git add . && git commit -m "初始源码 + AgentCore 适配"
> ```

---

## 9. 日常运维

```bash
# 查看日志
aws logs tail /aws/bedrock-agentcore/runtimes/你的agent-id-DEFAULT \
  --log-stream-name-prefix "$(date +%Y/%m/%d)/[runtime-logs]" \
  --since 1h --region ${AWS_REGION}

# 停止会话
agentcore stop-session

# 修改源码后重新部署（AgentCore 会重新构建容器镜像）
agentcore launch

# 完全清理
agentcore destroy
```

---

## 10. 故障排查

| 问题 | 原因和解决方法 |
|------|---------------|
| Quick Suite "Creation failed"，只有 listTools | AgentCore 的 `allowedClients` 未包含 M2M Client ID。编辑 `.bedrock_agentcore.yaml` 添加后重新 `agentcore launch` |
| Cognito 返回 "invalid_scope" | Resource Server 未创建或 scope 名称不匹配，确认步骤 11 已执行 |
| `ModuleNotFoundError: awslabs.billing_cost_management_mcp_server` | `pip install -e .` 未执行，或 `awslabs/` 目录结构不正确 |
| 修改代码后本地测试无变化 | 确认使用了 `pip install -e .`（可编辑模式），而非 `pip install .` |
| AgentCore 部署后修改未生效 | 需要重新执行 `agentcore launch` 重新构建容器镜像 |
| 容器启动报 PermissionError: logs 目录 | `logging_utils.py` 中 `get_server_directory()` 未处理 `PermissionError`，确认已按步骤 2 修改添加了 fallback 到 `/tmp` |
| 容器运行报 PermissionError: sessions 目录 | `sql_utils.py` 中 `get_session_db_path()` 未处理 `PermissionError`，确认已按步骤 2 修改添加了 fallback 到 `/tmp` |
| Quick Suite "Validation failed for type" | `metrics` 参数类型为 `Optional[str]` 时 Quick Suite 校验不通过，改为 `Optional[List[str]]` 后重新部署并重建集成 |
| curl 测试返回 400 Bad Request | Accept header 需同时包含 `application/json` 和 `text/event-stream` |
| agentcore invoke 返回 400 | 正常现象，`agentcore invoke` 跳过了 MCP initialize 握手，用 curl 发 initialize 请求验证 |
| 401 Unauthorized | Bearer Token 过期，重新获取 |
| 403 Access Denied | 检查执行角色是否包含所需的 API 权限 |
| CloudWatch 日志无 HTTP 请求记录 | MCP Server 可能监听了 127.0.0.1，确认 `server.py` 中使用 `host="0.0.0.0"` |
| Token 验证失败 | `client_credentials` token 可能缺少 AgentCore 期望的某些 claim，改用 User Auth 方式 |
| import 报错 `circular import` | 添加自定义工具时注意避免循环引用 |

> 如果 Service Auth 方式始终失败，最可能的原因是 AgentCore Runtime 的 JWT authorizer 对 `client_credentials` token 的验证方式与预期不符。
> 这种情况下请改用已验证通过的 User Authentication (3LO) 方式。

---

## 11. 参考文档

- [billing-cost-management-mcp-server 源码](https://github.com/awslabs/mcp/tree/main/src/billing-cost-management-mcp-server)
- [FastMCP 文档](https://github.com/jlowin/fastmcp) — 了解 `FastMCP`、`@tool` 装饰器、`import_server` 等 API
- [Deploy MCP servers in AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp.html)
- [MCP protocol contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp-protocol-contract.html)
- [IAM Permissions for AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html)
- [Authenticate and authorize with Inbound Auth and Outbound Auth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html)
- [Amazon Cognito as identity provider](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-idp-cognito.html)
- [Amazon Quick Suite MCP integration](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)
- [Connect Amazon Quick Suite to enterprise apps and agents with MCP](https://aws.amazon.com/blogs/machine-learning/connect-amazon-quick-suite-to-enterprise-apps-and-agents-with-mcp/)
