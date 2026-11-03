# mihomo 覆写脚本（B 站增强）

## 快捷覆写 URL

两个版本都基于 [AIsouler/MyClash](https://github.com/AIsouler/MyClash) 二次修改，均新增了哔哩哔哩（B 站）分流规则并放行 AAAA/IPv6。**按需二选一**，不要同时挂两个。

### 全量版（推荐，分流规则更细）

```
https://raw.githubusercontent.com/tashuo666/mihomo-/main/mihomoScript.js
```

加速镜像：

```
https://ghfast.top/https://raw.githubusercontent.com/tashuo666/mihomo-/main/mihomoScript.js
```

```
https://gh-proxy.com/https://raw.githubusercontent.com/tashuo666/mihomo-/main/mihomoScript.js
```

### 精简版（分组少、配置轻）

```
https://raw.githubusercontent.com/tashuo666/mihomo-/main/Script.js
```

加速镜像：

```
https://ghfast.top/https://raw.githubusercontent.com/tashuo666/mihomo-/main/Script.js
```

```
https://gh-proxy.com/https://raw.githubusercontent.com/tashuo666/mihomo-/main/Script.js
```

> 国内网络访问 `raw.githubusercontent.com` 常被干扰（实测直连超时 24s 无响应），建议直接用加速镜像。
> 上述镜像实测均返回完整文件，`ghfast.top` 通常 2s 左右、`gh-proxy.com` 约 5s。
> 镜像服务可用性会变化，失效时换用其他 GitHub 加速前缀，保留后面的原始路径即可。

> URL 引用方式会随仓库更新自动生效，无需重新导入。
> 若内核缓存了旧解析结果，更换脚本后请清一次 `cache.db` 或重启内核。

## 两个版本的区别

| | 全量版 `mihomoScript.js` | 精简版 `Script.js` |
| --- | --- | --- |
| 分流分组 | 约 27 个（YouTube / GitHub / Microsoft / Apple / TikTok / Twitter / Meta / Line / Netflix / Emby / PikPak / Spotify / Crypto / PayPal / Patreon / EHentai 等） | 约 7 个（Google / AI / Telegram / Steam / AdBlock） |
| 额外能力 | 负载均衡组、家宽节点组、不可见字符清洗 | 无 |
| 规则总数（同一订阅实测） | 68 条 | 约 25 条 |
| 适合 | 需要细分各国区服务 | 只要基本可用、想少占内存 |

两者都包含下文的 B 站规则与 PCDN 修复。

## 本次修改内容

### 1. 新增 B 站分流规则

注册三个 rule-provider：

| provider | 规则集 | 用途 |
| --- | --- | --- |
| `bilibili` | `geosite/bilibili.mrs` | 境内主站、接口、CDN 域名 |
| `bilibili-!cn` | `geosite/bilibili@!cn.mrs` | 境外站点（bilibili.tv 等） |
| `bilibili_ip` | `geoip/bilibili.mrs` | B 站相关 IP 段 |

新增策略组 `BiliBili`（`select` 类型，默认 `直连`），并在 `prefixRules` 中注册：

```js
'RULE-SET,bilibili-!cn,BiliBili',
...
'RULE-SET,bilibili,直连',
```

> **顺序很关键**：`bilibili-!cn` 必须排在 `bilibili` 之前。否则境外域名会被主规则集抢先直连，`bilibili.tv` 将永远无法走到代理组。

### 2. 修复 PCDN 漏网（视频卡顿的真实原因）

原脚本虽然用 `hosts` 屏蔽了 PCDN 域名，但实测发现：

- 真实播放域名形如 `b-xxxxxxxxxxxx.edge.mountaintoys.cn`，`hosts` 的 `+.edge.mountaintoys.cn` 在 **fake-ip 模式下对已缓存域名失效**
- 该域名漏到「漏网之鱼」策略组，由远端节点解析并回填缓存，导致 DNS 层策略同样失效
- 播放器实际连的是 PCDN 节点，而非 B 站官方 CDN → 卡顿

**修复方式**：在规则层直接 `REJECT`，使其不依赖任何 DNS 解析结果：

```js
'DOMAIN-SUFFIX,mcdn.bilivideo.com,REJECT',
'DOMAIN-SUFFIX,mcdn.bilivideo.cn,REJECT',
'DOMAIN-SUFFIX,edge.mountaintoys.cn,REJECT',
'DOMAIN-SUFFIX,h2.smtcdns.net,REJECT',
```

同时在 DNS 层做双重保险（`nameserver-policy` + `hosts`，只黑 A 记录，保留 AAAA）。

修复后实测：播放地址全部回落到官方 CDN（`upos-sz-mirror*.bilivideo.com`），下载速度稳定在 650~810 KB/s。

### 3. AAAA / IPv6 处理

- **完全不屏蔽 AAAA**：DNS 与规则均未加入任何切断 IPv6 的逻辑，实测 `www.baidu.com`、`www.bilibili.com`、`i0.hdslb.com` 的 AAAA 查询均正常返回 `240e:...` 等真实 IPv6 地址
- **`代理IPV6优先` 保持 `false`**：实测本机 IPv6 延迟 188ms、IPv4 39ms（约 4.8 倍差距），默认不强开；需要时可自行改为 `true`
- 直连组保留「双栈 / IPv4优先 / IPv6优先 / 仅IPv4 / 仅IPv6」五种选择，可随时切换
- B 站域名指定使用国内 DNS 解析（`nameserver-policy` 中 `rule-set:bilibili`），保留 AAAA 记录

## 实测验证结果

在 mihomo v1.19.33 内核中加载真实订阅后逐条验证：

| 域名 | 命中规则 | 出口 |
| --- | --- | --- |
| `www.bilibili.com` | `RuleSet(bilibili)` | 直连 |
| `api.bilibili.com` | `RuleSet(bilibili)` | 直连 |
| `i0.hdslb.com` | `RuleSet(bilibili)` | 直连 |
| `upos-sz-mirrorali.bilivideo.com` | `RuleSet(bilibili)` | 直连 |
| `www.bilibili.tv` | `RuleSet(bilibili-!cn)` | `BiliBili` → 代理节点 |
| `mcdn.bilivideo.com` | `DomainSuffix` | `REJECT` |
| `mcdn.bilivideo.cn` | `DomainSuffix` | `REJECT` |
| `h2.smtcdns.net` | `DomainSuffix` | `REJECT` |
| `*.edge.mountaintoys.cn` | `DomainSuffix` | `REJECT` |
| `www.baidu.com` | `RuleSet(geolocation-cn)` | 直连 |
| `www.google.com` | `RuleSet(google)` | 代理节点 |

配置已通过 `mihomo -t` 校验；50 个规则集全部下载成功（bilibili 53 条域名、bilibili-!cn 4 条、bilibili_ip 2 条）。

## 使用方法

### Bettbox

在「覆写脚本」处填入上面的 URL（推荐），或把 `.js` 文件放入覆写脚本位置，使 `function main(config)` 生效。

### Clash Verge Rev

1. 将 `.js` 保存到配置目录的 `profiles` 下
2. 在订阅上右键 → 编辑 → 选择该脚本作为覆写

### 通用注意

- 首次使用建议删除内核的 `cache.db`（或清空 DNS 缓存），避免旧的 PCDN 解析结果残留
- 若你的机场节点不含 B 站境外内容需求，可把 `BiliBili` 组内改选其他策略

## 可调开关

脚本顶部 `ruleOptionsEnable` 中：

```js
BiliBili: true,        // 关闭则完全不生成 B 站规则
代理IPV6优先: false,    // 你的网络 IPv6 更好时可改 true
```

## 署名

- 原脚本作者：[AIsouler](https://github.com/AIsouler/MyClash)
- 二次修改：新增 B 站分流规则、修复 PCDN 漏网、确认 AAAA 放行
- B 站分组图标：Koolson/Qure 图标集

## 免责声明

本项目仅供学习与技术研究使用，请遵守所在地法律法规与相关服务条款。
