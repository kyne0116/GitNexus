# GitNexus MCP 安装使用指南

## 快速开始

### 1. 安装 GitNexus

```bash
npm install -g gitnexus@latest
```

> 不要用 `npx -y gitnexus@latest mcp` 作为 MCP 启动命令。`npx` 每次需要检查/下载包，容易导致 Claude Code 连接超时（"Failed to connect"）。

### 2. 索引你的仓库

```bash
cd /path/to/your/repo
gitnexus analyze
```

索引数据存储在仓库根目录的 `.gitnexus/` 下，同时注册到全局注册表 `~/.gitnexus/registry.json`。

### 3. 配置 Claude Code MCP

推荐配置到用户级（所有项目通用）：

```bash
claude mcp add gitnexus --scope user -- gitnexus mcp
```

也可以配置到项目级（仅当前项目生效），在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "gitnexus": {
      "type": "stdio",
      "command": "gitnexus",
      "args": ["mcp"]
    }
  }
}
```

### 4. 验证连接

```bash
claude mcp list
```

应显示 `gitnexus: gitnexus mcp - ✓ Connected`。

## 常见问题

### "Failed to connect" 连接失败

| 原因 | 表现 | 解决方案 |
|------|------|----------|
| npx 启动慢 | 用 `npx gitnexus@latest mcp` 作为命令 | 全局安装后改用 `gitnexus mcp` |
| KuzuDB 文件锁冲突 | 多个 Claude Code 会话同时运行 | 关闭其他会话，或等待空闲会话自动释放锁 |
| 原生模块不兼容 | Node.js 版本与 KuzuDB 绑定不匹配 | 确保 Node.js >= 18，重新 `npm install -g gitnexus@latest` |

### 部分路径可用、部分路径不可用

MCP 配置有两个作用域：

- **用户级**（`~/.claude/settings.json`）— 所有路径生效
- **项目级**（项目根目录 `.mcp.json`）— 仅该项目目录内生效

如果只在项目级配置了 gitnexus，其他路径自然看不到。解决方案：配置到用户级。

### MCP 服务器架构说明

```
claude mcp list
    └─ 启动: gitnexus mcp (stdio 进程)
        └─ 读取: ~/.gitnexus/registry.json (全局注册表)
            └─ 加载: 所有已索引仓库的 .gitnexus/kuzu 数据库
                └─ 等待: stdin 上的 JSON-RPC 请求
```

MCP 服务器不依赖当前工作目录，它从全局注册表发现所有已索引仓库。索引新仓库后，运行中的服务器会在下次工具调用时自动发现。
