# tvbox-config

NAS（广州耀中 · 科龙空调事业部）自动同步的 **TVBox / FongMi 接口清单**。

**用途**：给电视端播放器提供多源清单（**50 多个点播源**，多源检索 + 播放失败自动换源，每天自动体检、死源剔除、新源自动补入）。
这是 **备用通道**（主通道是 NAS 上的直连地址：同 WiFi 走内网、异地走 Tailscale Funnel）。

## 接口地址（备用，按「新鲜度」排序）

```
https://raw.githubusercontent.com/Jamin-f/tvbox-config/main/tvbox.json                          ← 首选，直发 GitHub，几分钟内就是最新
https://ghproxy.net/https://raw.githubusercontent.com/Jamin-f/tvbox-config/main/tvbox.json     ← 同上，走加速代理
https://raw.githack.com/Jamin-f/tvbox-config/main/tvbox.json                                   ← 同上，另一家代理
https://cdn.jsdelivr.net/gh/Jamin-f/tvbox-config@main/tvbox.json                               ← jsDelivr CDN，边缘缓存最长 12h，可能慢半天
https://fastly.jsdelivr.net/gh/Jamin-f/tvbox-config@main/tvbox.json                            ← jsDelivr 另一边缘，同上
```

> 几条内容**完全一致**，区别只在缓存新鲜度。前三条直发 GitHub，随时是最新。
> 后两条是 CDN 边缘：jsDelivr 对 `@main` 默认缓存 12 小时，推送后脚本会自动调 purge，
> 但 purge 偶尔被限流（`throttled: true`），此时该边缘只能等 TTL 到期 —— 所以别把它们当首选。
> 要确认「刚推的成功没有」用按 commit sha 取（绕开一切缓存）：
> `https://raw.githubusercontent.com/Jamin-f/tvbox-config/<sha>/tvbox.json`

## 谁在维护

NAS 上三级定时流水线，全自动：

| 时间 | 任务 | 做什么 |
|---|---|---|
| **05:15** | `tvbox-pool.timer` → `tvbox_pool.py` | 拉 20+ 社区聚合仓 + GitHub 搜索**自动发现新源仓库**；去重、滤垃圾、滤擦边站、归一化域名 → 源池（约 120 个候选） |
| **05:30** | `tvbox-api.timer` → `tvbox_build.py` | 对池子里每个源**真实搜索一次**做全量探活，只把活源写进正式清单 |
| **05:45** | `tvbox-cdn-push.timer` | 把新清单推到这里 + 刷 CDN 缓存，并更新 `status.txt` 心跳 |

**全自动，无需人工干预。** 清单每天全量重建，所以源数量每天会在 **50~60 之间小幅浮动**
（死站下线、新站补入、源站自身抽风）—— 上游清单里本来就有一半是僵尸站，这正是「每天扫一遍存活」的意义。

## 文件

| 文件 | 说明 |
|---|---|
| `tvbox.json` | 播放器接口清单（50 多个点播源），`version` 字段为构建时间戳（yyyyMMddHHmm） |
| `live.m3u` | 直播源清单 |
| `status.txt` | 心跳：最后一次同步时间、版本号、源数量。**这个文件每天都变**，一看就知道 NAS 是否还在正常同步 |

## 三条通道（内容同一份）

| 通道 | 地址 | 适用 |
|---|---|---|
| Tailscale Funnel（主） | `https://trim-a22c.taileb0849.ts.net/tvbox.json` | 异地 / 手机，NAS 直发、实时 |
| 内网（家里） | `http://192.168.1.194:8099/tvbox.json` | 同一个 WiFi 下最快、零缓存延迟 |
| **本仓库（备用）** | 见上面五条地址 | 主通道不通时的兜底 |

## 播放器怎么填

1. 打开 FongMi / TVBox → **设置** → **配置地址**
2. 「点播」那一栏填上面的地址（首选主通道两条；死活不通再换本仓库的地址）
3. 确定；首页出现源名 + 顶部「电影 / 连续剧 / 综艺 / 动漫」分类 = 生效

> 清单带 `version` 字段，播放器发现版本变了会**自动重载** —— 源更新是自动生效的，不用手动重填。

## 内容底线

入库前做**两级擦边站过滤**（采源池时滤 + 发布前对成品 `tvbox.json` 再扫一遍），
确保不把成人/擦边站带进家里的电视。仓库历史也保持干净（发现污染会用孤儿分支重写 + 强推，而不是只推新版覆盖）。
