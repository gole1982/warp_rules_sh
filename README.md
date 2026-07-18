# Cloudflare WARP 分流规则（域名回退 + 隧道分割）

适用于 **Cloudflare WARP 消费版（Windows / Linux / macOS）** 的分流规则。

核心理念：**国内网站直连（绕过 WARP），国外 LLM / 搜索引擎 / 被墙站点走 WARP 隧道。**
由于消费版 WARP 仅支持 Exclude（排除）模式，实现方式是把国内域名从隧道中**排除**并加入 **DNS 回退**，其余流量默认走隧道。

---

## 一、两种机制的本质区别（实测结论，非推测）

| 机制 | WARP 命令 | 匹配方式 | 关键特性 |
|------|-----------|----------|----------|
| **DNS 回退** (Domain Fallback) | `warp-cli dns fallback add <域>` | **后缀匹配** | 裸 `abc.com` 自动覆盖 `1.abc.com`、`2.abc.com` 等所有子域 |
| **隧道排除** (Split Tunnel Exclude) | `warp-cli tunnel host add <域>` | **域名匹配**（解析后按返回 IP 放行） | 裸 `abc.com` **不**覆盖子域 `x.abc.com`，需 `*.abc.com` 才覆盖 |

### 验证证据

1. **DNS 回退后缀匹配**：配置裸 `qq.com` 后，解析 `mail.qq.com` → 日志显示 `via default`（本地解析器），证明子域命中。
2. **隧道排除不覆盖子域**：仅配置裸 `qq.com` 时，`tracert mail.qq.com` 首跳为 `104.28.0.0`（Cloudflare 边缘）→ 流量**进了隧道**；
   而同时配置 `*.qq.com` 时，`mail.qq.com` 首跳为本地路由器 → **直连**。
3. **国家后缀 `cn` 一条全覆盖**：配置裸 `cn` 后，`cloudflare.cn`、`baidu.com.cn`、`people.com.cn` 均 `via default` 且 tracert 直连。

---

## 二、域名添加方法论（本规则集采用）

对每个**国内常用服务**，按两种机制分别添加（二者匹配方式不同）：

```
DNS 回退列表:   主页          (如 qq.com)         —— 后缀匹配, 裸主页已覆盖所有子域, 不加通配符
隧道排除列表:   主页 + *.主页  (如 qq.com + *.qq.com) —— 域名匹配, 子域需通配符才覆盖
```

- **DNS 回退绝不加通配符**：回退是后缀匹配，裸 `qq.com` 已涵盖 `mail.qq.com` 等全部子域，加 `*.qq.com` 纯属冗余。
- **隧道排除必须加 `*.主页`**：隧道是域名匹配，裸 `qq.com` 不覆盖子域流量，需 `*.qq.com` 才直连子域。
- **不做**逐条枚举子域（mail./v./qzone. 等）——隧道侧用 `*.主页` 一次性覆盖。
- **国家后缀** `cn` 单条覆盖全部 `.cn` 层级（回退与隧道各一条）。
- **国外服务一律不加**直连列表，即使它有国内入口（如 `cn.bing.com`、`bing.com` 整体算国外服务 → 走隧道）。
- 中间含 `cn` 的国内 `.com` 站，按"主页"逻辑处理（如 `oss-cn-hangzhou.aliyuncs.com` → 用 `aliyuncs.com` + `*.aliyuncs.com`）。

### 分类简化对照

| 类型 | 示例 | 简化写法 |
|------|------|----------|
| 国内服务 | `qq.com` | 回退: `qq.com`；隧道: `qq.com` + `*.qq.com` |
| 国家后缀 | 所有 `.cn` | 回退: `cn`；隧道: `cn` |
| 国外服务（不加） | `bing.com` / `google.com` / `openai.com` | 不出现在列表，默认走隧道 |

---

## 三、当前生效规则集

> 从本机 WARP 导出的实际配置快照：DNS 回退与隧道排除各 **175** 条。

结构：
- 国家后缀：`cn`（覆盖全部 `.cn`）
- 国内服务主页 + `*.主页`，覆盖：百度、QQ、淘宝/天猫、京东、拼多多、微博、知乎、B站、抖音、快手、小红书、爱奇艺、优酷、芒果、携程、美团、阿里/腾讯/华为云、163、搜狐、新浪、360、搜狗、飞书、钉钉、WPS、CSDN、滴滴、高德、新华/央视/人民网、gov.cn、edu.cn、国内大模型（DeepSeek/智谱/月之暗面/通义/阶跃/零一万物等）等。

### 默认走 WARP 隧道的流量（未排除即走隧道）
- LLM API：`openai.com`、`api.anthropic.com`、`api.x.ai`(grok)、`generativelanguage.googleapis.com`(gemini)
- 安全平台：`hackthebox.com`、`tryhackme.com`
- 搜索引擎：`google.com`、`duckduckgo.com`、`bing.com`（国外服务，含 cn.bing.com 入口也走隧道）
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
3. 每个国内服务：DNS 回退只加裸 `主页`，隧道排除加 `主页` + `*.主页`；`cn` 单条覆盖国家后缀（两侧各一条）。
4. 修改后建议重启 WARP 使规则全量生效：`warp-cli disconnect` 后 `warp-cli connect`。

---

## 五、快速验证命令

```powershell
# 1. 确认 .cn / 国内子域走本地解析 (应 via default)
warp-cli dns log enable
Resolve-DnsName mail.qq.com
# 日志 (C:\ProgramData\Cloudflare\cfwarp_daemon_dns.txt) 对应行应含 "via default"

# 2. 确认外国域名走 WARP 隧道 (应 via primary)
Resolve-DnsName github.com
# 对应行应含 "via primary"

# 3. 确认国内子域流量直连 (首跳为本地路由器, 非 Cloudflare 边缘)
tracert -h 3 <某国内子域解析出的IP>
```
