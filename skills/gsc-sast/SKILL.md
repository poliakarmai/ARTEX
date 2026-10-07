---
name: gsc-sast
description: 在目标暴露源代码、Git 仓库、依赖清单或编译产物时，调用 GSC MCP 做 SAST/SCA/密钥扫描，产出可验证的代码级发现（含 PoC 复核）。
---

# GSC SAST（代码静态扫描）

面向**已部署 GSC 实例**的用户。通过 MCP 连接 [GSC（Git Security Checker）](https://github.com/poliakarmai/gsc-core)，让 ARTEX 代理在渗透过程中对发现的**代码资产**做静态安全扫描，弥补 ARTEX 只看外部面（recon / DAST）而看不到代码内部的问题。

## 1. 准备工作

### 1.1 部署 GSC MCP

GSC 的 MCP 服务为 Streamable HTTP（SSE 应答），默认端点 `/mcp`，例如 `http://<gsc-host>:8002/mcp`。启动方式：

```bash
GSC_MCP_TOKEN=<随机令牌> GSC_ALLOWED_ROOTS=/data python3 gsc_mcp_server.py \
  --transport http --host 0.0.0.0 --port 8002
```

### 1.2 配置 ARTEX MCP

ARTEX → 系统设置 → MCP → 添加服务器：

```json
{
  "url": "http://<gsc-host>:8002/mcp",
  "headers": {
    "Authorization": "Bearer <GSC_MCP_TOKEN>"
  }
}
```

配置后重载，确认工具列表出现 `scan_repo`、`verify_finding` 等。

---

## 2. 工具一览

| 工具 | 用途 |
| --- | --- |
| `scan_repo` | 对本地仓库路径做 GSC 扫描，返回按严重度汇总 + top 发现 |
| `scan_diff` | 只扫相对 git base 变更的文件（PR review 场景） |
| `list_findings` | 从 GSC 数据库快读最近发现（不重新扫描） |
| `verify_finding` | 在沙箱重跑某发现的 PoC，报告是否仍可利用 |
| `get_finding` | 取单条发现的完整详情（规则、位置、修复建议、置信度） |
| `list_detectors` | 列出扫描引擎可用的检测器（规则 ID + 语言） |

所有工具均为**只读**：GSC MCP 不暴露自动修复、自动提 PR 等破坏性动作，发现仅供人工确认。

---

## 3. 常用工作流

### 3.1 发现源码 / 仓库 → 静态扫描

当 recon 阶段发现目标暴露的源码（`.git` 泄露、公开 Git 仓库、源码压缩包、前端 bundle 内的 `.map`、依赖清单 `package.json`/`requirements.txt` 等）：

1. 将源码落到本地目录（须在 `GSC_ALLOWED_ROOTS` 允许的路径内）
2. `scan_repo(repo_path=<路径>)` 扫描
3. 读 `top` 里 CRITICAL/HIGH 的 `rule_id` / `file` / `line`

### 3.2 用发现的密钥 / 凭据做下一步

GSC 发现硬编码密钥、数据库连接串、JWT、云厂商 key、内部地址等时，把它们当作**后续攻击面线索**（认证绕过、越权、SSRF 打内网），而不是直接判定为漏洞——先 `verify_finding` 复核。

### 3.3 复核发现是否可利用

`verify_finding(repo_path=<路径>, finding_key=<key>)` 在沙箱执行 PoC，返回 `verified` / `not_reproducible`。只有 `verified` 的发现才进入报告，避免把静态规则命中当作已验证漏洞。

---

## 4. 边界与注意

- **只读**：GSC MCP 不写目标、不自动修复，发现只供人工确认。
- **授权**：仅对已授权目标使用，遵守项目 scope 与 rate limit。
- **路径**：`scan_repo` 只接受 `GSC_ALLOWED_ROOTS` 内的路径，否则拒绝。
- **语义**：GSC 是**静态**扫描（源码级），与 ARTEX 的动态面（recon/DAST/流量）互补——发现用于交叉验证，不替代动态确认。
