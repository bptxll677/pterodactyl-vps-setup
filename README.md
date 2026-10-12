# Pterodactyl面板安装：从干净的 VPS 装好翼龙面板与 Wings，配置怎么选、新手常踩的坑在哪

大多数人 Pterodactyl面板安装失败，原因不在命令敲错，而在买服务器那一步就已经注定了结果。

Pterodactyl（国内常叫翼龙面板）是免费开源的游戏服务器管理面板，MIT 协议，用 PHP + React + Go 写成，从 2015 年开发到现在。它的核心机制是：每个游戏服都跑在独立的 Docker 容器里，靠容器隔离保证宿主机环境干净，同时精确限制每个服的 CPU、内存和磁盘。这套机制决定了它的一条硬门槛——宿主机必须支持 Docker。

而 Docker 跑不起来的最常见原因，是你买了一台 OpenVZ 或 LXC 架构的廉价 VPS。官方系统需求文档写得很直白：不支持 Windows Server、不支持共享主机、不支持没有 Docker 支持的 OpenVZ/LXC 容器、不支持 32 位系统。这类机器在 `docker run hello-world` 那一步就会报错，前面的 PHP、数据库、NGINX 全白装。

所以这篇文章先把选机器讲清楚，再走一遍完整的安装路径。

## 一、动手之前，四个前置条件

### 1. 虚拟化方式：必须能跑 Docker

KVM 或独立物理机都可以，OpenVZ/LXC 不行。这一条没有任何绕过办法，因为 Wings 守护进程的全部工作就是给每个游戏服起一个 Docker 容器。

顺带一个容易混的点：Wings 也跑不了在多数托管 PaaS 上。它需要特权 Docker 访问、大量任意 TCP/UDP 端口以及 2022 端口的 SFTP，纯容器平台给不了这些。

### 2. 操作系统

官方明确支持的版本：

- Ubuntu 20.04、22.04、24.04
- Debian 11、12、13
- CentOS Stream 8、9
- Rocky Linux 8、9
- AlmaLinux 8、9

Windows Server 不在支持列表里，不是"可以试试"，是真的不行。

### 3. 软件栈

面板端要求 PHP 8.2 或 8.3（老版本不支持）、MySQL 5.7.22 以上（官方推荐 8.0+）、Redis 6 以上、NGINX 1.18 以上或 Apache、Composer 2.x。数据库不要直接拿 root 用户给面板用，官方反复强调要建独立账号。

Redis 严格说是可选项，但生产环境基本等于必装——缓存和队列都靠它。

### 4. 端口

别只放行 80 和 443。Pterodactyl 还需要：

| 端口 | 用途 |
| --- | --- |
| 80 / 443 | 面板网页与 HTTPS |
| 2022 | Wings 的 SFTP，给用户传整合包和存档 |
| 8080 | Wings 守护进程和面板通信 |
| 25565 / 19132 | Minecraft Java 版 / 基岩版默认端口 |

防火墙这块的原则很简单：游戏要用的端口必须开，数据库端口永远不要对公网开。

> 面板和 Wings 不是一回事。面板只是网页后台和数据库，真正启动游戏容器的是 Wings。只装 Panel 不接节点，你会得到一个能登录、能点按钮、但一个服都开不起来的后台，很多人卡在这里以为安装失败。

## 二、面板端安装的关键步骤

以 Ubuntu 22.04 / 24.04 为例，官方文档的流程拆开就是这九步：

1. **装依赖**：`php8.3` 加上 `common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip` 这一串扩展，再加 `mariadb-server`、`nginx`、`redis-server`、`tar unzip git`
2. **装 Composer**：`curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer`
3. **下载面板代码**到 `/var/www/pterodactyl`
4. **装 PHP 依赖**：`composer install --no-dev --optimize-autoloader`
5. **配置环境**：`php artisan p:environment:setup` 和 `p:environment:database`，这两步会写入 `.env`
6. **建表并创建管理员**：`php artisan migrate --seed --force`，然后 `php artisan p:user:make` 按提示填邮箱、用户名、姓名和密码
7. **改属主**：`chown -R www-data:www-data /var/www/pterodactyl/*`
8. **配队列和定时任务**：写一个 `pteroq.service`（`ExecStart` 指向 `artisan queue:work --queue=high,standard,low`），再加一条 cron：`* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1`
9. **配 NGINX 站点并上 SSL**。用 IP 直接跑 HTTPS 会一直有证书警告，正规做法是买个域名，加一条 A 记录指向服务器，再用 Let's Encrypt 签发

第 6 步之后有个容易被忽略的细节：记下 `.env` 里的 `APP_KEY`。这个值丢了，数据库里加密过的数据（比如集成用的 API 密钥）就恢复不了。这不是"建议"，是备份清单上的必选项。

## 三、Wings 节点：真正跑游戏的那台

Wings 是 Go 写的守护进程，装在每个要开服的节点上。要求比面板高一档：

- Docker 20.10.0 以上，推荐 24+
- 硬件最低 2 核、2GB 内存，但这 2GB 是"不含游戏服本身"的
- 磁盘最低 20GB，推荐 50GB 以上 SSD

安装本身很短：

bash
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64"
chmod u+x /usr/local/bin/wings


之后回到面板的 管理 → 节点 里新建节点，把生成的配置文件复制到节点机器上，启动守护进程。Minecraft 是典型的 I/O 密集型负载，机械盘上跑起来会很难受，SSD 这块不建议省。

## 四、配置怎么算：把内存账算清楚

官方给出的面板硬件要求其实很宽松：最低 1 核、1GB 内存、5GB 磁盘，推荐 2 核以上、2GB 以上、10GB 以上。面板本身占用不大，第三方部署指南给的估值是面板约 200–300MB，Wings 约 100–200MB，两者合计按 500MB 预留比较稳妥。

真正吃资源的是游戏服。按官方文档给的规模参考：

| 规模 | CPU | 内存 | 磁盘 |
| --- | --- | --- | --- |
| 1–5 个服 | 2–4 核 | 4–8GB | 50GB SSD |
| 5–20 个服 | 4 核以上 | 8GB 以上 | 100GB 以上 SSD |
| 20 个服以上 | 8 核以上 | 16GB 以上 | 200GB 以上 |

换算成一个具体场景：一台 4GB 内存的机器，扣掉系统加 Pterodactyl 自己的约 500MB，能给 Minecraft 的实际上限大约 3–3.5GB。想同时跑一个生存服、一个创造服加一个 Velocity 代理，8GB 起步才比较从容。

面板和节点放一台还是两台，新手不用纠结。一台就够，所有东西装在一起。等服多起来，再把面板挪到一台小机器上，节点各自独立——代价是两套防火墙、两套更新、节点 token 要同步，维护成本翻倍。

## 五、一键脚本还是手动装

手动装的好处是每一步都知道在干什么，出问题能定位。一键脚本的好处是省时间，坏处是脚本坏了你不一定看得懂哪一段。

社区里比较活跃的是 guldkage 的 Pterodactyl-Installer，支持 Ubuntu 20.04 到 24.04、Debian 11 到 12，PHP 8.3，NGINX 和 Apache 都能选，功能包括装/更新/卸载 Panel 与 Wings、切换域名、装 phpMyAdmin。它的 README 里有一句提醒值得抄下来：脚本是为全新系统准备的，非全新环境失败概率不低。

这条提醒反过来说明一件事——**装 Pterodactyl 的机器最好别用来装宝塔**。宝塔会占用 80/443、安装自己的 PHP 和 NGINX，和 Pterodactyl 的站点配置很容易互相打架。

## 六、机器从哪来：为什么这类需求常落到雨云

把前置条件列出来，选机器的标准其实就固定了：KVM 架构、能拿到 root、能自选 Ubuntu/Debian 系统镜像、有独立公网 IPv4、带宽别太寒酸、最好还能先试用。

雨云是 2018 年成立的国产云服务商，持有工信部 ISP 和 IDC 证，产品线包括云服务器、游戏云、裸金属服务器、虚拟主机、对象存储、CDN 和云应用。对装 Pterodactyl 这件事来说，几个实际用得上的点：

- 采用 KVM 虚拟化，支持 Linux 和 Windows 系统镜像，Docker 能正常跑
- 机房覆盖美国洛杉矶、日本东京、中国香港、韩国首尔、德国法兰克福，以及深圳、宿迁、十堰、襄阳、宁波、温州等国内节点，香港和美国节点免备案
- 云服务器/游戏云产品支持 **1 元试用一天**（每款限 1 次，配置越高试用价可能略高）
- 支持 7 天无理由退款，每个账号限 3 次

体验过雨云这套流程的人通常会提到试用机制的价值：装面板这件事，从系统初始化到能登录后台大约十几分钟，花一元钱先验证一遍 Docker 起不起得来、端口放不放得通，比付了月费再踩坑划算得多。

👉 [先花 1 元开一台机器试一遍安装流程](https://www.rainyun.com/MzA2MTc2_)

## 七、雨云当前在售套餐与价格

下面是雨云官网公开的热销方案。价格是原价，实际支付会因为新用户券和年付折扣而低一截；套餐和价格会随活动变动，下单前以结算页显示为准。

**云服务器 · 美国（AMD EPYC 7702，流量不限型）**

| 方案 | CPU | 内存 | 磁盘 | 带宽 | 价格/月 | 开通 |
| --- | --- | --- | --- | --- | --- | --- |
| KVM 入门版 | 1 核 | 1GB | 30–2000GB | 上行 20M / 下行 30M | ￥25 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 标准版 | 2 核 | 2GB | 30–2000GB | 上行 30M / 下行 50M | ￥34 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 中配版 | 2 核 | 4GB | 30–2000GB | 上行 40M / 下行 60M | ￥44 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 进阶版 | 4 核 | 4GB | 30–2000GB | 上行 60M / 下行 80M | ￥60 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 高配版 | 4 核 | 8GB | 30–2000GB | 上行 80M / 下行 100M | ￥113 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 旗舰版 | 8 核 | 16GB | 30–2000GB | 上行 100M / 下行 150M | ￥197 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |

**云服务器 · 中国香港（Xeon Gold，流量叠加型，免备案）**

| 方案 | CPU | 内存 | 磁盘 | 带宽 | 价格/月 | 开通 |
| --- | --- | --- | --- | --- | --- | --- |
| KVM 入门版 | 1 核 | 1GB | 30–2000GB | 上行 10M / 下行 10M | ￥28 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 标配版 | 2 核 | 2GB | 30–2000GB | 上行 20M / 下行 20M | ￥35 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 中配版 | 2 核 | 4GB | 30–2000GB | 上行 20M / 下行 20M | ￥48 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 高配版 | 4 核 | 4GB | 30–2000GB | 上行 30M / 下行 30M | ￥68 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 顶配版 | 4 核 | 8GB | 30–2000GB | 上行 30M / 下行 30M | ￥108 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| KVM 旗舰版 | 8 核 | 16GB | 30–2000GB | 上行 50M / 下行 50M | ￥188 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |

**游戏云 VPS · 固定计费（国内高防节点）**

| CPU 机型 | CPU | 内存 | 带宽 | 系统盘 | 数据盘 | 价格/月 | 开通 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| E5 2666v3 | 2 核 | 4GB | 10M | 30G | 20G | ￥65 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| Gold 6146 | 2 核 | 4GB | 10M | 30G | 20G | ￥80 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| Ryzen 5900X | 1 核 | 2GB | 5M | 30G | 20G | ￥125 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| Ryzen 7950X | 1 核 | 2GB | 5M | 30G | 20G | ￥139 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| i9 13900KF | 1 核 | 2GB | 5M | 30G | 20G | ￥145 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |
| i9 14900KF | 1 核 | 2GB | 5M | 30G | 20G | ￥145 | [ 查看套餐](https://www.rainyun.com/MzA2MTc2_) |

游戏云这条线的定价逻辑和云服务器不一样：Ryzen 5900X、7950X、i9 13900KF/14900KF 这几款是高频 U，单核性能强，价格自然高，因为它们面向的是 Minecraft 这种多核优化差、吃单核频率的场景。如果你只是想装 Pterodactyl 面板本身，用不到这些；如果面板和游戏服要挤在同一台机器上，E5 2666v3 或 Gold 6146 的 2 核 4G 起步档更划算。

另外要注意，游戏云的动态计费模式适合"不经常玩"的场景，闲时更省钱；固定计费适合长期开着的服。

## 八、优惠怎么叠加

根据多方公开测评和雨云论坛的说明，新用户走优惠通道注册可以拿到折扣券，绑定微信后还能领一张首月 5 折券；积分商城里通常能兑换 8 折新购券。重点在于 8 折券可以和官方的年付 7 折叠加，也就是年付大约 5.6 折。券的类型和额度会调整，以账户里实际发放的为准。

按 4 核 8G 的美国高配版算一笔账：原价 ￥113/月，年付 7 折约 ￥79/月，再叠 8 折券大约 ￥63/月。这个价位买 KVM、独立 IP、可用 root 装 Pterodactyl 的机器，属于合理区间。

👉 [注册后进积分商城看看能领到什么券](https://www.rainyun.com/MzA2MTc2_)

## 九、几个高频问题

**1 核 1G 能装面板吗？**
官方最低要求是 1 核 1GB，能装，能登录。但装完之后你几乎没有余量再开游戏服，只能当作一个纯管理后台用，游戏跑在别的节点上。

**装 Pterodactyl 需要备案吗？**
机器本身不需要域名就能访问面板，但要正经上 HTTPS 就得有域名指向服务器。香港和美国节点免备案，适合不想折腾备案的场合。

**502 报错怎么排查？**
按顺序看三处：PHP-FPM 有没有在跑、NGINX 站点配置里的 `root` 是否指向 `/var/www/pterodactyl/public`、`.env` 里的 `APP_URL` 是否和实际访问地址一致。绝大多数 502 出在最后一条。

**能不能先装宝塔再装 Pterodactyl？**
不建议。两者都要占 80/443 和 NGINX 配置，冲突概率高，排查起来很费时间。要用一键脚本，就在全新系统上跑。

**整合包服要注意什么？**
给 Java 参数留足内存，Forge 和大整合包比纯净服吃得多。分配内存时记得面板和 Wings 自己还要占约 500MB。

## 结语

Pterodactyl面板安装这件事，流程本身是标准的、有官方文档可查的，真正的变量在服务器上：虚拟化必须是 KVM，系统必须是全新安装的 Linux，端口必须放开，磁盘最好是 SSD。把这四点在上机器之前确认掉，剩下的就是照着文档敲命令，或者跑一次一键脚本。

配置选择上给一个直接的建议：只想先跑通面板，选美国 2 核 2G（￥34/月）或香港 2 核 2G（￥35/月）够用；要把面板和一到两个 MC 服放一起，从 4 核 8G 起步；打算多开几个服或者做长期社区服，8 核 16G 才不至于两个月就升配。拿不准的话，1 元试用一天先验证一遍，比看任何测评都准。

👉 [从雨云开通一台能装 Pterodactyl 的 VPS](https://www.rainyun.com/MzA2MTc2_)
