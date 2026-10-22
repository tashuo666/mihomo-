# B 站增强版 mihomo 覆写脚本

在 [AIsouler/MyClash](https://github.com/AIsouler/MyClash) 的 `Script.js` 基础上二次修改，新增哔哩哔哩（B 站）分流规则，并修正 PCDN 漏网导致的视频卡顿问题。

## 适用场景

- 使用 Clash Verge Rev / mihomo / [Bettbox](https://github.com/appshubcc/Bettbox) 的内核
- 希望 B 站：**境内直连、境外（港澳台 / 番剧）走代理**
- 希望 **IPv6 记录不被屏蔽**（AAAA 全链路放行）
- 被 B 站视频卡顿困扰，怀疑 PCDN 问题

## 本次修改内容

### 1. 新增 B 站分流规则

注册三个 rule-provider：

| provider | 规则集 | 用途 |
| --- | --- | --- |
| `bilibili` | `geosite/bilibili.mrs` | 境内主站、接口、CDN 域名 |
| `bilibili-!cn` | `geosite/bilibili@!cn.mrs` | 境外站点（bilibili.tv 等） |
| `bilibili_ip` | `geoip/bilibili.mrs` | B 站相关 IP 段 |

新增策略组 `BiliBili`（`select` 类型，默认 `直连`），并在 `prefixRules` 中按顺序注册：

```js
'RULE-SET,bilibili-!cn,BiliBili',
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

同时在 DNS 层做双重保险（`nameserver-policy` + `hosts`，A 与 AAAA 均黑洞）。

修复后实测：播放地址全部回落到官方 CDN（`upos-sz-mirror*.bilivideo.com`），下载速度稳定在 590~618 KB/s。

### 3. AAAA / IPv6 处理

- **完全不屏蔽 AAAA**：DNS 与规则均未加入任何切断 IPv6 的逻辑，实测 `www.baidu.com`、`www.bilibili.com` 的 AAAA 查询正常返回
- **`代理IPV6优先` 保持 `false`**：实测本机 IPv6 延迟 188ms、IPv4 39ms（约 4.8 倍差距），默认不强开；需要时可自行改为 `true`
- B 站域名指定使用国内加密 DNS 解析（`nameserver-policy` 中 `rule-set:bilibili`），保留 AAAA 记录

## 实测验证结果

在内核中加载后逐条验证：

| 域名 | 命中规则 | 出口 |
| --- | --- | --- |
| `www.bilibili.com` | `RuleSet(bilibili)` | 直连 · IPv6优先 |
| `api.bilibili.com` | `RuleSet(bilibili)` | 直连 · IPv6优先 |
| `i0.hdslb.com` | `RuleSet(bilibili)` | 直连 · IPv6优先 |
| `www.bilibili.tv` | `RuleSet(bilibili-!cn)` | `BiliBili` → 代理节点 |
| `mcdn.bilivideo.com` | `DomainSuffix` | `REJECT` |
| `h2.smtcdns.net` | `DomainSuffix` | `REJECT` |
| `*.edge.mountaintoys.cn` | `DomainSuffix` | `REJECT` |
| `www.baidu.com` | `RuleSet(geolocation-cn)` | 直连 |
| `www.google.com` | `RuleSet(google)` | 代理节点 |

配置已通过 `mihomo -t` 校验，规则集可正常下载（bilibili 53 条域名、bilibili-!cn 4 条、bilibili_ip 2 条）。

## 使用方法

### Bettbox

将 `Script.js` 放入覆写脚本位置，或在应用的覆写功能中粘贴内容使 `function main(config)` 生效。

### Clash Verge Rev

1. 将 `Script.js` 保存到配置目录的 `profiles` 下
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

## 免责声明

本项目仅供学习与技术研究使用，请遵守所在地法律法规与相关服务条款。
