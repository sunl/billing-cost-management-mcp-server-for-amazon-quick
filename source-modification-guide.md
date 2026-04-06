# 源码修改指南：适配 AgentCore Runtime 与 Quick Suite

本文档记录了从 [awslabs/mcp](https://github.com/awslabs/mcp/tree/main/src/billing-cost-management-mcp-server) 上游源码到当前仓库所做的全部修改，以及后续源码定制化的方法。

当前仓库 [billing-cost-management-mcp-server-for-amazon-quick](https://github.com/sunl/billing-cost-management-mcp-server-for-amazon-quick) 已包含以下所有修改，可直接克隆使用，无需手动操作本文档中的步骤。

---

## 1. 上游源码获取方式

源码最初从 awslabs/mcp 仓库提取：

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/awslabs/mcp.git /tmp/awslabs-mcp
cd /tmp/awslabs-mcp
git sparse-checkout set src/billing-cost-management-mcp-server

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

---

## 2. AgentCore Runtime 适配修改

源码中的 `server.py` 默认使用 stdio 传输，需要修改以适配 AgentCore Runtime。

### 修改 1：`awslabs/billing_cost_management_mcp_server/utilities/logging_utils.py`

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

### 修改 2：`awslabs/billing_cost_management_mcp_server/utilities/sql_utils.py`

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

### 修改 3：`awslabs/billing_cost_management_mcp_server/server.py`

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

### 修改 4：`awslabs/billing_cost_management_mcp_server/tools/cost_explorer_tools.py`

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

---

## 3. requirements.txt

源码部署时，`requirements.txt` 只列出直接依赖，不包含 MCP Server 包本身（源码会作为项目的一部分被 AgentCore 打包进容器）：

```
mcp[cli]>=1.23.0
fastmcp>=2.14.0
boto3>=1.34.0
pydantic>=2.10.6
loguru>=0.7.0
python-dotenv>=1.0.0
```

---

## 4. 源码定制化

### 4.1 移除不需要的工具

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

### 4.2 修改工具默认行为

例如，将 Cost Explorer 默认 Metric 从 `UnblendedCost` 改为 `AmortizedCost`：

```bash
vi awslabs/billing_cost_management_mcp_server/tools/cost_explorer_tools.py
```

修改后本地测试验证：

```bash
python awslabs/billing_cost_management_mcp_server/server.py
# 在另一个终端发送测试请求
```

### 4.3 添加自定义工具

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

### 4.4 修改 Server Instructions

`server.py` 中 `FastMCP()` 的 `instructions` 参数定义了 LLM 使用工具时的引导说明。Quick Suite Chat Agent 会参考这些说明决定何时调用哪个工具。你可以根据业务需求修改，例如调整默认的成本分析策略、添加公司特定的分析流程等。

### 4.5 同步上游更新

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

---

## 5. 跨账号查询与管理账号兼容性修改

### 修改 5：`awslabs/billing_cost_management_mcp_server/utilities/aws_service_base.py` — 本账号 ID 缓存与跳过 assume role

当用户通过 `target_account_id` 查询 MCP 部署所在账号时，原始逻辑也会尝试 assume `BillingMCPCrossAccountRole`，但本账号不需要跨账号访问。

新增模块级缓存 `_local_account_id` 和辅助函数 `_get_local_account_id()`，在 `create_aws_client` 中判断 `target_account_id` 是否等于本账号，如果是则跳过 assume role：

```python
# 模块级缓存
_local_account_id: Optional[str] = None


def _get_local_account_id(session) -> str:
    """Get the local AWS account ID, cached after first call."""
    global _local_account_id
    if _local_account_id is None:
        sts_client = session.client('sts')
        _local_account_id = sts_client.get_caller_identity()['Account']
    return _local_account_id
```

`create_aws_client` 中的 assume role 逻辑改为：

```python
    if target_account_id:
        local_account_id = _get_local_account_id(session)
        if target_account_id != local_account_id:
            # 跨账号 assume role（原有逻辑）
            ...
```

### 修改 6：`awslabs/billing_cost_management_mcp_server/tools/cost_explorer_tools.py` — 管理账号 LINKED_ACCOUNT 过滤防御

LLM 在后续查询中可能自行添加 `LINKED_ACCOUNT` filter 来过滤已通过 `target_account_id` 指定的账号。如果该账号是管理账号/付款账号，`LINKED_ACCOUNT` 维度中不包含管理账号，会导致返回空结果。

#### 6a. Prompt 引导

在工具 description 的 `IMPORTANT USAGE GUIDELINES` 中新增：

```
- When target_account_id is specified, DO NOT use LINKED_ACCOUNT dimension to filter the same account ID.
  target_account_id already scopes the query to that account. Adding a LINKED_ACCOUNT filter for the same
  account will return empty results if that account is a management/payer account, because management accounts
  do not appear in the LINKED_ACCOUNT dimension.
```

#### 6b. 代码防御

新增辅助函数 `_remove_redundant_linked_account_filter()` 和 `_clean_linked_account_filter()`，在 `cost_explorer` 函数入口处检测并移除冗余的 `LINKED_ACCOUNT` filter：

```python
def _remove_redundant_linked_account_filter(filter_str: str, target_account_id: str) -> Optional[str]:
    """Remove redundant LINKED_ACCOUNT filter when target_account_id is already specified."""
    ...

def _clean_linked_account_filter(filter_obj: dict, target_account_id: str):
    """Recursively clean LINKED_ACCOUNT filter from a filter object."""
    ...
```

在 `cost_explorer` 函数中调用：

```python
    if target_account_id and filter:
        filter = _remove_redundant_linked_account_filter(filter, target_account_id)
```

支持直接 filter、`And`/`Or` 复合 filter 和 `Not` filter 的递归清理。如果整个 filter 都是冗余的则置为 `None`，如果复合 filter 中只剩一个条件则自动展平。
