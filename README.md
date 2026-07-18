# Cloudflare WARP 分流规则（域名回退 + 隧道分割）

适用于 **Cloudflare WARP 消费版（Windows）** 的分流规则文档。

核心理念：**国内网站直连（绕过 WARP），国外 LLM / 搜索引擎 / 被墙站点走 WARP 隧道。**
由于消费版 WARP 仅支持 Exclude（排除）模式，实现方式是把国内域名从隧道中**排除**并加入 **DNS 回退**，其余流量默认走隧道。

---

## 一、两种机制的本质区别（实测结论，非推测）

| 机制 | WARP 命令 | 匹配方式 | 关键特性 |
|------|-----------|----------|----------|
| **DNS 回退** (Domain Fallback) | `warp-cli dns fallback add <域>` | **后缀匹配** | 裸 `abc.com` 自动覆盖 `1.abc.com`、`2.abc.com` 等所有子域 |
| **隧道排除** (Split Tunnel Exclude) | `warp-cli tunnel host add <域>` | **域名匹配**（解析后按返回 IP 放行） | 裸 `abc.com` **不**覆盖子域 `x.abc.com`，需 `*.abc.com` 才覆盖 |

### 验证证据

1. **DNS 回退后缀匹配**：配置裸 `qq.com` 后，解析 `www.qq.com` → 日志显示 `via default`（本地解析器），证明子域命中。
2. **隧道排除不覆盖子域**：仅配置裸 `qq.com` 时，`tracert www.qq.com` 首跳为 `104.28.0.0`（Cloudflare 边缘）→ 流量**进了隧道**；
   而同时配置 `tieba.baidu.com` 时，`tracert tieba.baidu.com` 首跳为 `192.168.50.1`（本地路由器）→ **直连**。
3. **国家后缀 `cn` 一条全覆盖**：配置裸 `cn` 后，`cloudflare.cn`、`baidu.com.cn`、`people.com.cn` 均 `via default` 且 tracert 直连，证明 `.cn` 任意层级被单条 `cn` 覆盖。

---

## 二、域名分类简化规则

| 类型 | 示例 | DNS 回退简化 | 隧道排除简化 |
|------|------|--------------|--------------|
| 不同后缀 | `abc.com` / `abc.net` | 各一条，不可合并 | 各一条，不可合并 |
| 子串相似但无关 | `abcdef.com` / `bcd.com` | 各一条（子串 ≠ 域名关系） | 各一条 |
| 同域 + 子域 | `abc.com` / `1.abc.com` / `2.abc.com` | **一条 `abc.com`** 即覆盖全部子域 | **两条**：`abc.com` + `*.abc.com` |
| 国家后缀 | 所有 `.cn` 站点 | **一条 `cn`** | **一条 `cn`** |

### 简化操作清单

- **DNS 回退列表**：凡是「已存在裸主域」的子域条目（如 `tieba.baidu.com`、`mail.qq.com`）全部删除，裸主域已后缀覆盖。
- **隧道排除列表**：子域条目**必须保留**（裸主域不匹配子域流量）。如需覆盖某主域全部子域，添加 `*.主域`。
- **`.cn` 体系**：仅需一条 `cn`（回退 + 隧道各一条），无需逐条列举中文站点 `.cn` 域名。
- **中间含 `cn` 的国内 `.com` 站**（如 `oss-cn-hangzhou.aliyuncs.com`、`ark.cn-beijing.volces.com`）不被 `cn` 覆盖，需按真实后缀（`.com`）逐条或加 `*.aliyuncs.com` 等。

---

## 三、当前生效规则集

> 以下为推送时从本机 WARP 导出的实际配置快照。

### DNS 回退列表（`warp-cli dns fallback list`）
共 **109** 条。结构：
- 裸后缀：`cn`（覆盖全部 `.cn`）
- 国内主域：`baidu.com`、`qq.com`、`taobao.com`、`jd.com`、`weibo.com`、`zhihu.com`、`bilibili.com`、`aliyun.com`、`tencentcloud.com`、`huaweicloud.com` 等（裸主域已后缀覆盖其子域）

### 隧道排除列表（`warp-cli tunnel host list`）
共 **122** 条。结构：
- 裸后缀：`cn`
- 国内主域：`baidu.com`、`qq.com` 等
- 必要子域：`tieba.baidu.com`、`map.baidu.com`、`mail.qq.com`、`v.qq.com`、`qzone.qq.com`、`lbs.amap.com`、`jr.jd.com`、`oss-cn-hangzhou.aliyuncs.com` 等（隧道排除需显式覆盖子域）

### 默认走 WARP 隧道的流量（未排除即走隧道）
- LLM API：`openai.com`、`api.anthropic.com`、`api.x.ai`(grok)、`generativelanguage.googleapis.com`(gemini)
- 安全平台：`hackthebox.com`、`tryhackme.com`
- 搜索引擎：`google.com`、`duckduckgo.com`
- 被墙站点：`youtube.com`、`facebook.com`、`x.com`、`huggingface.co`、`github.com` 等

---

## 四、配套脚本

| 文件 | 用途 |
|------|------|
| `apply_warp_rules.ps1` | Windows PowerShell 脚本，应用完整规则集 |
| `apply_warp_rules.sh` | Linux/macOS Bash 脚本，应用完整规则集 |

### 使用方式

**Windows (PowerShell, 需以管理员运行)：**
```powershell
.\apply_warp_rules.ps1
```

**Linux / macOS (Bash)：**
```bash
chmod +x apply_warp_rules.sh
./apply_warp_rules.sh
```

### 注意事项
1. 脚本依赖 `warp-cli` 已安装并注册（`warp-cli registration show` 可见状态）。
2. 消费版 WARP 仅支持 Exclude 模式，脚本通过 `tunnel host add` / `dns fallback add` 实现分流。
3. 隧道排除中子域必须逐条或用 `*.` 通配；DNS 回退中裸主域即覆盖子域。
4. 修改后建议重启 WARP 服务使规则全量生效：`warp-cli disconnect` 后 `warp-cli connect`。

---

## 五、快速验证命令

```powershell
# 1. 确认 .cn 走本地解析 (应显示 via default)
#    查看日志: C:\ProgramData\Cloudflare\cfwarp_daemon_dns.txt
warp-cli dns log enable
Resolve-DnsName cloudflare.cn
# 日志中对应行应含 "via default"

# 2. 确认外国域名走 WARP 隧道 (应显示 via primary)
Resolve-DnsName github.com
# 日志中对应行应含 "via primary"

# 3. 确认 .cn 流量直连 (首跳应为本地路由器, 非 Cloudflare 边缘)
tracert -h 3 <某.cn域名解析出的IP>
```
