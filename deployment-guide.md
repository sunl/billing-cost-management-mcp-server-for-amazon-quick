# 部署 Billing MCP Server 到 AgentCore Runtime 并接入 Quick Suite（Service Authentication）

## 1. 方案概述

### 1.1 目标

部署 [billing-cost-management-mcp-server-for-amazon-quick](https://github.com/sunl/billing-cost-management-mcp-server-for-amazon-quick) 到 Amazon Bedrock AgentCore Runtime，并通过 Amazon Quick Suite 的 Chat Agent 以 Service Authentication (2LO) 方式调用，使业务用户能在对话界面中直接查询 AWS 账单与成本数据。

本仓库基于 [awslabs/mcp](https://github.com/awslabs/mcp/tree/main/src/billing-cost-management-mcp-server) 上游源码，已完成 AgentCore Runtime 适配修改。修改详情参见 [source-modification-guide.md](./source-modification-guide.md)。

本仓库支持跨账号查询账单数据，配置方法参见 [cross-account-design.md](./cross-account-design.md)。

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
billing-cost-management-mcp-server (ARM64 容器)
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

## 3. 获取源码并安装依赖

### 步骤 1：克隆源码

```bash
git clone https://github.com/sunl/billing-cost-management-mcp-server-for-amazon-quick.git .
cd billing-cost-management-mcp-server-for-amazon-quick
```

### 步骤 2：安装依赖

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install bedrock-agentcore-starter-toolkit

# 以可编辑模式安装源码包
pip install -e .

agentcore --help
```

### 步骤 3：本地测试（可选）

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

### 步骤 4：创建信任策略和角色

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

### 步骤 5：添加 AgentCore 基础权限

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

### 步骤 6：添加 Billing and Cost Management API 权限

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

### 步骤 7：创建 User Pool 和测试用户

```bash
export POOL_NAME="<YOUR_POOL_NAME>"

export POOL_ID=$(aws cognito-idp create-user-pool \
  --pool-name "${POOL_NAME}" \
  --policies '{"PasswordPolicy":{"MinimumLength":8}}' \
  --region ${AWS_REGION} | jq -r '.UserPool.Id')

export CLIENT_ID=$(aws cognito-idp create-user-pool-client \
  --user-pool-id ${POOL_ID} \
  --client-name "BillingMCPClient" \
  --no-generate-secret \
  --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
  --region ${AWS_REGION} | jq -r '.UserPoolClient.ClientId')

export COGNITO_USERNAME="<YOUR_USERNAME>"
export COGNITO_PASSWORD="<YOUR_SECURE_PASSWORD>"

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

### 步骤 8：创建 Cognito Domain

```bash
export COGNITO_DOMAIN_PREFIX="billing-mcp-$(echo ${AWS_ACCOUNT_ID} | tail -c 9)"

aws cognito-idp create-user-pool-domain \
  --user-pool-id ${POOL_ID} \
  --domain ${COGNITO_DOMAIN_PREFIX} \
  --region ${AWS_REGION}

echo "Cognito Domain: ${COGNITO_DOMAIN_PREFIX}"
```

### 步骤 9：创建 Resource Server

Service authentication (2LO) 使用 `client_credentials` 授权流程，Cognito 要求必须有 Resource Server 定义 scope。

```bash
aws cognito-idp create-resource-server \
  --user-pool-id ${POOL_ID} \
  --identifier "billing-mcp" \
  --name "Billing MCP Server" \
  --scopes '[{"ScopeName":"invoke","ScopeDescription":"调用 Billing MCP Server"}]' \
  --region ${AWS_REGION}
```

### 步骤 10：创建 Machine-to-Machine App Client（Quick Suite 专用）

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

---

## 6. 配置并部署到 AgentCore Runtime

### 步骤 11：运行 agentcore configure

```bash
agentcore configure -e awslabs/billing_cost_management_mcp_server/server.py --protocol MCP
```

该命令会启动交互式引导，在项目根目录下生成 `.bedrock_agentcore.yaml` 配置文件。按以下方式回答各提示项：

| 提示项 | 输入内容 | 说明 |
|--------|----------|------|
| Agent name | `billing_mcp_server` | AgentCore 中的 agent 标识名 |
| Dependency file | 自动检测到 `requirements.txt`，直接回车确认 | |
| Execution role ARN | 粘贴 `${EXECUTION_ROLE_ARN}` 的实际值 | 步骤 4 创建的 IAM 角色 ARN |
| ECR repository | 直接回车（留空） | 工具会自动创建 ECR 仓库 |
| OAuth | 输入 `yes` | 启用 JWT 认证 |
| Discovery URL | 粘贴 `${DISCOVERY_URL}` 的实际值 | 步骤 7 输出的 OpenID Connect 发现端点 |
| Client ID | 粘贴 `${CLIENT_ID}` 的实际值 | 步骤 7 创建的 App Client ID 和步骤 10 创建的 M2M Client ID，以逗号分隔 |
| Memory Configuration| 输入 `s` 跳过 | 暂时不在本文讨论范围内 |

命令执行完成后，会在项目根目录生成 `.bedrock_agentcore.yaml` 文件，会包含以下类似内容：

```yaml
default_agent: billing_mcp_server

entry_point: awslabs/billing_cost_management_mcp_server/server.py
execution_role: arn:aws:iam::123456789012:role/BillingMCPServerAgentCoreRole
ecr_repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/bedrock-agentcore-billing_mcp_server
protocol_configuration:
  server_protocol: MCP

authorizer_configuration:
  customJWTAuthorizer:
    discoveryUrl: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXX/.well-known/openid-configuration
    allowedClients:
    - abcdef1234567890  # 步骤 7 创建的 CLIENT_ID，测试用
    - 你的QS_M2M_CLIENT_ID实际值 # Quick Suite 使用的是步骤 10 创建的 M2M Client ID，必须加入列表，否则 Quick Suite 连接时会因为 token 的 `client_id` 不在允许列表中而被拒绝。
```

> 如果 `agentcore configure` 因任何原因失败或需要重新配置，可以直接手动创建或编辑该文件。
> 文件路径为项目根目录下的 `.bedrock_agentcore.yaml`。

### 步骤 12：部署

```bash
agentcore launch
```

部署完成后记录 Agent ARN：

```bash
export AGENT_ARN="输出的ARN"
```

### 步骤 13：验证部署

```bash
# 获取 Bearer Token
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
> - Token 有效期默认 1 小时，过期后重新执行获取 Token 的命令

---

## 7. 接入 Amazon Quick Suite Chat Agent

### 步骤 14：构造端点 URL 和认证信息

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

### 步骤 15：在 Quick Suite 控制台创建 MCP Actions 集成

1. 登录 [Amazon Quick Suite 控制台](https://quicksight.aws.amazon.com/)（需要 Author Pro 角色）
2. 左侧导航栏 → Connections → Integrations → Actions 标签页
3. 在 "Model Context Protocol" 卡片上点击 "+"
4. 填写集成信息：
   - Name: 自定义名称
   - MCP server endpoint: 步骤 14 输出的 `MCP_SERVER_ENDPOINT`
5. 点击 "Next"
6. 认证方式选择 "Service authentication"，填写：
   - Client ID: `${QS_M2M_CLIENT_ID}`
   - Client Secret: `${QS_M2M_CLIENT_SECRET}`
   - Token URL: `${TOKEN_URL}`
7. 点击 "Create and continue"
8. 等待工具发现完成（约 1-2 分钟），确认能看到 25 个 Actions
9. 点击 "Next"，可选共享给其他用户，点击 "Done"

> 注意：Service authentication 不需要 Authorization URL 和 Redirect URL，配置比 User Auth 更简单。

### 步骤 16：在 Chat Agent 中使用

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

## 8. 日常运维

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

## 9. 故障排查

| 问题 | 原因和解决方法 |
|------|---------------|
| Quick Suite "Creation failed"，只有 listTools | AgentCore 的 `allowedClients` 未包含 M2M Client ID。编辑 `.bedrock_agentcore.yaml` 添加后重新 `agentcore launch` |
| Cognito 返回 "invalid_scope" | Resource Server 未创建或 scope 名称不匹配，确认步骤 9 已执行 |
| `ModuleNotFoundError: awslabs.billing_cost_management_mcp_server` | `pip install -e .` 未执行，或 `awslabs/` 目录结构不正确 |
| AgentCore 部署后修改未生效 | 需要重新执行 `agentcore launch` 重新构建容器镜像 |
| Quick Suite "Validation failed for type" | `metrics` 参数类型问题，参见 [source-modification-guide.md](./source-modification-guide.md) 中的修改 4 |
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

## 10. 参考文档

- [billing-cost-management-mcp-server 上游源码](https://github.com/awslabs/mcp/tree/main/src/billing-cost-management-mcp-server)
- [源码修改指南](./source-modification-guide.md)
- [跨账号查询方案](./cross-account-design.md)
- [FastMCP 文档](https://github.com/jlowin/fastmcp)
- [Deploy MCP servers in AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp.html)
- [MCP protocol contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp-protocol-contract.html)
- [IAM Permissions for AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html)
- [Authenticate and authorize with Inbound Auth and Outbound Auth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html)
- [Amazon Cognito as identity provider](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-idp-cognito.html)
- [Amazon Quick Suite MCP integration](https://docs.aws.amazon.com/quick/latest/userguide/mcp-integration.html)
- [Connect Amazon Quick Suite to enterprise apps and agents with MCP](https://aws.amazon.com/blogs/machine-learning/connect-amazon-quick-suite-to-enterprise-apps-and-agents-with-mcp/)
