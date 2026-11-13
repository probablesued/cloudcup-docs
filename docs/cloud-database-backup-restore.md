# 云数据库自动备份、保留期与恢复演练

自动备份决定的是“还能回到哪一段时间”，不是“删掉实例以后数据还在”。时间点恢复（PITR）要同时有一份全量备份和那之后连续的日志。手动快照、按需备份、跨地域备份，保留规则常常和自动备份不是同一套。实例删掉之后，有的云会按你事先选的策略留下备份，有的只留几天，有的连同备份一起清掉。恢复演练应落到一台新实例上；这台新实例从创建成功起就可能开始计费，演练结束要连同它带出来的备份一起清理。

云服务器磁盘快照的保留和计费见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。谁可以删除实例、改备份策略，见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。账单用哪种币种出，见 [云账号充值与多币种付款核对](multi-currency-top-up-verification.md)。这三篇的内容这里不重复。本文不写单价。

核对时间：2026-11-13。资料核对日期：2026-11-13。保留天数、PITR 粒度、删除实例后的备份行为、控制台路径和命令参数，都来自这一天打开的官方页面，并附有链接。打不开或正文没有写明的规则，不补。

范围：阿里云只核对了 RDS MySQL（help.aliyun.com）。RDS SQL Server、PostgreSQL、MariaDB 的保留天数没有逐页打开。腾讯云只核对了中国站 TencentDB for MySQL（cloud.tencent.com）；国际站 www.tencentcloud.com 这次没有打开。AWS 使用 docs.aws.amazon.com 与 aws.amazon.com 的全球文档，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 learn.microsoft.com 的全球文档，产品页按 Azure Database for MySQL Flexible Server 和 Azure Database for PostgreSQL Flexible Server 核对；由世纪互联运营的 Azure 没有单独打开产品页。Google Cloud 只按 Cloud SQL for MySQL 核对；Cloud SQL for PostgreSQL、SQL Server 的备份页这次没有按同样深度打开。页面改版后，以你账号所属站点的当前文档为准。

## 先看这四件事

1. 自动备份的保留天数，是 PITR 窗口的上限。日志备份关着、日志链断了，或者全量备份比日志更早过期，实际能回到的时间会短于你填写的天数。
2. 手动快照或按需备份要单独看保留。有的一直留到你删除，有的跟着自动备份的保留期一起过期。
3. 跨地域备份和跨地域只读副本不是同一项。下文只写这次打开过的跨地域备份或备份复制。只读副本的延迟和计费，没有另开页面的，不补。
4. 删除或释放实例时，先看这一家写的是“备份一起删”“保留几天”还是“按删除前的策略留下”。演练恢复出来的新实例、最终快照和“删除后仍保留”的备份，都会继续占备份存储。

## 对照

下面只列这次打开的页面写明的默认值和范围。同一家如果有两句互相冲突的话，放在后面各节里，不提前合成一个数。

| 产品 | 自动备份保留 | 日志或 PITR | 手动或按需备份 | 删除实例之后 |
| --- | --- | --- | --- | --- |
| 阿里云 RDS MySQL | 数据备份默认 7 天，不能关。云盘 7～730 天。MySQL 5.7 基础系列固定 7 天 | 日志备份默认开，7～730 天，且不能长于数据备份。打开后可恢复到保留期内任意一秒 | 保留 7～730 天，或不指定时永久保留 | 释放后是否保留，看释放前的策略。保留的是数据备份，不含 Binlog |
| 腾讯云 MySQL（中国站） | 双节点、三节点、四节点、云盘版：数据备份 7～1830 天，默认 7 天。单节点（云盘）：7～30 天，默认 7 天 | 日志备份：前一类 7～3650 天，单节点（云盘）7～30 天，默认都是 7 天。回档到秒 | 手动备份会一直保存，直到你删除 | 销毁时可选不保留或保留。未保留则备份随实例销毁 |
| AWS RDS 实例 | 控制台创建默认 7 天；API 或 CLI 未指定时默认 1 天。可设 0～35 天，0 表示关闭自动备份 | 保留期内任意时间点。事务日志每 5 分钟上传一次 | 手动快照一直留到删除。每区域最多 100 个 | 可选保留自动备份，保留长度等于删除当时的保留期。手动快照不随实例删除 |
| Azure MySQL Flexible Server | 1～35 天，默认 7 天 | 保留期内的时间点。快照默认每天一次，事务日志每 5 分钟 | 按需备份，门户最多 50 个，保留期跟服务器的备份保留期走 | 概念页常见问题写删除后备份一并删除；同一篇和恢复已删除服务器页又写 5 天内可恢复 |
| Azure PostgreSQL Flexible Server | 7～35 天，默认 7 天 | 保留期内的时间点。RPO 一般最多约 5 分钟 | 按需备份最多 7 个，保留期跟备份窗口走。可单独删除 | 删除后备份保留 5 天，只能从原订阅恢复 |
| Cloud SQL for MySQL，标准备份 | 自动备份 1～365 天。Enterprise 默认 7 天，Enterprise Plus 默认 15 天 | Enterprise 日志 1～7 天、默认 7 天；Enterprise Plus 1～35 天、默认 14 天。gcloud 把 PITR 写成可到秒的小数 | 按需备份一直留到你删除，或留到实例被删除 | 未开启“删除后保留”、也没有最终备份时，标准备份说明写明会删掉该实例的备份 |

## 阿里云 RDS MySQL

中国站规则来自 help.aliyun.com 的 RDS MySQL 文档。国际站英文自动备份页这次没有逐段打开。国际站账号不要只按下面的中国站菜单操作。

### 自动备份和日志

[备份指引](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/backup-guide) 写明：数据备份默认开启，不能关闭，最少保留 7 天，频率最低每周 2 次。日志备份默认开启，可以关闭，最少保留 7 天。日志备份开着时，可以恢复到备份保留期限内、从第一个全量备份开始的任意时间点。[日志备份](https://help.aliyun.com/zh/rds/support/use-the-log-backup-feature) 把这个时间点写成任意一秒。

[全量备份](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/full-backup) 的保留范围：

| 参数 | 文档写明的范围 |
| --- | --- |
| 数据备份保留天数 | 默认 7 天。云盘实例 7～730 天。MySQL 5.7 基础系列固定 7 天，不能改。高性能本地盘为 7 天或以上；不超过 730 天的是常规备份，超过 730 天的转为归档备份 |
| 备份周期 | 最低每周 2 次。云盘实例开启高频快照后，最高每 15 分钟 1 次 |
| 日志备份保留天数 | 7～730 天，默认 7 天，必须小于或等于全量备份保留天数。MySQL 5.7 基础系列固定 7 天 |

只读实例只支持设置本地日志保留，不支持设置自动备份策略。备份期间不要做 DDL。表数量超过 5 万张时不能做库表恢复，整实例恢复不受这句限制；超过 60 万张时无法备份。修改备份策略会立刻触发一次全量备份。

[任意时间点保护](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/any-point-in-time-recovery-protection) 是日志备份的保留方式升级，不是另一种备份。传统按周期过期时，可恢复天数会随着全量备份的间隔变短。升级后，系统在你设置的日志保留天数之外，再留一个更早的全量备份和中间的连续日志，用来把可恢复窗口固定在你设置的天数。以日志保留 7 天为例，实际会留下大约 7～9 天的备份数据，但只对一个全量备份和最多额外一周日志计费。该功能在各地域陆续发布，高级备份策略（稀疏备份）页暂不支持这个开关。2024-01-11 及之后新建的实例可以按该页设置；更早的实例要从备份策略页的弹窗升级，升级后不能回退。

关闭日志备份后，该页写明系统会在 1～3 分钟内删除已有日志备份，删掉的不能恢复，按时间点恢复随之不可用。本地 Binlog 仍可能用 `SHOW BINARY LOGS` 看到，那是实例磁盘上的日志，不是备份空间里的日志备份。

按量付费实例在延期免停额度内（欠费 7 天内）仍会自动备份；超过 7 天后停服并停止自动备份。这句来自全量备份页的常见问题，和其他产品的欠费天数不要混用。

### 手动备份

全量备份页写明：手动备份的保留时长和自动备份策略无关，由你指定，支持 7～730 天和永久保留；不指定时默认永久保留。控制台入口在实例基本信息页右上角的「备份实例」。备份任务在任务中心按「手动备份实例」查看。完成后出现在备份恢复 > 基础备份列表。

[CreateBackup](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/api-rds-2014-08-15-createbackup-mysql) 把 MySQL 手动备份集的 `BackupRetentionPeriod` 写成 7～730 天，或 `-1`（长期保留）。实例须为运行中，且没有正在执行的备份。MySQL 引擎这一页写一天内可创建的备份集数量没有限制；同一页开头又写单个实例一天不超过 20 个，并注明 MySQL 无限制。以 MySQL 那句为准。

这次没有打开阿里云 CLI 的命令行参考，不写一条未核对的 `aliyun rds` 命令。改策略用的是 OpenAPI [ModifyBackupPolicy](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/api-rds-2014-08-15-modifybackuppolicy-mysql)：`BackupPolicyMode` 取 `DataBackupPolicy` 时，`BackupRetentionPeriod` 和 `PreferredBackupPeriod`、`PreferredBackupTime` 必传。数据备份保留天数参数表写 7～730。日志保留 `LogBackupRetentionPeriod` 同样是 7～730，且不大于数据备份保留天数。`ReleasedKeepPolicy` 的取值是 `None`（不保留）、`Lastest`（保留最后一个）、`All`（全部保留）。参数表上的拼写是 `Lastest`。2024-02-01 及之后新购的 RDS MySQL 云盘实例，该参数默认 `Lastest`；高性能本地盘默认 `None`。

同一页的错误码 `InvalidBackupRetentionPeriod.Malformed` 又把说明写成“备份周期不能小于 1 天或大于 30 天”。参数表和这条错误说明不一致。控制台能设置的范围以备份策略页为准。

### 跨地域备份

[跨地域备份](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/use-the-cross-region-backup-feature-of-an-apsaradb-rds-for-mysql-instance) 默认关闭。它把本地备份复制到另一个地域，不替换默认备份。保留天数 7～1825 天。跨地域日志备份是单独的开关。

这一页同时有两套说法，操作时对一下自己的存储类型：

- 对比表写：云盘实例释放后，已有跨地域备份不再保留，不再计费。高性能本地盘实例释放后，仍按原保留策略留下跨地域备份，继续产生存储费用，不再产生流量费用。
- 同一页后文又写：跨地域备份保留时间不受实例到期或释放影响，可在跨地域备份菜单看到未到期的备份。关闭跨地域备份的说明则把“实例释放后异地备份不再保留”标在云盘实例上，把“无论是否释放都保留到过期”标在高性能本地盘上。

云盘和高性能本地盘不要套用同一句。关闭跨地域备份不会立刻删掉已有备份，已有备份仍按关闭前的保留时间留到过期，到期前存储费用还在。想缩短这段时间，该页写把保留天数改成最短的 7 天，等过期清理。

### 释放实例以后

[实例释放后备份保留](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/configure-backup-retention-policies-for-released-instances)：

- 2024-02-01 及之后新购的 RDS MySQL 云盘实例，默认「保留最后一个」。
- 高性能本地盘新购实例默认「不保留」。购买页上可选不保留、保留最后一个、全部保留。
- 释放前在备份恢复 > 备份策略 > 基础备份里编辑「实例释放后保留备份文件」。
- 释放后在 RDS 控制台左侧备份管理 > 已删除实例备份里改策略。
- 保留下来的备份，该页写暂不支持指定保留时长，会按策略长期保存。释放后 7 天内存储免费，超过 7 天按量计费。不需要时把策略改为不保留。
- 这条策略只保留数据备份，不保留 Binlog。释放后的备份不能再做依赖 Binlog 的任意时间点恢复。
- 刚释放时，已删除实例备份页可能要等后台任务完成，通常 30 分钟内出现。
- 账户欠费持续超过 30 天，该页写明会终止已删除实例的备份保留，并永久删除这些备份。

[释放或退订](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/release-or-unsubscribe-from-an-instance) 还写了回收站：到期释放或手动退订的包年包月实例、手动释放的按量付费实例和 Serverless 实例，其中符合条件的会进入回收站，可解锁或重建。哪些实例符合条件，该页指向实例回收站。这次没有打开回收站页，不写回收站保留天数。没有进入回收站、也没有保留备份、跨地域备份也不在保留期内的，该页写明数据会被清除。

跨地域备份对比表里还有一句“默认备份在实例释放后默认保留 7 天”。这和上面“云盘默认保留最后一个、本地盘默认不保留”不是同一句。释放前以备份策略页上的「实例释放后保留备份文件」为准。

### 备份费用怎么看

备份放在备份空间，不占实例存储。备份使用量 = 数据备份 + 日志备份。查看位置在实例基本信息页的备份使用量。MySQL 基础系列或 Serverless 基础系列升级小版本后，这个数字可能暂时显示为 0，下一次定时备份完成后恢复。

[备份指引](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/backup-guide) 的免费额度只抵扣 730 天以内的常规备份，不抵扣归档备份：

| 存储类型 | 未开存储压缩 | 开启存储压缩 |
| --- | --- | --- |
| 高性能云盘 | 存储空间的 200% | 存储空间的 400% |
| 高性能本地盘 | 存储空间的 50% | 存储空间的 100% |

未超出免费额度不收备份费。超出部分按使用量、按小时计。单价在 RDS 定价页的「备份与恢复」，这里不抄。该页的杭州演算里有具体单价，只说明算法。

账单：在账单明细里按计费项和明细搜索实例名或实例 ID，找计费项为「实例备份」的行。同一页列出的计费项 Code 包括同地域基础备份 `BackupCharged`、跨地域备份存储、跨地域备份网络流量 `NetworkOutDuplicationSize`、备份下载外网流量，以及已删除实例备份保留。跨地域和已删除实例备份不要只在「实例备份」一行里找。

缩短保留天数后备份用量不变，通常是因为还没有超出新天数的备份被删掉。实例已经释放仍在扣备份费，先到已删除实例备份页看策略是不是「保留最后一个」或「全部保留」，并且是否已经过了 7 天免费期。

## 腾讯云 TencentDB for MySQL

以下来自中国站。国际站账号请再打开 www.tencentcloud.com，不要把下面的中文菜单直接套过去。

### 自动备份

[备份数据库](https://cloud.tencent.com/document/product/236/35172) 的路径：MySQL 控制台，实例 ID，备份恢复 > 自动备份设置。

自动备份不能关闭，也不能手动删除，到期自动删。每周至少备份 2 次。备份开始时间默认由系统在 0:00～12:00 分配，也可以自定义一段时间；写的是进入调度队列的时间，不是保证在这一分钟开始。单次任务最长不超过 24 小时。只读实例不支持数据库备份。

| 架构 | 数据备份保留 | 日志备份保留 |
| --- | --- | --- |
| 双节点、三节点、四节点、云盘版 | 7～1830 天，默认 7 天 | 7～3650 天，默认 7 天 |
| 单节点（云盘） | 7～30 天，默认 7 天 | 7～30 天，默认 7 天 |

回档范围由备份周期，加上数据备份和日志备份都还在的那段时间决定。该页的例子是：周期选周一和周四、保留 7 天，则可回档到 7 天内任意时刻。可恢复时间从当前往前、最远的一个有效物理备份或全量逻辑备份算起。缩短日志保留会缩短回档窗口。

单节点（云盘）和云盘版不支持「定期备份」。双节点、三节点、四节点可以另开定期保留：定期备份保留 90～3650 天，默认 1080 天，并且必须长于常规保留。关闭定期备份后，不再产生新的定期备份，旧的仍按原策略留到到期。

2019-02-26 起，自动备份默认是物理备份。需要逻辑备份时，用手动备份或 API。

### 回档和克隆

两件事不要混在一次演练里。

[回档](https://cloud.tencent.com/document/product/236/7276) 把库表写回源实例，原库表还在，回档结果是一套新名字的库表。它不支持整实例回档，也不支持只读实例和灾备实例。单节点云盘版不支持回档。时间可以设到秒。该页的规则是：设成 `2025-08-05 23:59:59` 时，这一秒的 `.000` 毫秒及其后的毫秒不在恢复范围内。回档方式分普通、快速、极速；快速和极速如果漏选了有跨库或跨表操作的对象，会失败。同一实例一次最多 500 个库或表。同一个 APPID 最多同时 5 个回档任务。包年包月实例未到期但账号欠费时，禁止回档，余额为正之后才能做。

[克隆](https://cloud.tencent.com/document/product/236/41207) 用备份和 binlog 生成一台新实例，不改源实例。按时间点克隆的范围由日志保留决定，按备份集克隆的范围由数据备份保留决定。支持的架构包括单节点、双节点、三节点、云盘版，所以单节点可以克隆，但不能按上一节回档。新实例规格和磁盘不能小于源实例，可用区、版本、复制方式和默认参数须与源实例相同。克隆过程中新实例不出现在列表里，成功后才计费。计费模式在克隆页选包年包月或按量。账户余额须大于 0。路径：备份恢复 > 数据备份列表，左上角克隆，或某一条备份的操作列。

季度演练用克隆到新实例。回档会把数据写回正在使用的实例。

### 跨地域备份

[跨地域备份](https://cloud.tencent.com/document/product/236/73100) 默认关闭。单节点云盘版不支持。路径：备份恢复 > 跨地域备份。

备份地域最少 1 个、最多 2 个，不能选源地域自己。保留时长默认 7 天，可设 3～1830 天。开启跨地域备份时，备份 binlog 会自动打开，之后可以单独关掉。本地自动备份完成后才转储到异地。开通时会把最近一次有效备份，以及从那次备份到当前的 binlog，同步到目标地域。再次开启后，该页写明开启时间点之前的异地备份不能再用于时间点恢复。

异地备份和 binlog 不能占用免费备份空间。关闭后不会立刻删除，已有文件按保留时长到期再删，到期前仍可能计费。源地域和目的地域的对照表在这一页，具体以控制台可选地域为准。

### 销毁实例以后

[销毁实例](https://cloud.tencent.com/document/product/236/30305)：包年包月退还后进入回收站 7 天，按量计费退还后进入回收站 24 小时。进入「隔离中」后实例不能访问，也不再产生与该实例相关的费用。只读实例会一起销毁。灾备实例会断开同步并升级为主实例。

销毁对话框里可以选择不保留，或保留。保留的范围是：全部、最近一个，或某个时间之后产生的备份。选「保留全部」或「保留时间段内的」时，可以再选是否保留 Binlog；「保留最近一个」默认不能选保留 Binlog。

[备份保留设置](https://cloud.tencent.com/document/product/236/118344) 的前提写成双节点、三节点，并且已经产生过备份。销毁页的对话框没有再写一遍这个架构限制。云盘版、四节点、单节点在销毁时能不能看到同一组选项，以控制台为准。

选择保留后，备份不会马上出现在数据库备份 > 备份列表 > 已销毁实例备份。延迟期间仍可在数据备份列表里看到。超过 24 小时还没有出现，该页写提交工单。

[备份空间收费说明](https://cloud.tencent.com/document/product/236/36263) 另有一套到期和欠费时间，和上面的“自己点销毁”不是同一条：

- 包年包月到期后 7 天内备份仍进行，超出赠送空间的部分仍计费。第 8 天起隔离进回收站，停止自动备份，禁止回档和手动备份，仍可下载。回收站再隔离 7 天后下线，数据备份一并销毁。
- 按量计费欠费 24 小时内备份仍进行。之后隔离，停止自动备份，禁止回档和手动备份，仍可下载。回收站隔离 7 天后下线并销毁备份。
- 包年包月未到期但账号欠费时，禁止回档、手动备份和下载，自动备份继续，超出赠送空间的部分仍计费。

自己点销毁后的 7 天或 24 小时，不要和到期、欠费后的隔离天数加在一起。选了保留备份的，下线后留下的是已销毁实例备份，不是回收站里的实例。

### 备份费用怎么看

备份空间按地域存放自动数据备份、手动数据备份和日志备份。

[备份空间收费说明](https://cloud.tencent.com/document/product/236/36263) 的免费额度：

- 云盘（单节点云盘、云盘版）：按实例，免费空间为该实例存储空间的 200%。在实例的备份恢复页查看。
- 本地盘：按地域，免费空间为该地域双节点和三节点主实例、灾备实例的存储空间之和。只读实例不赠送。在数据库备份页查看。落冷备份不能用这份赠送空间。

[计费概述](https://cloud.tencent.com/document/product/236/18335) 把本地盘免费额度写成双节点、三节点、四节点的存储空间之和。收费说明的公式没有写四节点。对账时以数据库备份页上显示的赠送空间为准。

超出后按量计费。收费说明和计费概述里有按地域列出的单价和演算，这里不抄。跨地域备份、已销毁实例备份、落冷后的标准存储，该页写明不占用赠送空间，单独计费。

费用中心里，备份费用的资源 ID 形如 `backup_mysql_<region>_<账号数字>`。该页的例子是 `backup_mysql_ap-shanghai_` 加一串账号数字。用这个资源 ID 对明细，不要只在实例 ID 上找。回收站里的实例，其备份仍计入备份空间。

## AWS RDS

本节是 Amazon RDS 数据库实例，不是 Aurora。PITR 页写明 Aurora 的恢复在集群文档里。CLI 也写明 `--backup-retention-period` 不适用于 Aurora，Aurora 的保留期在集群上改。这次没有打开 Aurora 的备份页。

全球商业区文档在 docs.aws.amazon.com。跨区域复制页的地域表里有中国（北京）和中国（宁夏）互为对方。AWS 中国区自己的文档在 docs.amazonaws.cn，这次没有打开，中国区账号的默认保留天数不要只凭全球文档推断。

### 自动备份和 PITR

[备份保留期](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.BackupRetention.html)：用 API 或 AWS CLI 创建实例且没有设置保留期时，默认 1 天。用控制台创建时，默认 7 天。实例可设 0～35 天。设为 0 会关闭自动备份。Multi-AZ DB 集群可设 1～35 天，不能设 0。从 0 改成非 0，或从非 0 改成 0，会发生中断。

实例处于 `stopped` 时不创建自动备份。保留期的计算不包含停留在 `stopped` 的时间，所以停机可能让已有备份留得比保留期更长。实例必须是 `available`，自动备份才会执行；`storage_full` 等状态不会做自动备份。同一区域正在复制快照时，也不会给同一个数据库做自动备份。见 [备份简介](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)。

[恢复到指定时间](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html)：可以恢复到保留期内的任意时间点，并且会创建一台新实例，不修改源实例。事务日志每 5 分钟上传到 Amazon S3 一次。最近可恢复时间因此会落后于当前时间，看 CLI `describe-db-instances` 的 `LatestRestorableTime`，或控制台的 Automated backups。最早可恢复时间也在 Automated backups 里。这一页没有把可选的恢复时间写成只能落在 5 分钟整点上。控制台路径：打开 RDS 控制台 <https://console.aws.amazon.com/rds/>，导航窗格选 Automated backups，选中实例，Actions，Restore to point in time。时间按你本地时区显示，页面会标出相对 UTC 的偏移。

恢复后的存储卷会在后台从 S3 继续加载数据。此期间实例可用，性能可能还没到正常水平。`DescribeDBInstances` 里 `StorageOperationStatus` 为 `Initializing` 表示仍在加载，这两个字段消失表示加载完成。

用 CLI 修改保留期。两个非 0 值之间的修改会尽快异步生效。从 0 改为非 0 或反过来，除非带 `--apply-immediately`，否则放到下一个维护窗口；开启或关闭备份时会有几秒到几分钟的 I/O 暂停。见 [modify-db-instance](https://docs.aws.amazon.com/cli/latest/reference/rds/modify-db-instance.html)。实例如果是只读副本的源，不能把保留期设为 0。

```bash
aws rds modify-db-instance \
  --db-instance-identifier <db-instance-id> \
  --backup-retention-period 7
```

恢复到新实例。`--restore-time` 用 UTC 的 ISO8601 时间，下面的时间只说明格式：

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier <source-db-instance-id> \
  --target-db-instance-identifier <new-db-instance-id> \
  --restore-time 2026-11-13T00:00:00Z
```

也可以改用 `--use-latest-restorable-time`。恢复不能把存储改小。加大存储时至少增加 10%，RDS for SQL Server 在恢复时不能加大存储。SQL Server 把库设成 `OFFLINE`、`EMERGENCY` 或 `SINGLE_USER`，会使整个实例的最近可恢复时间停止前移。

实例如果已经关联 AWS Backup 的备份计划，PITR 使用那份计划。这类备份的名称以 `awsbackup:` 加作业号结尾。

### 手动快照

手动快照一直保留到你删除它。每区域最多 100 个，这个上限不计算自动备份。见备份简介。

```bash
aws rds create-db-snapshot \
  --db-instance-identifier <db-instance-id> \
  --db-snapshot-identifier <snapshot-id>
```

源实例须为 `available` 或 `storage-optimization`。见 [create-db-snapshot](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-snapshot.html)。快照标识 1～255 个字母、数字或连字符，以字母开头，不能以连字符结尾，也不能连续两个连字符。

### 跨区域复制自动备份

[复制自动备份到另一个区域](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html)：开启后，快照和事务日志一就绪就复制到你选的目的区域。支持 RDS for Db2、MariaDB、MySQL、Oracle、PostgreSQL、SQL Server 的 Multi-AZ 实例部署。不支持 Multi-AZ DB 集群。每个账户默认最多 20 份跨区域自动备份。

源区域和目的区域的对照表在这一页，也可以用 `describe-source-regions` 查。复制按快照复制收取数据传输费，复制完成后在目的区域按备份存储计费。单价不抄。

这和跨区域只读副本不是同一项。只读副本的页面这次没有打开。

### 删除实例以后

[删除数据库实例](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DeleteInstance.html)：

- 可以选是否做最终快照。最终快照和以前的手动快照都会留下。`creating`、`deleting`、`failed` 以及该页列出的几种 `incompatible-*`、`insufficient-capacity`、`restore-error`、`upgrade-failed` 状态下不能做最终快照。
- 可以选是否保留自动备份。保留时，留下的长度等于删除当时实例上的保留期。保留下来的系统快照和事务日志会按原规则过期，没有新的快照和日志产生，最后一份系统快照过期后，整组保留备份被系统删除。最终快照不会过期。见 [保留自动备份](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.Retaining.html)。
- 不保留时，同区域的自动备份被删除，不能恢复。已经复制到其他区域的自动备份仍会留下。
- 手动快照不随实例删除。
- 每个区域最多 40 份保留的自动备份，不占数据库实例配额。保留备份里没有参数组或选项组的信息。不能修改一份已经保留的自动备份。

保留的自动备份和手动快照在删除前都会计费。演练实例如果点了保留自动备份或创建了最终快照，删掉实例并不会停掉这笔备份费。

CLI 里 `--no-delete-automated-backups` 表示保留自动备份，`--skip-final-snapshot` 表示不创建最终快照。删除页给了组合示例。本文不给删除命令，避免把演练命令复制到生产实例上。清理演练实例时，在控制台确认没有勾选保留自动备份、没有留下最终快照，再单独删除演练期间创建的手动快照。

### 备份费用怎么看

[备份简介](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html) 写明：每个区域的备份存储是该区域自动备份和手动快照之和。把快照复制到另一个区域，增加的是目的区域的备份存储。

同一页的免费额度：每个账户、每个区域，免费备份存储等于该账户在该区域已预置数据库存储的 100%。额度按该区域全部快照存储的比例，分摊成每个快照上的折扣，不是从某一个快照里扣掉一块固定容量。该页用 100 GB 预置存储、两份快照合计 125 GB 做了比例演算，演算是算法说明，不是价格。

[RDS 常见问题](https://aws.amazon.com/rds/faqs/) 用另一句说明同一件事：免费备份存储等于该账户在该区域的已预置数据库存储合计，只对超出的备份存储收费，按天折算。常见问题同时写明，备份存储大小不在 RDS 控制台显示，也不在 `DescribeDBSnapshots` 的返回里。实例停机后，仍对预置存储和备份存储收费，不对数据库实例小时收费。

用户指南写，快照存储费在 Cost Explorer 和成本与使用情况报告（CUR）里按快照计。CUR 的 `ResourceId` 是快照 ARN 或集群快照 ARN，不是数据库实例 ARN。源实例已经删除的，费用记在仍产生费用的那份快照 ARN 上。按实例 ARN 过滤的预算或成本类别会匹配不到这些行。要按实例汇总，给快照加成本分配标签。

因此，控制台里看不到“备份已用 GB”是这一页写明的情况。对账去 CUR 或 Cost Explorer，按快照 ARN 看，再和保留期、手动快照、跨区域副本对上。

## Azure Database for MySQL 与 PostgreSQL Flexible Server

两份产品都核对了。保留天数的下限不同，删除后的写法也不同。不要把 MySQL 的 1 天下限套到 PostgreSQL 上。

由世纪互联运营的 Azure 没有单独打开产品页。[恢复已删除的 MySQL 服务器](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/how-to-restore-dropped-server) 列出了世纪互联的资源管理器地址 `https://management.chinacloudapi.cn`。这只说明调用哪个地址，不表示下面的保留天数已经在世纪互联文档里核对过。

### MySQL Flexible Server

[备份和还原](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/concepts-backup-restore)：自动备份默认开启，保留期 1～35 天，默认 7 天。第一份快照在服务器创建后马上安排，之后默认每天一次。事务日志每 5 分钟备份一次。备份窗口由 Azure 管理，不能自定义。不能把这些备份文件导出，只能在这项服务里还原。

备份间隔可以改成 12 小时或 6 小时，默认 24 小时。路径：Settings > Compute + Storage 里的 Backup Interval。只支持 General Purpose 和 Business Critical。Burstable 不支持。同一页的冗余选项又把层级写成 Basic、General Purpose、Memory-Optimized。两套层级名称都在这一页上，以门户里的名称为准。

按需备份跟着服务器的保留期走，不是永久保留。门户里最多触发 50 个。保留 7 天时，该页写快照和事务日志会存到最近 8 天（窗口再往前 1 天）。

冗余：同区域高可用或没有高可用时，默认本地冗余；区域冗余高可用使用区域冗余备份存储。异地冗余可以在创建时开启。创建之后，可以从本地冗余改成异地冗余；不能在创建之后用 Compute + Storage 把区域冗余备份直接改成异地冗余，该页给出的办法是 PITR 到新服务器，或新建一台再迁移。异地冗余服务器的备份存储用量是本地冗余的两倍。

还原总是创建一台新服务器。可以选最近还原点、保留期内的自定义还原点，或某一天最快的全量备份还原点。最快还原点只恢复全量快照，不再重放日志。新服务器继承计算层、vCore、存储大小、备份保留期和备份冗余，也继承虚拟网络和防火墙等设置。`time_zone` 会回到 `SYSTEM`。还原完成后的高可用服务器会变成非高可用。

异地还原只在开启了异地冗余时可用。可以还原到配对区域，或除 `Brazil South`、`USGov Virginia`、`West US 3` 以外、Flexible Server 可用的其他区域。复制有延迟，最多可能丢失 1 小时数据。异地还原用的是已复制过去的最新备份，不是源区域上任意一秒。

长期保留（用 Azure Backup 保留最多 10 年）这一页标成预览且已暂停，写明在另行通知前不要配置新的长期备份。已有备份数据该页写仍然可以还原。本文不把这套暂停中的预览写成当前的操作步骤。

删除服务器：同一页的常见问题写，删除服务器后属于它的全部备份一并删除，不能恢复。正文的 Important 又写，删除后 5 天内可以恢复，并指向恢复已删除服务器。恢复页写：备份最多在服务里保留 5 天，只能从原来的订阅恢复；资源组如果已经不在，要先建一个同名资源组。步骤是从 Monitor 的活动日志里找出删除事件，用删除时间减去 15 分钟作为 `restorePointInTime`，再调用创建服务器的 REST。两处都在 2026-11-13 打开的页面上。误删后先按恢复页查这 5 天里备份是否还在，不要只根据常见问题那句就认为已经没有备份。

服务器停止后不再做新备份。停止当时还在保留窗口内的旧备份会留到重新启动。重新启动后，仍按保留窗口管理。停止期间仍对预置存储和保留窗口内的备份存储收费。免费备份空间只适用于运行中的服务器。

CLI 参考页写明 `--backup-retention` 是 1～35 天，默认 7。`--backup-interval` 接受 24、12、6，默认 24。见 [az mysql flexible-server](https://learn.microsoft.com/en-us/cli/azure/mysql/flexible-server)。

```bash
az mysql flexible-server update \
  --resource-group <resource-group> \
  --name <server-name> \
  --backup-retention 7

az mysql flexible-server restore \
  --resource-group <resource-group> \
  --name <new-server> \
  --source-server <source-server> \
  --restore-time 2026-11-13T00:00:00Z
```

`--restore-time` 是 UTC 的 ISO8601 时间。不传时，该参考写默认用运行命令的机器上的当前时间。参考页的命令列表里有 `az mysql flexible-server backup create`，这次打开的正文没有展开它的参数，不补参数。

### PostgreSQL Flexible Server

[备份和还原](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore)：保留期 7～35 天，默认 7 天。下限是 7，不是 MySQL 的 1。停止的服务器也会保留已有备份。第一份快照在创建后马上安排，目前每天一次；如果上次快照之后没有任何数据库被修改，快照会暂停，一旦有修改就再拍一份。事务日志在 WAL 可以归档时备份。该页写 RPO 的延迟一般最多约 5 分钟。

异地冗余只能在创建服务器时选择，创建后不能改备份存储冗余。异地还原不是 PITR：目前不能对异地冗余备份做时间点还原，只能还原到配对区域里已经复制过去的最新数据。创建后至少等 1 小时再做第一次异地还原。之后的延迟最多约 1 小时。主区域故障时，配对区域里不能新建异地冗余服务器，因为存储要在主区域预配；仍可以把源服务器异地还原到配对区域。

PITR 创建同区域的新服务器，可以选择可用区，不覆盖原服务器。可选最近还原点、自定义还原点，或备份列表里的快速还原点。公网访问的服务器只能还原成公网访问；专用虚拟网络的服务器可以还原到同一虚拟网络或另一个虚拟网络，不能改成公网。还原后，防火墙、专用终结点和虚拟网络规则不会从原服务器复制过来，参数也不会自动带过去。这和 MySQL 那篇“继承防火墙和虚拟网络”不同。

按需备份：在自动备份之外再拍一份，保留时长仍由备份窗口决定，用户可以单独删除按需备份。自动备份由服务按保留期管理。最多 7 个。Burstable 计算层和 SSDv2 存储不支持按需备份。

长期保留由 Azure Backup 提供，最多 10 年，和最多 35 天的自动备份可以同时存在。还原目前是 Restore as Files 到存储账户。这套步骤这次没有逐页打开，不写点击顺序。

[恢复已删除的服务器](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/how-to-restore-deleted-server)：删除后备份保留 5 天，只能从原来的订阅访问和恢复。超过 5 天会因为找不到备份而失败。

CLI 的 `--backup-retention` 范围是 7～35 天，默认 7。见 [az postgres flexible-server](https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server)。

```bash
az postgres flexible-server update \
  --resource-group <resource-group> \
  --name <server-name> \
  --backup-retention 7

az postgres flexible-server restore \
  --resource-group <resource-group> \
  --name <new-server> \
  --source-server <source-server> \
  --restore-time 2026-11-13T00:00:00Z
```

不传 `--restore-time` 时，[还原到最新还原点](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/how-to-restore-latest-restore-point) 写明默认是运行命令的系统当前时间。参考页列出 `az postgres flexible-server backup create`，正文没有展开参数，不补。

### 备份费用怎么看

两篇概念页都写：备份存储在不超过已预置服务器存储的 100% 时不另收费，超出部分按 GB·月计。MySQL 的例子是预置 250 GB 则有 250 GB 备份存储不另收费；每天备份用量 25 GB 时，大约可以有 10 天落在这份额度里。这是页面上的算法例子。PostgreSQL 用 GiB 写了同一结构的例子。异地冗余时，PostgreSQL 页写备份大小按本地副本的两倍计算，计费是本地备份的两倍减去已预置存储。单价在定价页，不抄。

查看用量：Azure 门户的监视指标。MySQL 用 Backup Storage used，PostgreSQL 用 Backup Storage Used。它是保留期内数据库备份和日志备份占用的合计。事务很多时，备份存储会随日志变大，不随数据库文件大小封顶。

MySQL 停止后，免费备份空间不再按运行中服务器的规则适用，停止期间备份存储仍收费。PostgreSQL 页写停止的服务器仍保留备份；停止期间是否继续拍新快照，这一页没有写成和 MySQL 相同的“不再做新备份”，这里不补。

## Google Cloud SQL for MySQL

备份选项页和备份概览的最后更新时间都是 2026-11-12 UTC。Cloud SQL for PostgreSQL、SQL Server 没有按本节的深度打开。

每个实例选一种备份选项。2026-11-13 打开的 [选择备份选项](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/backup-options) 写明：增强备份用 Backup and DR，备份放在备份保管库；标准备份由 Cloud SQL 管理，放在实例所在项目。增强备份推出之前创建的实例使用标准备份。一个实例不能同时用两种。

### 标准备份

自动备份保留 1～365 天。Enterprise 默认 7 天，Enterprise Plus 默认 15 天。按需备份一直保留，直到你删除它，或直到它所属的实例被删除。

如果开启“实例删除后保留备份”，自动备份仍按 1～365 天滚动删除，按需备份则无限期保留，直到手动删除或项目被删除。见备份选项页，细节在 [备份概览](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/backups) 的 Retain backups after instance deletion。

没有指定备份位置时，备份放在离实例最近的多区域。该页的例子：实例在 `us-central1`，默认备份在 `us` 多区域。自定义位置可以选其他区域或多区域，用来满足数据驻留。改位置之后，已有备份留在原位置。

最终备份默认保留 30 天。标准备份可把最终备份的保留设为 1～365 天；增强备份可设为 1 天到 10 年。最终备份只适用于主实例，不适用于副本。可以在实例设置里事先开启，也可以在删除时再选。删除时的选择要和实例上已经开启的最终备份设置一致，否则删除会失败；把删除对话框里的最终备份留空，则沿用实例设置。

备份频率限制：每个实例、每个项目，每 50 分钟最多 5 次备份操作。失败的不计入。令牌桶最多 5 个，每 10 分钟补充 1 个。

不开启删除后保留、也不做最终备份时，标准备份一节写明 Cloud SQL 会自动删除该实例的全部备份。概览另有一句：为了在误删后连同 IP 一起恢复，Cloud SQL 会把已删除实例的备份保留 4 天；要恢复到删除当时的状态并保留 IP，须在 4 天内联系 Google Cloud Customer Care。从备份还原不会保留已删除实例的 IP。两句都在概览和选项页里。4 天是联系支持的窗口；它不代替你自己打开的“删除后保留”或最终备份。

### PITR

[配置 PITR](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/configure-pitr)：PITR 依赖自动备份。在已有主实例上开启 PITR，实例会重启。

默认是否开启，取决于创建方式：

- Enterprise Plus：不论用控制台、gcloud、Terraform 还是 API 创建，PITR 默认开启。要关须自己关。
- Enterprise：在控制台创建时 PITR 默认开启。启用了高可用的 Cloud SQL for MySQL 也默认开启。用 gcloud、Terraform 或 Admin API 创建时，PITR 默认关闭，要自己开。
- 自动备份同样：控制台创建默认开启；gcloud、Terraform、API 创建时要自己开。

日志保留必须小于或等于备份保留。同一页先写“必须小于”，紧接着的 Caution 又写“必须小于或等于”。日志每天清理一次，不是连续清理。日志保留天数和备份份数设成同一个数时，实际留下的日志会短于这个数。该页的例子：日志 7 天、备份 7 份，实际保留 6～7 天的日志。建议备份保留比日志保留多 1。

Enterprise 的日志保留是 1～7 天，默认 7 天。Enterprise Plus 是 1～35 天，默认 14 天。gcloud 的 `--retained-transaction-log-days` 把总范围写成 1～35，并注明 Enterprise 的默认和上限都是 7，Enterprise Plus 的默认是 14、上限是 35。见 [gcloud sql instances create](https://docs.cloud.google.com/sdk/gcloud/reference/sql/instances/create)。`--enable-bin-log` 和 `--enable-point-in-time-recovery` 把恢复精度写成可以到秒的小数。产品页写的是恢复到指定时间点。

```bash
gcloud sql instances patch <instance-id> \
  --enable-bin-log \
  --retained-transaction-log-days=7

gcloud sql backups create \
  --instance=<instance-id> \
  --description=drill
```

`--backup-start-time` 用 UTC 的 24 小时制。按需备份的 `--location` 默认是离你最近的多区域，只在需要时再指定。见 [gcloud sql backups create](https://docs.cloud.google.com/sdk/gcloud/reference/sql/backups/create)。`--retain-backups-on-delete` 出现在 `gcloud sql instances create` 的参数里。这次没有打开 `gcloud sql instances patch` 的参考页，不把这个旗标写成 patch 子命令上已经核对过的参数。

[还原概览](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/restore)：用备份还原可以还原到已有实例，也可以还原到新实例。新实例可以在另一个区域或另一个项目，CPU 和内存也可以不同。PITR 一定创建新实例，不能 PITR 到已有实例。把备份还原到已有实例时，已有的归档日志会被删掉，还原完成之后新产生的日志才能再用于 PITR。演练不要把备份还原到生产实例上。

每个实例、每个区域、每个项目，每 30 分钟最多 3 次还原。失败的不计入。

二进制日志的存放位置分磁盘和 Cloud Storage。2023-08-11 之后新建且开启 PITR 的 Enterprise，以及 Enterprise Plus，日志放在 Cloud Storage，和主实例同一区域。放在 Cloud Storage 的这些日志，还原页写明不按实例另收一笔费用，保留上限分别是 Enterprise Plus 35 天、Enterprise 7 天；实际保留天数仍由 `transactionLogRetentionDays` 决定。更早创建、日志仍在磁盘上的实例，日志占实例磁盘。从磁盘迁到 Cloud Storage 时，同一页一处写会有几分钟停机并丢掉已有 PITR 日志，另一处写可以用 gcloud 或 API 在不停机的情况下切换。关闭再打开 PITR 会删除已有日志，并且要重启。磁盘上的日志删掉以后，已经预配的磁盘容量不会跟着变小。

副本没有自己的备份。副本提升成独立实例之后，要单独配备份，不能使用原主实例的备份。

### 增强备份

增强备份的自动计划可以是小时、天、周、月或年。按需备份的保留由备份计划设置，最长 10 年（3653 天）。备份保管库的最短强制保留如果超过 10 年，Cloud SQL 实例不能选用该保管库。

删除实例后，实例存活期间产生的备份会自动留下，并继续按删除前的备份计划过期。这和标准备份“要自己打开删除后保留”不同。最终备份的保留也可以设到 10 年。

跨区域：备份保管库可以建在另一个区域，备份计划必须和实例在同一区域，并且和保管库在同一项目。用跨区域备份做 PITR 时，只能还原到源实例所在区域的新实例。实例使用客户管理的加密密钥时，备份用的是实例那把密钥；实例所在区域不可用时，不能还原这些跨区域备份。增强备份不能和灾难恢复副本同时使用。

费用按保管库里的备份大小，由 Backup and DR 计。保管库在另一个区域时，另有区域间数据传输费。单价见 Backup and DR 定价，不抄。

### 删除以后备份还在不在

标准备份要事先打开删除后保留，或在删除时做最终备份，实例删除后才能按对应规则留下备份。打开删除后保留时，自动备份按删除前的保留设置滚动删除。概览的例子：自动备份保留设为 7，则最新一份自动备份在实例删除 7 天后删除。按需备份在这种设置下无限期保留，直到手动删除或项目删除。保留期在实例删除后不能再改。

增强备份在删除实例后自动保留存活期间的备份，规则见上一节。

概览还写了 4 天的 Customer Care 窗口，见标准备份一节。没有最终备份、也没有可保留的备份时，故障排查表写明删除之后不能再给这个实例做备份；如果之前做过导出，可以新建实例再导入。导出是 Cloud Storage 里的文件，生命周期由你自己管，和 Cloud SQL 备份不是同一项。

删除后仍在计费的，是保留下来的自动备份、按需备份和最终备份。单份备份的大小就是该份备份的计费大小。见备份概览。

### 备份费用怎么看

[Cloud SQL 定价](https://cloud.google.com/sql/pricing) 有一行 Backups (used)，按 GiB·小时计，金额随页面上选择的区域变化。这一页的表还分单区域和 HA。这里不抄表里的金额。

标准备份的费用由备份总大小、存放位置和保留设置决定。增强备份走 Backup and DR。实例磁盘上的二进制日志占的是实例存储，不是这一行 Backups。放在 Cloud Storage、供 PITR 使用的日志，还原页写明不按实例另收费。

控制台里看某一份备份的大小。结算上的明细在结算账号的费用表里按 Cloud SQL 或 Backup and DR 筛选。预算到了阈值不会自动停备份，这点已写在 [CDN 费用配置与防超支](cdn-cost-configuration.md)，这里不重复预算的做法。

## 备份策略检查清单

每个实例做一遍。菜单以你登录的站点为准。

1. 引擎和架构先对上。阿里云本节的天数是 RDS MySQL。腾讯云单节点（云盘）的保留上限是 30 天，并且不能回档、不能做跨地域备份。AWS 的 0～35 天是 RDS 实例，不是 Aurora。Azure PostgreSQL 的保留从 7 天起，MySQL 从 1 天起。Cloud SQL 用 gcloud 或 API 创建的 Enterprise 实例，PITR 和自动备份默认是关的。
2. 自动备份开着，保留天数覆盖你需要回退的最远时间。阿里云、腾讯云每周至少 2 次。阿里云日志保留不能长于数据备份保留。Cloud SQL 的日志保留不要大于备份保留；天数和份数设成同一个数时，实际日志会短一截。
3. PITR 真的可用。阿里云日志备份是开的；升级了任意时间点保护的，可恢复窗口应按设置的天数理解。腾讯云回档依赖数据备份和 binlog 都还在。AWS 看 `LatestRestorableTime`。事务日志每 5 分钟上传一次，最近可恢复时间会落后于当前时间。Azure 看 Backup and Restore 里列出的成功全量备份，失败的备份不会出现在列表里。Cloud SQL 在实例上确认 binary log 或 PITR 已启用。
4. 手动备份的保留是你以为的那样。阿里云不指定则永久保留。腾讯云手动备份会一直保存。AWS 手动快照不随实例删除。Azure 的按需备份跟着服务器保留期走，MySQL 门户最多 50 个，PostgreSQL 最多 7 个。Cloud SQL 标准按需备份在实例删除时会被删掉，除非开了删除后保留。
5. 跨地域备份是否开启、保留多久、关了以后旧文件是否还在计费。阿里云云盘和本地盘在实例释放后的跨地域备份行为不同。腾讯云异地备份不占免费空间。AWS 不保留同区域自动备份时，其他区域的复制备份仍会留下。Azure MySQL 可以在创建后把本地冗余改成异地冗余；Azure PostgreSQL 的异地冗余创建后不能改，异地还原也不是 PITR。Cloud SQL 增强备份的跨区域 PITR 只能回到源区域的新实例。
6. 删除或释放时的默认策略已经看过。阿里云云盘新实例默认保留最后一个数据备份，不含 Binlog；本地盘默认不保留。腾讯云销毁对话框里的“不保留”会在下线后清掉云上备份。AWS 要同时看最终快照和“保留自动备份”。Azure 按 5 天恢复窗口理解，并记住 MySQL 常见问题有一句相反的话。Cloud SQL 标准备份要自己打开删除后保留或最终备份。
7. 知道费用出现在哪一页。阿里云看基本信息页的备份使用量和账单里的「实例备份」，跨地域、已删除实例另有计费项 Code。腾讯云看数据库备份页或实例备份恢复页的赠送空间，费用中心用 `backup_mysql_` 开头的资源 ID。AWS 的备份大小不在 RDS 控制台显示，CUR 里按快照 ARN 计。Azure 看 Backup Storage used。Cloud SQL 看单份备份大小和结算里的 Backups 行。
8. 有权限的人才能做恢复和删除。最小权限怎么收，见账号那篇。演练使用单独的身份，不把主账号、根用户或管理员密钥写进命令、工单和仓库。

## 每季度做一次恢复演练

目标是证明“在保留期内的某个时间点，能在新实例上读到当时的数据”，然后把演练实例清掉，避免它和它的备份继续计费。不要把回档、还原做到生产实例上。

1. 记下源实例的标识、区域、引擎版本，以及这一刻的最早、最近可恢复时间。阿里云在备份恢复里看可恢复范围。腾讯云在自动备份设置里确认数据和日志保留都覆盖这一季要演练的时间点。AWS 看 Automated backups 的最早和最近可恢复时间。Azure 看 Backup and Restore 列表。Cloud SQL 看实例的备份和 PITR 配置。
2. 选一个已经过去、并且落在可恢复窗口内的时间。用业务自己能核对的标记：某张表的行数、某条当日写入的业务单号，或应用写下来的校验和。标记不要包含密码、密钥或个人身份信息。
3. 恢复到新实例，名字用演练前缀，例如 `drill-` 加日期。腾讯云用克隆，不要用回档。AWS、Azure、Cloud SQL 的 PITR 都会新建实例。Cloud SQL 如果用的是“把备份还原到已有实例”，不要选生产实例。
4. 新实例就绪后，用只读账号连接演练实例，核对第 2 步的标记。对不上就停，不要改应用的连接串。Azure PostgreSQL 还原后防火墙和参数不会自动带上，连不上时先看网络规则，而不是先判断备份损坏。AWS 在存储初始化完成前，查询会比平时慢。
5. 这一季如果只演练、不切换：记下核对结果，然后删除演练实例。删除时按这一家的选项处理最终快照和“保留备份”。AWS 不要勾选保留自动备份，并删掉演练时创建的手动快照。阿里云如果演练实例会进入回收站或默认保留最后一个备份，到已删除实例备份里改成不保留。腾讯云销毁时选不保留，并确认已销毁实例备份里没有这台演练实例。Azure 删除后 5 天内备份仍可能在，确认不再需要即可；不要为了演练去调用恢复已删除服务器的接口。Cloud SQL 标准备份确认没有打开删除后保留，删除对话框里也不要做最终备份；如果实例设置强制最终备份，演练结束后到已删除实例的备份列表里删掉它。
6. 这一季如果演练包含切换：先在新实例上完成第 4 步，再改应用配置里的主机名。旧实例先停止写入，保留到新实例做出一次成功的自动备份之后再释放。切换不是把旧实例的连接名自动交给新实例。Cloud SQL 从备份还原不会保留被删实例的 IP。
7. 第二天看账单或用量。演练实例的计算、存储和备份应已停止新增。阿里云看已删除实例备份和跨地域备份。腾讯云看 `backup_mysql_` 资源是否仍在增长。AWS 在 CUR 里搜演练快照的 ARN。Azure 看 Backup Storage used 是否还挂在已删除的服务器上。Cloud SQL 看项目级保留备份和最终备份。仍在计费就按上面的清理入口处理。

演练记录只留时间点、实例标识、核对是否通过、清理是否完成。不要留下主账号密码、连接串里的口令或访问密钥。

## 常见坑

- 把磁盘快照当成数据库的 PITR。云盘或 EBS 快照的保留写在云服务器那篇。数据库的可恢复时间由这一篇的日志备份决定。
- 阿里云关掉日志备份后，1～3 分钟内历史日志备份会被删掉，任意时间点恢复就没了。MySQL 5.7 基础系列的保留天数固定为 7，改不了。
- 阿里云释放后如果保留了数据备份，Binlog 不在其中，不能按任意秒再恢复。云盘实例的跨地域备份在释放后不再保留；高性能本地盘的跨地域备份会留到过期并继续计存储费。释放后的备份过了 7 天就开始计费。账号欠费超过 30 天，已删除实例的保留备份会被永久删除。
- 腾讯云回档写回源实例。单节点不能回档，但可以克隆。手动备份和跨地域备份、已销毁实例备份都不走那份免费空间。关闭跨地域备份后，旧文件在到期前仍占空间。
- 腾讯云自己销毁的回收站时间（包年包月 7 天、按量 24 小时）和到期、欠费后的隔离时间不是同一张表。选了不保留，下线后云上备份没有了。
- AWS 用 CLI 创建且不写保留期时，默认是 1 天，控制台创建默认是 7 天。把保留期改成 0 或从 0 改回来会发生中断，而且实例如果已有只读副本，不能设成 0。
- AWS 删除实例时如果保留自动备份，备份会留到删除当时的保留期结束，最终快照则一直留到你删除。其他区域的自动备份复制，不会因为你没保留源区域的自动备份而消失。控制台不显示备份存储大小。
- Azure 的按需备份不是 AWS 那种永久手动快照，它跟着服务器的保留期。MySQL 常见问题写删除后备份全部消失，恢复已删除服务器页写最多 5 天。PostgreSQL 的异地还原不能选任意时间点。还原出的 MySQL 高可用服务器会变成非高可用。PostgreSQL 还原后要自己补防火墙和参数。
- Cloud SQL 用命令行创建的 Enterprise 实例，自动备份和 PITR 默认关闭。把备份还原到生产实例会清掉已有的 PITR 日志。日志保留和备份保留设成同一个数时，实际可恢复时间会短于这个数。标准备份在删除实例时会一起删掉，除非开了删除后保留或最终备份；增强备份则会自动留下。4 天的支持窗口不能代替这个设置。
- 演练实例、最终快照和“删除后保留”的备份会继续计费。停止或关机通常停不掉已经产生的备份存储费。AWS 停机后仍收备份存储；Azure MySQL 停止后不再拍新备份，窗口内的旧备份仍计费，而且免费额度只适用于运行中的服务器。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。资料核对日期：2026-11-13。

阿里云中国站（RDS MySQL）：

- [备份指引](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/backup-guide)
- [数据库备份的自动配置与手动执行](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/full-backup)
- [查看或管理日志备份](https://help.aliyun.com/zh/rds/support/use-the-log-backup-feature)
- [设置任意时间点保护](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/any-point-in-time-recovery-protection)
- [跨地域备份](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/use-the-cross-region-backup-feature-of-an-apsaradb-rds-for-mysql-instance)
- [设置实例释放后备份保留策略](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/configure-backup-retention-policies-for-released-instances)
- [释放或退订实例](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/release-or-unsubscribe-from-an-instance)
- [ModifyBackupPolicy](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/api-rds-2014-08-15-modifybackuppolicy-mysql)
- [CreateBackup](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/api-rds-2014-08-15-createbackup-mysql)

这次没有写入正文的阿里云材料：实例回收站的保留天数、RDS SQL Server / PostgreSQL / MariaDB 的备份策略页、国际站逐段对照、阿里云 CLI 命令行参考。OpenAPI 错误码里“1～30 天”和参数表“7～730 天”不一致，正文已标出。

腾讯云中国站：

- [备份数据库](https://cloud.tencent.com/document/product/236/35172)
- [回档数据库](https://cloud.tencent.com/document/product/236/7276)
- [克隆实例](https://cloud.tencent.com/document/product/236/41207)
- [跨地域备份](https://cloud.tencent.com/document/product/236/73100)
- [备份保留设置](https://cloud.tencent.com/document/product/236/118344)
- [销毁实例](https://cloud.tencent.com/document/product/236/30305)
- [备份空间收费说明](https://cloud.tencent.com/document/product/236/36263)
- [计费概述](https://cloud.tencent.com/document/product/236/18335)

腾讯云国际站这次没有打开。收费说明和计费概述对“四节点是否计入本地盘免费额度”的写法不一致，正文已标出。单价没有抄。

AWS：

- [备份简介](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)
- [备份保留期](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.BackupRetention.html)
- [恢复到指定时间](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html)
- [保留自动备份](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.Retaining.html)
- [删除数据库实例](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DeleteInstance.html)
- [跨区域复制自动备份](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReplicateBackups.html)
- [modify-db-instance](https://docs.aws.amazon.com/cli/latest/reference/rds/modify-db-instance.html)
- [create-db-snapshot](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-snapshot.html)
- [RDS 常见问题](https://aws.amazon.com/rds/faqs/)

这次没有打开：Aurora 备份页、跨区域只读副本页、docs.amazonaws.cn、RDS 定价页上的单价表。免费套餐里另行给出的自动备份存储额度没有写入正文。

Microsoft Azure：

- [Azure Database for MySQL Flexible Server 备份和还原](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/concepts-backup-restore)
- [恢复已删除的 MySQL Flexible Server](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/how-to-restore-dropped-server)
- [az mysql flexible-server](https://learn.microsoft.com/en-us/cli/azure/mysql/flexible-server)
- [Azure Database for PostgreSQL Flexible Server 备份和还原](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/concepts-backup-restore)
- [恢复已删除的 PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/how-to-restore-deleted-server)
- [还原到最新还原点](https://learn.microsoft.com/en-us/azure/postgresql/backup-restore/how-to-restore-latest-restore-point)
- [az postgres flexible-server](https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server)

由世纪互联运营的 Azure 产品页没有打开。MySQL 长期保留预览已暂停，没有写成操作步骤。`backup create` 的参数表没有展开。

Google Cloud：

- [选择备份选项](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/backup-options)（Last updated 2026-11-12 UTC）
- [备份概览](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/backups)（Last updated 2026-11-12 UTC）
- [配置 PITR](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/configure-pitr)
- [还原概览](https://docs.cloud.google.com/sql/docs/mysql/backup-recovery/restore)
- [gcloud sql instances create](https://docs.cloud.google.com/sdk/gcloud/reference/sql/instances/create)
- [gcloud sql backups create](https://docs.cloud.google.com/sdk/gcloud/reference/sql/backups/create)
- [Cloud SQL 定价](https://cloud.google.com/sql/pricing)

Cloud SQL for PostgreSQL 和 SQL Server 的备份页、`gcloud sql instances patch` 参考页、Backup and DR 单价表这次没有打开。定价页上的金额没有写入正文。

如需为阿里云、腾讯云、Google Cloud、AWS、Microsoft Azure 的国际站账号开户或充值，可访问海宝云（CloudCup）官网 <https://www.haibaocloud.com>。官网常见问题当前写明：充值仅支持 USDT-TRC20，单笔最低 50 USDT；其他币种或法币需联系客服定制。具体以官网当前说明为准。Telegram 客服：@cloudcup。
