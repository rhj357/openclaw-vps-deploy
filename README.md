# OpenClaw VPS部署：选什么配置、一年花多少钱，RackNerd年付方案从开机到跑通的全流程

OpenClaw 这类要 24/7 在线的 AI 代理，第一个问题就是它跑在哪。放在自己电脑上，合上盖子代理就下线了，定时任务、消息回复、自动化流程全部断掉。所以部署到一台 VPS 几乎是必选项：机器不睡觉、有固定公网 IP，Stripe、GitHub 这类服务的 webhook 也能稳定送达。

配置这件事不用纠结：OpenClaw 官方给的建议是 2 核 CPU、2GB 内存起步，模型推理全部交给上游的 Anthropic、OpenAI 等服务商，VPS 本身只做消息路由、技能执行、浏览器自动化和记忆读写，负载很轻。真正要做的决定是选哪家、花多少钱——这正好是本文要算清楚的事。

> **选购速览**：个人自用、不跑本地模型，RackNerd 的 2.5 GB 年付方案就够了，约 $22.88/年，折合每月不到 2 美元。只想快速下单的，👉 [查看 RackNerd 当前促销套餐与价格](https://bit.ly/my-rn)。

## OpenClaw 对 VPS 的真实配置要求是什么

官方的推荐线是 2 核 CPU、2GB 内存、Ubuntu 20.04 以上或 Debian，这个配置足够一个重度用户日常使用。原因前面提过：语言模型的推理在云端完成，你的 VPS 干的是消息路由、技能执行、浏览器自动化、记忆读写这些轻量活。

真正的门槛在内存。OpenClaw 官方文档专门写过 Linux 内存压力与 OOM kill 的诊断——进程莫名消失、日志里出现 exit 137，就是内存被内核回收了。1GB 的机器装是装得上，但跑起来之后没什么余量，技能开多了或者浏览器自动化一上，随时可能被杀进程。预算允许的话，2GB 是省心线；确实想跑本地模型的，至少 4GB 起步，这个最后单独说。

存储和流量反而是最不用操心的部分。OpenClaw 本体加工作区数据，10GB 磁盘都绰绰有余，消息类流量一个月几个 GB 到头了。选套餐时真正该看的是内存、机房稳定性和价格，存储大小排不上号。

## RackNerd 的套餐和价格一览

先说结论：RackNerd 走的是低价年付路线，一台 2.5GB 内存的机器一年 $22.88，比主流云厂商的入门款便宜一半以上。作为对比：DigitalOcean、Vultr 这类按月计费的厂商，1GB 内存入门款普遍 $4–6/月，一年就是 $48–72，配置还更低；Hetzner 的 2GB 套餐约 €4/月，一年也要 €48 左右。年付锁价换低价，就是这类商家的生存空间。

下面是它长期促销的 KVM VPS 系列套餐（均含独立 IPv4，1Gbps 带宽，结账时可选机房）：

| 套餐 | CPU / 内存 | SSD 存储 | 月流量 | 年付价 | 购买 |
| --- | --- | --- | --- | --- | --- |
| 1 GB | 1 核 / 1 GB | 17 GB | 1 TB | $10.99/年 | [开通 1GB 方案](https://bit.ly/my-rn) |
| 1.5 GB | 1 核 / 1.5 GB | 25 GB | 2 TB | $15.88/年 | [开通 1.5GB 方案](https://bit.ly/my-rn) |
| 2.5 GB | 2 核 / 2.5 GB | 40 GB | 4.5 TB | $22.88/年 | [开通 2.5GB 方案](https://bit.ly/my-rn) |
| 3 GB | 2 核 / 3 GB | 50 GB | 5 TB | $26.88/年 | [开通 3GB 方案](https://bit.ly/my-rn) |
| 4 GB | 2 核 / 4 GB | 65 GB | 7 TB | $35.88/年 | [开通 4GB 方案](https://bit.ly/my-rn) |
| 5 GB | 3 核 / 5 GB | 80 GB | 9 TB | $46.88/年 | [开通 5GB 方案](https://bit.ly/my-rn) |
| 6 GB | 3 核 / 6 GB | 95 GB | 11 TB | $56.88/年 | [开通 6GB 方案](https://bit.ly/my-rn) |
| 8 GB | 4 核 / 8 GB | 130 GB | 14 TB | $76.88/年 | [开通 8GB 方案](https://bit.ly/my-rn) |

这些都是促销年付价，不同活动期和机房会有小幅浮动，下单页以实时价格为准；另外商家不定期还会放出位置限定的特价机。对 OpenClaw 来说，2.5 GB 这一档内存够用、CPU 有两核，价格也压得住，是大多数人的舒适区：👉 [开通 RackNerd 2.5 GB 年付方案](https://bit.ly/my-rn)。

## OpenClaw VPS部署六步走

整套流程在一台全新机器上十几分钟能走完。还没开机器的话，👉 [先把 RackNerd 的 VPS 开出来](https://bit.ly/my-rn)，系统模板选 Ubuntu 22.04，开通后欢迎邮件里会有 IP 和 root 密码。另外提前准备好 AI 服务商的 API key（Anthropic、OpenAI 或 OpenRouter 都行），后面向导要用。

### 第一步：SSH 连上服务器

bash
ssh root@你的服务器IP


首次连接确认指纹，然后更新一遍系统：

bash
apt-get update && apt-get -y upgrade


### 第二步：安装 Node.js 22

bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs


### 第三步：安装 OpenClaw 并注册守护进程

bash
npm install -g openclaw@latest
openclaw onboard --install-daemon


`--install-daemon` 会把 OpenClaw 注册成 systemd 服务，开机自启、崩溃自动拉起。onboard 向导会让你选 AI 服务商、填 API key，想接 Telegram、WhatsApp 或 Discord 也可以在这一步配。

### 第四步：把网关绑回本机，别暴露公网

先生成一个安全的 token：

bash
openssl rand -base64 32


写入 `~/.openclaw/openclaw.json`：

json
{
  "gateway": {
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "换成上面生成的token"
    }
  },
  "session": {
    "dmScope": "per-channel-peer"
  }
}


这一步的意义：网关只监听 127.0.0.1:18789，别人扫到你的公网 IP 也摸不到面板。网上有些教程让你直接用「公网IP:18789?token=xxx」访问控制台，这种写法等于把整个管理界面挂在公网上，token 再强也只是唯一的门锁，别这么干。

### 第五步：启动并验证

bash
openclaw status
journalctl -u openclaw -f


status 显示 running、日志没有报错，服务端就算跑起来了。也可以在服务器本机 `curl http://127.0.0.1:18789/health`，返回 200 或 401（取决于认证状态）说明网关活着。

### 第六步：做一次安全自检

bash
ss -lntp


输出里 18789 应该只出现在 127.0.0.1 上；如果看到 0.0.0.0:18789，说明绑定没生效，回去改配置。再从你自己的电脑测一下外部可达性：

bash
nc -zv 服务器IP 18789


返回 Connection refused 才是正确状态。最后打开防火墙，只留 SSH：

bash
ufw allow 22 && ufw enable


## 小内存 VPS 值得加的两条优化

如果你选的是 1GB 或 1.5GB 这类低配方案，官方文档给过两条专门针对小型 VPS 的调优，能明显改善命令响应速度。

第一条是开启 Node 的模块编译缓存，把下面两行写进 `~/.bashrc`：

bash
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1


第一行让重复执行的 CLI 命令启动更快（首次运行先建缓存），第二行让网关的常规重启在进程内完成，省掉额外的进程交接，小机器上尤其明显。

第二条是 systemd 层面的重启策略。用官方 onboard 安装的守护进程是用户级服务，可以这样调：

bash
systemctl --user edit openclaw-gateway.service


加入：

ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90


Restart=always 保证崩了自动拉起，RestartSec 设短一点恢复更快。低配机器最好也确认磁盘是 SSD 承载，减少随机读写的冷启动惩罚。

## 部署完之后，从外面怎么连

最省事的是 SSH 隧道。在你自己的电脑上执行：

bash
ssh -N -L 18789:127.0.0.1:18789 root@服务器IP


然后本地直接访问 ws://127.0.0.1:18789，效果和网关跑在本机一样，加密和认证全部走 SSH。

第二选择是 Tailscale，VPS 和本地设备都装上：

bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up


之后在任何设备上通过 Tailscale 分配的 100.x.x.x 地址访问 18789，手机流量下也能连。两条路选一条就够，共同点是都不需要在公网开 18789 端口。

## 下单 RackNerd 之前，先确认这几件事

**机房位置在结账时选。** 它在洛杉矶、圣何塞、西雅图、达拉斯、芝加哥、纽约、亚特兰大、阿姆斯特丹等地都有节点，每个机房提供 test IP 和 looking glass，下单前先 ping 一下选延迟最低的。国内用户一般美西（洛杉矶、圣何塞）表现相对好，但以你自己的实测为准。

**它是年付模式。** 低价促销套餐按年收费，等于一次锁定一年成本。省钱的前提是你确定会用满一年；想先低成本试水，$10.99 的 1GB 方案是一年里最便宜的学费。

**数据备份要自己做。** 跑在 VPS 上的 OpenClaw，状态和工作区都在机器上，官方的建议是把 VPS 当作唯一数据源、定期备份。核心是 `~/.openclaw` 目录，里面装着配置、渠道凭据、代理记忆和会话：

bash
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw


用 cron 定时跑，再传一份到对象存储。商家再稳，快照和自己主动备份也是两回事。

**1GB 方案能跑但别指望太多。** 轻量用法（定时提醒、简单消息回复）配上前面那两条调优可以撑住；技能和自动化开多了，升级到 2.5GB 比反复排查 exit 137 划算。

## 常见问题

**1GB 内存的套餐到底能不能跑 OpenClaw？**
能装能启动，官方写的最低线就是 1 vCPU、1GB 内存、10GB 磁盘。但官方推荐 2GB 起，1GB 机器上开浏览器自动化或多个技能时容易触发 OOM。轻量用法可以，重度使用直接上 2GB 以上。

**18789 端口需要在防火墙上打开吗？**
不要。网关保持 loopback 绑定，远程访问走 SSH 隧道或 Tailscale，这个端口在公网上应该是拒绝连接的状态。

**用 Docker 部署可以吗？**
可以，一条命令起服务：

bash
docker run -d --name openclaw -v openclaw-data:/home/node/.openclaw -p 127.0.0.1:18789:18789 --restart unless-stopped openclaw/openclaw:latest


注意端口映射同样要绑在 127.0.0.1 上，写成 `-p 18789:18789` 就把容器暴露到公网了。

**重装系统怎么办？**
配置在 `~/.openclaw/openclaw.json`，渠道凭据在 `~/.openclaw/credentials/`，代理记忆和会话在 `~/.openclaw/agents/`。重装前把整个 `~/.openclaw` 打包下载，装好再放回去就能恢复。

**想跑本地模型选什么？**
RackNerd 这类入门年付方案不适合本地推理，内存和 CPU 都不够。想零成本折腾可以看 Oracle Cloud 的免费层（ARM 架构 Ampere A1，最高 4 核、24GB 内存，长期免费）；付费路线则直接选 4GB 内存以上的方案。

## 最后的选择建议

按情况对号入座就行：只想验证 OpenClaw 值不值得长期用，1GB 年付方案 $10.99，买的是一年的试错机会；个人日常重度的 24/7 代理，2.5GB 方案在内存和价格之间最平衡；要跑本地模型或多人共用，别在入门款里挑，4GB 内存以上更合适。

下单前确认两件事：机房先用 test IP 实测延迟再选，系统模板选 Ubuntu 22.04。都定了就 👉 [去 RackNerd 看当前的年付特价](https://bit.ly/my-rn)，机器开出来后从第一步的 SSH 连接开始走就行。
