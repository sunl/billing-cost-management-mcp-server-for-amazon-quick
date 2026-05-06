# 跨账号查询账单数据方案设计

## 1. 需求

- 用户在 Quick Suite 提问时提供目标账号 ID，查询该账号的账单数据
- 不提供账号 ID 时，查询当前账号（MCP Server 部署所在账号）
- 支持一次提问查询多个账号（LLM 自动拆分为多次工具调用）
- 跨账号角色名统一命名，用户只需提供账号 ID，不需要知道 role ARN

## 2. 常量与环境变量

| 环境变量 | 必填 | 说明 | 默认值 |
|---------|------|------|--------|
| `CROSS_ACCOUNT_ROLE_NAME` | 否 | 目标账号中的 IAM 角色名 | `BillingMCPCrossAccountRole` |

代码中读取方式：

```python
CROSS_ACCOUNT_ROLE_NAME = os.environ.get('CROSS_ACCOUNT_ROLE_NAME', 'BillingMCPCrossAccountRole')
```

逻辑：
- 传了 `target_account_id` → 拼 `arn:aws:iam::{target_account_id}:role/{CROSS_ACCOUNT_ROLE_NAME}`，assume role 后查询
- 没传 `target_account_id` → 不 assume role，查当前账号
- 传了但目标账号没创建对应 role → assume 失败，返回错误

安全性由 AWS IAM 保障：源账号的策略限制能 assume 哪些账号，目标账号的信任策略限制谁能 assume 进来。

## 3. 代码改动

### 3.1 aws_service_base.py

`create_aws_client` 新增 `target_account_id` 可选参数：

```python
def create_aws_client(service_name: str, region_name: Optional[str] = None, target_account_id: Optional[str] = None) -> Any:
    # ... 现有的 allowed_services 校验 ...

    region = region_name or os.environ.get('AWS_REGION', 'us-east-1')

    # Create AWS session
    profile_name = os.environ.get('AWS_PROFILE')
    if profile_name:
        session = boto3.Session(profile_name=profile_name, region_name=region)
    else:
        session = boto3.Session(region_name=region)

    # 跨账号 assume role
    if target_account_id:
        cross_account_role_name = os.environ.get('CROSS_ACCOUNT_ROLE_NAME', 'BillingMCPCrossAccountRole')
        role_arn = f"arn:aws:iam::{target_account_id}:role/{cross_account_role_name}"
        sts_client = session.client('sts', region_name=region)
        assumed = sts_client.assume_role(
            RoleArn=role_arn,
            RoleSessionName=f'billing-mcp-{target_account_id}'
        )
        credentials = assumed['Credentials']
        session = boto3.Session(
            aws_access_key_id=credentials['AccessKeyId'],
            aws_secret_access_key=credentials['SecretAccessKey'],
            aws_session_token=credentials['SessionToken'],
            region_name=region
        )

    # ... 现有的 Config 和 return session.client(...) ...
```

### 3.2 工具函数改动

每个需要支持跨账号的工具函数新增参数：

```python
target_account_id: Optional[str] = None,
```

然后在调用 `create_aws_client` 时透传：

```python
# 改动前
ce_client = create_aws_client('ce')

# 改动后
ce_client = create_aws_client('ce', target_account_id=target_account_id)
```

### 3.3 已改动的文件清单

| 文件 | create_aws_client 调用次数 | 说明 |
|------|--------------------------|------|
| `utilities/aws_service_base.py` | 1（函数定义） | 核心改动 |
| `tools/cost_explorer_tools.py` | 1 | Cost Explorer 查询 |
| `tools/cost_comparison_tools.py` | 1 | 成本对比 |
| `tools/cost_anomaly_tools.py` | 1 | 成本异常 |
| `tools/budget_tools.py` | 2 | STS + Budgets |
| `tools/compute_optimizer_tools.py` | 1 | Compute Optimizer |
| `tools/cost_optimization_hub_tools.py` | 1 | Cost Optimization Hub |
| `tools/recommendation_details_tools.py` | 3 | 推荐详情（含内部函数透传） |
| `tools/ri_performance_tools.py` | 1 | RI 性能 |
| `tools/sp_performance_tools.py` | 1 | SP 性能 |
| `tools/free_tier_usage_tools.py` | 1 | Free Tier |

总计：1 个核心文件 + 10 个工具文件。

## 4. AWS 侧配置

默认跨账号角色名为 `BillingMCPCrossAccountRole`，以下配置均使用此名称。如需自定义，将 4.1 和 4.2 中的角色名替换为你的自定义名称。

并且在 AgentCore 部署的时候，需要在容器环境变量中设置 `CROSS_ACCOUNT_ROLE_NAME`

### 4.1 源账号（部署 MCP Server 的账号）

给 `BillingMCPServerAgentCoreRole` 添加 assume role 权限, 将此策略追加到步骤 6 创建的角色上：

```bash
cat > cross-account-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountAssumeRole",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": [
      "arn:aws:iam::111111111111:role/BillingMCPCrossAccountRole",
      "arn:aws:iam::222222222222:role/BillingMCPCrossAccountRole"
    ]
  }]
}
EOF

aws iam put-role-policy \
  --role-name BillingMCPServerAgentCoreRole \
  --policy-name CrossAccountAssumeRolePolicy \
  --policy-document file://cross-account-policy.json
```

### 4.2 每个目标账号

创建统一命名的角色 `BillingMCPCrossAccountRole`：

信任策略（trust policy）：

```bash
# 在目标账号中执行

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<源账号ID>:role/BillingMCPServerAgentCoreRole"
    },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name BillingMCPCrossAccountRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Cross-account role for Billing MCP Server"
```

权限策略（与 README.md 步骤 6 中的 `billing-mcp-policy.json` 相同）：

创建命令：

```bash
# 在目标账号中执行

cat > billing-mcp-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Sid": "CostExplorer", "Effect": "Allow", "Action": ["ce:GetCostAndUsage","ce:GetCostAndUsageWithResources","ce:GetCostAndUsageComparisons","ce:GetCostComparisonDrivers","ce:GetCostForecast","ce:GetUsageForecast","ce:GetDimensionValues","ce:GetTags","ce:GetCostCategories","ce:GetReservationPurchaseRecommendation","ce:GetReservationCoverage","ce:GetReservationUtilization","ce:GetSavingsPlansUtilization","ce:GetSavingsPlansCoverage","ce:GetSavingsPlansUtilizationDetails","ce:GetSavingsPlansPurchaseRecommendation","ce:GetAnomalies"], "Resource": "*"},
    {"Sid": "CostOptimizationHub", "Effect": "Allow", "Action": ["cost-optimization-hub:GetRecommendation","cost-optimization-hub:ListRecommendations","cost-optimization-hub:ListRecommendationSummaries"], "Resource": "*"},
    {"Sid": "ComputeOptimizer", "Effect": "Allow", "Action": ["compute-optimizer:GetAutoScalingGroupRecommendations","compute-optimizer:GetEBSVolumeRecommendations","compute-optimizer:GetEC2InstanceRecommendations","compute-optimizer:GetECSServiceRecommendations","compute-optimizer:GetRDSDatabaseRecommendations","compute-optimizer:GetLambdaFunctionRecommendations","compute-optimizer:GetEnrollmentStatus","compute-optimizer:GetIdleRecommendations"], "Resource": "*"},
    {"Sid": "Budgets", "Effect": "Allow", "Action": ["budgets:ViewBudget"], "Resource": "*"},
    {"Sid": "Pricing", "Effect": "Allow", "Action": ["pricing:DescribeServices","pricing:GetAttributeValues","pricing:GetProducts"], "Resource": "*"},
    {"Sid": "FreeTier", "Effect": "Allow", "Action": ["freetier:GetFreeTierUsage"], "Resource": "*"},
    {"Sid": "BCMPricingCalc", "Effect": "Allow", "Action": ["bcm-pricing-calculator:GetPreferences","bcm-pricing-calculator:GetWorkloadEstimate","bcm-pricing-calculator:ListWorkloadEstimateUsage","bcm-pricing-calculator:ListWorkloadEstimates"], "Resource": "*"}
  ]
}
EOF

aws iam put-role-policy \
  --role-name BillingMCPCrossAccountRole \
  --policy-name BillingMCPServerPolicy \
  --policy-document file://billing-mcp-policy.json
```

## 5. 用户体验示例

```
查看账号 111111111111 上个月的费用，按服务分组
→ LLM 调用 cost-explorer，target_account_id=111111111111

查看上个月的费用
→ LLM 调用 cost-explorer，target_account_id=None（查当前账号）

对比账号 111111111111 和 222222222222 上个月的费用
→ LLM 调用两次 cost-explorer，分别传入两个账号 ID，汇总结果

账号 111111111111 有哪些 EC2 可以优化？
→ LLM 调用 compute-optimizer，target_account_id=111111111111
```

## 6. 不需要改动的工具

以下工具不涉及账号维度，不需要加 `target_account_id` 参数：

- `aws_pricing_operations.py` — Pricing API 是公共定价数据，不区分账号
- `billing_conductor_operations.py` — 管理账号功能，成员账号无权限
- `unified_sql_tools.py` — 本地 session SQL 查询，不调用 AWS API
- `bcm_pricing_calculator_tools.py` — BCM 定价计算器，查询的是定价模型而非账单数据
- `storage_lens_tools.py` — S3 Storage Lens，通过 Athena 查询，场景较特殊

## 7. 改动总结

| 类别 | 改动 |
|------|------|
| 核心文件 | `aws_service_base.py`：`create_aws_client` 新增 `target_account_id` 参数 + assume role 逻辑 |
| 工具文件 | 10 个文件的工具函数新增 `target_account_id` 参数并透传 |
| 不改的文件 | `aws_pricing_operations.py`、`billing_conductor_operations.py`、`unified_sql_tools.py`、`bcm_pricing_calculator_tools.py`、`storage_lens_tools.py`、`server.py`、prompts、其他 utilities |
| AWS 配置 | 源账号加 `sts:AssumeRole` 权限，每个目标账号创建对应角色（默认名 `BillingMCPCrossAccountRole`） |
| 环境变量 | `CROSS_ACCOUNT_ROLE_NAME`（可选，有默认值） |
