# mihomo- / js分流规则

存放 mihomo / Bettbox 用的 JS 覆写脚本。

## Script.js（Bettbox 覆写脚本）

基于 [AIsouler/MyClash](https://github.com/AIsouler/MyClash) 定制，供 Bettbox「URL 脚本覆写」引用：

```
https://raw.githubusercontent.com/tashuo666/mihomo-/main/Script.js
```

定制点：
- DNS 全加密（DoH / DoT）：规避明文 DNS 被网络中间层干扰导致的解析失败
- 直连策略组默认 IPv6 优先
- fake-ip-range6 使用 ULA 网段；关闭 prefer-h3

> 个人定制版，不自动跟随上游更新。
