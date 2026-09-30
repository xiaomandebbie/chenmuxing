# 09 · 接入 Drivesoid（情绪）

[Drivesoid](https://github.com/A1batr055/Drivesoid) 是一个独立的情绪服务。它怎么安装、怎么配置、情绪怎么算，都看它自己的仓库。

这一篇只讲晨暮星这边怎么接。

## 接上之后会发生什么

- **聊天时**：`vesper-gateway` 每记下一条你说的话，就把它连同最近约 600 字的前文报给 Drivesoid；TA 回完一句，再报一次。上报不等结果，Drivesoid 挂了也不会拖慢聊天
- **醒来时**：phosphor 先让 Drivesoid 刷新一次，再取 TA 此刻最明显的三项情绪（比如「满足 74、喜悦 70、玩乐 43」），放进做决定的上下文。TA 这次的心情以它为底色，为什么会这样由 TA 自己去最近的对话和记忆里找；数字不会写进推送、动态和回复里
- **动态页**：标题下的「TA此刻」显示同样的三项，点进去是心绪页 `/drives`
- **心绪页 `/drives`**：五组情绪卡片（精力、依恋、警觉、愉悦、低落），每项 0–100，标出平常的水平和比上一次的变化。登录和动态页一样
- **不接**：`DRIVES_URL` 留空，上面这些都不做，其他照常

晨暮星这边不会因此多调模型。给消息打情绪标签是 Drivesoid 自己做的，用它自己配置的模型和 key，费用算在那边：每条聊天一次小模型调用，输入大约一千 token。

打开动态页、心绪页只读 Drivesoid 现成的数据，刷新再勤也不会多调模型。

## 前提

- Drivesoid 和晨暮星装在**同一台机器**上。它的上报接口只接受本机请求
- 按 Drivesoid 仓库的说明装好，并让它常驻运行（比如用 pm2）
- 端口用它默认的 `24601`，别和晨暮星的 3001 / 3002 撞

确认它在跑：

```bash
curl -s http://127.0.0.1:24601/api/drives/status | head -c 300; echo
```

返回一段 JSON 就可以往下走。

> Drivesoid 的填表页 `/setup` 和看板只监听本机。在自己电脑上打开要先开隧道：`ssh -L 24601:localhost:24601 用户@服务器`，再访问 `http://127.0.0.1:24601/setup`。

## 接入：两步

**1. 在晨暮星的 `.env` 里加地址**

```bash
cd ~/chenmuxing && sed -i '/^DRIVES_URL=/d' .env && echo "DRIVES_URL=http://127.0.0.1:24601" >> .env
```

**2. 重启三个进程**

```bash
cd ~/chenmuxing && pm2 restart vesper vesper-gateway phosphor --update-env
pm2 logs phosphor --lines 20 --nostream | grep Drivesoid
```

看到 `情绪（Drivesoid）：已接入` 就好了。刷新动态页，「TA此刻」会变成三项数字。

想断开：删掉 `DRIVES_URL` 这一行，再执行一次第 2 步。

## 晨暮星用到的接口

Drivesoid 升级后如果它们变了，这边要跟着改，代码在 `src/drives.js`。

| 谁调 | 接口 | 什么时候 |
|---|---|---|
| gateway | `POST /internal/drives/event`，`{"type":"msg_user","payload":{"text":"…","context":[…]}}` | 记下你的一条消息后 |
| gateway | `POST /internal/drives/event`，`{"type":"msg_assistant","payload":{}}` | 记下 TA 的回复后 |
| phosphor | `POST /internal/drives/session-start` | 醒来读情绪之前，先刷新（最多等 15 秒） |
| phosphor、vesper | `GET /api/drives/status` | 取此刻的情绪，算最明显的三项；心绪页也用它 |
| vesper | `GET /api/dashboard/config` | 心绪页取各项"平常的水平"，读不到就用默认值 |

只有走网关 `chat` 线路的聊天会被上报。直连上游的聊天，Drivesoid 收不到。

## 安全

- 防火墙**不要**开 24601。它只需要在本机被访问，心绪页是 vesper 在服务器上读好再给你看的
- Drivesoid 的 key 存在它自己目录的配置里，和晨暮星的 `.env` 是两份，互不影响

## 排错

| 现象 | 原因 |
|---|---|
| 启动日志是 `情绪（Drivesoid）：未接入` | `DRIVES_URL` 没填，或者改完没带 `--update-env` 重启 |
| 心绪页提示"还没接上情绪系统" | 同上，vesper 没读到 `DRIVES_URL` |
| 心绪页提示"连不上情绪系统" | Drivesoid 没在跑，`pm2 status` 看一下 |
| `drives: 上报 msg_user 失败（Drivesoid 在运行吗？）` | Drivesoid 没在跑，或者地址、端口不对 |
| `drives: 情绪快照还没有或已过期，这次不带情绪` | Drivesoid 刚启动，或者它那边卡住了。偶尔一次没关系 |
| `drives: 刷新情绪快照失败，读现有快照` | 刷新超过 15 秒。会直接读现有快照，不影响这次醒来 |
| 情绪一直不怎么变 | 聊天没走网关的 `chat` 线路，Drivesoid 收不到上报 |

Drivesoid 自己的报错，看它的日志和它仓库的说明。
