# 云服务器账单为什么超预算：计算、磁盘、公网流量与快照费用清单

云服务器账单超出预期，通常是因为一张账单里叠了几类独立费用：计算（实例）、磁盘、公网、快照，有时还有镜像许可证和公网 IP。停机、卸载磁盘、删除云盘，都不会自动停掉其余计费项。

价格随地域、规格、冗余方式和时间变化。本文只说明计费机制和排查顺序。文中出现的免费额度、最短计费时长和官方示例，都来自撰写时打开的官方页面，并附有链接。这些规则可能调整，购买和对账时请以当前官方定价页、价格计算器和账单明细为准，不要把文档示例里的单价当成现行统一标价。

核对时间：2026-11-08。阿里云、腾讯云同时存在中国站和国际站，结算币种、欠费流程和部分控制台文案可能不同。下文阿里云、腾讯云的主体规则来自所列中文文档；国际站账号请再打开该账号所属站点的定价页核对。

## 先把账单拆开

按这个顺序看费用与成本或 Billing 控制台，比先看实例单价更有用：

1. 按产品、地域、计费项过滤，确认超支来自实例、磁盘、公网还是快照。
2. 找出状态不是“运行中”，但仍在计费的计算资源。各厂商对“关机”的定义不同，见下一节。
3. 列出未挂载、已分离或随实例保留下来的磁盘。容量费多数按预置大小计算，不按文件系统里已用空间计算。
4. 区分公网是按带宽保底，还是按出方向流量后付费。入方向通常不计公网流量费，出方向和跨地域流量才是常见超支点。
5. 按地域加总快照容量，并检查自动快照策略、自定义镜像和跨地域复制。删除磁盘不一定删除快照。

镜像许可证、空闲公网 IP、负载均衡、NAT 网关不在这四类里面，但会单独出账。固定公网 IP 的规则不能直接套到弹性公网 IP 或 NAT。

## 计算：实例停了，不一定停止收计算费

计算费按实例规格计，通常包含 vCPU 和内存；带本地盘或 GPU 的规格还会把这些资源算进规格费。实例空闲但状态仍是运行中，计算费继续走。

### 阿里云

按量付费实例从创建开始到释放为止，按秒计量。帮助中心写明：即使没有跑业务，实例规格也会持续计费，除非开启节省停机模式。包年包月则按购买时长预付。本地盘不能单独购买，费用已含在实例规格里。见 [计费概述](https://help.aliyun.com/zh/ecs/billing-overview) 和 [按量付费](https://help.aliyun.com/zh/ecs/pay-as-you-go-1)。

主动关机要看是否启用节省停机：

| 计费项 | 运行中 | 普通停机 | 节省停机 |
| --- | --- | --- | --- |
| 实例规格（vCPU、内存、GPU） | 计费 | 计费 | 不计费 |
| 云盘容量 | 计费 | 计费 | 仍计费 |
| 固定公网带宽 | 计费 | 计费 | 不计费 |
| 按使用流量的公网带宽 | 按实际出网流量 | 不产生流量费 | 不产生流量费 |
| 镜像许可证 | 计费 | 计费 | 不计费 |

欠费停机和主动关机不是同一件事。按量付费文档写明，欠费停机期间实例不计费；未开节省停机时，自己在控制台或 `StopInstance` 关机仍按原规则计费。节省停机可能带来启动失败、公网 IP 变化、突发性能实例 CPU 积分清零，以 [按量付费](https://help.aliyun.com/zh/ecs/pay-as-you-go-1) 中的风险说明为准。

释放当小时如果实际使用短于最短计费时长，该计费周期按最短时长计：1 vCPU 不足 10 分钟按 10 分钟，2 vCPU 不足 5 分钟按 5 分钟，4 vCPU 及以上不足 2 分钟按 2 分钟。这是计费规则，不是折扣。

国际站同一篇按量付费说明还写了后付费结算：默认支付方式为银行卡时，扣款阈值为 1000 USD；结算日扣款失败后还会在 T+7、T+14 再试，三次都失败则自 T+15 起欠费停机并停止计费。中国站文档的出账描述与此不同。以账号所在站点为准：[国际站按量付费](https://www.alibabacloud.com/help/zh/ecs/user-guide/pay-as-you-go)、[中国站按量付费](https://help.aliyun.com/zh/ecs/pay-as-you-go-1)。

当前规格单价见 [ECS 定价](https://www.aliyun.com/price/detail/ecs)。国际站账号不要直接用中国站价目表下单。

### 腾讯云

计费概述列出包年包月、按量计费、竞价实例（波动型）、竞价实例（计划型）和包销计费。见 [计费概述](https://cloud.tencent.com/document/product/213/2179)。

按量实例只有选择“关机不收费”后，才停止收取 CPU、内存费用；异构计算实例此时也停止收取 GPU 或 FPGA 费用。系统盘、数据盘、按带宽计费的公网带宽和镜像继续计费。该功能不适用于在实例内部开关机、搭载本地盘的实例，以及欠费关机。批量关机时，不支持该功能的实例关机后继续收费。见 [关机不收费说明](https://cloud.tencent.com/document/product/213/19918)。

API `StopInstances` 的 `StoppedMode` 默认是 `KEEP_CHARGING`（关机后继续收费），要显式传 `STOP_CHARGING` 才是关机不收费。见 [关机不收费操作](https://cloud.tencent.com/document/product/213/19922)。

关机不收费会释放 CPU、内存和普通公网 IP，再次启动可能失败或换公网 IP。绑定的 EIP 会保留，该说明写明此期间不计 EIP 空置费用。做变配、重装等运维关机时，不执行关机不收费策略。

### AWS

按需实例按启动到终止或停止的时间计费。Linux、Windows，以及带 SQL Enterprise、SQL Standard、SQL Web 的 Windows，不足一小时按秒计，最少 60 秒；文档写明其余操作系统类型按小时计。见 [EC2 按需定价](https://aws.amazon.com/cn/ec2/pricing/on-demand/)。

[实例状态](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html) 把计算费和磁盘费分开：

- `running` 对实例使用计费。实例闲置且没人登录，只要仍是 `running`，也按秒计费，每次进入 `running` 至少 1 分钟。
- `stopped` 不再收取实例使用费和数据传输费，但 Amazon EBS 卷的存储继续计费。从 `stopped` 再次启动，重新按秒计费，每次启动至少 1 分钟。
- 休眠时，`stopping` 阶段仍收使用费，进入 `stopped` 后不收实例使用费；EBS 存储（含写入根卷的内存数据）继续计费。
- 进入 `shutting-down` 或 `terminated` 后，不再产生该实例的费用。预留实例应用到已终止实例时，仍按其付款选项计到期限结束。
- 重启不新开计费周期，也不再另收最低 1 分钟。

终止实例时，根卷默认删除，其余 EBS 卷默认保留。保留下来的卷继续按预置容量计费。弹性 IP 无论使用中还是空闲，按需定价页都写明需要付费，单价见该页指向的 VPC 定价。

同区域跨可用区传输不是“内网免费”。[全球网络常见问题](https://aws.amazon.com/about-aws/global-infrastructure/global-network/faqs/) 写明，同一区域里 EC2 等资源跨可用区的传入和传出都可能计费；跨区域传输另计。每月 100 GB 免费额度只针对从区域到互联网的出站，见下一节。

估算用 [AWS 定价计算器](https://calculator.aws/)。

### Microsoft Azure

虚拟机价格不包含托管磁盘。定价页常见问题写明：临时磁盘（资源磁盘）不计存储费，VM 关闭后上面的数据会丢失；操作系统磁盘按磁盘的正常费率另计。见 [Linux 虚拟机定价](https://azure.microsoft.com/zh-cn/pricing/details/virtual-machines/)。

同一页对“已停止”的说明是：

- 状态为“已停止”或“已停止(已分配)”时，仍对已分配的虚拟核心计费，软件许可证本身按该说明不计。
- 只有“已停止(已取消分配)”或“已删除（已取消分配）”不再对核心计费。
- 在虚拟机操作系统里关机，可能只进入仍会计费的“已停止”。要从管理门户停止，或使用会取消分配的关机操作，才能停止计算费。

该页还写明按整分钟计费：运行 6 分 45 秒按 6 分钟计，不足 1 分钟的秒数不另计。页面同时保留了部分较早的状态用词，对账时以门户里的实际电源状态和当前文档为准。

现成 VM（Spot）在容量不足或出价低于现价时会被收回，计算单价规则与按需 VM 不同，以同一定价页为准。估算用 [Azure 定价计算器](https://azure.microsoft.com/zh-cn/pricing/calculator/)。

### Google Cloud

Compute Engine 对 vCPU、内存和 GPU 至少计 1 分钟，之后按秒。正常运行时间是启动到进入“正在停止”之间的秒数。实例空闲但状态仍是“正在运行”，按正常运行时间收费。预留从创建到删除按标准费率计费，不论有没有开机。机器类型价格不含磁盘、映像、网络和 GPU。见 [Compute Engine 价格](https://cloud.google.com/products/compute/pricing?hl=zh-cn)。

[停止或重启实例](https://docs.cloud.google.com/compute/docs/instances/stop-start-instance?hl=zh-cn) 写明：停止后实例保留挂接的资源、配置和内部 IP，但内存中的数据会丢失；已停止实例上挂接的资源继续计费，不用的资源要分离并删除。API 说明进一步写明：停止期间不收 VM 使用费，永久性磁盘和静态 IP 等资源在删除前继续计费。见 [`instances.stop`](https://docs.cloud.google.com/compute/docs/reference/rest/v1/instances/stop)。

付费映像（如部分 Windows、SQL Server、RHEL、SLES）另计，且至少 1 分钟。SQL Server 许可在实例 vCPU 少于 4 个时，按 4 个 vCPU 计，见 [磁盘和映像价格](https://cloud.google.com/compute/disks-image-pricing?hl=zh-cn)。

估算用 [Google Cloud 价格计算器](https://cloud.google.com/products/calculator?hl=zh-cn)。

## 磁盘：创建或预置之后，卸载通常停不了容量费

磁盘费多数按配置容量和使用时长计算，不按文件系统显示的已用空间计算。未挂载的盘、已停止实例上的系统盘，都要单独看。

### 阿里云

[块存储计费](https://help.aliyun.com/zh/ecs/block-storage-devices) 写明：云盘创建成功后开始计费，与是否挂载、用作系统盘还是数据盘无关。

- 随实例创建的云盘，计费方式与实例相同。
- 单独创建且不挂载的云盘，只支持按量付费，创建后就按容量计费。
- 包年包月实例可以挂包年包月或按量数据盘；按量实例只能挂按量云盘。
- 扩容后对新容量计费。ESSD AutoPL、ESSD PL-X 如果配置了预配置 IOPS，另收预配置性能费。
- ESSD AutoPL 默认可能开启性能突发。官方规则是：每小时突发 IO 总量不超过 10 万时，这 10 万 IO 不收突发费；超过后另有封顶规则。10 万是免费额度，不是磁盘单价。

本地盘费用含在实例规格中。弹性临时盘创建成功后按容量计费。

帮助中心有华东 1（杭州）ESSD 的演算示例，并注明示例只说明算法，实际以定价页和账单为准。不要把示例里的元/GiB 抄成当前目录价。容量单价见块存储价格，入口在上述文档和 [ECS 定价](https://www.aliyun.com/price/detail/ecs)。

节省停机不停云盘费，见上一节的对照表。

### 腾讯云

云硬盘按容量计费，支持包年包月和按量计费。按量计费购买时冻结 1 小时费用，之后按小时结算，时间粒度到秒。不同地域、不同类型单价不同。见 [云硬盘计费概述](https://cloud.tencent.com/document/product/362/32361) 和 [价格总览](https://cloud.tencent.com/document/product/213/112630)。

创建云硬盘时可以选择“暂不挂载”，盘会处于待挂载。计费模式在创建时确定，卸载不会代替销毁。[销毁云硬盘](https://cloud.tencent.com/document/product/362/5752) 写明：销毁后不再对该盘收费；按量云硬盘手动销毁立即生效；销毁前盘必须是待挂载。包年包月销毁后可能在回收站保留 7 天，此期间的退费和恢复以该页规则为准。

关机不收费期间，系统盘和数据盘继续计费。非弹性云硬盘生命周期跟随实例，不能单独销毁。

价格总览里的存储空间、额外性能和数据备份点是分开的计费项。页面上的单价按地域列出，会更新，对账时打开该页，不要引用旧表。

### AWS

[EBS 定价](https://aws.amazon.com/cn/ebs/pricing/) 写明：卷存储按每月预置的 GB 计费，直到释放存储。gp3 含 3,000 IOPS 和 125 MB/s 基准，超出部分另计；io1、io2 的预置 IOPS 另计。预置存储按秒累计，最少 60 秒。定价页的演算使用“在每月每 GB 收取某金额的区域中”这种假设费率，只说明公式，不是所有区域的现行标价。

因此，把卷从实例上分离但未删除，预置容量仍在。实例 `stopped` 后，卷存储继续计费，见 [实例状态](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)。

同一页还描述了 EBS 免费套餐：30 GB 存储、200 万 I/O 和 1 GB 快照存储。并写明自 2025 年 7 月 15 日起，新客户可获得最高 200 美元免费套餐抵扣金，适用于包括部分 EBS 卷类型在内的合格服务，免费计划在账户创建后 6 个月内有效，抵扣金须在 12 个月内使用。适用条件和卷类型以该页及免费套餐条款为准，不要把旧账户的额度套到新账户，也不要反过来。

### Microsoft Azure

托管磁盘按类型、性能层、冗余（LRS 或 ZRS）和事务计费，和 VM 计算费分开。标准托管磁盘按预置容量收费，不按盘上实际数据量收费。见 [磁盘常见问题](https://learn.microsoft.com/zh-cn/azure/virtual-machines/faq-for-disks) 和 [了解磁盘计费](https://learn.microsoft.com/zh-cn/azure/virtual-machines/disks-understand-billing)。

空盘有一个例外：首次创建的空托管磁盘，在第一次使用前只存元数据。第一次挂到 VM 或上传 VHD 之后才分配存储并开始收磁盘费。从未使用、尚未分配存储的空盘，不产生磁盘存储费。盘一旦开始计费，从 VM 拆下并保留，费用不会跟着 VM 消失。[删除 VM](https://learn.microsoft.com/zh-cn/azure/virtual-machines/delete) 写明：删除选项设为 Detach 时，磁盘会留下来并继续产生适用费用。

其他和账单有关的差异：

- 高级 SSD v2 对全部磁盘提供 3,000 IOPS 和 125 MB/s 基准，超出后 IOPS 和吞吐量另计。
- 超级磁盘按预置容量、IOPS 和吞吐量计费。
- 标准 SSD、标准 HDD 除容量外还有事务费，并设有每小时可计费事务上限，超过上限的事务在该小时不再计费。具体上限以定价页为准。
- 高级 SSD 只有显式打开按需突发才产生突发启用费和突发事务费；基于积分的突发不按该说明另计。
- 共享高级 SSD 会随挂载的 VM 数量增加费用。共享超级磁盘和共享高级 SSD v2 不按每台 VM 另收一笔挂载费，但共享后可能出现需要另付费的性能上限。

快照按已用数据计费，见下一节。当前目录价见 [托管磁盘定价](https://azure.microsoft.com/zh-cn/pricing/details/managed-disks/)。该页声明价格是估算，实际受协议、购买日期和汇率影响。

### Google Cloud

[磁盘类型](https://cloud.google.com/compute/docs/disks?hl=zh-cn) 的费用说明写明：从创建卷到删除卷，按预置容量计费，适用于本地 SSD、Persistent Disk 和 Hyperdisk。以下情况继续计费：

- 创建实例时自动生成的启动卷
- 挂在已停止或已暂停实例上的持久卷
- 没有挂到任何虚拟机的持久卷

[磁盘和映像价格](https://cloud.google.com/compute/disks-image-pricing?hl=zh-cn) 写明：预置空间包含已用和未用空间，200 GB 的盘在删除前按 200 GB 计，与实际写入量无关。成功删除永久性磁盘后，不再为该盘付费。标准、SSD 和平衡永久性磁盘的 I/O 含在容量价里；Hyperdisk Balanced 等类型对超出基准的预置 IOPS 和吞吐量另计。磁盘费用按秒分摊。

该价格页的“标准预配空间”列出每个账号每月 0 至 30 GiB·月为免费，超出部分按所列费率计。这是标准永久性磁盘容量的免费档，不要套到 SSD、平衡盘或 Hyperdisk。页内美国区示例（例如 200 GB 标准盘整月费用）是示例，非美元账单要查 Cloud Platform SKU。

## 公网：通常收出方向，入方向免费不等于所有流量都免费

“入方向免费”只覆盖各文档写明的公网入站。出方向、跨地域、跨可用区、NAT、负载均衡和空闲公网 IP 可能另计。按固定带宽购买时，流量小也要为带宽上限付钱；按流量购买时，出方向放量会直接抬高账单。

### 阿里云

固定公网 IPv4 只对出方向计费，入方向不计费。出方向包括对外提供访问和响应用户下载。上传到 ECS 的公网入流量，在该说明中不计 ECS 公网带宽费。见 [公网带宽计费](https://help.aliyun.com/zh/ecs/public-bandwidth)。

两种模式：

- 按固定带宽：按配置的带宽上限和时长计费，适合流量稳定的业务。带宽是上限，不是承诺一定跑满。
- 按使用流量：只对实际出网流量后付费。带宽峰值只是上限，网络拥塞时不保证跑到峰值。包年包月实例若选按流量，购买时可以没有这笔预付，使用中仍会出后付费账单。

按量实例的固定带宽按秒计量，并适用与实例相同的最短计费时长（1 vCPU 10 分钟、2 vCPU 5 分钟、4 vCPU 及以上 2 分钟）。帮助中心的杭州地域演算标明“价格仅为示例”。把带宽改为 0 Mbit/s 可以释放固定公网 IP 并停止这项费用。

EIP 和 NAT 网关不走这套固定公网 IP 规则，要看各自的计费文档。开通云数据传输 CDT 后，账单产品名会变成云数据传输；该页只说明 CDT 有免费流量额度和阶梯价，没有给出具体免费 GB 数，这里不补数字。

### 腾讯云

[网络计费问题](https://cloud.tencent.com/document/product/213/17282) 写明：

- 同一私有网络内，云服务器之间以及云服务器与其他腾讯云产品之间的流量免费。
- 公网入网流量不收费，包括从实例访问公网下载，以及用户向实例上传。
- 公网出网流量收费，包括实例对外提供访问和用户从实例下载。
- 只产生入网流量的攻击不计费；攻击若打出公网出方向流量，出方向会计费。

[公网计费模式](https://cloud.tencent.com/document/product/213/10578) 有三种：按固定带宽、按流量、按共享带宽包。按流量是按小时结算公网出流量；按小时带宽是按小时结算公网出带宽。超过设定带宽上限的流量默认丢弃且不计算费用。文档建议带宽利用率高于 10% 时优先考虑按带宽，低于 10% 时优先考虑按流量。10% 是选型建议，不是计费阈值。

包年包月实例的带宽从包月带宽改为按流量，每台只允许转换 1 次。按流量的包年包月实例在账户欠费时，实例可以继续运行，但公网带宽服务会暂停，结清后自动恢复。

关机不收费会继续收取按带宽计费的公网带宽，并释放普通公网 IP。当前流量和带宽单价以控制台定价和 [计费概述](https://cloud.tencent.com/document/product/213/2179) 所链的价格页为准。国际站若使用单独的价格计算器，以登录后看到的价格为准。

### AWS

[AWS 定价说明](https://aws.amazon.com/pricing/) 写明：数据传入（data transfer IN）始终免费。出站按阶梯计价，用得越多，单位价格可以越低。

[EC2 按需定价](https://aws.amazon.com/cn/ec2/pricing/on-demand/) 和 [全球网络常见问题](https://aws.amazon.com/about-aws/global-infrastructure/global-network/faqs/) 写明：

- 所有 AWS 服务和区域合计，每月有 100 GB 出站到互联网的免费额度，中国区域和 GovCloud 除外。这 100 GB 是全球合计，不是每个区域各 100 GB，未用部分不结转到下月。
- 另外，CloudFront 每月有 1 TB 出站免费额度，规则以该常见问题为准。
- 符合条件的客户在把数据全部迁出 AWS 时，可以另行申请出站免费，需要走支持流程，不是默认账单规则。
- 同一区域跨可用区传输对 EC2 等资源按传入和传出计费。同区域里 EC2 与 S3、DynamoDB 等通过指定端点直接传输可以免费，路径上如果有 NAT 网关、PrivateLink 或 Transit Gateway，按那些服务的处理费计。
- 实例处于 `stopped` 时，不再收取该实例的数据传输费，但停止前已经产生的出站仍会计入账单。

不要把 100 GB 免费出站理解成跨可用区、跨区域或 NAT 处理费也免费。

### Microsoft Azure

[带宽定价](https://azure.microsoft.com/zh-cn/pricing/details/bandwidth/) 写明：

- 数据传入免费。
- 同一可用性区域内的数据传输免费。
- 同一区域内 Azure 服务之间的数据传输不另收传输费，例如同一区域的 Azure SQL。
- 两个区域之间按出站费率计费，入站免费。
- 通过 Microsoft 全球网络把数据送到互联网时，每月前 100 GB 出站免费，之后按源大洲分档。路由偏好选 ISP 网络时，免费额度同样是前 100 GB，后续单价不同。

该页声明价格是估算，不是报价。超过 100 GB 之后的美元单价会随大洲和路由变化，以页面当前表格为准，不要只记一个“每 GB 单价”。虚拟机定价页同时写明收取标准出口费用，并说明额外的实例级公共 IP、保留 IP 需要另付费；自动分配的动态 VIP 在该说明中免费。公共 IP 的现行价格以 IP 地址定价页为准，本文不另抄单价。

### Google Cloud

[VPC 网络价格](https://cloud.google.com/vpc/network-pricing?hl=zh-cn) 写明：入站数据传输不产生费用，对请求的响应计入出站并计费。高级层级是默认层级，标准层级要显式指定。

- 标准层级：入站免费，并列出每个账号每月前 200 GiB 免费，所有区域合并计算。同页写明“始终免费”用量限制不适用于标准层级，因此不要把别的免费计划再叠加进去。
- 高级层级：按目的地分档。部分目的地列出每月首 1 GiB 免费；前往澳大利亚、印度尼西亚、韩国、南美洲、沙特阿拉伯，以及中国（不含香港）的表格，从付费档开始，没有同样的首 1 GiB 免费。不能把北美目的地的免费额度用到所有出站。
- 同一可用区内，使用内部 IPv4 或任意 IPv6 的流量免费。使用外部 IP 跨 VPC 的流量按互联网出站费率计。
- 已分配但未使用的静态外部 IP，价格高于使用中的地址。免费档只写了每个账号每月 0 到 1 小时，且 `northamerica-northeast1` 和 `northamerica-northeast2` 不在该免费档内。

负载均衡器处理的数据另有费用，和 VPC 出站不是同一行账单。

## 快照：和磁盘分开计费，删盘不一定停

快照按自身保留的数据计费。自动策略会按计划继续产生新快照。跨地域复制通常在目标侧产生存储费，并可能另收复制流量。增量快照删掉其中一份，费用不一定等额下降，因为被其他快照引用的数据会留下来。

### 阿里云

开通快照服务不收费，创建快照后才开始计费。手动快照和自动快照策略默认都是标准快照，按地域、容量和时长计费。见 [快照计费](https://help.aliyun.com/zh/ecs/snapshots-1)。

- 标准快照按小时计费，不足 1 小时按 1 小时。可用 OSS 标准本地冗余存储包或存储容量单位包抵扣，抵扣顺序以该页为准。
- 归档快照最短保留 60 天（1,440 小时）。60 天内删除，除已发生的存储费外，还要支付不足规定时长的费用。归档快照不能复制。
- 把标准快照复制到其他地域，目标地域同时产生复制流量费和标准快照存储费。
- 释放云盘不改变快照的计费周期。只有云盘打开了“自动快照随云盘释放”，释放云盘时才会一并删除自动快照并停止这部分费用。手动快照不会因此消失。
- 计费从创建时间所在整点开始，到删除时间所在整点结束。文档中的例子是 11:02 创建、15:20 删除，计 11:00 到 16:00，共 5 小时。

该页的杭州地域单价演算标明“本示例价格仅为示例”。自动快照数量突然增加时，文档建议先看策略的保留周期和执行时间，再按地域核对快照列表：名称以 `auto2.0_` 开头的是自动快照。

### 腾讯云

快照按每个地域的总存储容量、按量后付费、以小时整点结算。自定义镜像底层占用快照容量，保留自定义镜像会产生快照费。见 [云硬盘计费概述](https://cloud.tencent.com/document/product/362/32361)。

中国地域提供 80GB 免费额度，超出部分按量计费。文档示例把华北（北京）和中国香港列为享有该额度，把新加坡列为不享有；并写明预计后续取消免费额度，覆盖范围以快照概览为准。计费概述的镜像小节把国内地域自定义镜像相关赠送额度写作 80GiB。对账时以控制台快照概览的当前额度为准，不要自行换算后当成多出来的一份额度。

[快照相关问题](https://cloud.tencent.com/document/product/362/17820) 写明：

- 销毁云硬盘不会删除关联快照，快照要单独删除，否则继续按容量计费。
- 定期快照策略会按设定自动创建快照，并在策略要求删除时删除。策略上限是单地域 30 个策略、每个策略最多绑定 200 块云硬盘。
- 自定义镜像的关联快照按实际容量收费，且必须先删镜像才能删关联快照。
- 快照支持跨地域复制。复制完成前不能删除源快照。

[价格总览](https://cloud.tencent.com/document/product/213/112630) 把快照费用拆成容量，以及可选的跨地域复制流量。复制费按源地域和目的地域区分，单价以该页当前表为准。

### AWS

快照是增量备份，费用取决于存储的数据量。删除一份快照不一定降低存储费：只有仅被这份快照引用的数据会删除，其他快照仍引用的数据会保留，费用改记到后续快照。见 [EBS 快照](https://docs.aws.amazon.com/zh_cn/ebs/latest/userguide/ebs-snapshots.html) 和 [删除快照](https://docs.aws.amazon.com/zh_cn/ebs/latest/userguide/ebs-deleting-snapshot.html)。

删除快照不影响原卷，删除卷也不影响已经生成的快照。注册中的 AMI 会占住根卷快照，要先注销 AMI 才能删。由 AWS Backup 管理的快照不能在 EC2 里直接删。符合回收站规则的快照删除后仍会保留一段时间。

[EBS 定价](https://aws.amazon.com/cn/ebs/pricing/) 写明：复制快照按跨区域传输的数据收费，复制完成后目标区域按标准快照存储计费。快照归档有 90 天最短保留期，提前删除或永久恢复要补足剩余时间的归档存储费；快速快照还原按启用期间的 DSU 小时计费，最少 1 小时。页内美元单价都写在具体示例区域或“收取某费率的区域”之下，不是全球统一价。

Amazon Data Lifecycle Manager 或 AWS Backup 会按策略继续创建快照。策略没有到期删除时，快照会一直占存储。

### Microsoft Azure

托管磁盘快照独立于源磁盘存在，按已用容量计费，不按磁盘预置大小计费。例如预置 64 GiB、实际使用 10 GiB 时，快照按 10 GiB 计。见 [创建快照](https://learn.microsoft.com/zh-cn/azure/virtual-machines/snapshot-copy-managed-disk)。

[了解磁盘计费](https://learn.microsoft.com/zh-cn/azure/virtual-machines/disks-understand-billing) 补充：完整快照按已用数据计费；增量快照只包含自上次快照以来的差异。500 GiB 磁盘若只用了 50 GiB，完整快照按 50 GiB 计。增量快照存在标准 HDD 存储上。删除链中的一份增量快照，不会使后续快照失效，后台会把数据合并到后一份快照。

跨区域复制只支持增量快照。托管复制只对跨区域带宽和源快照读取事务计费，复制完成前不能删除源快照，并且必须按创建顺序复制。见 [将快照复制到新区域](https://learn.microsoft.com/zh-cn/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions)。

删除 VM 时如果选择保留磁盘，磁盘费用还在；快照本来就是独立资源，不会随磁盘选项自动消失。快照单价见 [托管磁盘定价](https://azure.microsoft.com/zh-cn/pricing/details/managed-disks/) 的快照部分。

### Google Cloud

项目里只要还有标准快照，就按快照总大小产生存储费，和磁盘容量费分开。标准快照和归档快照是增量的，按变化数据压缩后的大小计，不按磁盘预置容量计。删除一份快照时，部分数据可能移到链上的下一份快照，存储费可能因此上升。见 [磁盘和映像价格](https://cloud.google.com/compute/disks-image-pricing?hl=zh-cn)。

- 标准快照存储最短结算 1 小时。
- 归档快照至少计 90 天存储费，用归档快照创建磁盘时另收检索费。
- 快照放在与磁盘不同的区域或多区域位置时，创建或恢复会产生网络费。同一区域创建标准快照，该页写明不产生这笔网络费。
- 多区域快照上传、下载有单独价格。不需要跨区域恢复时，文档建议指定区域级位置。

[快照时间表](https://docs.cloud.google.com/compute/docs/disks/about-snapshot-schedules) 写明：时间表创建的是标准快照。不设置保留策略时，自动快照会一直保留，存储费持续到手动删除。保留策略要在创建时间表时设置。

快照产品说明见 [磁盘快照](https://docs.cloud.google.com/compute/docs/disks/snapshots?hl=zh-cn)。价格页中的每 GiB 小时费率是列表价，非美元账户以 SKU 页面为准。

## 排查清单

对每个地域做一遍。数字只用来对照账单，单价回到上一节的官方页面查。

1. 计算。列出运行中、已停止但未释放、已停止仍分配的实例。阿里云确认是否节省停机；腾讯云确认关机模式是继续收费还是关机不收费，以及是不是在系统内关机；AWS 确认是 `stopped` 还是只是闲置的 `running`；Azure 确认是“已停止(已取消分配)”；Google Cloud 确认状态不是“正在运行”，并检查未删除的预留。
2. 磁盘。列出未挂载、待挂载、已分离和随实例保留的系统盘、数据盘。阿里云从创建起计费；Azure 先确认空盘是否从未挂载或上传，已分配存储的盘拆下来仍计费；其余三家按预置容量计到删除或销毁。核对预配置 IOPS、吞吐量、性能突发和备份点，这些会单独成行。
3. 公网。确认每台实例是按带宽还是按流量。按带宽时，看长期用不满的带宽上限。按流量时，看出方向字节数、跨地域复制、跨可用区和 NAT。阿里云可把不再使用的固定带宽设为 0。AWS 同时看空闲弹性 IP。Google Cloud 看未挂到资源上的静态外部 IP。腾讯云看出网流量，入网下载本身按文档不计公网流量费。
4. 快照。按地域汇总容量，标出自动策略、自定义镜像和跨地域副本。阿里云看是否打开“自动快照随云盘释放”，归档快照未满 60 天不要直接删。腾讯云销毁云盘后快照仍在，中国地域免费额度以快照概览为准。AWS 删除一卷快照后，用账单而不是快照个数判断费用是否下降。Azure 按已用大小看增量链。Google Cloud 检查快照时间表有没有保留策略。
5. 再估算。用 [阿里云 ECS 定价](https://www.aliyun.com/price/detail/ecs)、腾讯云 [价格总览](https://cloud.tencent.com/document/product/213/112630)、[AWS 定价计算器](https://calculator.aws/)、[Azure 定价计算器](https://azure.microsoft.com/zh-cn/pricing/calculator/) 和 [Google Cloud 价格计算器](https://cloud.google.com/products/calculator?hl=zh-cn) 按当前地域重算。国际站账号使用国际站计算器。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。

阿里云：

- [计费概述](https://help.aliyun.com/zh/ecs/billing-overview)
- [块存储计费](https://help.aliyun.com/zh/ecs/block-storage-devices)
- [公网带宽计费](https://help.aliyun.com/zh/ecs/public-bandwidth)
- [快照计费](https://help.aliyun.com/zh/ecs/snapshots-1)
- [按量付费](https://help.aliyun.com/zh/ecs/pay-as-you-go-1)
- [国际站按量付费](https://www.alibabacloud.com/help/zh/ecs/user-guide/pay-as-you-go)
- [ECS 定价](https://www.aliyun.com/price/detail/ecs)

腾讯云：

- [计费概述](https://cloud.tencent.com/document/product/213/2179)
- [公网计费模式](https://cloud.tencent.com/document/product/213/10578)
- [网络计费问题](https://cloud.tencent.com/document/product/213/17282)
- [关机不收费说明](https://cloud.tencent.com/document/product/213/19918)
- [关机不收费操作](https://cloud.tencent.com/document/product/213/19922)
- [云硬盘计费概述](https://cloud.tencent.com/document/product/362/32361)
- [价格总览](https://cloud.tencent.com/document/product/213/112630)
- [销毁云硬盘](https://cloud.tencent.com/document/product/362/5752)
- [快照相关问题](https://cloud.tencent.com/document/product/362/17820)

AWS：

- [EC2 按需定价](https://aws.amazon.com/cn/ec2/pricing/on-demand/)
- [EBS 定价](https://aws.amazon.com/cn/ebs/pricing/)
- [EBS 快照](https://docs.aws.amazon.com/zh_cn/ebs/latest/userguide/ebs-snapshots.html)
- [删除 EBS 快照](https://docs.aws.amazon.com/zh_cn/ebs/latest/userguide/ebs-deleting-snapshot.html)
- [实例状态更改](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)
- [AWS 定价说明](https://aws.amazon.com/pricing/)
- [全球网络常见问题](https://aws.amazon.com/about-aws/global-infrastructure/global-network/faqs/)
- [AWS 定价计算器](https://calculator.aws/)

Microsoft Azure：

- [了解磁盘计费](https://learn.microsoft.com/zh-cn/azure/virtual-machines/disks-understand-billing)
- [托管磁盘定价](https://azure.microsoft.com/zh-cn/pricing/details/managed-disks/)
- [带宽定价](https://azure.microsoft.com/zh-cn/pricing/details/bandwidth/)
- [Linux 虚拟机定价](https://azure.microsoft.com/zh-cn/pricing/details/virtual-machines/)
- [创建快照](https://learn.microsoft.com/zh-cn/azure/virtual-machines/snapshot-copy-managed-disk)
- [将快照复制到新区域](https://learn.microsoft.com/zh-cn/azure/virtual-machines/disks-copy-incremental-snapshot-across-regions)
- [磁盘常见问题](https://learn.microsoft.com/zh-cn/azure/virtual-machines/faq-for-disks)
- [删除 VM 和附加资源](https://learn.microsoft.com/zh-cn/azure/virtual-machines/delete)
- [Azure 定价计算器](https://azure.microsoft.com/zh-cn/pricing/calculator/)

Google Cloud：

- [Compute Engine 价格](https://cloud.google.com/products/compute/pricing?hl=zh-cn)
- [磁盘和映像价格](https://cloud.google.com/compute/disks-image-pricing?hl=zh-cn)
- [VPC 网络价格](https://cloud.google.com/vpc/network-pricing?hl=zh-cn)
- [磁盘类型与费用](https://cloud.google.com/compute/docs/disks?hl=zh-cn)
- [磁盘快照](https://docs.cloud.google.com/compute/docs/disks/snapshots?hl=zh-cn)
- [停止或重启实例](https://docs.cloud.google.com/compute/docs/instances/stop-start-instance?hl=zh-cn)
- [`instances.stop`](https://docs.cloud.google.com/compute/docs/reference/rest/v1/instances/stop)
- [快照时间表](https://docs.cloud.google.com/compute/docs/disks/about-snapshot-schedules)
- [Google Cloud 价格计算器](https://cloud.google.com/products/calculator?hl=zh-cn)

如需为这些云平台的国际站账号开户或充值，可访问海宝云官网 <https://www.haibaocloud.com>。充值币种和最低金额以官网当前说明为准。
