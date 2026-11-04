# tvbox-config

NAS（广州耀中 · 科龙空调事业部）自动同步的 **TVBox / FongMi 接口清单**。

**用途**：给电视端播放器提供多源清单（**近 60 个点播源**，每天自动体检、死源剔除、新源自动补入）。
这是 **备用 CDN 通道**（主通道是 NAS 上的 Tailscale Funnel 直连地址）。

## 接口地址（备用，按推荐顺序）

```
https://fastly.jsdelivr.net/gh/Jamin-f/tvbox-config@main/tvbox.json   ← 首选（边缘实时性好）
https://cdn.jsdelivr.net/gh/Jamin-f/tvbox-config@main/tvbox.json      ← 同一 CDN 的另一边缘，偶滞后
https://raw.githack.com/Jamin-f/tvbox-config/main/tvbox.json
https://ghproxy.net/https://raw.githubusercontent.com/Jamin-f/tvbox-config/main/tvbox.json
```

> 这几条是**各不相同的 CDN 边缘、各自独立缓存**，一条抽风换另一条即可，内容完全一致。

## 谁在维护

NAS 上三级定时流水线，全自动：

| 时间 | 任务 | 做什么 |
|---|---|---|
| **05:15** | `tvbox-pool.timer` → `tvbox_pool.py` | 拉 20+ 社区聚合仓 + GitHub 搜索**自动发现新源仓库**；去重、滤垃圾、滤成人站、归一化域名 → 源池（当前 **123 个候选**） |
| **05:30** | `tvbox-api.timer` → `tvbox_build.py` | 对池子里每个源**真实搜索一次**做全量探活，只把活源写进正式清单 |
| **05:45** | `tvbox-cdn-push.timer` | 把新清单推到这里 + 刷 jsDelivr 缓存，并更新 `status.txt` 心跳 |

**全自动，无需人工干预。** 清单每天全量重建，所以源数量每天会小幅浮动（死站下线、新站补入）—— 正常现象。

## 文件

| 文件 | 说明 |
|---|---|
| `tvbox.json` | 播放器接口清单（约 60 个点播源），`version` 字段为构建时间戳（yyyyMMddHHmm） |
| `live.m3u` | 直播源清单 |
| `status.txt` | 心跳：最后一次同步时间、版本号、源数量。**这个文件每天都会变**，一看就知道 NAS 是否还在正常同步 |

## 三条通道（内容同一份）

| 通道 | 地址 | 适用 |
|---|---|---|
| Tailscale Funnel（主） | `https://trim-a22c.taileb0849.ts.net/tvbox.json` | 异地 / 手机，走 NAS 直发 |
| **CDN（备用，本仓库）** | 见上面四条地址 | Funnel 不通时的兜底，全球加速 |
| 内网 | `http://192.168.1.194:8099/tvbox.json` | 同一个 WiFi 下最快、无缓存延迟 |

## 播放器怎么填

1. 打开 FongMi / TVBox → **设置** → **配置地址**
2. 「点播」那一栏填上面的地址（首选 Funnel 那条；异地用不了就换 CDN 四条里的任意一条）
3. 确定；首页出现「电影 / 连续剧 / 综艺 / 动漫」分类 = 生效

> **缓存说明**：jsDelivr 对 `@main` 默认缓存 12 小时，本仓库推送后会自动调用 purge API 刷新。
> 但 `cdn.jsdelivr.net` 的边缘 purge 偶尔被限流（throttled），此时该边缘可能要等较久才刷新 ——
> 所以另有 `fastly.jsdelivr.net` / githack / ghproxy 三条独立缓存的地址兜底。
> 想确认云端内容是否已是最新：按 commit sha 取 `https://raw.githubusercontent.com/Jamin-f/tvbox-config/<sha>/tvbox.json`（绕开所有缓存）。
