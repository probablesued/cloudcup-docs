# 云盘快照与自定义镜像：备份、回滚和新实例恢复

高危操作之前留一份磁盘备份，误删或系统损坏之后找得回那个时间点，或者用同一套系统盘环境再开一台实例，靠的是云盘快照和自定义镜像（AWS 文档里的 AMI，Azure 文档里的映像版本，Google Cloud 文档里的 custom image）。快照是某一时刻已经写进磁盘的副本。镜像或 AMI 是用来启动实例的模板，通常还关联着快照，两者不是同一项资源。

这篇只写云服务器磁盘的手动快照、自动策略、多盘一致备份、回滚或用快照建新盘、以及用系统盘快照做自定义镜像再开新实例。托管数据库的自动备份、时间点恢复和删除实例后的备份保留，见 [云数据库自动备份、保留期与恢复演练](cloud-database-backup-restore.md)。快照单价、免费额度、归档最短保留和提前删除补费，见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md) 里「快照：和磁盘分开计费，删盘不一定停」一节。谁有权创建或删除这些资源，见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。那些内容这里不重复。

核对时间：2026-11-23。资料核对日期：2026-11-23。崩溃一致性还是应用一致性、能不能对源盘回滚、系统盘快照能不能做镜像、删盘之后快照还在不在，都来自这一天打开的官方页面，并附有链接。打不开、正文没有写明，或同一主题两页说法不同的，下面按读到的原文分开写，不补成一套各云通用的「回滚按钮」。

范围：阿里云中国站用 help.aliyun.com 上这次能打开的现行页面。国际站打开了 www.alibabacloud.com 的英文快照概述、手动创建快照、回滚云盘、自动快照策略、快照一致性组概述、应用一致性快照，以及用快照创建自定义镜像。腾讯云中国站用 cloud.tencent.com。国际站打开了 www.tencentcloud.com 与 intl.cloud.tencent.com 的创建快照页（两页正文和更新时间相同），以及 www.tencentcloud.com 的回滚页和快照问答。用中国站文档编号打开 `https://www.tencentcloud.com/document/product/362/8191` 和 `https://intl.cloud.tencent.com/document/product/362/8191`，均返回 404，国际站定期快照策略的操作页这次没有打开，不要把中国站的保留类型和「关联实例」菜单套到国际站。AWS 使用 docs.aws.amazon.com 的全球文档，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 learn.microsoft.com 的全球文档，这次通读的是中文页；由世纪互联运营的 Azure 没有打开。Google Cloud 使用 docs.cloud.google.com。页面改版后，以你账号所属站点的当前文档为准。

## 先看这几件事

1. 快照是某一时刻的磁盘副本，默认多半只包含已经写入磁盘的数据。内存里还没落盘的内容不在里面。阿里云把未另做应用一致性的快照写成崩溃一致性快照。AWS 创建快照页写的是：快照只包含提出请求时已经写到卷上的数据，不含应用程序或操作系统缓存。腾讯云创建快照页写的是：只保存该时刻已经写入的数据，不含内存中尚未写入的数据，并举例 Linux 的 `/run`。Google Cloud 最佳实践页把应用仍在运行时拍下的快照称为 crash consistent。Azure 这次打开的托管磁盘快照页没有使用这两个名称；虚拟机还原点页才分开写了应用程序一致性和崩溃一致性。
2. 回到快照时刻，快照点之后的写入会丢掉。阿里云回滚页用警告写明，回滚不可逆，并建议回滚前再为云盘做一份快照。腾讯云回滚页写明，源盘会回到创建快照时的数据，此时刻之后的数据被清除。AWS、Azure、Google Cloud 这次打开的页面没有把恢复写成对源盘按一个回滚按钮覆盖：AWS 常见是用快照建新卷，或在实例仍为 running 时替换根卷；Azure 托管磁盘快照页写的是用快照创建新的托管磁盘；Google Cloud 恢复页写明，从快照恢复不会覆盖源盘，也不会删除源快照。
3. 多块盘、RAID 或逻辑卷要同一时刻一起备份时，只用各云页面写明的产品名。阿里云是快照一致性组，并可以在组里再启用应用一致性快照或文件系统一致性快照。腾讯云是快照组，页面写明目前为灰度内测。AWS 多卷快照在 RAID 页被写成 crash-consistent；Windows 上另有基于 VSS 的应用一致性快照。Google Cloud 最佳实践页对多盘应用一致性写的是：先手动暂停应用并冻结各盘文件系统，全部快照完成后再恢复。Azure 单盘快照页没有多盘一致性产品名；虚拟机还原点页写的是一台虚拟机上全部已附加托管磁盘各有一个磁盘还原点。
4. 镜像、AMI、映像和快照不是同一项资源。阿里云和腾讯云这次打开的页面都写明：只有系统盘快照能创建自定义镜像，数据盘快照不能单独当成自定义镜像的系统盘来源。腾讯云使用限制页同时写明，只有数据盘快照能创建新的弹性云硬盘，系统盘快照不能用来新建云硬盘。AWS 可以从实例或根卷快照注册 EBS-backed AMI，AMI 会占用快照，注册中的 AMI 会挡住根卷快照的删除。Azure 映像版本可以把磁盘或快照当作源。Google Cloud 自定义映像可以从磁盘、快照或其他映像创建；快照概述把「整台虚拟机或多块盘一起备份」指到 machine image，machine image 的操作页这次没有打开。
5. 自动快照策略、定期快照策略、快照时间表会按计划继续产生新快照。不设到期删除时，快照会一直留着。删除或释放云盘，通常不会把已经生成的快照一起删掉。阿里云只有打开了「自动快照随云盘释放」，释放云盘时才会把未到期的自动快照提前删掉。腾讯云问答写明，销毁云硬盘不会删除关联快照，自动快照也不随释放云硬盘而删除。AWS 删除卷不影响已有快照。Google Cloud 标准快照和归档快照不随源盘删除；即时快照会随源盘删除而消失。费用是否因此停掉，见账单一文，这里不展开单价。

## 对照

下表只填这次打开的页面写明的规则。单元格写「这次没读到」，是因为对应页面没有打开，或打开了但正文没有这句。恢复方式以各章原文为准，这里不合成一个按钮。

| 云 | 这次读到的默认一致性 | 多盘产品名 | 恢复到源盘 | 用快照做新实例环境 | 删盘或删实例之后 |
| --- | --- | --- | --- | --- | --- |
| 阿里云中国站 | 手动创建页：默认崩溃一致性，不含内存。一致性组里可再做成应用一致性或文件系统一致性 | 快照一致性组。同可用区，最多 128 块盘、合计不超过 256 TiB。控制台不能跨实例做应用一致性 | 只能回滚到原云盘。已挂载时要先停实例。共享快照和复制快照不能回滚 | 必须是系统盘快照。可另加数据盘快照。共享快照不能做镜像 | 只有设置了自动快照随云盘释放，释放云盘时才提前删除自动快照。手动快照这一页没有写成会一起删 |
| 阿里云国际站 | 与中国站对应页一致：默认 crash-consistent。标签写法见阿里云一章，两边不完全相同 | 快照一致性组概述已打开，盘类型名单与中国站一致性组页对齐；中国站概述页的名单更短，见正文 | 同样只能回滚源盘，已挂载须停机。对话框要求勾选确认。登录凭证两段文字见正文 | 同样必须是系统盘快照。Last Updated: Aug 24, 2026 | 自动策略页写了随盘释放会删除未到期自动快照。手动快照是否一起删除，这次打开的国际站页面没有写成会 |
| 腾讯云中国站 | 创建页不使用「崩溃一致性」这个词。只保留已写入数据，建议先关机，或 sync / 暂停写入后再拍 | 快照组。一台 CVM 的多块云硬盘，保证写入时序和崩溃一致性。页面写明灰度内测 | 只能回滚到源云硬盘。已挂载须关机。未挂载可直接回滚。跨地域复制出来的新快照不能回滚 | 只有系统盘快照能做自定义镜像。控制台创建镜像页是从实例制作，默认可以包含数据盘 | 销毁或释放云硬盘不会删除关联快照。自动快照按策略保留，不随盘释放 |
| 腾讯云国际站 | 创建快照页与中国站同方向：只捕获已写入数据，建议关机或暂停 I/O。定期策略操作页这次是 404 | 这次没读到快照组页面 | 回滚页已打开：只能回到源盘；已挂载须关机 | 问答写明只有系统盘快照能做自定义镜像，系统盘快照不能新建云硬盘。创建镜像的控制台页这次没有按国际站打开 | 问答写终止 CVM 不会自动删除关联快照。自定义镜像的关联快照要先删镜像 |
| AWS（全球文档） | 单卷快照不含缓存。多卷快照在 RAID 页写成 crash-consistent。Windows 可用 VSS 做成应用一致性，并打上 `AppConsistent:true` | 多卷快照，最多 128 块 EBS 卷。一套里有一块失败，其余进入 `error` | 这次打开的页面没有源盘原地回滚。根卷可以在实例 running 时替换。数据盘留在实例上 | 从实例或根卷快照创建 EBS-backed AMI。AMI 仍关联快照，先注销 AMI 才能删根卷快照 | 删除卷不影响已有快照。删除快照不影响原卷 |
| Azure（全球文档，中文页） | 托管磁盘快照页没写一致性模式。还原点页：省略 `consistencyMode` 时默认应用程序一致性；设为 `CrashConsistent` 时为崩溃一致性 | 还原点为每块已附加托管磁盘各做一个磁盘还原点。单盘快照页没有多盘一致性组 | 快照页写的是只读副本，用来创建新的托管磁盘。用快照创建新 VM 前要先彻底关闭源 VM | 计算图库的映像版本可以把磁盘或快照当作源。映像里的数据磁盘不能超过 1 TB | 快照页写明快照独立于源磁盘存在。删盘之后快照是否自动消失，这一页没有另写一句「会删除」 |
| Google Cloud | 应用仍在跑时拍下的快照，最佳实践页称为 crash consistent。要应用一致性需暂停应用；Linux 持久盘可用 `guest-flush`，Windows 持久盘可用 VSS。Hyperdisk 不支持这两项 | 快照概述把整机或多盘备份指到 machine image，操作页这次没打开。多盘应用一致性要手动冻结每块盘 | 从快照新建磁盘，不覆盖源盘，不删除源快照 | 启动盘快照可以在创建实例时选作启动盘。要批量开实例，恢复页建议先做自定义映像 | 标准快照和归档快照不随源盘删除。即时快照随源盘删除。时间表默认在删盘后仍永久保留已生成的自动快照，除非改了源盘删除规则 |

## 阿里云

中国站规则来自 help.aliyun.com。国际站英文页与中国站一致的部分会注明 Last Updated；只在一边读到的，不抄到另一边。

### 手动快照只包含已经写入云盘的数据

[快照概述](https://help.aliyun.com/zh/ecs/user-guide/snapshot-overview/)（页上「更新时间」栏为空）把快照写成云盘在某个时刻的完整拷贝。ECS 实例不会自动备份云盘数据。第一份标准快照是全量，后续标准快照是增量，但任一快照都能恢复到它自己那个时间点的全部云盘数据，原理见 [快照原理](https://help.aliyun.com/zh/ecs/user-guide/how-do-snapshots-work)（页上未见更新日期）。快照默认存进用户看不见的 OSS Bucket，不能改成自己的 Bucket。

[手动创建单个快照](https://help.aliyun.com/zh/ecs/user-guide/create-a-snapshot)（页上未见更新日期）写明：快照默认为崩溃一致性快照，只包含已写入云盘的数据，不含内存中的数据。业务要求更严时，该页建议创建应用一致性快照。创建瞬间可能对云盘 IO 产生一次毫秒级抖动。创建期间不要停止或重启实例，否则快照会失败；这段时间新写入的增量数据不会进入这份快照；正在创建快照的云盘不能扩容。

云盘要处于使用中或待挂载。使用中时，实例须为运行中或已停止。待挂载的盘必须曾经挂载过 ECS，从未挂载过的盘不能做快照。ESSD PL-X、本地盘和弹性临时盘不能做快照。这句来自中国站创建页和概述页。国际站 [Manually create a single snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/create-a-snapshot)（Last Updated: Oct 29, 2026）的限制句是本地盘和弹性临时盘，没有写出 ESSD PL-X。不要把 PL-X 补进国际站账号。

控制台路径：ECS 控制台快照页，选地域，云盘快照页签，创建云盘快照。资源类型默认是云盘。保留时间可以永久，也可以自定义天数。永久保留在额度用尽后不能再新建。自定义保留还剩 3 天时，列表会提示即将自动释放。ESSD 系列（ESSD、ESSD AutoPL、ESSD Entry 和 ESSD 同城冗余）默认开启极速可用，快照不必等到上传 OSS 100% 就能用于回滚、建盘或共享。进度列到「上传至 OSS：100%」才表示上传完成。正在创建的任务可以用 DeleteSnapshot 取消。

国际站同一页的控制台用词是 Disk Snapshots、Create Disk Snapshot。极速可用的英文是 Instant Access，默认开启的范围写的是 ESSD、ESSD AutoPL、ESSD Entry 和 regional ESSD。

### 自动快照策略会继续产生新快照

[创建自动快照策略](https://help.aliyun.com/zh/ecs/user-guide/create-an-automatic-snapshot-policy-1)（页上「更新时间」栏为空）：当前地域还没有任何策略时，第一次给云盘指定策略会创建名为 `default_policy` 的默认策略。控制台路径是 ECS 控制台的自动快照策略页。重复日期在周一到周日里选。开始时间是 UTC+8 的整点，快照在该整点之后的一小时内创建。一天要多次时，可以在 00:00～23:00 里选多个整点。

保留时间默认持续保留，也可以自定义。持续保留时，自动快照数量达到额度上限后，系统删除最早创建且没有被锁定的自动快照。中国站这一页没有写出「每块盘 2000 份」这个数字，额度指向使用限制页，那一页这次没有打开，不补数字。改保留时间只影响之后新建的自动快照。无论保留方式如何，不用的快照要另行删除，否则存储费继续，计费见账单文。

关联类型一旦选定就不能改，一个策略只能选一种。关联云盘可以随时增减。关联实例标签时，多个标签是逻辑与，只支持自定义标签，最少 1 个、最多 5 个；到点时按标签找出实例，为实例上挂载的全部云盘做快照。盘被卸载、实例解绑标签或策略解绑标签后，从下一个时间点起不再为这些盘做自动快照。高效云盘、SSD 云盘、普通云盘不支持并发创建快照，同时绑了多个策略时，自动快照可能被跳过。ESSD 系列可以手动和自动并发，但有并发上限；上一代云盘在自动快照进行中时，要等它完成才能手动再做。自动快照时间点如果正好有别的快照任务，这个时间点会跳过，下一个时间点再做。

可选的跨地域复制会在目标地域产生流量费和存储费，见账单文，这里不抄单价。归档快照不能复制，最短保留和提前删除补费也只在账单文里核对。

若设置了自动快照随云盘释放，手动释放云盘、云盘随实例释放或更换系统盘时，未到期的自动快照也会被提前删除。没有这句设置时，不要默认「盘释放了，自动快照费就停了」。

国际站 [Create an automatic snapshot policy](https://www.alibabacloud.com/help/en/ecs/user-guide/create-an-automatic-snapshot-policy-1)（Last Updated: Oct 08, 2026）的执行时间同样写 UTC+8，默认策略名同样是 `default_policy`，关联类型同样创建后不能改。保留方式里「直到达到额度后删除最早的未锁定自动快照」这一项，国际站正文写了 up to 2000 automatic snapshots per disk。这个 2000 只出现在国际站该页，不回写成中国站配额。

### 多盘用快照一致性组

[快照一致性组](https://help.aliyun.com/zh/ecs/user-guide/snapshot-consistency-group-overview/)（页上未见更新日期）用于同可用区内一台或多台 ECS 的多块云盘，在同一时间点做快照，保证崩溃一致性和时序一致性。创建组本身不收费，组里的快照按容量和时长计存储费。

盘类型有两页不同的名单，分开保留：

- 概述页的使用限制：目前仅 ESSD、ESSD AutoPL 和 ESSD Entry 支持快照一致性组，且未开启多重挂载。
- 一致性组专页：ESSD、ESSD AutoPL、ESSD Entry 和 ESSD 同城冗余都支持；同城冗余只能和同类型云盘放在一组。仅 ESSD 类型支持应用一致性快照。本地盘和弹性临时盘两者都不支持。

组内快照不支持自定义保留时间，默认永久保留。单个组最多 128 块盘，总容量不超过 256 TiB。单块盘正在执行的快照任务（手动、自动、一致性组）上限 10 个，到了就不能再被选进新的组。删除一致性组会删掉组内全部快照，删除后不能恢复。

国际站 [Snapshot-consistent group overview](https://www.alibabacloud.com/help/en/ecs/user-guide/snapshot-consistency-group-overview)（Last Updated: Apr 27, 2026）的盘类型名单与中国站一致性组专页一致，包含 ESSD zone-redundant，并写明 zone-redundant 只能与同类型放在一组。国际站概述页的限制表却只写了 ESSD、ESSD AutoPL、ESSD Entry。和中国站一样，概述和专页不要合成一张表。

云盘被做成逻辑卷或 RAID 时，手动创建页建议使用快照一致性组并开启应用一致性快照。回滚这类盘之前，回滚页要求先停掉使用该盘的服务，确认没有读写在进行。

### 应用一致性快照和文件系统一致性快照

[创建应用一致性快照](https://help.aliyun.com/zh/ecs/user-guide/create-application-consistent-snapshots-in-the-ecs-console/)（页上未见更新日期）在创建一致性组时开启。它解决的是内存或尚未落盘的应用数据。目前仅未开启多重挂载的 ESSD 支持。ECS 控制台不能跨实例创建应用一致性快照。

操作系统范围：Windows Server 2012 R2 及以上；Linux 为 CentOS 7.6 及以上、Ubuntu 18.04 及以上或 Alibaba Cloud Linux 2。实例须在运行中，云助手正常。实例要绑定一个 RAM 角色，策略至少覆盖查询快照、创建快照、打标签和查询云盘。未装云助手时，勾选启用应用一致性快照会自动安装插件。

Linux 靠固定路径的脚本：`/tmp/prescript.sh` 在快照前暂停应用写入，`/tmp/postscript.sh` 在完成后恢复。名称、类型和路径不能改，权限是仅 root 可读写执行。脚本成功，快照打上 `APPConsistent:True`；没配脚本或脚本失败，得到的是文件系统一致性快照，标签为 `FsConsistent: True`。应用一致性是否成立，由你自己的脚本负责。还可以勾选文件系统 IO 暂停与恢复。中国站的说明是：若把超时设为 16 秒，在这 16 秒内暂停没完成，则暂停失败，可能不一致。

Windows 用卷影复制服务 VSS。同时勾选启用应用一致性快照和「是否默认包含 Writer」时，创建应用一致性快照；只勾选启用、不包含 Writer 时，创建文件系统一致性快照。没有默认 Writer 时，背景表写的也是文件系统一致性快照。

控制台路径：快照一致性组页签，创建快照一致性组，资源类型为实例，选一台实例上的 ESSD，在高级配置里启用。返回的云助手命令执行 ID 里，ExitCode 为 `0` 表示成功。不为 `0` 时按该页错误码表排查，例如 9 是 prescript.sh 失败，11 是暂停 I/O 失败，14 是快照个数超限。API 走云助手 RunCommand，Linux 插件名是 `app-snapshot-plugin`，Windows 是 `app-snapshot-plugin-win`。`EnableFsFreeze` 默认 True，`TimeoutInSeconds` 默认 30。

国际站 [Create application-consistent snapshots](https://www.alibabacloud.com/help/en/ecs/user-guide/create-application-consistent-snapshots-in-the-ecs-console)（Last Updated: Aug 25, 2026）的操作系统范围、脚本路径、控制台不能跨实例、ExitCode 表和默认超时 30 秒，与中国站对应。标签字符串不同：国际站写的是 `AppConsistent:True` 和 `FsConsistent:True`。中国站写的是 `APPConsistent:True` 和 `FsConsistent: True`。以账号所在站点快照上的实际标签为准，不要互换大小写。超时那一句也不同：国际站写的是超时超过 16 秒且在该时间内暂停没完成则失败。不把两句收成一个「官方就是 16 秒上限」。

### 回滚只作用于源云盘，而且会丢掉之后的写入

[使用快照回滚云盘](https://help.aliyun.com/zh/ecs/user-guide/roll-back-a-disk-by-using-a-snapshot)（页上未见更新日期）：只能把快照回滚到创建它的那块原云盘。要恢复到别的盘，该页写的是用快照另建一块云盘，创建云盘的逐步点击这次没有打开。已经更换过操作系统的，更换前的系统盘快照不能回滚更换后的系统盘，只能用历史快照建一块按量付费云盘，挂上去拷数据，用完释放。共享快照和复制快照都不能回滚。

前提：快照已创建完成，没有正在创建的快照；原盘还在，释放了就无法回滚；盘已经挂载时，必须先停止 ECS。回滚不可逆。从快照创建时间到回滚之间的数据会丢失。该页建议回滚前再手动做一份当前快照。

控制台：快照页，云盘快照页签，操作列回滚云盘，确定。回滚后，快照之后的主机名、SSH 配置、密码、网络、软件源、时钟源等都会回到快照时刻，需要按当前环境重配。若快照之后扩容过，回滚会把容量退回快照时的大小，扩容出来的空间丢失，还要再扩容文件系统。系统盘回滚后，该页写默认会自动绑定实例当前的 SSH 密钥对或用户名密码，用这套认证登录。

国际站 [Roll back a disk with a snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/roll-back-a-disk-by-using-a-snapshot)（Last Updated: Sep 21, 2026）同样只能回滚源盘，已挂载必须停机，共享快照和复制快照不能回滚，并建议回滚前再做快照。对话框比中国站多一句：要勾选确认再点 OK。登录相关有两段，都留在这里：一张表写密码和 SSH 设置会回到快照时刻；后文又写回滚系统盘不影响实例当前的 SSH 密钥对、用户名和密码，用现有凭证登录。不把这两段改写成一句。

只恢复某个文件夹时，两边都写：不要整盘回滚，用快照做一块临时按量盘，挂上，拷文件，再卸载并释放临时盘。

### 只有系统盘快照能做自定义镜像

[使用快照创建自定义镜像](https://help.aliyun.com/zh/ecs/user-guide/create-a-custom-image-from-a-snapshot)（页上未见更新日期）：必须是系统盘快照。数据盘快照不能创建自定义镜像。共享快照也不能。加密快照做成的镜像默认加密，密钥与源快照相同。镜像和快照必须同一地域；要换地域，先做好镜像再复制镜像。源实例到期或释放后，已经用该快照做成的镜像，以及用该镜像开出的实例，不受影响。

控制台可以从镜像页、实例详情或快照页进入。快照页的路径是：云盘快照里找到云盘属性为系统盘的快照，操作列创建自定义镜像。可选增加数据盘并指定数据盘快照 ID，镜像会带上同样数量的数据盘。不选数据盘快照 ID 时，新数据盘是空盘，默认容量 5 GiB。选了则容量等于快照容量。镜像检测默认「创建后执行检测」，只覆盖部分操作系统。

若快照已经配置了自动释放时间，必须勾选已阅读快照计费通知。快照到期前要先手动删除由它创建的镜像，否则快照不会按计划释放，费用继续。快照存储费本身见账单文。

国际站 [Create a custom image from a snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/create-a-custom-image-from-a-snapshot)（Last Updated: Aug 24, 2026）的限制和 5 GiB 空数据盘默认值与中国站一致。自动释放那句同样要求在到期前删掉相关镜像，否则快照不会释放并继续计费。

## 腾讯云

以下中国站步骤来自 cloud.tencent.com。国际站只采用这次打开的 www.tencentcloud.com 与 intl.cloud.tencent.com 页面。定期快照的点击顺序不要套到国际站。

### 创建快照时，内存里的数据不在里面

[创建快照](https://cloud.tencent.com/document/product/362/5755)（最近更新时间：2026-05-19 17:00:51）用增量方式保存指定时刻的云硬盘数据。删除某一份快照后，没被删除的快照仍能把盘恢复到各自的时间点。任何状态下都能创建，但只能保存已经完成写入的数据。应用程序正在写入的部分可能进不了这份快照。该页强烈建议创建前关机，或确认内存数据已经写入并暂停读写。

数据库示例是自建 MySQL：先执行 `FLUSH TABLES WITH READ LOCK;`，再做快照，然后 `UNLOCK TABLES;`。系统层示例是执行 `sync`，把文件系统缓冲区写进云硬盘，并在创建前不要再写入。这是云盘快照前的准备，不是托管数据库的自动备份。TencentDB 的备份见数据库一文。

控制台：云硬盘控制台，目标盘操作列创建快照，填写名称，选择保留时间，确定。API 是 CreateSnapshot。快照控制台可以看状态、硬盘属性、关联硬盘、保留时间和关联镜像。本地盘不支持快照，这句在 [快照使用问题](https://cloud.tencent.com/document/product/213/44127)（页上未见更新日期）。制作快照会占用少量 I/O，[快照相关问题](https://cloud.tencent.com/document/product/362/17820)（页上未见更新日期）建议放在业务相对空闲时。不需要为了创建而先卸载；该页同时建议系统盘先关机，以便更完整。

国际站 [Creating Snapshot](https://www.tencentcloud.com/document/product/362/5755) 与 [intl.cloud.tencent.com 同编号](https://intl.cloud.tencent.com/document/product/362/5755) 都打开了，Last updated 都是 2023-12-22 10:19:47，正文相同：CBS 控制台里对目标盘 Create a snapshot，对话框只写了填写名称后点 OK，没有中国站那句「选择保留时间」。`FLUSH TABLES WITH READ LOCK` 和 `sync` 的建议还在。国际站账号按这两页操作，不要补上中国站的保留时间下拉框。

### 定期快照会持续创建，禁用策略也不会删掉已有快照

[定期快照](https://cloud.tencent.com/document/product/362/8191)（最近更新时间：2026-08-14 15:03:00）只在中国站打开。控制台：云硬盘控制台，快照，定期快照策略，选地域，新建。备份周期可以按天（间隔 1～365 天）、按周或按月。备份时间点是 00:00～23:00 的整点，控制台实际创建时间可能和设定整点有差异，快照里的数据以控制台创建时间为准。

保留类型有按数量、按天、永久保存。按天的保留时间该页默认写保留 30 天。永久保存不自动删除。高级保留策略可选，和普通保留一起生效，但优先级更高。停止重复可以永不停止、计次或到截止时间。自动快照名称形如 `snap_yyyyMMdd_HH`。

关联云硬盘后，对勾选的盘按策略执行。关联云服务器时，会对这台云服务器的所有云硬盘创建快照组；该功能页写明为灰度内测，并指向创建快照组。关闭策略开关后不再创建新的定期快照，已经创建的仍按原规则保留或销毁。手动快照默认长期保存，只要账户余额充足。定期快照和手动快照使用上不冲突，但同一块盘不能并行：一边在做时，另一边要等它完成。一次快照如果跨过了下一个时间点，那个时间点不再补做。该页举例：9:00、10:00、11:00 三个点，9:00 那次用了 70 分钟，10:00 就不做，下一次是 11:00。

[使用限制](https://cloud.tencent.com/document/product/362/5145)（最近更新时间：2026-10-08 11:49:00）：单块云硬盘快照配额 1000 个。单地域定期快照策略最多 30 个。每个策略最多绑定 200 块云硬盘。一块云硬盘最多绑定 10 个所属地域的定期策略。快照总容量该页写无限制。定期快照页另写：快照数量达到上限后，自动快照任务会持续失败。快照使用问题页则写：自动快照达到上限后，创建时间最早的自动快照会自动删除。两句都在中国站已打开的页面里，条件不完全相同，不合成一个算法。

国际站定期快照操作页两个地址都是 404。国际站 [Snapshot](https://www.tencentcloud.com/document/product/362/17820)（Last updated: 2023-12-22 11:00:17）仍然写了 scheduled snapshot 的存在，以及单地域最多 30 个策略、每个策略最多 200 块盘，具体操作指向 Scheduling Snapshots。那一页这次没打开，30 和 200 只当作这则问答里的数字，不补中国站的按天默认 30 天、高级保留和关联实例菜单。

### 快照组

[创建快照组](https://cloud.tencent.com/document/product/362/115828)（最近更新时间：2025-01-21 17:38:52）只在中国站打开。它为一台 CVM 上的多块云硬盘同时做快照，保证跨盘写入的时序一致性和崩溃一致性。页面写明功能处于灰度内测，要联系客户经理或提交工单申请。实例须为运行中或已关机，云硬盘状态为使用中。

控制台：云服务器控制台，实例行更多，创建快照组，选盘，填快照组名称和保留时间，确认。结果在快照列表的快照组里。创建前的数据库锁和 `sync`，与创建单份快照页相同。国际站没有打开对应页面，不把「更多 > 创建快照组」写成国际站菜单。

### 回滚源盘，或用数据盘快照建新盘

[从快照回滚数据](https://cloud.tencent.com/document/product/362/5756)（最近更新时间：2025-05-28 16:18:43）：只支持回滚到创建该快照的源云硬盘。源盘须处于正常生命周期，不是过期或隔离。未挂载（待挂载）可以直接回滚。已挂载时，云服务器必须关机。控制台：云硬盘控制台，快照，快照列表，行内回滚，确认。API 是 ApplySnapshot。回滚会清除快照时刻之后的数据。

使用限制页补充：要用已有快照得到一块新盘，走「从快照创建云硬盘」，新盘容量必须大于或等于源盘。只有数据盘快照能创建新的弹性云硬盘。系统盘快照不能新建云硬盘。从快照创建云硬盘的逐步点击这次没有打开，不补菜单。

[快照使用问题](https://cloud.tencent.com/document/product/213/44127) 写了一则广州地域的例子：旧实例到期释放后，同地域新买的实例不能用旧数据盘快照回滚出原来的数据。回滚对象是原来的盘。可行的做法是用数据盘快照创建云硬盘，再挂到新实例。跨地域复制完成后，新快照和源盘不再关联，回滚功能不可用；复制完成前不能删除源快照。这几句在快照相关问题页同样有。

国际站 [Rolling back Data from Snapshot](https://www.tencentcloud.com/document/product/362/5756)（Last updated: 2023-12-22 10:20:44）同样只回到源盘，未挂载可直接回滚，已挂载须关机。控制台入口写成 Snapshot List 页的 Rollback。它没有中国站那句「非过期或隔离」。国际站问答确认：系统盘快照不能用来创建云硬盘，只能做自定义镜像；终止 CVM 不会自动删除关联快照。

### 自定义镜像会带出关联快照

[创建自定义镜像](https://cloud.tencent.com/document/product/213/4942)（最近更新时间：2026-06-09 15:48:01）是中国站控制台说明。镜像底层使用云硬盘快照。创建时会默认创建关联快照，保留镜像会产生快照费用，赠送额度和单价见账单文，这里不重算。每个地域该页写暂支持 500 个自定义镜像。Linux 只做系统盘镜像时，确认 `/etc/fstab` 里没有数据盘配置，否则用该镜像开出的实例可能起不来。

2018 年 7 月之后创建、系统盘为云硬盘的实例，支持在线制作镜像。除此以外的实例，要先关机再制作。控制台有两个入口：实例列表的更多（或页签视图的更多操作）里，镜像/操作系统，创建自定义镜像；或镜像页上方的创建自定义镜像。实例含数据盘时，默认做系统盘加数据盘的整机镜像，也可以改成只要系统盘。做好后在镜像列表用创建实例购买新服务器。配置 25Gbps 网卡的裸金属该页写暂不支持用控制台和 API 创建自定义镜像。

快照相关问题页写明：系统盘和数据盘都能做快照，但只有系统盘快照能创建自定义镜像。镜像和关联快照是绑定关系，删除快照前要先删除关联镜像。创建过云硬盘的快照可以单独删除；创建过自定义镜像的快照必须先删镜像。用定期快照创建了自定义镜像或云硬盘，不会导致快照策略执行失败。

[创建镜像 API](https://cloud.tencent.com/document/product/213/16726)（最近更新时间：2025-09-19 11:38:37）是另一份说明，不是控制台点击稿。CreateImage 里 `InstanceId` 和 `SnapshotIds` 二选一。基于快照时，`SnapshotIds` 必须包含一个系统盘快照，不能和 `InstanceId` 同时传。`ForcePoweroff` 默认 `false`，表示开机状态制作镜像；该页写开机状态制作可能导致部分数据未备份。`DataDiskIds` 只在基于实例做整机镜像时使用。

国际站问答有「只有系统盘快照能做镜像」「先删镜像再删关联快照」，没有打开国际站的创建自定义镜像控制台页，不把中国站的「2018 年 7 月之后可在线制作」和整机镜像默认值套过去。

欠费后的隔离天数只在中国站快照使用问题页读到：欠费后快照进入已隔离，保留 30 天，期间余额没有回到大于等于 0，到期删除该账户下除镜像快照以外的快照。国际站这次打开的问答没有这句，不套用 30 天。

## AWS

以下来自 docs.aws.amazon.com。没有打开 docs.amazonaws.cn。中国区账号不要把下面的控制台路径当成中国区文档已经核对过。

### 单卷快照不含缓存，AWS 也不会自动备份

[Amazon EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)（页上未见更新日期）把快照写成 EBS 卷的时间点副本，增量保存自上一份快照以来变化的块。快照存在你不能直接访问的 S3 桶里，不能用 S3 控制台或 S3 API 去读。数据会在该区域的全部可用区复制。从快照创建的新卷，一开始是创建快照时那块源卷的副本。AWS 不会自动备份 EBS 上的数据。要定期自己做，或用 Amazon Data Lifecycle Manager、AWS Backup 自动做。这两份产品的操作页这次没有按步骤打开，不写策略向导。

[Create Amazon EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)（页上未见更新日期）：创建是异步的。快照马上出现，但在数据传到 S3 之前状态是 `pending`，可能要数小时。`pending` 期间可以继续使用卷，快照内容不受影响。快照只包含提出请求时已经写到卷上的数据，不含应用程序或操作系统缓存的数据。该页建议创建前暂停写入；做不到时，在实例里先卸载卷，等快照进入 `pending` 再挂回去。根卷建议先停止实例再拍。同一卷上多份 `pending` 快照会拉低卷性能。加密状态跟随源卷，不能在创建对话框里改。未加密卷要得到加密快照，先做未加密快照，再复制成加密副本。

[Create a snapshot of an EBS volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshot.html)（页上未见更新日期）的控制台：EC2 控制台 <https://console.aws.amazon.com/ec2/>，Snapshots，Create snapshot，Resource type 选 Volume，选 Volume ID。卷在区域里时，没有 Snapshot destination，快照就在该区域。CLI 示例是 `aws ec2 create-snapshot --volume-id ...`。示例里的卷 ID 要换成你的。

### 多卷是崩溃一致，Windows 可以再做应用一致

[Create multi-volume EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshots.html)（页上未见更新日期）：Resource type 选 Instance，默认为该实例上全部 EBS 卷做快照，可以排除根卷或指定的数据卷。一套最多 128 块，含根卷和最多 127 块数据卷。其中一块失败，其余进入 `error`。每份快照仍是独立快照。该页写可以借助 Systems Manager 命令文档，为 Windows 实例上的全部 EBS 卷做应用一致性快照。

[Amazon EBS and RAID](https://docs.aws.amazon.com/ebs/latest/userguide/raid-config.html)（页上未见更新日期）写明：RAID 里的卷如果各拍各的，恢复时对不齐会破坏阵列完整性。该页给出的做法是多卷快照：跨实例上多块 EBS 卷的时间点、数据协调、crash-consistent 快照，不必为了让各卷彼此对齐而先停实例。RAID 5、RAID 6、RAID 1 在这页被写成不建议用于 EBS，理由是校验写或双写占用 IOPS。这是该页的建议，不是一条禁止创建的配额。

[Application consistent Windows VSS based snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/application-consistent-snapshots.html)（页上未见更新日期）只支持 Windows。通过 Systems Manager Run Command 调用 VSS，把应用程序和磁盘之间尚未完成的事务写入快照。不必关机，也不必断开磁盘。组件包名是 `AwsVssComponents`，须装在实例上。推荐文档是 `AWSEC2-VssInstallAndSnapshot`。过程中 I/O 暂停最多十秒，然后为实例上全部卷做快照，成功的快照打上标签 `AppConsistent:true`。使用这项功能没有另收一笔功能费，仍按由此产生的 EBS 快照计费，单价见账单文。Linux 的应用一致性步骤这次没有打开专页，不把 VSS 或十秒暂停写成 Linux 的官方步骤。创建快照页对 Linux 的建议仍是暂停写入或先卸载。

### 恢复不是对源卷按回滚，根卷可以替换

这次打开的 EBS 页面没有「把源卷原地回滚到快照」的按钮。从快照得到数据的路径，概述页写的是创建新的 EBS 卷。新卷的创建点击页这次没有单独打开，不补菜单。

根卷有另一条路径。[Replace the root volume](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html)（页上未见更新日期）：实例必须处于 `running`。过程中实例会自动重启，内存内容被清掉，不需要你再手动重启。实例存储卷、非根 EBS 卷、网卡和 IP 保持不动。只能用于 EBS 根卷，不能替换实例存储根卷。

控制台：Instances，Actions，Monitor and troubleshoot，Replace root volume。Restore 可以选：

- Launch state：用创建当前根卷时的那份快照。
- Snapshot：用你指定的快照。只能用直接从该实例当前或以前的根卷创建的快照，不能用这些快照再复制出来的副本。
- Image：用你指定的 AMI。AMI 的产品代码、计费信息、架构和虚拟化类型要与实例一致。多块盘的 AMI 只用它的根卷，其余卷忽略。不能用于带 AWS Marketplace 产品代码的 AMI。

还可以用一块已经准备好的、同一可用区、当前未挂载且状态为 `available` 的 EBS 卷来换。加密根卷不能换成未加密卷。可以选择在替换成功后删除被换下来的根卷；选了删除，原根卷不可恢复。不选则原根卷仍留在账号里，要自己再删，存储费不会跟着替换结束而消失。任务状态从 `pending`、`in-progress` 到 `succeeded`。状态为 `failed-detached` 时，该页写明替换失败，实例上没有挂着根卷。这和阿里云、腾讯云「先关机，再把源盘覆盖回快照」不是同一套操作。

### AMI 占用根卷快照

[Create an Amazon EBS-backed AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)（页上未见更新日期）可以从实例做，也可以从根卷快照做。从实例做时：Instances，Actions，Image and templates，Create image。Reboot instance 默认勾选，创建时会重启，以便在数据静止时拍快照。取消勾选则不关机、不重启，该页警告不能保证文件系统完整。过程中会为根卷和已挂载的其他 EBS 卷做快照。在注销 AMI 并删除这些快照之前，快照持续计费。实例存储卷里的数据不会进入新实例，EBS 卷上的数据会。新实例的根卷和额外 EBS 在投入使用前要初始化，初始化专页这次没有打开。

从快照做：Snapshots 里选快照，Actions，Create image from snapshot。要填架构、根设备名和虚拟化类型。根卷容量不能小于快照。Windows、Red Hat、SUSE 和 SQL Server 要核对新 AMI 的 Platform details；为空或与预期操作系统不符时，该页说这次创建不成功，应丢弃并改从实例创建。CLI 示例是 `aws ec2 register-image`，块设备映射里带上根卷的 SnapshotId。

[Delete an Amazon EBS snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-snapshot.html)（页上未见更新日期）：删除快照不影响原卷，删除卷也不影响已经生成的快照。注册中的 AMI，即使已弃用或禁用，也会占住根设备快照，要先注销 AMI。禁用的 AMI 在 AMIs 页面要打开 Disabled images 过滤才看得见。AWS Backup 管理的快照不能在 EC2 里直接删。别人共享给你的快照不能删；你删掉自己共享出去的快照后，对方失去访问。符合回收站规则的快照删除后会先进回收站，回收站专页这次没有打开，不写保留天数。控制台删除时要输入 `delete`。增量快照删掉一份后，仍被其他快照引用的数据会留下来，费用改记到后续快照，所以份数减少不等于账单等额下降。算法和示例见账单文。

## Microsoft Azure

以下来自 learn.microsoft.com 的全球文档，通读的是中文页。由世纪互联运营的 Azure 没有打开，不把下面的一致性默认值和门户名称套到那套环境。

### 托管磁盘快照是独立的只读副本

[创建 Azure 托管磁盘的快照](https://learn.microsoft.com/zh-cn/azure/virtual-machines/snapshot-copy-managed-disk)（页上未见更新日期）把快照写成托管磁盘的完整只读副本，可以拍 OS 磁盘或数据磁盘，独立于源磁盘存在，用来创建新的托管磁盘。该页没有写「回滚源盘」。用快照创建新的 VM 之前，要先彻底关闭源 VM，以便清掉仍在进行的进程。

这次打开的正文，包括带 `pivots=azure-portal` 的地址，给出的创建步骤是 PowerShell，不是门户点击顺序。示例名是文档示例：资源组 `myResourceGroup`，区域 `eastus`，VM `myVM`，快照名 `mySnapshot`。`New-AzSnapshotConfig` 的 `-SourceUri` 在示例里取的是 OS 磁盘的托管磁盘 ID，`-CreateOption copy`。默认使用本地冗余的标准存储。该页建议用标准存储而不是高级存储，并写高级快照会额外产生费用，没有在这页给出单价。要区域冗余时，在支持可用性区域的区域加上 `-SkuName Standard_ZRS`。拍完用 `Get-AzSnapshot` 核对。

计费按已用容量、与源盘分开。该页用 64 GiB 预配、10 GiB 已用做了用量报告演算。那是该页示例，单价和免费额度见账单文，这里不把演算当成你的账单。

门户里从哪一个磁盘菜单点创建快照，这次读到的正文没有写出，不补。从快照创建新磁盘的点击或 CLI 专页这次也没有打开。

### 虚拟机还原点才写了两种一致性

[使用虚拟机还原点](https://learn.microsoft.com/zh-cn/azure/virtual-machines/virtual-machines-create-restore-points)（页上未见更新日期）是另一项资源，不是上面那份单盘快照 cmdlet 的别名。还原点保存捕获时的虚拟机配置，以及每块已附加托管磁盘的时间点快照，每块盘一个磁盘还原点。还原点放在作用域为这台虚拟机的还原点集合里。第一个还原点是完整副本，之后是增量。可以排除单块盘。

一致性模式表：

- 应用程序一致性：Windows 用 VSS 写入器，Linux 用预/后处理脚本，在快照前刷新正在处理的应用数据。省略 `consistencyMode` 时这是默认值。
- 崩溃一致性：捕获所有磁盘的写入顺序一致性快照，相当于断电或崩溃后的状态。创建请求里把 `consistencyMode` 设为 `CrashConsistent`。

即时访问只适用于把高级 SSD v2 或超级磁盘用作数据磁盘的 VM，同时覆盖应用程序一致性和多磁盘崩溃一致性（预览）。`instantAccess` 设在还原点集合上，默认 `false`。`instantAccessDurationMinutes` 设在每个还原点上，有效范围 60～300 分钟，默认 300。即时访问目前通过 REST、SDK、CLI 和 ARM 模板，该页写当前不支持门户。即时访问的存储费和一次性还原费该页有说明，单价不抄。

限制里和操作直接相关的几条：只支持托管磁盘；崩溃一致性不支持写入加速磁盘、临时 OS 磁盘和共享磁盘；NVMe 磁盘控制器或 Azure Boost 上的 VM 不支持多磁盘崩溃一致性还原点；应用程序一致性同样不支持临时 OS 磁盘和共享磁盘；一台虚拟机最多保留 500 个还原点，不论集合数和一致性类型；不能为同一台虚拟机并发创建还原点；统一业务流程的虚拟机规模集不支持。虚拟机上还有还原点时，不能在资源组或订阅之间移动这台虚拟机。新建还原点的限速是每小时 3 次（应用程序一致和崩溃一致各有一行），超出返回 HTTP 429。

还原点的门户创建步骤和「从还原点建新磁盘」的点击，这次打开的章节没有写到操作清单的那一步，不补。故障排查该页指向 VM 代理和扩展，专页没有打开。

### 映像版本可以用磁盘或快照当源

[创建映像定义和映像版本](https://learn.microsoft.com/zh-cn/azure/virtual-machines/image-version)（页上未见更新日期）写的是 Azure 计算图库，不是一个单独的「自定义镜像」按钮。映像源可以是 VM、托管映像、快照、托管磁盘、存储账户里的 VHD，或其他映像版本。许多能力该页写明只通过计算库提供，不适用于托管映像。映像如果包含数据磁盘，数据磁盘大小不能超过 1 TB。

从 VM 捕获的门户步骤，该页指向「捕获 VM 的映像」，那一页这次没有打开。非 VM 源的门户路径是：搜索 Azure 计算图库，选库，添加，VM 映像定义。操作系统状态按源选择通用化或专用化。通用化前的 Linux 或 Windows 准备，该页指向各自的指南，这里不展开 Sysprep 命令。版本号格式是 Major.Minor.Patch。源选「磁盘或快照」时，OS 磁盘下拉框里选磁盘或快照，数据盘要填 LUN 再选。创建前的最佳实践是先停止并解除分配 VM。CLI 用 `az sig image-definition create` 和 `az sig image-version create`；专用化 Linux 示例带 `--os-state specialized`。复制区域必须包含源所在区域。同一托管映像要等当前版本复制完成，才能再用来创建另一个版本。

因此，Azure 上「用快照开一台新实例」至少分成两件事：用快照创建新的托管磁盘，或把快照放进映像版本再用来创建 VM。两条都不是把源盘覆盖回快照时刻。

## Google Cloud

以下来自 docs.cloud.google.com。

### 标准快照不随源盘删除，即时快照会

[磁盘快照](https://docs.cloud.google.com/compute/docs/disks/snapshots)（页上未见更新日期）写明：标准快照对单块盘做增量、异地冗余备份，盘可以挂在正在运行的实例上。要备份整台虚拟机或多块盘，该页指向 machine image。machine image 的创建页这次没有打开，不写它的菜单。

三种快照都是某一时刻的磁盘内容，差别在删盘之后还在不在、恢复快慢和存放位置：

- 标准快照：不随源盘删除。可以用快照时间表创建。
- 归档快照：同样不随源盘删除。不能用快照时间表创建。删除不可恢复。
- 即时快照：存在到源盘被删除为止。不能用快照时间表创建。Hyperdisk ML 和 Hyperdisk Throughput 不能做即时快照。

从快照恢复的意思是用快照创建新磁盘，新盘可以马上达到其最大性能，不需要预热。恢复不会覆盖源盘，也不会删除源快照。标准快照默认是全球资源。区域范围（regionally scoped）这次在快照概述和创建页都标成 Preview。

快照默认增量。该页同时写，为了快照历史的可靠性，某一份快照有时会自动拍成全量，以前的增量快照不变。你不用在创建时选择全量还是增量。删掉一份仍被其他快照依赖的快照时，恢复其他快照所需要的数据会留下，不再被引用的数据才删除，总容量可能下降，但不是整份都消失。标准快照删除后，回收站（Preview）默认保留 3 天，项目、标签或组织可以改成 1～7 天。归档快照没有这段可恢复时间。

### 崩溃一致是默认结果，应用一致要自己暂停

[快照最佳实践](https://docs.cloud.google.com/compute/docs/disks/snapshot-best-practices)（页上未见更新日期）：应用还在运行时做快照，可能抓不到内存中尚未写到盘上的数据。这种快照是 crash consistent，相当于快照时刻机器崩溃。不必为了崩溃一致性去暂停负载。能不能用，取决于应用能否从崩溃状态把日志重放回来。

把应用暂停，让事务完成并把待写数据刷到盘上，得到的是 application consistent。支持范围按盘和操作系统分开：

- Hyperdisk：不支持 `guest-flush`，也不支持 VSS。要应用一致性，只能在创建前手动暂停应用。
- 挂在 Windows Server 上的 Persistent Disk：使用 VSS 快照。VSS 操作页这次没有打开。
- 挂在 Linux 上的 Persistent Disk：用快照前后的 shell 脚本，并在创建时打开 `guest-flush`。脚本专页这次没有打开。

多块盘之间也要应用一致性时，该页写明必须手动处理：冻结每块盘上的文件系统，这些盘的快照都完成之后再恢复应用。不必关机。手动暂停之后，要等快照资源进入 `UPLOADING` 再恢复负载。`UPLOADING` 表示快照已经创建，但还没保存到 Cloud Storage。`READY` 要等操作 `DONE`。

单块盘每 60 分钟最多做 6 次快照，超过返回 `RESOURCE_OPERATION_RATE_EXCEEDED`。从磁盘创建映像、从即时快照做标准或归档快照，也算在这个限制里。快照时间表不算。该页的实践是大约每小时一次，并避开业务结束和午夜的高峰。从同一份快照创建新的区域级 Persistent Disk 或 Hyperdisk，每个目标区域每 60 分钟最多 6 次；创建区域级 Persistent Disk 不受这句限制。要一次性做很多块盘时，该页建议先用快照做映像，再从映像做盘。

[创建快照](https://docs.cloud.google.com/compute/docs/disks/create-snapshots)（页上未见更新日期）的控制台：Create a Snapshot，填写名称，类型默认 `STANDARD`，也可选归档，选择源盘和存储位置，Create。存储位置不选时，用项目快照设置里的默认位置；在你第一次改设置之前，预定义默认是离源盘最近的多区域。gcloud 是 `gcloud compute snapshots create`，源盘用 `--source-disk` 和 `--source-disk-zone`。若创建过程失败，在拍到一份干净快照之前不能删除源盘。创建前如果暂停了应用，同样要等到 `UPLOADING` 再恢复。

### 快照时间表只产生标准快照

[快照时间表](https://docs.cloud.google.com/compute/docs/disks/about-snapshot-schedules)（页上未见更新日期）：时间表创建的是标准快照，不能创建归档或即时快照。频率只能在 hourly、daily、weekly 里选一种，不能叠用。开始时间按 UTC，实际在该小时内开始。例如设 12:00，可能在 12:00 到 12:59 之间。美国夏令时切换时，开始时间可能前移或后移一小时，即使你的本地时区不使用夏令时。小时频率的间隔是 1～23 的整数，该页建议用能整除 24 的数。

不设置保留策略时，自动快照会一直保留，直到你手动删除，存储费一直在。保留策略必须在创建时间表时设置，不能事后补到已经生成的快照上。设置了最长保留天数后，到期删除还要求当时已经有更新的一份快照；没有更新的一份，就继续留着。最短可以设 1 天。保留策略只作用于自动快照。

源盘删除规则默认是：源盘删除后，这个时间表已经生成的快照改为永久保留，原来的保留策略不再适用，除非你在创建时改成「删盘后仍应用保留策略」。从盘上解除时间表再删盘，原来的源盘删除规则仍然作用于已生成的快照。时间表挂在盘上时不能删除时间表，要先解除。控制台一次只能给一块盘挂一个时间表；gcloud 或 REST 一块盘最多 10 个。一个时间表最多挂 1000 块盘。每个区域最多 1000 个在用的时间表。CSEK 加密的盘不能做时间表。挂到 Windows 实例的盘，该页写用 `guest-flush` 让 VSS 生成卷影副本，细节在 Windows 应用一致性专页，这次没有打开。

创建时间表的向导页这次没有按点击打开。上面这些是属性页写明的规则。

### 恢复是新磁盘或新实例，自定义映像用来批量复制

[从快照恢复](https://docs.cloud.google.com/compute/docs/disks/restore-snapshot)（页上未见更新日期）：新磁盘至少要和快照的源盘一样大。更大时，还要按操作系统扩展文件系统，工具该页没有指定。给定快照创建新的区域级或地带级磁盘，该页写最多每十分钟一次；60 分钟内突发最多 6 次。创建区域级磁盘不受这句限制。这句和最佳实践页的「每个目标区域每 60 分钟 6 次」都在已打开的页面里，数字口径一个写「每十分钟一次」、一个写「每 60 分钟 6 次」，不合并成一个配额公式。

控制台建盘：Snapshots 页找到快照，再到 Disks，Create new disk，Source type 选 Snapshot，容量大于或等于源盘，Create。非启动盘可以再挂到实例上。新盘可以选与快照存储位置不同的区域和可用区。

启动盘快照可以直接用来创建实例：Create an instance，OS and storage，Change，Snapshots，选快照和启动盘类型、大小。要一次开出多台相同启动盘的实例，该页建议先做自定义映像，再从映像创建，而不是反复用快照。允许 HTTP 或 HTTPS 会给实例加上网络标记，并创建放行 `tcp:80` 或 `tcp:443` 入方向的防火墙规则。规则怎么收紧见 [安全组与防火墙规则](security-group-firewall-least-exposure.md)，这里不重复。

[创建自定义映像](https://docs.cloud.google.com/compute/docs/images/create-custom)（页上未见更新日期）开篇写明：要做磁盘数据的增量备份，用快照，而不是映像。映像可以从持久化磁盘（盘仍挂在 VM 上也可以）、持久化磁盘的快照、项目里的其他映像、其他项目共享来的映像，或 Cloud Storage 里的压缩 RAW 创建。创建映像同样受每 10 分钟一次、60 分钟最多 6 次的频率限制。

从磁盘做映像前，该页要求关掉磁盘的 auto-delete，否则删除 VM 时盘会一起被删。控制台：Create an image，Source 选磁盘、快照或其他映像。源盘挂在正在运行的 VM 上时，要勾选 Keep instance running。gcloud 从快照创建是 `gcloud compute images create IMAGE_NAME --source-snapshot=SOURCE_SNAPSHOT`。从正在运行的实例的磁盘创建时，默认不允许，要加 `--force`。不指定存储位置时，映像放在离源最近的多区域。

标准快照和归档快照的存储费、跨区域恢复的网络费，见账单文。同一区域创建标准快照是否另收网络费，以账单文所引的价格页为准，这里不重抄。

## 检查清单

按这个顺序做。每一步先确认账号在中国站、国际站，还是全球文档对应的环境。腾讯云国际站的定期快照策略页、AWS 中国区文档和世纪互联运营的 Azure 这次没有打开。

1. 高危变更前先有一份当前快照。阿里云手动创建页把回滚、改关键系统文件、换操作系统列成要事先做快照的场景，默认得到的是崩溃一致性。腾讯云创建页建议先关机，或至少 `sync` 并暂停写入；自建 MySQL 的示例是先全局读锁。AWS 根卷建议先停实例；做不到暂停写入时，先在实例里卸载再拍。Google Cloud 若不能接受崩溃一致性，按最佳实践页暂停应用，并等到快照进入 `UPLOADING` 再恢复。Azure 单盘快照页要求在用快照创建新 VM 之前彻底关闭源 VM。
2. 多块盘或 RAID 不要各拍各的。阿里云用快照一致性组，应用一致性只在未多重挂载的 ESSD 上、且控制台只能针对单台实例。腾讯云快照组要先确认账号在灰度内测里。AWS 用多卷快照，RAID 页写明这是 crash-consistent；Windows 再装 `AwsVssComponents` 走 VSS。Google Cloud 多盘应用一致性要手动冻结每块盘。Azure 要一台虚拟机上多块盘同一时刻的副本时，看还原点，不要把单盘 `New-AzSnapshot` 理解成已经包含了其余盘。
3. 核对一致性结果，而不是只看任务成功。阿里云看标签：中国站页面写 `APPConsistent:True` 或 `FsConsistent: True`，国际站页面写 `AppConsistent:True` 或 `FsConsistent:True`。脚本失败时你得到的是文件系统一致性，不是应用一致性。AWS 成功的 VSS 快照带 `AppConsistent:true`。Azure 还原点要看创建时有没有把 `consistencyMode` 设为 `CrashConsistent`；省略时该页的默认是应用程序一致性。
4. 恢复前再留一份「现在」的副本，并确认恢复方式。阿里云回滚页明确建议回滚前再做一份快照，然后停机，只回滚源盘。腾讯云回滚会清掉快照时刻之后的数据，已挂载必须关机，也不能回滚到新买的另一台实例的盘上。AWS 替换根卷会重启并清空内存；不勾选删除被替换根卷时，旧根卷还在计费。Google Cloud 是新磁盘，源盘还在。Azure 是新的托管磁盘或映像版本。
5. 系统盘和数据盘分开走。阿里云、腾讯云：系统盘快照做自定义镜像，数据盘快照做新盘（腾讯云使用限制写明系统盘快照不能新建云硬盘）。阿里云不选数据盘快照时，镜像附带的新数据盘默认 5 GiB 空盘。腾讯云中国站从实例做镜像时，有数据盘则默认整机镜像。AWS 从快照注册 AMI 时根卷不能小于快照，Windows 和部分商业 Linux 要核对 Platform details。Azure 映像里的数据磁盘不能超过 1 TB。Google Cloud 批量开实例时先把启动盘快照做成自定义映像。
6. 自动策略还在，就会继续产生快照。阿里云改保留时间不影响已有快照。腾讯云中国站关掉定期策略开关，已有快照仍按原规则留或删。Google Cloud 时间表没有保留策略时一直保留；删盘后的默认规则还会把已生成的自动快照改成永久保留。AWS 的 Data Lifecycle Manager 和 AWS Backup 这次没有打开操作页，账号里如果已经有策略，到对应服务里看有没有到期删除，不要只在 EC2 快照列表里找一个总开关。
7. 删除顺序不要反。阿里云快照设了自动释放时，先删由它创建的镜像，否则快照不会释放。删除快照一致性组会连组内快照一起删。腾讯云先删自定义镜像，再删它关联的快照；复制没完成不能删源快照。AWS 先注销 AMI，再删根卷快照；Backup 管理的快照到 Backup 里删。Google Cloud 先把时间表从盘上卸下，才能删时间表。归档快照删了不能恢复。
8. 删盘之后再看一次快照列表。阿里云没开「自动快照随云盘释放」时，自动快照还在。腾讯云释放云硬盘不删除快照。AWS 删卷不影响快照。Google Cloud 标准快照和归档快照还在，即时快照不在。Azure 快照独立于源盘。第二天的账单若还有这一项，按账单文对容量和策略，而不是只看磁盘列表已经空了。

## 常见坑

- 把各云的恢复都做成「停机然后回滚」。阿里云和腾讯云的回滚确实要求已挂载的盘先停机，并且只能覆盖源盘。AWS 替换根卷要求实例处于 running，并会自己重启。Azure 快照页和 Google Cloud 恢复页写的是创建新磁盘，源盘还在。
- 阿里云中国站和国际站的登录凭证说明不要并成一句。中国站回滚页写系统盘回滚后默认绑定实例当前的密钥或密码。国际站同一主题既写密码和 SSH 设置回到快照时刻，又写当前密钥、用户名和密码不受影响。操作前以你站点当前页的警告为准。
- 应用一致性没做成，任务却是成功的。阿里云脚本路径、权限或名称不对时，得到文件系统一致性快照，ExitCode 不一定是你以为的成功语义，要看标签。国际站和中国站的标签大小写在文档里就不一样。
- 腾讯云快照组、定期策略「关联实例」仍是灰度内测。菜单里没有，不代表文档写错，也不要把中国站菜单写进国际站。国际站 8191 这次是 404。
- 腾讯云系统盘快照去做「从快照创建云硬盘」。使用限制页写这条不支持，系统盘快照的出口是自定义镜像。数据盘快照才用来建新盘，容量不能小于源盘。
- AWS 取消了创建 AMI 时的重启，或在根卷替换时勾了删除原根卷。前者不保证文件系统完整。后者成功之后原根卷不能再找回。多卷快照里一块失败，整套其余快照会进入 `error`，不能假设另外几块仍然可用。
- Azure 把单盘快照和还原点当成同一个默认。单盘快照页没有 `consistencyMode`。还原点省略该字段时，默认是应用程序一致性，不是崩溃一致性。即时访问当前不支持门户。
- Google Cloud 删了磁盘，以为标准快照会跟着没。即时快照会没，标准快照和归档快照不会。时间表若保持默认源盘删除规则，删盘之后已有自动快照改为永久保留，原来的保留天数不再约束它们。
- 快照份数减少了，就以为费用等额下降。增量链里被其他快照引用的数据会留下。阿里云、AWS、Google Cloud 已打开的原理页都写了这件事。账单怎么对，见账单文。
- 用云盘快照代替托管数据库的时间点恢复。自建库可以在快照前按各云页面做读锁或刷盘。RDS、TencentDB、Azure Database、Cloud SQL 的备份保留和恢复不在这篇。

## 参考来源

资料核对日期：2026-11-23。下面只列出这次实际打开并据以写稿的页面。页上能看到更新日期的，写在标题后；看不见的，不补。

阿里云中国站：

- [快照概述](https://help.aliyun.com/zh/ecs/user-guide/snapshot-overview/)（页上「更新时间」栏为空）
- [手动创建单个快照](https://help.aliyun.com/zh/ecs/user-guide/create-a-snapshot)（页上未见更新日期）
- [使用快照回滚云盘](https://help.aliyun.com/zh/ecs/user-guide/roll-back-a-disk-by-using-a-snapshot)（页上未见更新日期）
- [创建自动快照策略](https://help.aliyun.com/zh/ecs/user-guide/create-an-automatic-snapshot-policy-1)（页上「更新时间」栏为空）
- [创建应用一致性快照](https://help.aliyun.com/zh/ecs/user-guide/create-application-consistent-snapshots-in-the-ecs-console/)（页上未见更新日期）
- [快照一致性组](https://help.aliyun.com/zh/ecs/user-guide/snapshot-consistency-group-overview/)（页上未见更新日期）
- [使用快照创建自定义镜像](https://help.aliyun.com/zh/ecs/user-guide/create-a-custom-image-from-a-snapshot)（页上未见更新日期）
- [快照原理](https://help.aliyun.com/zh/ecs/user-guide/how-do-snapshots-work)（页上未见更新日期）

没有打开中国站「使用快照创建云盘」的逐步页面，也没有打开「快照使用限制」配额专页。每块盘能保留多少份手动快照，中国站自动策略页只指向使用限制，正文不补数字。ESSD PL-X 不能做快照，来自概述页和手动创建页。

阿里云国际站：

- [Overview](https://www.alibabacloud.com/help/en/ecs/user-guide/snapshot-overview)（Last Updated: Oct 26, 2026）
- [Manually create a single snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/create-a-snapshot)（Last Updated: Oct 29, 2026）
- [Roll back a disk with a snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/roll-back-a-disk-by-using-a-snapshot)（Last Updated: Sep 21, 2026）
- [Create an automatic snapshot policy](https://www.alibabacloud.com/help/en/ecs/user-guide/create-an-automatic-snapshot-policy-1)（Last Updated: Oct 08, 2026）
- [Snapshot-consistent group overview](https://www.alibabacloud.com/help/en/ecs/user-guide/snapshot-consistency-group-overview)（Last Updated: Apr 27, 2026）
- [Create application-consistent snapshots](https://www.alibabacloud.com/help/en/ecs/user-guide/create-application-consistent-snapshots-in-the-ecs-console)（Last Updated: Aug 25, 2026）
- [Create a custom image from a snapshot](https://www.alibabacloud.com/help/en/ecs/user-guide/create-a-custom-image-from-a-snapshot)（Last Updated: Aug 24, 2026）

国际站创建页没有写 ESSD PL-X。自动策略页里的 2000 份只属于国际站正文。应用一致性标签以国际站页面的 `AppConsistent:True` 为准，不与中国站的 `APPConsistent:True` 合并。

腾讯云中国站：

- [创建快照](https://cloud.tencent.com/document/product/362/5755)（最近更新时间：2026-05-19 17:00:51）
- [从快照回滚数据](https://cloud.tencent.com/document/product/362/5756)（最近更新时间：2025-05-28 16:18:43）
- [创建镜像 API](https://cloud.tencent.com/document/product/213/16726)（最近更新时间：2025-09-19 11:38:37）
- [创建自定义镜像](https://cloud.tencent.com/document/product/213/4942)（最近更新时间：2026-06-09 15:48:01）
- [快照相关问题](https://cloud.tencent.com/document/product/362/17820)（页上未见更新日期）
- [快照使用问题](https://cloud.tencent.com/document/product/213/44127)（页上未见更新日期）
- [定期快照](https://cloud.tencent.com/document/product/362/8191)（最近更新时间：2026-08-14 15:03:00）
- [创建快照组](https://cloud.tencent.com/document/product/362/115828)（最近更新时间：2025-01-21 17:38:52）
- [使用限制](https://cloud.tencent.com/document/product/362/5145)（最近更新时间：2026-10-08 11:49:00）

没有打开「从快照创建云硬盘」和「删除快照」的逐步页面。系统盘快照不能建云硬盘、先删镜像再删快照，来自使用限制页和两篇问答。

腾讯云国际站：

- [Creating Snapshot](https://www.tencentcloud.com/document/product/362/5755)（Last updated: 2023-12-22 10:19:47）
- [Creating Snapshot（intl.cloud.tencent.com）](https://intl.cloud.tencent.com/document/product/362/5755)（Last updated: 2023-12-22 10:19:47）
- [Rolling back Data from Snapshot](https://www.tencentcloud.com/document/product/362/5756)（Last updated: 2023-12-22 10:20:44）
- [Snapshot](https://www.tencentcloud.com/document/product/362/17820)（Last updated: 2023-12-22 11:00:17）

`https://www.tencentcloud.com/document/product/362/8191` 与 `https://intl.cloud.tencent.com/document/product/362/8191` 均返回 404。国际站创建自定义镜像控制台、快照组、使用限制专页这次没有打开。

AWS：

- [Amazon EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)（页上未见更新日期）
- [Create Amazon EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html)（页上未见更新日期）
- [Create a snapshot of an EBS volume](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshot.html)（页上未见更新日期）
- [Create multi-volume EBS snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-create-snapshots.html)（页上未见更新日期）
- [Amazon EBS and RAID configuration](https://docs.aws.amazon.com/ebs/latest/userguide/raid-config.html)（页上未见更新日期）
- [Application consistent Windows VSS based Amazon EBS snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/application-consistent-snapshots.html)（页上未见更新日期）
- [Delete an Amazon EBS snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-snapshot.html)（页上未见更新日期）
- [Create an Amazon EBS-backed AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)（页上未见更新日期）
- [Replace the root volume](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html)（页上未见更新日期）

没有打开 docs.amazonaws.cn、从快照创建卷的专页、Amazon Data Lifecycle Manager 操作页、AWS Backup 操作页、Linux 应用一致性专页，以及回收站专页。crash-consistent 一词来自 RAID 页对多卷快照的描述。

Microsoft Azure：

- [创建 Azure 托管磁盘的快照](https://learn.microsoft.com/zh-cn/azure/virtual-machines/snapshot-copy-managed-disk)（页上未见更新日期）
- [使用虚拟机还原点](https://learn.microsoft.com/zh-cn/azure/virtual-machines/virtual-machines-create-restore-points)（页上未见更新日期）
- [创建映像定义和映像版本](https://learn.microsoft.com/zh-cn/azure/virtual-machines/image-version)（页上未见更新日期）

同一快照地址加上 `pivots=azure-portal` 后，读到的仍是 PowerShell 示例，没有门户菜单。没有打开英文页作为对照，没有打开从快照创建托管磁盘的专页、捕获 VM 映像专页，以及由世纪互联运营的 Azure 文档。

Google Cloud：

- [磁盘快照](https://docs.cloud.google.com/compute/docs/disks/snapshots)（页上未见更新日期）
- [创建快照](https://docs.cloud.google.com/compute/docs/disks/create-snapshots)（页上未见更新日期）
- [从快照恢复](https://docs.cloud.google.com/compute/docs/disks/restore-snapshot)（页上未见更新日期）
- [快照最佳实践](https://docs.cloud.google.com/compute/docs/disks/snapshot-best-practices)（页上未见更新日期）
- [快照时间表](https://docs.cloud.google.com/compute/docs/disks/about-snapshot-schedules)（页上未见更新日期）
- [创建自定义映像](https://docs.cloud.google.com/compute/docs/images/create-custom)（页上未见更新日期）

没有打开 machine image 操作页、Linux `guest-flush` 脚本专页、Windows VSS 快照专页，以及创建快照时间表的向导页。区域范围快照和回收站在已打开的页面里标为 Preview。

如需为阿里云、腾讯云、AWS、谷歌云开户或使用 USDT 充值，可访问自助站 <https://www.cloudcup.online>。海宝云官网：<https://www.haibaocloud.com>。Telegram：@cloudcup。充值币种、网络和最低金额以各站点当前说明为准。
