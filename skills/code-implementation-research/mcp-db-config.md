# MCP 数据库连接配置参考

当需要连接测试/生产环境数据库时，可参考以下 MCP 配置。具体配置以用户环境为准。

```json
{
  "mcpServers": {
    "scp-test": {
      "command": "C:/nodejs/npx.cmd",
      "args": [
        "-y",
        "mcp-mysql-server",
        "--host=10.7.60.157",
        "--port=3306",
        "--user=scp_ips",
        "--password=scp_ips"
      ]
    },
    "scp-prod": {
      "command": "C:/nodejs/npx.cmd",
      "args": [
        "-y",
        "mcp-mysql-server",
        "--host=10.7.60.158",
        "--port=3306",
        "--user=scp_ips",
        "--password=scp_ips"
      ]
    }
  }
}
```

若 MCP 工具调用失败，可尝试在 `temp-work/` 下创建临时 Python 脚本连接；若仍无法连接，须明确告知主会话和用户，并在调研报告中写明无法进行数据库调研。

## 在 Claude Code 中配置 MCP

若需在 Claude Code 中使用 MySQL MCP，将上述配置添加到 Claude Code 的 MCP 配置文件中（通常在 `.claude/settings.json` 或通过 Claude Code 设置界面配置）。配置完成后，可在对话中直接调用 MCP 工具查询数据库。
