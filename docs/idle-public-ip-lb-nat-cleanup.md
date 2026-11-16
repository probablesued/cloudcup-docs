# 闲置公网 IP、负载均衡与 NAT 网关：空转计费与清理清单

实例停掉或删掉以后，弹性公网 IP、静态公网 IP、负载均衡和 NAT 网关经常还是独立资源。它们的保有费或小时费，不跟着“这段时间没有业务流量”一起停。固定公网 IP 跟着实例走，弹性或静态公网 IP 往往要单独释放。删掉 NAT 网关或负载均衡，也不自动等于关联的公网 IP 已经停止计费。

磁盘、快照和云服务器固定公网带宽的账单拆解见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。谁可以释放 EIP、删除 NAT 或删除实例，见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。账单用哪种币种出，见 [云账号充值与多币种付款核对](multi-currency-top-up-verification.md)。这三篇的内容这里不重复。地域价目表不在这里抄成一份现行统一价。页面上出现的单价，只作为撰写时该页的目录价或示例，以购买页和定价页为准。

核对时间：2026-11-16。资料核对日期：2026-11-16。计费项、未绑定或已停止时是否仍收费、最短计费时长、删除后资源是否还在，都来自这一天打开的官方页面，并附有链接。打不开、正文没有写明，或同一主题两页说法不同的，下面按读到的原文分开写，不补成一个数。

范围：阿里云中国站用 help.aliyun.com。国际站只打开了英文按量付费 EIP 页（www.alibabacloud.com，页眉 Last Updated: Oct 26, 2026）；ALB、CLB、NLB、NAT 的国际站页面这次没有打开，不要把中国站菜单套到国际站。腾讯云只核对了中国站 cloud.tencent.com。用中国站文档编号去打开 www.tencentcloud.com/document/product/1199/41692，返回 404，国际站计费页这次没有打开。AWS 使用 aws.amazon.com 与 docs.aws.amazon.com 的全球文档，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 azure.microsoft.com 与 learn.microsoft.com 的全球文档，中文定价页是同一套全球定价的中文界面；由世纪互联运营的 Azure 没有打开。Google Cloud 使用 cloud.google.com 与 docs.cloud.google.com。页面改版后，以你账号所属站点的当前文档为准。

## 先看这几件事

1. 没流量，保有费或小时费仍可能继续。按流量计的出方向费用会随着流量下降；按带宽峰值、按实例小时、按公网 IP 保有计的项目，不看这一小时有没有人访问。
2. 删除实例，不一定释放弹性公网 IP、负载均衡或 NAT 网关。固定公网 IP 和弹性/静态公网 IP 不是同一套规则。前者多半随实例回收，后者通常要再执行一次释放。
3. 删除 NAT 网关，常见结果是解除和公网 IP 的关联，公网 IP 仍留在账号里继续计保有费。删除负载均衡，也不要默认它名下的 EIP 已经释放。
4. 释放之前先确认没有业务依赖：DNS、防火墙白名单、SNAT/DNAT、负载均衡后端、监控探活。确认不再使用，再按后面的清单释放。

## 对照

下表只填这次打开的页面写明的规则。单元格写“这次没读到”，是因为对应页面没有打开，或打开了但正文没有这句。

| 云 | 公网 IP：没流量或未绑定 | 负载均衡：没有业务流量 | NAT：没有业务流量 | 删掉实例或网关之后 |
| --- | --- | --- | --- | --- |
| 阿里云中国站 | 按量 EIP 未绑定仍收配置费（公网 IP 保有费），除非页面列出的豁免成立。包年包月 EIP 不收这项配置费。固定公网 IP 随实例释放 | ALB：实例未释放就按小时收实例费；无流量不产生 LCU 费。CLB：实例还在就收实例费；公网实例另收公网 IP 保有费。NLB：实例费从创建到释放按小时计，无流量问答这页没写 | 实例未删除就继续收实例费。CU 按该小时处理流量。只解绑 EIP，NAT 和 EIP 都不停止计费 | 释放 ECS 不释放 EIP。删除 NAT 后要另释放 EIP。NLB 自动分配的 EIP 随 NLB 释放；选用的已有 EIP 会留下 |
| 阿里云国际站 | 英文按量付费页同样是流量或带宽费，加配置费。单价用美元。欠费后的天数和中国站不是同一段文字 | ALB、CLB、NLB 的国际站页面这次没读到 | NAT 的国际站页面这次没读到 | 国际站 EIP 页写释放后停止计费，并可能仍收到已发生周期的账单 |
| 腾讯云中国站 | 按流量、共享带宽包：未绑定收 IP 资源费；全地域配额大于 2000 时，绑定后也收。按小时带宽、包月带宽不收 IP 资源费。普通公网 IP 不是 EIP | 无访问或未绑定后端，仍按小时收实例费。LCU 和网络费页这次没打开 | 标准型实例费按使用时长。CU 按活跃连接、新建连接、处理流量三者取高。EIP 另行计费 | 销毁进回收站后，普通公网 IP 会释放，EIP 保留。删除 NAT 会解除 EIP 关联，不从账号释放该 EIP |
| AWS | 在用和闲置的公共 IPv4，定价页都写成每小时 0.005 美元，按秒计、最少 60 秒。当前页没有“绑定到运行中实例的第一个 EIP 免费” | ALB、NLB、Gateway LB、Classic LB：运行的每个小时或不足一小时都按一小时计，再加容量单位或 Classic 的流量 | 网关处于已预置且可用，就收 NAT Gateway 小时费，不足一小时按一小时，另加数据处理 | 停止实例会释放自动分配的公网 IPv4，EIP 仍关联并计费。终止实例会解绑 EIP。删除 NAT 会解绑 EIP，不从账号释放 |
| Microsoft Azure | 静态公共 IP：不论关联资源是否处于已停止（解除分配），都收费，删除 IP 资源后停止。动态公共 IP：关联的虚拟机已停止（解除分配）时不计。定价页单价这次显示为占位 | Basic 负载均衡的产品页写 2025-09-30 已退役。Standard、Gateway 的定价页这次超时，小时费不填 | 概述和 FAQ 写 Standard 与 StandardV2 同价，并指向定价页。定价页这次超时，小时单价不填 | 删除 NAT 网关不删除其公共 IP 或前缀。删除虚拟机是否一并删除公共 IP 资源，这次没有打开专门页面 |
| Google Cloud | 未分配到资源上的静态外部 IP，单价高于正在使用的。停着的虚拟机上，静态外部 IP 仍算在用。临时外部 IP 在实例停止或删除时释放 | 转发规则按小时收费，不按这一小时有没有流量。绑在转发规则上的静态外部 IP 不计 IP 费 | Public NAT 的网关小时费按拿到 NAT 分配的虚拟机数量计，另加处理的 GiB，以及网关使用的外部 IP 小时费 | 删除实例不自动释放静态外部 IP。Cloud NAT 的删除命令页这次没有打开 |

## 阿里云

中国站规则来自 help.aliyun.com。国际站只对照了英文 EIP 按量付费页。国际站账号不要只按下面的中文菜单操作。

### 弹性公网 IP

[弹性公网 IP](https://help.aliyun.com/zh/eip/elastic-ip-address) 把 EIP 写成可以独立购买和持有的公网 IPv4。它能绑到专有网络 ECS、公网 NAT 网关、负载均衡等资源上，也能再解开。

[按量付费](https://help.aliyun.com/zh/eip/pay-as-you-go/) 的计费项是公网网络费，加 EIP 配置费（公网 IP 保有费）。

- 按使用流量：按出方向流量计，入方向不计。计费周期 1 小时，不足 1 小时按 1 小时。2024-12-17 00:00（UTC+8）之后首次创建按使用流量 EIP 的，流量费由云数据传输 CDT 出账。此时间点之前已经创建过的，仍按该页规则，也可以自行升级到 CDT。
- 按固定带宽：按设定的带宽峰值和持有时长计，与这一小时的实际流量无关。当天改过带宽峰值时，计费峰值取当天最大值。不足 1 小时按 1 小时，按天结算。
- 配置费：按量 EIP 申请成功后就开始计。未绑定任何云资源时，该页写明仍要支付配置费。释放后停止计费，但按流量的可能在下一小时收到本小时账单，按带宽的可能在次日收到当日账单。

BGP（多线）和 BGP（多线）精品不收配置费的情况，按量付费页写了三条：直接绑定到专有网络 ECS 或 ECI，并且主账号可保有的 EIP 数量（配额，不是实际持有量）小于或等于 2000；从 IP 地址池分配的 EIP，改为按地址池的公网 IP 保有费计；本地自带公网 IP 上云。该页的例子是，绑到 CLB 的 EIP 仍收配置费；绑到弹性网卡时，如果选的是已绑定主网卡的弹性网卡，也仍收。配额提升到 2000 以上之后，连直接绑到 ECS 的 EIP 也改为收取。本地运营商线路不区分是否绑定，都收配置费。包年包月 EIP 不收这项配置费。

地域单价表在按量付费页上，并写明仅供参考、以购买页为准。这里不抄。

释放只适用于按量 EIP。包年包月不能直接释放。本地运营商线路创建成功后 30 天内不能释放，接口错误码 `Forbidden.MinimumHoldingPeriod` 写的是满 30 天后再操作。被安全锁定的不能释放。连续 EIP 组里的单个地址不能单独释放。

控制台路径在产品页：EIP 控制台选择地域，操作列选择释放。释放前先移出共享带宽、解除绑定、去掉关联的 SNAT 或 DNAT。强制释放会先解绑并移出共享带宽；如果地址还被 NAT 的 SNAT 或 DNAT 占用，仍要先删条目。连续 EIP 组不支持强制释放。生产地址可以在详情页开删除保护。[ReleaseEipAddress](https://help.aliyun.com/zh/eip/developer-reference/api-vpc-2016-04-28-releaseeipaddress-eips) 只释放状态为 Available 的按量 EIP。

产品页还写：加入共享带宽后，EIP 不再收取自身的公网网络费（流量费和带宽费）。这句没有写配置费是否同时停止，这里不补。释放后，指向该 EIP 的 DNS A 记录不会自动改掉。

欠费：中国站按量付费页写，超出延期免停额度后进入暂停服务，暂停满 7 天仍未缴清会被自动释放。延期免停额度本身的天数，这一页没有给出数字。国际站英文页写的是另一套：欠费后服务再保持 15 天，然后暂停；暂停后再满 15 天自动释放。两页不要合成一个天数。

### 固定公网 IP 和 EIP

[IP 地址](https://help.aliyun.com/zh/ecs/user-guide/ip-address/) 把专有网络 ECS 的公网地址分成固定公网 IP 和 EIP。固定公网 IP 在实例释放时回收。带宽设为 0 Mbit/s 时也会回收。按量实例使用节省停机后，固定公网 IP 被释放，再次启动不保证还是同一个。EIP 不会随实例释放而回收。

[DeleteInstance](https://help.aliyun.com/zh/ecs/developer-reference/api-ecs-2014-05-26-deleteinstance) 同样写明：EIP 不随实例释放，要用 ReleaseEipAddress 释放。云盘和自动快照是否一起删，由磁盘上的参数决定，见账单那篇，这里不展开。

[释放实例时保留弹性网卡](https://help.aliyun.com/zh/ecs/user-guide/eni-retention-and-reuse)：网卡随实例释放时，EIP 只是解除绑定，还要手动释放或改绑；网卡保留时，EIP 的绑定关系也保留。

### 应用型负载均衡 ALB、传统型 CLB、网络型 NLB

这次打开的是这三种。没有把没打开的负载均衡产品写进同一句。

[ALB 计费规则](https://help.aliyun.com/zh/slb/application-load-balancer/alb-billing-rules)：只支持按量付费。费用是实例费、LCU 费，公网实例再加公网网络费。实例费从创建完成到释放结束按小时计，不足 1 小时按 1 小时。该页目录价把基础版、标准版、WAF 增强版写成每小时 0.049 元、0.147 元、0.245 元，并写明以购买页为准。LCU 按新建连接、并发连接、处理数据量、规则评估四项取高。无流量时，该页问答写明不产生 LCU 费。同一页后文又写，转发规则即使无流量也会占用控制面并纳入规则评估数参与 LCU 计费。两句都留在这里。公网 ALB 通过 EIP 提供公网能力；没有业务流量时，只要还绑着 EIP，仍产生公网 IP 保有费。

[CLB 按量付费](https://help.aliyun.com/zh/slb/classic-load-balancer/product-overview/pay-as-you-go)：按使用量计费，以及已于 2026-01-01 00:00:00（UTC+8）停止新购的按规格计费。2024-12-01 00:00:00 起，原来的实例费改名为公网 IP 保有费，并新增一项实例费。实例费是实例还在就收，该页把单价写成 0.147 元/小时。2024-12-01 之前创建的实例，该页写暂时不收实例费，限时到 2027-11-30 23:59:59。公网 IP 保有费只对公网按量实例收取，和这个 IP 有没有被使用无关。私网实例不收公网 IP 保有费、流量费和带宽费；私网 CLB 另绑的 EIP 在 EIP 账单里，不在 CLB 账单里。

同一页的问答：停止实例只暂停转发。实例还在时，公网 IP 保有费、实例费、按规格实例的规格费、按固定带宽的带宽费继续收；流量费和 LCU 费在停机期间不再产生。删除（释放）之后，这些计费停止，并释放公网 IP。

[NLB 计费规则](https://help.aliyun.com/zh/slb/network-load-balancer/product-overview/nlb-billable-items) 把实例费写成从创建完成到释放结束按小时收取，不足 1 小时按 1 小时。目录价写 0.147 元/小时，并注明限时免实例费已于 2023-10-01 结束。LCU 按用量。公网网络费走 EIP。这一页没有 ALB 那种“无流量是否仍收实例费”的问答，不把 ALB 的问答套过来。

[创建和管理 NLB 实例](https://help.aliyun.com/zh/slb/network-load-balancer/user-guide/create-and-manage-an-nlb-instance)：系统自动分配的 EIP，在释放 NLB 或从公网改成私网时一并释放。创建时选的已有 EIP 会保留。释放入口是 NLB 控制台、实例操作列的释放。删除保护开着时不能释放。被容器服务 Kubernetes 版管理的 NLB，释放后那个云服务不能继续用这台负载均衡。

### NAT 网关

[NAT 网关计费](https://help.aliyun.com/zh/nat-gateway/nat-gateway-billing) 覆盖公网 NAT 和 VPC NAT。两项都是实例费和容量单位 CU 费。公网 NAT 上的 EIP 另计。付费类型是后付费按使用量。资源包只抵扣，不改变付费类型。单可用区容灾只支持按量，不能用资源包。包年包月和按固定规格已停止新购。

计费周期 1 小时，不足 1 小时按 1 小时。实例费按创建完成到释放结束的时长。自 2025-09-26 起，CU 只按该小时处理流量计算，处理流量含入方向和出方向，1 GB 按 2^30 字节。该页的例子里，处理流量为 0 的 VPC NAT 没有 CU 费，实例费仍按 1 小时计。

停止计费这一节写了三句：删除实例后 NAT 停止计费；要停 EIP，必须再释放 EIP，包年包月 EIP 到期后才自动停止服务；后端已停或没有流量时，只要 NAT 还在，实例费继续。只把 EIP 从 NAT 上解绑，两边都不停止计费。删除后仍收到账单，该页解释为删除前那个周期的延迟出账。

[使用公网 NAT 网关实现公网访问](https://help.aliyun.com/zh/nat-gateway/user-guide/use-internet-nat-gateway-for-public-network-access) 的清理顺序：

1. 在实例详情的 SNAT、DNAT 页签删除条目。删除不可恢复。
2. 在绑定的弹性公网 IP 页签解除绑定。只解绑仍收 EIP 配置费，要到 EIP 控制台再释放。
3. 在实例操作列删除。未解绑、未删条目时，可以选择强制删除。删除保护要先关掉。

[DeleteNatGateway](https://help.aliyun.com/zh/nat-gateway/developer-reference/api-vpc-2016-04-28-deletenatgateway-natgws) 在 `Force=true` 时自动删除 SNAT、DNAT，解绑 EIP，并删除未删的 NAT 带宽包。参数说明写的是解绑，不是释放 EIP。控制台那句“删除 NAT 网关及其包含资源”没有逐项列出包含的是不是 EIP 本身，不把这句理解成 EIP 已经被释放。

## 腾讯云

以下来自中国站。国际站页面这次没有打开。

账户分标准账户和传统账户。[计费概述](https://cloud.tencent.com/document/product/1199/41692) 写 2020-06-17 零点后注册的是标准账户，更早的账号到控制台看自己的类型。两种账户的价格该页写成完全相同，计费主体不同。

### 弹性公网 IP 和普通公网 IP

[IP 资源费用](https://cloud.tencent.com/document/product/1199/51694)（页眉最近更新 2026-08-21）写：按小时带宽和包月带宽的 EIP 不收 IP 资源费。按流量和共享带宽包的 EIP，在账号全地域 EIP 配额小于或等于 2000 时，只在未绑定云资源时收 IP 资源费；配额大于 2000 时，绑定与否都收。计费从申请开始，配额不超过 2000 时在绑定期间暂停、解绑后恢复，释放时停止。时长算到秒，每小时结算上一小时，不足一小时按秒数除以 3600。

该页价目表不区分线路类型：中国大陆地区、雅加达、首尔、法兰克福、圣保罗、利雅得、新山为 0.20 元/小时；中国香港、新加坡为 0.30 元/小时；弗吉尼亚、硅谷、曼谷、东京、大阪、克雷塔罗为 0.25 元/小时。这是这一页列出的价格。

计费概述的标准账户表还有一句：按小时带宽、包月带宽在未绑定时免 IP 资源费，公网网络费仍在 EIP 上收取。同一表里“已绑定”那一格，这次打开的正文是空的。费用组成另一句写的是“绑定云资源后，访问公网时，会收取公网网络费用”。两句都保留，不合成一个条件。传统账户未绑定时收 IP 资源费、不涉及公网网络费；已绑定且配额不超过 2000 时，公网网络费在 CVM 等被绑资源上收取。

[释放按量计费 EIP](https://cloud.tencent.com/document/product/1199/41704)（页眉最近更新 2023-10-31）：先解绑，再在公网 IP 控制台选择地域，操作栏更多、释放，勾选确定释放以上 IP。页首写释放后不再收取 EIP 的任何费用。费用表里，传统账户写不收取任何费用；按小时带宽写停止 IP 资源费和公网网络费；包月带宽指向退还流程。按流量那一格在这次打开的表格里没有独立说明。[解绑 EIP](https://cloud.tencent.com/document/product/1199/41703)（页眉最近更新 2025-06-11）写：NAT 网关上如果这是唯一的 EIP，不能解绑；处于 DDoS 封堵的 EIP 可以解绑，不能释放。

普通公网 IP 是另一项。[公网服务](https://cloud.tencent.com/document/product/213/5224) 写：用户不能主动释放和实例关联的公网 IP。销毁按量实例，或包年包月到期后销毁，才会释放它。把 EIP 关联到实例时，原有公网 IP 被释放；取消关联后，实例会再拿到一个新的公网 IP，原来的不能拿回来。

[回收或恢复实例](https://cloud.tencent.com/document/product/213/4931)：包年包月销毁后进回收站 15 天，按量选择 2 小时后释放或欠费销毁的进回收站 2 小时。超过保留时长后，普通公网 IP 自动释放；EIP 和自行创建的弹性网卡保留，不再需要就另行释放。实例进入回收站后，不主动解除和负载均衡的搭载关系。同一页对“立即销毁并立即释放”写的是所有资源将被释放、不进回收站。这句没有再说明其中是否包含 EIP，不补。

### 负载均衡 CLB

[实例费用](https://cloud.tencent.com/document/product/214/8848)（页眉最近更新 2026-09-02）写：负载均衡购买后，即使处于闲置状态（无访问，或者未绑定后端主机），仍按小时收实例费。公网实例可以按量或包年包月，内网实例只有按量。按量每小时结算，不足一小时按一小时，创建时预先扣除一小时实例费。计费从创建成功到你发起销毁。该页按地域列出了每小时和每月的实例费，例如广州等中国大陆若干地域的按量实例费是 0.2 元/小时、包年包月是 144 元/月。完整地域表以该页为准，这里不逐行抄。

LCU 单价页和网络费用页这次没有打开，不写 LCU 系数和带宽单价。删除实例的点击步骤这次也没有打开删除页。

### NAT 网关

[公网 NAT 网关计费概述](https://cloud.tencent.com/document/product/552/18172)（页眉最近更新 2025-10-17）：创建时预先扣除一小时实例费。

标准型收实例费和 CU 费，都是后付费、按小时。实例费等于单价乘使用时长。CU 取该小时活跃连接、新建连接、处理流量三者换算后的最大值。该页同时列出“单价”和自 2023-06-01 起新购使用的“官网优惠价”，存量价格不变，并写最终优惠还和账号上的其他优惠有关。公网 NAT 的计费不包含 EIP。

传统型收实例费和网络费用。实例费按小型、中型、大型和地域列出小时价。标准账户的网络费在所绑 EIP 上计，不在 NAT 上重复计。传统账户的网络费在 NAT 上按流量、按小时结算。这一节末尾写：双机热备每 3 秒向主备各发一个 5 KB 探测包，每天 0.2747 GB，并给出中国大陆（不含港澳台）、中国香港、北美对应的费用。这句放在传统型网络费用下面，没有写标准型的 CU 是否同样计入这笔探测流量。

[公网 NAT 使用限制](https://cloud.tencent.com/document/product/552/12955)（页眉最近更新 2025-11-05）：删除 NAT 网关会解除弹性公网 IP 的关联，不会从用户账号释放该弹性 IP。控制台删除的逐步点击，这次没有另开操作页。

## AWS

本节使用全球商业文档。VPC 定价页写，在用和闲置公共 IPv4 的这条价格适用于全部 AWS 商业区域、美国 GovCloud、AWS 中国区域和 Local Zones。操作步骤来自 docs.aws.amazon.com，中国区文档 docs.amazonaws.cn 没有打开，中国区账号不要只凭全球文档推断控制台差异。

### 公共 IPv4 和 Elastic IP

[Amazon VPC 定价](https://aws.amazon.com/vpc/pricing/) 的 Public IPv4 Address：账号下每个公共 IPv4 按小时计费，按秒计算，最少 60 秒。在用（关联到你拥有的 AWS 资源）和闲置（在账号里、没有关联到资源）的小时价，该页都写成 0.005 美元。不专属于你的资源的公共 IPv4 不计，例子是没有按桶专用的 Amazon S3 地址。

这一页没有“绑定到运行中实例的第一个 Elastic IP 免费”。[Elastic IP 地址](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) 写的是：无论在用还是闲置都收费；解绑后地址仍分配在账号里，直到你明确释放。

同一份 VPC 定价还把 Amazon 提供的连续 IPv4 地址块写成另一个价格：块里每个地址每小时 0.008 美元，按小时计，不足一小时按一小时，在地址块分配到账号时就开始计，不再单独区分在用和闲置。不要和上面的 0.005 美元合成一个价。

[实例停止和启动](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-ec2-instance-stop-start-works.html)：停止后，启动时自动分配的公网 IPv4 会丢失；EIP 仍留在网卡上，停止期间继续为已关联的 EIP 计费。[实例状态变化](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)：停止或休眠后，除非实例还有辅助网卡或已关联 EIP 的辅助私有 IPv4，否则会换一个新的公网 IPv4。终止实例时，EIP 与实例解绑。这张表没有写终止会释放 EIP。[实例 IP 地址](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html) 写，自动分配的公网 IPv4 在停止、休眠或终止时释放回地址池，不能再指定拿回来。

[release-address](https://docs.aws.amazon.com/cli/latest/reference/ec2/release-address.html)：默认 VPC 里，释放会自动解绑。非默认 VPC 要先解绑，否则返回 `InvalidIPAddress.InUse`。VPC 的示例是：

```bash
aws ec2 release-address \
    --allocation-id eipalloc-64d5890a
```

把示例里的分配 ID 换成你要释放的那一个。`--public-ip` 在该页标为已弃用，对应的是 EC2-Classic 示例。

IPAM 免费套餐包含 Public IP Insights，可以看到公共 IPv4 用量。该页写免费套餐本身不收费，你使用的其他 AWS 服务仍照常计费，公共 IPv4 的小时费不因为开了 IPAM 而取消。

### Elastic Load Balancing

[Elastic Load Balancing 定价](https://aws.amazon.com/elasticloadbalancing/pricing/) 写了四种：Application Load Balancer、Network Load Balancer、Gateway Load Balancer、Classic Load Balancer。前三种都是运行的每个小时或不足一小时按一小时计，再按分钟计各自的容量单位（LCU、NLCU、GLCU）。开了容量预留时，预留的容量按分钟计，超出部分另计。Classic Load Balancer 是运行小时加流过的数据。Gateway Load Balancer Endpoint 该页写单独计价，那一页这次没有打开。

定价页的演算使用 US East (N. Virginia) 的数字，例如 ALB 小时费 0.0225 美元、每 LCU 0.008 美元。这是该页示例，不是所有区域的统一价。新客户免费套餐该页写：750 小时由 Classic 和 Application Load Balancer 共用，另有 15 个 ALB LCU。这是注册时的套餐说明，不表示现有账号一定仍在套餐内。

[删除 Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-delete.html)：负载均衡一旦可用，就按你保留它运行的每个小时或不足一小时计费；删除之后，这项费用停止。删除不影响已注册目标，目标组要另删。删除保护开着时不能删。控制台是 EC2 控制台的 Load Balancers，Actions，Delete load balancer，确认时输入 `confirm`。

```bash
aws elbv2 delete-load-balancer \
    --load-balancer-arn load-balancer-arn
```

这一页没有写关联的公共 IPv4 是否同时释放。NLB 和 Gateway Load Balancer 的删除页这次没有打开，不把 ALB 的点击顺序套过去。

### NAT Gateway

[VPC 定价](https://aws.amazon.com/vpc/pricing/)：NAT Gateway 按它处于已预置且可用的每个小时计费，不足一小时按一小时。流过的每 GB 另收数据处理费，不区分流量从哪来、到哪去。经 NAT 传出的数据再按普通数据传输计。要停止这项费用，用控制台、CLI 或 API 删除网关。

区域级 NAT Gateway 按它在每个可用区被配置的小时计。三个可用区同时存在一小时，就是三个 NAT Gateway 小时。某个可用区不再承担后，该可用区的小时费停止。

定价示例写的是 US East (Ohio)：该区域每小时 0.045 美元，每 GB 数据处理 0.045 美元。示例说明网关一旦可用，小时费就一直在。这是该区域示例。

[使用 NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html)：删除后，控制台里的记录大约再留一小时。删除会解绑 Elastic IP，不从账号释放。原来指向它的路由会变成 `blackhole`，直到你删掉或改掉路由。控制台路径是 VPC 控制台 <https://console.aws.amazon.com/vpc/>，NAT gateways，选中后 Actions，Delete NAT gateway，确认时输入 `delete`。不再需要那个 Elastic IP 时，再按上一节释放。

## Microsoft Azure

全球文档。中文定价页和英文定价页是同一产品的两种语言，由世纪互联运营的 Azure 没有打开。

### 公共 IP

[公共 IP 地址](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses) 写：2025-09-30，Basic SKU 公共 IP 已退役，仍在使用的应升级到 Standard。Standard 的分配方式是静态。Basic 的 IPv4 可以是动态或静态。NAT 网关这一行只支持静态 IPv4，不支持动态。公共 IPv4 有费用，公共 IPv6 不计费。

[IP 地址定价](https://azure.microsoft.com/en-us/pricing/details/ip-addresses/) 和[中文定价页](https://azure.microsoft.com/zh-cn/pricing/details/ip-addresses/) 这次都能打开，表格结构仍列出 Basic（经典）、Basic (ARM)、Standard、Standardv2 和全局。单价单元格显示为“$-/小时”，没有读到具体数字，这里不补单价。

两页 FAQ 的计费条件一致：

- ARM 模型里，关联的虚拟机处于已停止（解除分配）时，动态公共 IP 不计费。静态公共 IP 不论关联的是哪个资源，都计费。
- 经典模型里，虚拟机已停止（解除分配）时，实例级公共 IP（ILPIP）不计费。
- ARM 静态公共 IP 和经典保留 IP：计费从创建后的第二个小时开始，删除该 IP 资源后停止。
- 其他公共 IP：从关联资源启动时开始，关联资源被删除或已停止（解除分配）时停止。
- 不足 1 小时按 1 小时。

产品页写 Basic 已退役，定价页仍保留 Basic 列。以产品页的退役日期判断还能不能新建，以定价页看存量如果仍出现在账单里时的计费条件。两页不改写成“Basic 已经从价目上消失”。

公共 IP 是独立资源。这次没有打开“删除虚拟机是否默认删掉网卡和公共 IP”的页面，不把下面 NAT 的那句套到虚拟机上。门户里查找公共 IP 的入口，在 NAT 管理页的步骤里是：顶部搜索 Public IP address，打开 Public IP addresses。

### 负载均衡

[负载均衡 SKU](https://learn.microsoft.com/en-us/azure/load-balancer/skus)：2025-09-30，Basic Load Balancer 已退役。现存的三种是 Basic（已退役）、Standard、Gateway。Standard 公共负载均衡的前端公共 IP 必须是 Standard。Gateway SKU 面向第三方网络虚拟设备，概述页这次没有按计费打开。

Load Balancer 定价页和 Application Gateway 的定价、删除页这次都超时或没有打开。Standard 负载均衡在没有规则、没有流量时收不收固定小时费，这里不填。公共 IP 如果单独存在，仍按上一节的公共 IP 规则看。

### NAT 网关

[什么是 Azure NAT 网关](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview)：Standard 是单区域，StandardV2 是区域冗余，并要求 StandardV2 公共 IP 或前缀。不能从 Standard 升级成 StandardV2，只能新建一台换上。NAT 优先于负载均衡出站规则、实例级公共 IP 和 Azure Firewall，但用户自定义路由把 `0.0.0.0/0` 指向虚拟设备或虚拟网络网关时，那条路由优先于 NAT。不能和 Basic 负载均衡、Basic 公共 IP 放在同一子网上使用。

[FAQ](https://learn.microsoft.com/en-us/azure/nat-gateway/faq)：Standard 和 StandardV2 价格相同，具体见定价页。定价页这次抓取超时，小时单价和每 GB 单价不写。删除前要先从全部子网解除关联。

[管理 NAT 网关](https://learn.microsoft.com/en-us/azure/nat-gateway/manage-nat-gateway) 写明：删除 NAT 网关时，关联的公共 IP 或前缀不会被删除。该页的 CLI 示例先从子网去掉 NAT，再删网关。示例里的资源组名和名字是文档示例：

```bash
az network vnet subnet update \
    --resource-group test-rg \
    --vnet-name vnet-1 \
    --name subnet-1 \
    --remove natGateway

az network nat gateway delete \
    --name nat-gateway \
    --resource-group test-rg
```

公共 IP 要另删。这次没有打开 `az network public-ip delete` 的参考页，不写那条命令的参数。

## Google Cloud

### 外部 IP

[VPC 定价](https://cloud.google.com/vpc/pricing) 写：静态和临时外部 IP 都收费。预留了静态外部 IP、又没有分配给虚拟机或转发规则时，单价高于正在使用的静态和临时外部 IP。分配给转发规则的静态外部 IP 不计 IP 费。外部 IPv6 地址范围分配给子网、外部 IPv6 分配给虚拟机，以及静态区域级 IPv6（预览），该页写不计费。静态或临时内部 IP 不计费。非美元结算时，以 Cloud Platform SKU 上你的币种价格为准。

该页表格这次读到的单元格是：未使用的静态 IP 为 `$0.01 / 1 hour`，标准虚拟机上正在使用的静态和临时 IP 为 `$0.005 / 1 hour`，抢占式和 Spot 虚拟机上为 `$0.0025 / 1 hour`，Cloud NAT 使用的为 `$0.005 / 1 hour`，转发规则或 Cloud VPN 隧道公网 IP 为不计费。未使用和标准虚拟机两格的末尾还连着 “per 1 month / account”。不把这两格改写成另一套月价。

在用还是未用：静态外部 IP 只要还关联在虚拟机上，虚拟机停着也算在用。实例删除或地址与实例解除关联后，变为未使用。临时外部 IP 只在关联的虚拟机运行时算在用；实例停止或删除时，临时地址被释放。

查看状态：

```bash
gcloud compute addresses list
```

该页示例里，`IN_USE` 和 `RESERVED` 都会按外部 IP 价目计费，只是单价不同。

[配置静态外部 IP](https://docs.cloud.google.com/compute/docs/ip-addresses/configure-static-external-ip-address)：删除实例不会自动释放静态外部 IP。控制台释放时，地址必须没有被其他资源使用。路径是 IP addresses，External IP addresses，可用 `static` 过滤，然后 Release static address。gcloud 或 API 即使地址仍被使用也可以发起释放：未被使用的回到地址池；仍被使用的会一直挂在资源上，直到该资源被删除。

```bash
gcloud compute addresses delete ADDRESS_NAME
```

同一页的 API 把区域地址和全球地址分成 `addresses.delete` 与 `globalAddresses.delete`。释放后不保证还能再预留到同一个地址。

### 负载均衡和转发规则

[Cloud Load Balancing 定价](https://cloud.google.com/load-balancing/pricing) 覆盖：内部和外部直通网络负载均衡、内部和外部代理网络负载均衡，以及全球、经典、区域外部应用负载均衡。内部应用负载均衡有单独一节，这次没有按同样深度摘价。Private Service Connect 的转发规则另计，不放进这一节。

转发规则按小时计。该页和 VPC 定价页的例子都是：最多 5 条转发规则一共 0.025 美元/小时；只有 1 条或 3 条，也是 0.025 美元/小时；超出的每条再加 0.01 美元/小时。全球转发规则和区域转发规则分开计，也按项目分开计。转发规则只要存在就按这个小时费计，无论它是给负载均衡用还是给其他用途（例如 Packet Mirroring）。数据处理另按处理的 GiB 计。负载均衡定价页里，爱荷华（us-central1）所在分组的入站和出站处理费单元格都是每 GiB 0.008 美元。其他地域不在这里抄。

因此，没有业务流量时，处理费会跟着 GiB 下降；转发规则的小时费不跟着下降。绑在转发规则上的静态外部 IP 不再另收 IP 费，转发规则自己的小时费仍在。

### Cloud NAT

[Cloud NAT 定价](https://cloud.google.com/nat/pricing)：Public NAT 的费用是网关小时费、网关处理的每 GiB（入方向和出方向）、网关使用的每个静态或临时外部 IP 的小时费，再加上离开网络的数据传输。网关小时费按拿到 NAT 分配的虚拟机数量计，每小时 0.0014 美元乘以这个数量，32 台及以上封顶为每小时 0.044 美元。处理费该页写成每 GiB 0.045 美元。每个外部 IP 每小时 0.005 美元。该页没有写“一台虚拟机都没有时网关小时费是否为零”，这里不补。

Private NAT 该页写成网关每小时 0.045 美元，另加每 GiB 0.045 美元，没有外部 IP 这项。非美元以 SKU 上的币种价格为准。

查看 NAT 正在使用的 IP：

```bash
gcloud compute routers get-nat-ip-info
```

删除 Cloud NAT 的命令页这次没有打开，不写未核对的删除命令。不再使用的外部 IP，按上一节释放静态地址。

## 清理清单

按这个顺序做。每一步先确认没有业务还在用，再释放。

1. 列出还在的地址、负载均衡和 NAT，并对照 DNS、防火墙白名单、SNAT/DNAT 和监控。阿里云用 EIP 控制台按地域看未绑定地址。腾讯云用公网 IP 控制台。AWS 用账号里尚未释放的 Elastic IP，以及 IPAM 免费套餐里的 Public IP Insights。Azure 在门户搜索 Public IP addresses。Google Cloud 用 `gcloud compute addresses list`，看 `RESERVED`。
2. 确认当前身份有权释放。权限范围见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。阿里云释放按量 EIP 的接口是 ReleaseEipAddress，删除公网 NAT 的接口是 DeleteNatGateway。
3. 先处理明确无依赖的公网 IP。阿里云按量 EIP 要处于可释放状态，并先解绑、移出共享带宽、去掉 NAT 条目。腾讯云先解绑再释放；NAT 上唯一的 EIP 解绑页写明不能解绑。AWS 非默认 VPC 要先解绑再 `release-address`。Azure 静态公共 IP 要删除 IP 资源本身，虚拟机已停止（解除分配）不会停掉静态 IP 的费用。Google Cloud 控制台只能释放未被使用的静态地址。
4. 再处理没有监听、没有后端、确认不再转发的负载均衡。阿里云 ALB 未释放就继续收实例费；CLB 要删除而不只是停止；NLB 自动分配的 EIP 会随释放一起走，已有 EIP 要再看 EIP 控制台。腾讯云 CLB 的实例费到你发起销毁为止，进回收站的 CVM 不会自动解除和 CLB 的关系。AWS 删除 ALB 后，这台负载均衡的小时费停止，目标实例还在。Azure Standard 负载均衡的小时费这次没有从定价页读到，不在这一步假设它已经是 0。Google Cloud 删掉不再需要的转发规则，小时费才停止。
5. 最后删确认不再出网的 NAT，然后回去释放被解绑、但还在账号里的公网 IP。阿里云、腾讯云、AWS、Azure 的已打开页面都写了：删除 NAT 不会把公网 IP 的保有费一并停掉。Google Cloud 的 NAT 删除命令这次没有打开；网关用过的外部 IP 仍按地址状态计费。
6. 释放或删除后的下一小时、下一天再看一次账单。阿里云、腾讯云、AWS 的已打开页面都写了延迟出账或按上一周期结算。账单里还有这一项，先对消费时间，再判断资源是不是还在。

## 常见坑

- 把“没有流量”当成“这项资源已经免费”。按小时的实例费、按带宽峰值的费用、未绑定的公网 IP 保有费，都不靠流量下降来停止。
- 删掉 ECS、CVM 或 EC2 之后只查实例列表。阿里云 EIP、腾讯云 EIP、AWS Elastic IP 都会留下。腾讯云普通公网 IP 会随销毁释放，和 EIP 不是同一项。
- 阿里云配额提到 2000 以上，或腾讯云全地域 EIP 配额大于 2000 之后，原来“绑到实例就免保有费”的条件不再适用。本地运营商线路的阿里云 EIP 则不论是否绑定都收配置费，并且 30 天内不能释放。
- 阿里云中国站 EIP 暂停满 7 天可能被自动释放；国际站英文页写的是欠费后再保留 15 天、暂停后再满 15 天才自动释放。不要用一边的天数操作另一边的账号。
- AWS 当前 VPC 定价页对在用和闲置公共 IPv4 使用同一个小时价。停掉实例只能停掉自动分配的那个公网 IPv4，停不掉仍关联着的 Elastic IP。
- Azure 产品页写 Basic 公共 IP 和 Basic 负载均衡已在 2025-09-30 退役，定价页仍列出 Basic 列，而且这次打开的单价是占位符。不把占位符当成 0，也不把 Basic 列当成还能新建。
- Google Cloud 上，虚拟机停止后临时外部 IP 会释放，静态外部 IP 仍按在用价格计。用 gcloud 对仍被占用的地址执行删除，地址会留在资源上直到资源被删，控制台则要求地址未被使用才能释放。
- 释放之后 DNS 还指着旧地址。阿里云 EIP 页写明，释放不会自动改 DNS。AWS 删除负载均衡的页面要求先改 DNS 并等 TTL。地址回到地址池之后，可能被别人拿到。

## 参考来源

资料核对日期：2026-11-16。

阿里云中国站：

- [弹性公网 IP](https://help.aliyun.com/zh/eip/elastic-ip-address)
- [按量付费](https://help.aliyun.com/zh/eip/pay-as-you-go/)
- [ReleaseEipAddress](https://help.aliyun.com/zh/eip/developer-reference/api-vpc-2016-04-28-releaseeipaddress-eips)
- [IP 地址](https://help.aliyun.com/zh/ecs/user-guide/ip-address/)
- [DeleteInstance](https://help.aliyun.com/zh/ecs/developer-reference/api-ecs-2014-05-26-deleteinstance)
- [释放实例时保留弹性网卡](https://help.aliyun.com/zh/ecs/user-guide/eni-retention-and-reuse)
- [ALB 计费规则](https://help.aliyun.com/zh/slb/application-load-balancer/alb-billing-rules)
- [CLB 按量付费](https://help.aliyun.com/zh/slb/classic-load-balancer/product-overview/pay-as-you-go)
- [NLB 计费规则](https://help.aliyun.com/zh/slb/network-load-balancer/product-overview/nlb-billable-items)
- [创建和管理 NLB 实例](https://help.aliyun.com/zh/slb/network-load-balancer/user-guide/create-and-manage-an-nlb-instance)
- [NAT 网关计费](https://help.aliyun.com/zh/nat-gateway/nat-gateway-billing)
- [使用公网 NAT 网关实现公网访问](https://help.aliyun.com/zh/nat-gateway/user-guide/use-internet-nat-gateway-for-public-network-access)
- [DeleteNatGateway](https://help.aliyun.com/zh/nat-gateway/developer-reference/api-vpc-2016-04-28-deletenatgateway-natgws)

阿里云国际站这次只打开了 [Pay-as-you-go](https://www.alibabacloud.com/help/en/eip/product-overview/pay-as-you-go)（Last Updated: Oct 26, 2026）。国际站 ALB、CLB、NLB、NAT 页面没有打开。中国站和国际站对欠费后多少天自动释放的写法不同，正文已分开。

腾讯云中国站：

- [计费概述](https://cloud.tencent.com/document/product/1199/41692)（最近更新 2024-10-29）
- [IP 资源费用](https://cloud.tencent.com/document/product/1199/51694)（最近更新 2026-08-21）
- [释放按量计费 EIP](https://cloud.tencent.com/document/product/1199/41704)（最近更新 2023-10-31）
- [解绑 EIP](https://cloud.tencent.com/document/product/1199/41703)（最近更新 2025-06-11）
- [公网服务](https://cloud.tencent.com/document/product/213/5224)
- [回收或恢复实例](https://cloud.tencent.com/document/product/213/4931)
- [实例费用](https://cloud.tencent.com/document/product/214/8848)（最近更新 2026-09-02）
- [公网 NAT 网关计费概述](https://cloud.tencent.com/document/product/552/18172)（最近更新 2025-10-17）
- [公网 NAT 使用限制](https://cloud.tencent.com/document/product/552/12955)（最近更新 2025-11-05）

腾讯云国际站这次没有打开。CLB 的 LCU 页、网络费用页和删除实例页没有打开，所以没有写 LCU 系数、带宽单价和删除点击顺序。

AWS：

- [Amazon VPC 定价](https://aws.amazon.com/vpc/pricing/)
- [Elastic IP 地址](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- [实例 IP 地址](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html)
- [实例停止和启动](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-ec2-instance-stop-start-works.html)
- [实例状态变化](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)
- [release-address](https://docs.aws.amazon.com/cli/latest/reference/ec2/release-address.html)
- [Elastic Load Balancing 定价](https://aws.amazon.com/elasticloadbalancing/pricing/)
- [删除 Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-delete.html)
- [使用 NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html)

没有打开 docs.amazonaws.cn、NLB 与 Gateway Load Balancer 的删除页、Gateway Load Balancer Endpoint 的单独定价页。VPC 定价页写明公共 IPv4 小时价也适用于 AWS 中国区域，操作步骤仍只来自全球文档。

Microsoft Azure：

- [公共 IP 地址](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses)
- [IP 地址定价（英文）](https://azure.microsoft.com/en-us/pricing/details/ip-addresses/)
- [IP 地址定价（中文）](https://azure.microsoft.com/zh-cn/pricing/details/ip-addresses/)
- [负载均衡 SKU](https://learn.microsoft.com/en-us/azure/load-balancer/skus)
- [什么是 Azure NAT 网关](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview)
- [Azure NAT 网关 FAQ](https://learn.microsoft.com/en-us/azure/nat-gateway/faq)
- [管理 NAT 网关](https://learn.microsoft.com/en-us/azure/nat-gateway/manage-nat-gateway)

Load Balancer 定价页、NAT Gateway 定价页抓取超时，Application Gateway 定价和删除页没有打开，`az network public-ip delete` 的参考页没有打开。由世纪互联运营的 Azure 没有打开。定价页的单价单元格是占位符。

Google Cloud：

- [VPC 定价](https://cloud.google.com/vpc/pricing)
- [配置静态外部 IP](https://docs.cloud.google.com/compute/docs/ip-addresses/configure-static-external-ip-address)
- [Cloud Load Balancing 定价](https://cloud.google.com/load-balancing/pricing)
- [Cloud NAT 定价](https://cloud.google.com/nat/pricing)

内部应用负载均衡的单独定价节、Cloud NAT 删除命令页没有按同样深度打开。

如需为阿里云、腾讯云、Google Cloud、AWS、Microsoft Azure 的国际站账号开户或充值，可访问海宝云（CloudCup）官网 <https://www.haibaocloud.com>。官网常见问题当前写明：充值仅支持 USDT-TRC20，单笔最低 50 USDT；其他币种或法币需联系客服定制。具体以官网当前说明为准。Telegram 客服：@cloudcup。
