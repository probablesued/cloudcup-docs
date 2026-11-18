# 对象存储存储类型与生命周期：标准、低频、归档如何降本且不丢数据

对象存储的账单里，存储容量往往只是一部分。标准、低频、归档、冷归档这些类型，单价通常随“越少访问”下降，但页面同时写了最短存储时长、最小计费规格、取回或解冻费用、转换和删除的请求费用。把还在频繁读取的数据转去低频或归档，或者在最短时长之前再转一层、再删除，节省下来的存储费，可能被取回费、请求费和不足时长费用抵消。未完成的分片上传如果一直不合并、不取消，也会继续占容量计费。

私有桶、预签名 URL 和临时访问链接见 [对象存储私有桶与临时访问链接](object-storage-private-bucket-presigned-url.md)。云盘、快照和云服务器固定公网带宽的账单拆解见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。CDN 流量、请求和回源见 [CDN 费用配置与防超支](cdn-cost-configuration.md)。这几篇的内容这里不重复。对象存储不是云盘或快照，下面只写各云对象存储的存储类型与生命周期。地域价目表不在这里抄成一份现行统一价。页面上出现的单价，只作为撰写时该页的目录价或示例，以购买页和定价页为准。

核对时间：2026-11-18。资料核对日期：2026-11-18。存储类型名称、最短存储时长、提前删除或提前转换规则、取回或解冻时间、生命周期转换与过期删除的生效条件和限制，都来自这一天打开的官方页面，并附有链接。打不开、正文没有写明，或同一主题两页说法不同的，下面按读到的原文分开写，不补成一个数。

范围：阿里云中国站用 help.aliyun.com。国际站打开了 www.alibabacloud.com 的英文存储类型、生命周期概述、存储费用和生命周期计费页；国际站解冻专页这次没有打开，解冻优先级的细项只写中国站解冻页上的原文。腾讯云只核对了中国站 cloud.tencent.com。用中国站文档编号去打开 www.tencentcloud.com/document/product/436/33417 和 intl.cloud.tencent.com/document/product/436/33417，均返回 404，国际站页面这次没有打开，不要把中国站菜单套到国际站。AWS 使用 aws.amazon.com 与 docs.aws.amazon.com 的全球文档，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 azure.microsoft.com 与 learn.microsoft.com 的全球文档；英文和中文定价页已打开，单价单元格是占位符 `$-`；由世纪互联运营的 Azure 没有打开。Google Cloud 使用 cloud.google.com 与 docs.cloud.google.com。页面改版后，以你账号所属站点的当前文档为准。

## 先看这几件事

1. 先看访问频率，再看存储单价。低频、归档通常更便宜，但读一次可能另计取回或解冻费用，生命周期每次转换或删除也可能计请求费。阿里云转换存储类型页写明：如果对象每月平均访问频率高于 1 次，转换成低频、归档、冷归档或深度冷归档后的使用成本可能高于标准存储。
2. 设计生命周期时，把最短存储时长算进去。提前删除、覆盖、改存储类型或再往下转一层，已打开的阿里云、腾讯云、AWS、Azure、Google Cloud 页面都写了可能补收剩余时长，但起算点和例外并不相同。Google Cloud 定价页另外写明：由 Object Lifecycle Management 变更存储类型时，不收取提前删除费用；这一条不要套到其他云。
3. 先确认冷数据能不能直接读。腾讯云归档和深度归档、AWS 的 S3 Glacier Flexible Retrieval 与 S3 Glacier Deep Archive、Azure 的 Archive 层，都要先恢复或再水化才能读内容。阿里云归档可以解冻（约 1 分钟）或开启归档直读后直接访问。Google Cloud 的 Archive storage 页写明数据在毫秒级可用，不是数小时或数天的离线取回。
4. 小对象和海量请求可能把存储降本吃掉。阿里云和腾讯云的低频、归档等类型有 64 KB 最小计量单位；AWS 的 S3 Standard-IA、S3 One Zone-IA、S3 Glacier Instant Retrieval 最小计费对象是 128 KB，生命周期默认也不转换小于 128 KB 的对象。转换请求按对象次数计，小文件很多时，请求费可能高于省下的存储费。
5. 给未完成的分片上传单独写一条清理规则。阿里云、腾讯云、AWS 和 Google Cloud 已打开的页面都写了：未合并的分片会继续占空间并计费，可以用生命周期的中止或清理动作删除。Azure 未完成分片或未提交块的清理规则，这次没有打开对应页面，不补。

## 对照

下表只填这次打开的页面写明的规则。单元格写“这次没读到”，是因为对应页面没有打开，或打开了但正文没有这句。取回时间和最短时长以各云章节的原文为准，这里只做对照，不把不同站点的数字合成一套。

| 云 | 本次读到的存储类型或访问层 | 最短存储时长 | 取回或解冻（页上写明的时间） | 提前删除或提前转换 | 生命周期对小对象 | 未完成分片上传 |
| --- | --- | --- | --- | --- | --- | --- |
| 阿里云中国站 | 标准、低频访问、归档、冷归档、深度冷归档 | 标准无；低频 30 天；归档 60 天；冷归档和深度冷归档 180 天 | 低频可实时访问但计取回费；归档解冻通常 1 分钟，也可归档直读；冷归档高优先级通常 1 小时内、标准通常 2～5 小时、批量通常 5～12 小时；深度冷归档高优先级通常 12 小时内、标准通常 48 小时 | 不足规定时长前转换或删除，收取不足规定时长容量费用。低频和归档从 Object 的 Last Modified 起算；冷归档和深度冷归档从转为该类型的时间起算 | 低频、归档、冷归档、深度冷归档不足 64 KB 按 64 KB 计。生命周期是否跳过小文件，这次没有在已打开的生命周期概述页读到一条总规则 | 可用 AbortMultipartUpload 清理超过指定时间仍未合并的分片；分片按实际大小计费，无最小计量限制 |
| 阿里云国际站 | Standard、IA、Archive、Cold Archive、Deep Cold Archive | 与中国站存储类型页一致：IA 30 天，Archive 60 天，Cold / Deep Cold Archive 180 天 | 国际站存储类型对照表：Archive 解冻约 1 分钟或可直读；Cold Archive 1 到 12 小时；Deep Cold Archive 12 或 48 小时。各优先级的细项国际站解冻专页这次没有打开 | 国际站存储费用页和生命周期计费页写明提前转换或删除收取剩余时长费用，起算点与中国站同类页面一致 | 不足 64 KB 按 64 KB 计 | 国际站生命周期概述写明可用 AbortMultipartUpload 清理未完成分片。分片最小计量这次以中国站删除碎片页为准，国际站对应专页这次没有打开 |
| 腾讯云中国站 | 标准 / 标准（多 AZ）、低频 / 低频（多 AZ）、智能分层 / 智能分层（多 AZ）、归档 / 归档（多 AZ）、深度归档 | 标准和智能分层不限制；低频 30 天；归档 90 天；深度归档 180 天 | 低频毫秒级，但读时计取回费；归档快速 1～5 分钟、标准 3～5 小时、批量 5～12 小时；深度归档标准 12～24 小时、批量 24～48 小时 | 最短时长前删除、覆盖、改属性、改类型或生命周期沉降，按剩余时长补收提前删除费用。页面写明 COS 不会检查规则是否短于 30/90/180 天，仍按配置执行 | 低频、归档、深度归档不足 64 KB 按 64 KB 计。为降低成本，生命周期不对小于 64 KB 的对象做存储类型转换 | 文件碎片占用存储并计费；可用生命周期按碎片创建后天数清理。示例页写的是创建 30 天后删除，这是该页操作示例，不是必须写成 30 天 |
| 腾讯云国际站 | 这次没读到 | 这次没读到 | 这次没读到 | 这次没读到 | 这次没读到 | 这次没读到 |
| AWS（全球文档） | S3 Standard、S3 Express One Zone、S3 Intelligent-Tiering、S3 Standard-IA、S3 One Zone-IA、S3 Glacier Instant Retrieval、S3 Glacier Flexible Retrieval、S3 Glacier Deep Archive；RRS 页上写不建议使用 | Standard、Intelligent-Tiering、Express One Zone 无最短时长；Standard-IA 与 One Zone-IA 为 30 天；Glacier Instant Retrieval 与 Flexible Retrieval 为 90 天；Deep Archive 为 180 天 | Instant Retrieval 毫秒级；Flexible Retrieval：Expedited 通常 1～5 分钟（小于 250 MB）、Standard 通常 3～5 小时、Bulk 通常 5～12 小时；Deep Archive：Standard 通常 12 小时内，Bulk 通常 48 小时内，无 Expedited | 最短时长前删除、覆盖或再转换，按剩余时长按比例收取。生命周期不允许在同一条规则里、在最短时长结束前再转出该类型 | Standard-IA、One Zone-IA、Glacier Instant Retrieval 最小计费 128 KB。生命周期默认不转换小于 128 KB 的对象（页上写从 2024 年 9 月起的默认行为），可用对象大小过滤器改 | 建议用 AbortIncompleteMultipartUpload；示例是发起后 7 天中止。页上写明中止未完成分片不收取 early delete charges |
| Azure（全球文档） | Hot、Cool、Cold、Archive；另有 Smart tier，以及预览中的 Turbo tier | 通用 v2 账户：Cool 30 天，Cold 90 天，Archive 180 天。页上注记：在 Blob Storage 账户中，Cool 和 Cold 没有最短保留时长 | Hot / Cool / Cold 为毫秒级；Archive 需再水化到在线层。标准优先级：小于 10 GB 的对象可能最多 15 小时；高优先级：小于 10 GB 的对象可能不到 1 小时。高优先级在账户级有每小时 10 GiB 的限制 | 在最短天数前删除、覆盖或改层，按剩余天数按比例收取。若启用软删除，页上写提前删除罚金在保留期结束、视为删除之后才计 | 生命周期对小对象的最小体积限制，这次没读到 | 未完成分片或未提交块的生命周期清理，这次没读到 |
| Google Cloud | Rapid、Standard、Nearline、Coldline、Archive；另有 Autoclass | Standard 与 Rapid 无；Nearline 30 天；Coldline 90 天；Archive 365 天 | 存储类型页写明各类均为低延迟、无离线取回；Archive 也是毫秒级可用。读取 Nearline / Coldline / Archive 会另计取回费 | 最短时长前删除、替换或移动，按最短时长计费。例外：Standard / Rapid、由 Object Lifecycle Management 变更存储类型、已启用 Autoclass 的桶不计提前删除费；对象变为非当前版本或软删除时，要到符合永久删除条件才计 | 存储类型页写明无最小对象大小。Autoclass 下小于 128 KiB 的对象不向更冷类型转换，一直放在 Standard | 可用 AbortIncompleteMultipartUpload，条件仅限 age、matchesPrefix、matchesSuffix。XML API 分片在完成或中止前计入存储；未用于组装、被覆盖或上传被中止的分片可能产生提前删除费用。软删除不保护这类分片 |

## 阿里云

以下中国站规则来自 help.aliyun.com。国际站英文页与中国站一致的部分会注明国际站页眉的 Last Updated；只在一边读到的，不抄到另一边。

### 存储类型

[OSS有哪几种存储类型](https://help.aliyun.com/zh/oss/user-guide/overview-53/)（页上「更新时间」栏未见具体日期）写明，OSS 提供五种类型：标准存储（Standard）、低频访问存储（Infrequent Access）、归档存储（Archive）、冷归档存储（Cold Archive）、深度冷归档存储（Deep Cold Archive）。上传到 Bucket 的 Object 默认是标准存储。

该页对照表和正文写明的计费与访问特点如下。

- 标准存储：按对象实际大小计算，无最低存储时间，可实时访问。同城冗余（多 AZ）服务可用性写的是 99.995%，同城冗余（2 AZ）和本地冗余写的是 99.99%。无地域属性的标准存储服务可用性写的是 99.00%。
- 低频访问：最小计量单位 64 KB，最低存储时间 30 天，可实时访问，访问时产生数据取回费用。页上适用场景写的是平均每月访问频率 1 到 2 次；对照表「数据访问特点」写的是单文件月访问不到 1 次。这两处表述不一致，这里分开保留，不合成一个数。
- 归档存储：最小计量单位 64 KB，最低存储时间 60 天。可以解冻后访问，解冻时间写的是 1 分钟；也可以通过归档直读实时访问。解冻后访问产生归档存储数据取回容量费用，直接访问产生归档直读数据取回容量费用。
- 冷归档存储：最小计量单位 64 KB，最低存储时间 180 天，仅支持本地冗余。需解冻后访问，对照表写解冻时间 1～12 小时；解冻产生数据取回费用以及取回请求费用。
- 深度冷归档存储：最小计量单位 64 KB，最低存储时间 180 天，仅支持本地冗余。对照表写解冻时间 12/48 小时。该页列出当前采用单可用区机制的地域，包括华东 1（杭州）、华东 2（上海）、华北 2（北京）、华北 3（张家口）、华北 6（乌兰察布）、华南 1（深圳）、新加坡等，完整名单以该页为准，这里不逐行抄成可用性保证。

国际站 [Storage classes](https://www.alibabacloud.com/help/en/oss/user-guide/overview-53/)（Last Updated: Sep 23, 2026）五种类型的英文名称、64 KB 最小计量、30/60/180 天最短时长，以及 Archive 约 1 分钟、Cold Archive 1 到 12 小时、Deep Cold Archive 12 或 48 小时，与中国站对照表一致。冷归档和深度冷归档在国际站同样写明只支持本地冗余。

### 最短时长与不足规定时长容量费用

[存储费用](https://help.aliyun.com/zh/oss/storage-fees)（页上「更新时间」栏未见具体日期）写明：只要文件存储在 OSS Bucket 就会产生存储费用，与是否被访问、执行何种操作无关。低频访问最低 30 天、归档最低 60 天、冷归档以及深度冷归档最低 180 天。在不满足规定时长前转换类型或删除，会产生不足规定时长容量费用。

计算示例：低频访问类型存储不足 30 天（即 720 小时）被覆写或删除，收取剩余时间（720 小时减去已存储时间）的存储费用。归档、冷归档、深度冷归档规则类似，区别在最低时长。

该页对起算点写得很具体，不要混用：

- 低频访问和归档存储的不足规定时长，以文件存储在 OSS 的 Last Modified 时间开始计算。
- 冷归档和深度冷归档，以文件转为冷归档或者深度冷归档类型的时间开始计算。

因此，对象从标准转为低频再转为归档时，归档的 60 天不是从“第一次上传”单独再算一套，而是跟 Last Modified 走；转到冷归档后，180 天则从转为冷归档的那一刻重新起算。[通过生命周期规则转换或删除Object是如何计费的](https://help.aliyun.com/zh/oss/user-guide/fees-related-to-lifecycle-rules)（页上「更新时间」栏未见具体日期）给了例子：标准存储 10 天后转低频，再存 5 天转归档，再存 5 天删除，会收 40 天归档不足规定时长容量费用；标准存储 10 天后转冷归档，只存 1 天就删除，会收 179 天冷归档不足规定时长容量费用。若冷归档存了 10 天再转深度冷归档、只再存 1 天就删除，会同时出现 170 天冷归档不足规定时长费用，以及 179 天深度冷归档不足规定时长费用。

该页还写：删除操作会一次性收取不足规定时长容量费用，不会连续出现；覆写则可能连续出现。例如低频对象存 1 天后被同名上传覆盖，收取 29 天不足规定时长费用；再过 3 天再次覆写或删除，还会再收 27 天。

国际站 [Storage fees](https://www.alibabacloud.com/help/en/oss/storage-fees)（Last Updated: Oct 20, 2026）和 [Billing for transforming or deleting objects using lifecycle rules](https://www.alibabacloud.com/help/en/oss/user-guide/fees-related-to-lifecycle-rules)（Last Updated: Sep 21, 2026）对 30/60/180 天、Last Modified 与“转为冷归档或深度冷归档的时间”这两种起算点，以及上面几组天数示例，与中国站一致。存储费用页有一处把 Standard LRS 的月单价换算成小时单价的计算示例，那是该页当时的换算方法说明，不是这里要抄的现行统一价，实际单价以 OSS 产品定价页和购买页为准。OSS 产品定价页这次没有打开，不抄价目表。

### 解冻与取回

[解冻文件](https://help.aliyun.com/zh/oss/user-guide/restore-objects-for-access)（页上「更新时间」栏未见具体日期）写明：归档、冷归档或深度冷归档中的文件处于冻结状态，无法直接读取。解冻是创建临时可读副本，不改变原始文件的存储类型。状态包括冷冻、解冻中、已解冻、副本过期。首次请求返回 `202 Accepted`；解冻中再次提交相同优先级请求会返回 `409 Conflict`，但允许提交更高优先级来加速；已解冻后再请求返回 `200 OK`，并延长临时副本有效期。

该页写的解冻完成时间“请以实际为准”，同时给出通常值：

- 归档类型：通常 1 分钟。
- 冷归档：高优先级（Expedited）通常 1 小时内；标准（Standard）通常 2～5 小时内；批量（Bulk）通常 5～12 小时内。
- 深度冷归档：高优先级（Expedited）通常 12 小时内；标准（Standard）通常 48 小时内。

副本有效期：归档为 1～7 天；冷归档和深度冷归档为 1～365 天。有效期内可访问且无需再次支付数据取回费用。计费项包括：按实际解冻量一次性收取的数据取回容量费用；发起 `RestoreObject` 的请求费用（归档计为 Put 请求，冷归档和深度冷归档计为取回请求）；原类型的存储费用；冷归档和深度冷归档在副本有效期内另计临时存储费用。

配额参考值也只在这页读到：单个账号单个地域，冷归档平均每秒 500 个 Object，三种优先级每天合计 100 TB～120 TB；深度冷归档平均每秒 100 个 Object，两种优先级每天合计 10 TB～15 TB。超出后仍可提交请求，但会进入队列，完成时间可能超出对应优先级的通常时间。

[转换文件存储类型](https://help.aliyun.com/zh/oss/user-guide/convert-storage-classes)（页上「更新时间」栏未见具体日期）补充：访问低频会产生数据取回费用；解冻归档、冷归档和深度冷归档会额外收取数据解冻费用；开启归档直读后直接访问归档，会收取归档直读费用。这些费用与流出流量费用是两个独立计费项。冷归档和深度冷归档解冻时会生成一份标准存储类型副本，在解冻时间结束前按标准存储费率计算临时存储费用。国际站解冻专页这次没有打开，上面的优先级名称、副本有效期和配额只来自中国站解冻页。

### 生命周期规则

[什么是OSS生命周期](https://help.aliyun.com/zh/oss/user-guide/overview-54/)（页上「更新时间」栏未见具体日期）写明：规则创建后的 24 小时内，OSS 会加载规则；加载完成后，通常在次日 UTC 时间 0 点（北京时间 8 点后）扫描并执行。国际站 [Lifecycle overview](https://www.alibabacloud.com/help/en/oss/user-guide/overview-54/)（Last Updated: Oct 21, 2026）对加载和执行时间的写法相同。

一条规则包含三部分：控制对象、执行动作、触发策略。控制对象可按前缀（Prefix）、对象标签（Tag）或文件大小（ObjectSize）筛选。该页写明：目前不支持通配符匹配、后缀匹配以及正则匹配。执行动作包括存储类型转换（Transition）、过期删除（Expiration）、清理碎片（AbortMultipartUpload）。触发策略可以基于最后一次修改时间（Last-Modified-Time），或在启用访问跟踪后基于最后一次访问时间（Last-Access-Time）。

如果 Bucket 已开启对象级别保留策略（ObjectWorm），Transition 不受影响，但保留期内的 Object 不会被 Expiration 自动删除。

多条规则同时匹配同一文件时，优先级为：删除 Object > 转为深度冷归档 > 转为冷归档 > 转为归档 > 转为低频访问 > 转为标准。删除始终优先于存储转换。该页建议把删除规则的时间设得比转储规则更长。基于最后一次修改时间的规则和基于最后一次访问时间的规则同时命中时，该页举例：30 天后删除与 30 天后转低频相比，OSS 按较低开销原则只让删除生效。

使用 ossutil、SDK 或直接调用 API 配置时，每次 PutBucketLifecycle 都会完全覆盖当前 Bucket 的全部生命周期配置。只提交新规则、未包含原有规则，原有规则会被删除。控制台会读取当前配置再合并提交。

[转换文件存储类型](https://help.aliyun.com/zh/oss/user-guide/convert-storage-classes) 对基于最后一次修改时间的本地冗余转换路径写明：标准可转为低频、归档、冷归档或深度冷归档；低频可转为归档、冷归档或深度冷归档；归档可转为冷归档或深度冷归档；冷归档可转为深度冷归档。同一 Bucket 同时配置多级转换时，周期必须满足：转为低频的周期 < 转为归档的周期 < 转为冷归档的周期 < 转为深度冷归档的周期。同城冗余的对象转到冷归档或深度冷归档时，目标是本地冗余，因为这两种类型只支持本地冗余。基于最后一次访问时间、把标准或低频转为归档、冷归档或深度冷归档，该页写明需要先提交工单申请转换权限。基于最后一次修改时间的生命周期规则不支持把低频转回标准，该页给出的办法是 CopyObject 或 ossutil 的 set-props。

生命周期概述的「应用于生产环境」一节给出了阶梯时间的正反例：标准 10 天转低频、再 5 天转归档，归档只存了很短时间，会不足 60 天并产生不足规定时长容量费用；该页给出的正确示例是标准 30 天转低频，再至 90 天转归档，以便低频阶段满足归档最低 60 天要求。这是该页的配置示例，不是必须采用的业务天数。

生命周期规则本身不收费。执行转换、删除 Object 或删除 Part 时，按 Put 类型请求计费。概述页提醒：对于包含海量小文件的 Bucket，这笔请求费可能较为显著。生命周期计费页还写明：从低频、归档、冷归档转为其他类型的 Put 请求费用，高于从标准转为这些类型的费用；在中国内地各地域，用生命周期删除低频、归档、冷归档 Object 或 Part 的 Put 类请求费用，高于删除标准类型；删除深度冷归档不收取 Put 类请求费用。在中国香港以及海外地域，用生命周期删除各存储类型 Object 或碎片时不收取 Put 类请求费用。国际站生命周期计费页对 Chinese mainland 与 China (Hong Kong) and regions outside China 的这一区分，与中国站一致。生命周期规则本身不会产生数据取回费用；取回发生在转换之后你再去读或解冻这些对象时。

### 小对象与未完成分片

存储费用页写明，低频、归档、冷归档、深度冷归档有 64 KB 最小计量单位，计费容量可能大于实际容量。标准存储按实际大小计算。

[删除碎片](https://help.aliyun.com/zh/oss/user-guide/delete-parts)（页上「更新时间」栏未见具体日期）写明：分片上传时切出的 Part 在调用 CompleteMultipartUpload 前不能被读取。OSS 会根据 Part 的存储类型、实际大小和时长收取存储费用。Part 的存储类型与 Object 一致，但 Part 无最小计量限制，小于 64 KB 仍按实际大小计算。已上传但未完成或未取消的分片会占用存储空间并产生费用。除控制台、SDK、ossutil 手动删除外，可通过生命周期自动清理长时间未执行 CompleteMultipartUpload 的 Part。删除 Bucket 前必须先删除 Object 和 Part。Bucket 和 Object 删除后无法恢复，该页用警告标出，操作前要确认对象确实不再需要。

## 腾讯云

以下只来自中国站 cloud.tencent.com。国际站对应文档这次打开为 404，不把下面的菜单、天数或地域名单套到国际站账号。

### 存储类型

[存储类型概述](https://cloud.tencent.com/document/product/436/33417)（最近更新时间：2026-10-12 14:51:04）写明，COS 提供：标准存储（多 AZ）（`MAZ_STANDARD`）、低频存储（多 AZ）（`MAZ_STANDARD_IA`）、归档存储（多 AZ）（`MAZ_ARCHIVE`）、智能分层存储（多 AZ）（`MAZ_INTELLIGENT_TIERING`）、智能分层存储（`INTELLIGENT_TIERING`）、标准存储（`STANDARD`）、低频存储（`STANDARD_IA`）、归档存储（`ARCHIVE`）、深度归档存储（`DEEP_ARCHIVE`）。上传时如不设定存储类型，默认是标准存储。

该页对照表写明的最短计费时间和最小计量如下。

- 标准存储、标准存储（多 AZ）：按实际大小，最短计费时间不限制，毫秒级响应，无数据取回费用。
- 低频存储、低频存储（多 AZ）：不足 64 KB 按 64 KB 计，最短 30 天；不足 30 天按 30 天计算。响应毫秒级，但访问时收取数据取回费用。正文适用场景写平均每月访问频率 1 到 2 次。
- 智能分层存储、智能分层存储（多 AZ）：按实际大小计费，最短计费时间不限制，毫秒级。COS 根据访问频次在层级间自动转换。智能分层存储（多 AZ）可位于高频访问层和低频访问层；智能分层存储除这两个默认层级外，还可按需开启归档层和深度归档层（需添加智能分层配置规则，且受地域支持情况限制）。除归档层快速取回外，无数据取回费用；归档层快速取回的刊例价与同地域归档存储快速取回相同，归档层标准/批量取回及深度归档层取回均不计费。该页还写会收取智能分层对象监控费用。自动转换的具体天数，要看「智能分层存储简介」专页，这次没有打开，不补天数。
- 归档存储、归档存储（多 AZ）：不足 64 KB 按 64 KB 计，最短 90 天；不足 90 天按 90 天计算。读取前需要恢复。快速取回 1～5 分钟，标准取回 3～5 小时，批量取回 5～12 小时。
- 深度归档存储：不足 64 KB 按 64 KB 计，最短 180 天；不足 180 天按 180 天计算。标准取回 12～24 小时，批量取回 24～48 小时。

多 AZ 类型和深度归档都不是全部地域可用。该页列出了各自当前支持的地域，并注明中国香港、新加坡等需申请加白；归档存储写明除大阪外其他地域支持。完整地域名单以该页为准，账号里是否已开通，要在控制台和当前文档再确认一次。

存储类型转换是单向沉降：标准存储 > 低频存储 > 智能分层存储 > 归档存储 > 深度归档存储，也允许跳级，例如标准直接到归档，不支持逆向。较冷类型回到较热类型，非归档要用 PUT Object - Copy，归档或深度归档要用 POST Object restore。归档类型要先恢复到标准类型，才能再改成其他类型。

### 提前删除与存储容量计费

[存储容量费用](https://cloud.tencent.com/document/product/436/53482)（最近更新时间：2026-10-12 14:51:03）把提前删除费用定义为：在达到最短存储时长之前被删除，按剩余时长补收一次性存储容量费用。触发场景包括：手动彻底删除；修改自定义头部、修改存储类型等属性；生命周期沉降（沉降后删除原来的同名对象）；未开启版本控制时上传同名对象覆盖。

计费公式该页写为：提前删除费用 = 提前删除量 × 剩余时长 × 对应存储类型价格 × 适用折扣。剩余时长 = 最短存储时长 − 已存储时长。最短时长：低频 / 低频（多 AZ）30 天，归档 / 归档（多 AZ）90 天，深度归档 180 天。账单不单独列出“提前删除费用”，而是合并在对应存储类型的存储容量费用中。该页有一个北京地域低频存储的计算示例，用了当时页上的刊例价做乘法，那只是示例，不以这里的算式结果作为现行统一价。

### 取回与恢复

[数据取回费用](https://cloud.tencent.com/document/product/436/53862)（最近更新时间：2026-08-18 17:21:57）写明：读取低频、低频（多 AZ）、归档、归档（多 AZ）和深度归档时会产生数据取回费用，按实际读取或恢复的数据量计算。若对低频数据配置了生命周期，执行沉降到归档或深度归档时，也会产生低频数据的取回费用和请求费用。归档和深度归档在未恢复前不可读取和下载，需先恢复为标准存储类型副本。费用按日结算，以取回任务实际完成时间为准，不以请求发起时间为准。取回单价以产品定价中心为准，该页写明数据取回功能无资源包抵扣。该页计费案例里出现的单价标注为仅供参考，这里不抄成现行价。

[恢复归档对象](https://cloud.tencent.com/document/product/436/32430)（最近更新时间：2026-04-08 15:18:42）写明：恢复会新增一个标准存储类型的对象副本，在设定的有效期内可读取、下载；到期后副本自动删除。归档恢复后，副本按标准存储计费；深度归档恢复后，副本的请求费用按指定单价收取，流量费用与标准存储一致，具体以产品定价为准。副本有效期最短 1 天、最长 365 天，恢复完成后可再次单击恢复来更改有效期。快速取回不支持深度归档。数据恢复请求 QPS 限制为 100 次/秒。对象恢复完成后，在副本过期删除之前修改存储类型（例如改为标准存储），原对象类型将由归档存储替换为标准存储类型。

### 生命周期规则

[生命周期概述](https://cloud.tencent.com/document/product/436/17028)（最近更新时间：2025-08-04 17:57:32）写明：每个存储桶最多可添加 1000 条生命周期规则，支持基于最后修改时间和最后访问时间。规则可作用于当前版本、历史版本、删除标记和碎片文件，筛选方式包括对象前缀和对象标签。

沉降支持公有云地域；金融云地域仅支持沉降至低频存储。沉降是单向的，也支持跳级，不支持逆向。沉降不会改变对象原先的上传或修改时间。对同一组对象配置多条冲突规则且不含过期删除时，优先执行沉降至最冷存储类型的规则。过期删除的效力大于转换存储类型，冲突时以最短过期时间为准。页面强烈提醒：不要针对同一组对象配置多个含冲突条件的规则。支持设置的最长天数是 3650 天。历史版本的沉降和删除，以对象成为历史版本的时间计算，而不是最后修改时间。规则中的天数以 24 小时为准，未满 24 小时不算 1 天。

规则生效分为每日扫描和执行：COS 在北京时间（GMT+8）每日 0 点拉取规则并扫描；扫到达到指定日期的对象后再执行。请勿在扫描和执行期间变更规则状态，变更会导致原规则终止，无法保证操作正确执行。该页写明：目前腾讯云对于生命周期的执行效力不提供账单承诺，计费在生命周期执行完成时才改变。大量存量对象或意外情况可能导致未完成，可用 GET 或 HEAD Object 查看当前状态。

为降低用户成本，生命周期不会对小于 64 KB 的对象执行存储类型转换。但这些小对象如果直接上传为低频、归档或深度归档，仍按 64 KB 最小占用计费。

生命周期执行删除或沉降时会产生请求次数费用。同一天命中同一对象的多条规则，只执行删除或最冷的那一次，只收一次请求费；如果安排在不同天，会执行多次、收取多次。提前删除费用一节再次写明：低频至少 30 天、归档至少 90 天、深度归档至少 180 天；生命周期提前删除或沉降仍要补收。COS 不会检查规则是否包含少于这些天数的配置，所有配置都按你的要求执行。

### 未完成分片

[删除碎片文件](https://cloud.tencent.com/document/product/436/17313)（最近更新时间：2026-03-13 15:24:42）写明：暂停或取消上传的文件会显示在文件碎片管理项；文件碎片和正常对象一样占用存储空间，产生存储容量费用。删除碎片后不可恢复。除手动删除或清空碎片外，可在生命周期里为整个存储桶配置规则，按最后修改时间清理碎片。该页操作示例是碎片创建 30 天后删除，这是示例天数，不是必须值。删除存储桶前，需要先删掉未完成上传的碎片和已上传文件。

## AWS

以下来自 aws.amazon.com 与 docs.aws.amazon.com 的全球文档，没有打开 docs.amazonaws.cn。定价页上的地区单价不在这里抄成全区现行统一价。

### 存储类型

[Understanding and managing Amazon S3 storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)（页上未见更新日期）写明，S3 中每个对象都有存储类型，默认是 S3 Standard（`STANDARD`）。该页对照表写明的最短存储时长和最小计费大小如下。

- S3 Standard：为频繁访问、毫秒级读取设计，无最短存储时长，无最小计费对象大小。
- S3 Express One Zone（`EXPRESS_ONEZONE`）：单可用区、个位数毫秒访问，无最短存储时长，无最小计费对象大小。数据存放在你选择的单个可用区。
- S3 Intelligent-Tiering（`INTELLIGENT_TIERING`）：用于访问模式未知、变化或不可预测的数据，无最短存储时长，对照表最小计费大小为 None。按对象收取监控和自动化费用，无取回费用。小于 128 KB 的对象不被监控，始终存放在 Frequent Access 层。自动分层：上传或转入后先在 Frequent Access；连续 30 天未访问进入 Infrequent Access；连续 90 天未访问进入 Archive Instant Access。可选的 Archive Access 至少连续 90 天未访问，Deep Archive Access 至少连续 180 天未访问；这两层是异步取回，取回前要先 `RestoreObject`。
- S3 Standard-IA（`STANDARD_IA`）和 S3 One Zone-IA（`ONEZONE_IA`）：适合大于 128 KB、计划至少存储 30 天、需要毫秒级访问的低频数据。小于 128 KB 仍按 128 KB 计费。30 天内删除、覆盖或转换到其他类型，除正常存储费用外，还按剩余天数按比例收费。两者都按 GB 收取取回费。Standard-IA 跨多个可用区；One Zone-IA 只在一个可用区，不能抵御该可用区因灾害造成的物理损失。
- S3 Glacier Instant Retrieval（`GLACIER_IR`）：为每季度访问一次、仍需毫秒级取回的归档数据设计，最短 90 天，最小计费 128 KB，按 GB 收取取回费，可实时访问。
- S3 Glacier Flexible Retrieval（`GLACIER`）：取回时间为数分钟到数小时，最短 90 天，必须先恢复才能访问，按 GB 收取取回费。每个归档对象需要 40 KB 额外元数据：32 KB 按 Flexible Retrieval 费率，8 KB 按 S3 Standard 费率。
- S3 Glacier Deep Archive（`DEEP_ARCHIVE`）：取回时间为数小时，最短 180 天，必须先恢复才能访问，按 GB 收取取回费。同样需要 40 KB 额外元数据，其中 32 KB 按 Deep Archive 费率，8 KB 按 S3 Standard 费率。
- Reduced Redundancy Storage（`REDUCED_REDUNDANCY`）：该页用 Important 写明不建议使用，并称 S3 Standard 更有成本效益。

S3 on Outposts 的 `OUTPOSTS` 类型仅用于 Outposts 上的桶，这次不展开本地 Outposts 部署。

[Amazon S3 定价](https://aws.amazon.com/s3/pricing/)（页上未见更新日期）与文档页一致：Intelligent-Tiering 对小于 128 KB 的对象不监控、始终按 Frequent Access 计费且无监控费；Standard-IA 与 One Zone-IA 最小计费 128 KB、最短 30 天，提前删除、覆盖或转换要补收剩余天数。该页还写明：使用 PUT、COPY 或生命周期规则把数据迁入任何 S3 存储类型时，都有按请求计的 ingest 费用，转换前要考虑这笔成本。具体地区单价以该定价页和计算器为准，这里不列表。

### 取回时间

[Understanding archive retrieval options](https://docs.aws.amazon.com/AmazonS3/latest/userguide/restoring-objects-retrieval-options.html)（页上未见更新日期）写明：S3 Glacier Instant Retrieval 可用 `GET` 立即访问；Flexible Retrieval 和 Deep Archive 必须先用 RestoreObject 取回。可选 Expedited、Standard、Bulk。

- Expedited：用于 Flexible Retrieval 或 Intelligent-Tiering 的 Archive Access 层。小于 250 MB 的对象通常在 1～5 分钟内可用；大于或等于 250 MB 的对象通常以最高约每秒 300 MB 的吞吐取回。这是付费的高级功能。也可购买 Provisioned Capacity，每个容量单位保证每 5 分钟至少 3 次 Expedited，并提供最高约每秒 300 MB 的吞吐。没有预置容量时，高峰期 Expedited 请求可能不被接受。
- Standard：未指定取回选项时的默认值。Flexible Retrieval 或 Archive Access 层通常 3～5 小时；Deep Archive 或 Deep Archive Access 层通常 12 小时内。对 Intelligent-Tiering 中的对象，Standard 取回免费。若用 S3 Batch Operations 发起，Flexible Retrieval 的 Standard 通常在数分钟内开始、3～5 小时内完成；Deep Archive 的 Standard 通常在 9 小时内开始完成，吞吐最高约每天 1～2 PB。
- Bulk：该页称为 S3 Glacier 存储类型中成本最低的取回选项。Flexible Retrieval 或 Archive Access 通常 5～12 小时；Deep Archive 或 Deep Archive Access 通常 48 小时内。对存储在 Flexible Retrieval 或 Intelligent-Tiering 中的对象，Bulk 取回免费。

Deep Archive 和 Deep Archive Access 没有 Expedited。该页还写：恢复由数千万个对象或数百 TB 组成的数据集，可能超过各档通常时间；账户有取回请求配额，S3 Glacier 支持每秒 1000 笔恢复请求，超出可能被节流或拒绝并返回 `ThrottlingException`。恢复开始后的吞吐最高约每个账户每天 1～2 PB；大于 5 TB 的对象可能更久，最高约每秒 300 MB，例如一个 50 TB 的 Flexible Retrieval 对象可能需要最多 48 小时。需要提高限制时可联系 AWS Support。这些吞吐和配额数字都来自该页，实际以账号当前限制为准。

恢复归档对象时，对象的存储类型仍是 Flexible Retrieval 或 Deep Archive；你同时为归档本身和临时恢复出的副本付费，临时副本按 S3 Standard 费率。生命周期转换页也写了这一点。

### 生命周期转换与过期

[Transitioning objects using Amazon S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)（页上未见更新日期）写明，生命周期按瀑布模型向下转换，不能逆向走到更热的类型。支持的路径包括：

- Standard → Standard-IA、Intelligent-Tiering、One Zone-IA、Glacier Instant Retrieval、Flexible Retrieval 或 Deep Archive；
- Standard-IA → Intelligent-Tiering、One Zone-IA、Glacier Instant Retrieval、Flexible Retrieval 或 Deep Archive；
- One Zone-IA → Flexible Retrieval 或 Deep Archive；
- Glacier Instant Retrieval → Flexible Retrieval 或 Deep Archive；
- Flexible Retrieval → Deep Archive。

Intelligent-Tiering 内部各访问层能转到哪些类型，该页按 Frequent/Infrequent、Archive Instant Access、Archive Access、Deep Archive Access 分别列出，配置前要对照该页，不在这里简化成“都可以”。已启用或已暂停版本控制的桶中，复制状态为 `Pending` 或 `Failed` 的对象不能转换。加密对象在转换过程中保持加密。

默认行为：生命周期配置会阻止小于 128 KB 的对象转换到任何存储类型。该页不建议转换小于 128 KB 的对象，因为每个对象都要付转换请求费，请求费可能超过存储节省。从 2024 年 9 月起这是默认行为；之前的默认是小于 128 KB 的对象只允许转到 Flexible Retrieval 和 Deep Archive。若要允许更小对象转换，可在规则里加 `ObjectSizeGreaterThan` 或 `ObjectSizeLessThan`。

某些存储类型有最短存储时长。在时长结束前转出，要为剩余时长付费。不能在同一条生命周期规则里，把对象转进某类型后再在最短时长结束前转出。例如 Glacier Instant Retrieval 最短 90 天，不能写成 4 天后转入 Instant Retrieval、20 天后转入 Deep Archive；转到 Deep Archive 必须至少在 94 天之后。

转到 Flexible Retrieval 或 Deep Archive 是异步的，配置中的转换日期和物理转换之间可能有延迟。但一旦规则条件满足，就按目标存储类型计费，最短时长（90 或 180 天）和每对象 40 KB 开销也从这一刻开始算。唯一例外是转到 Intelligent-Tiering，计费在物理转换完成后才改变。

[Lifecycle configuration elements](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)（页上未见更新日期）写明：每个桶一个生命周期配置，最多 1000 条规则，这一限制不可调整。规则包含 ID（最长 255 个字符）、Status（`Enabled` 或 `Disabled`）、Filter，以及一个或多个 Transition 或 Expiration 动作。过滤器可用对象大小、键前缀、一个或多个对象标签，或它们的组合；多个标签是逻辑 AND。也可以指定一个日期，到期后对所有符合过滤条件的对象执行动作；如果日期已经过去，合格对象会立即符合条件。基于日期的动作不是一次性的，只要规则仍是 `Enabled`，S3 会继续对之后新创建的对象执行。日期必须符合 ISO 8601，时间始终是 UTC 午夜。控制台不能创建基于日期的规则，但可以查看、禁用或删除。

### 未完成的分段上传

[Configuring a bucket lifecycle configuration to delete incomplete multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html)（页上未见更新日期）建议配置 `AbortIncompleteMultipartUpload`，在发起后指定天数仍未完成时中止上传并删除已上传分片。该规则对已有和之后发起的未完成分段上传都适用，对已经完成的分段上传没有影响，也不会删除对象。该页写明：移除这些未完成分片时，不会产生 S3 Lifecycle 的 early delete charges。控制台和 CLI 示例使用的是 7 天，这是该页示例，不是必须值。

## Microsoft Azure

以下来自 learn.microsoft.com 与 azure.microsoft.com 的全球文档。由世纪互联运营的 Azure 没有打开，不要把下面的规则直接套到世纪互联账户。

### 访问层

[Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)（页上未见更新日期）写明，块 Blob（包括版本和快照）可以设置访问层，追加 Blob 和页 Blob 不支持更改层。访问层包括：

- Hot：在线层，适合频繁访问或修改，存储费用最高、访问费用最低。
- Cool：在线层，适合不频繁访问或修改，应至少存储 30 天；存储费用低于 Hot，访问费用高于 Hot。
- Cold：在线层，适合很少访问或修改但仍需快速取回，应至少存储 90 天；存储费用低于 Cool，访问费用高于 Cool。
- Archive：离线层，适合很少访问、可接受数小时延迟的数据，应至少存储 180 天。
- Smart tier：根据使用模式在 Hot、Cool、Cold 之间自动移动。该页指向单独的 Smart tier 文档，这份专页这次没有打开，自动分层的天数不补。
- Turbo tier（Preview）：页上写这是预览功能，为需要低延迟和高吞吐的工作负载提供 SSD 性能。预览资格和开通步骤以该页当时说明为准，这里不写成已对所有订阅默认可用。

Hot、Cool、Cold 都是在线层，可以立即访问。Archive 中的 Blob 不能读取或修改；要读或下载，必须先再水化（rehydrate）到 Hot、Cool 或 Cold。该页写 Archive 再水化最多可能需要 15 小时，取决于指定的优先级。Archive 的元数据仍可读取，元数据的存储费用按 Cool 层费率计；快照不支持已归档的 Blob。只有配置为 LRS、GRS 或 RA-GRS 的存储账户支持把 Blob 移到 Archive；ZRS、GZRS、RA-GZRS 账户不支持 Archive。

对照表把「建议的最短保留期」写为：Hot 为 N/A，Cool 30 天，Cold 90 天，Archive 180 天。脚注写明：在通用 v2 账户中，Cool 最短 30 天、Cold 最短 90 天；在 Blob Storage 账户中，Cool 和 Cold 没有最短保留时长。延迟（首字节时间）：Hot / Cool / Cold 为毫秒，Archive 为数小时。

在最短天数前删除、覆盖或移到其他层，会收取按比例计算的提前删除罚金。该页例子：Cool 层的 Blob 在 21 天后删除，收取相当于 9（30 减 21）天 Cool 存储的费用；Archive 在 45 天后删除或改到 Hot，收取相当于 135（180 减 45）天 Archive 存储的费用。该页还写：在 120 天后删除已归档 Blob，将导致该对象按 180 天计费。若账户启用了软删除，Blob 在保留期结束前只是软删除，不算作触发提前删除罚金的删除。

从更热层改到更冷层是立即完成的，从 Cold 或 Cool 改到 Hot 也是立即完成；从 Archive 再水化到在线层可能最多 15 小时。把 Blob 从更热层改到更冷层，通常用 Set Blob Tier 或生命周期策略；生命周期策略不能用来把已归档 Blob 再水化到在线层。从 Archive 再水化，或从 Cool / Cold 移到 Hot，该页更推荐 Copy Blob，因为复制可以避免源 Blob 尚未满最短间隔时的提前删除罚金，但会同时为源和目标两份容量付费。

更改层的计费方向该页归纳为：向更冷层移动按写入（操作 + 访问）计；向更热层移动按读取（操作 + 访问）计，并可能加上源层的取回费和提前删除费。再水化过程中，在层真正改变之前，数据仍按 Archive 计费。改变存储账户的默认访问层，会作用到所有未显式设置层的 Blob；在通用 v2 账户中，默认层改到更冷层会收取这些 Blob 的写操作费，改到更热层会收取读操作费和按 GB 的数据取回费。Archive 不能作为账户默认访问层。

[Azure Blob Storage pricing](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/) 与中文页 [Azure Blob 存储定价](https://azure.microsoft.com/zh-cn/pricing/details/storage/blobs/) 这次都打开了，标题可确认，页上未见更新日期。两页都写了提前删除惩罚：移到 Archive 受 180 天约束，移到 Cold（中文页作「寒层」）受 90 天约束，移到 Cool（中文页作「冷层」）受 30 天约束，并按对应层存储价按比例计算。这次打开时，单价单元格是占位符 `$-`，不把占位符当成 0，也不把占位符算成现行统一价。实际应付以你账号的定价计算器和购买页为准。

### 再水化

[Blob rehydration from the archive tier](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)（页上未见更新日期）写明两种方法：用 Copy Blob 把已归档 Blob 复制到在线层的新 Blob，源仍留在 Archive；或用 Set Blob Tier 把同一 Blob 的层改到 Hot、Cool 或 Cold。不能直接再水化已归档的快照或早期版本，必须用 Copy Blob 复制到在线层的新 Blob。

优先级用可选请求头 `x-ms-rehydrate-priority`：

- Standard：按到达顺序处理，小于 10 GB 的对象可能最多 15 小时完成。这是默认值。
- High：优先于标准请求，小于 10 GB 的对象可能在不到 1 小时内完成，但费用更高；也可能因 Blob 大小和当前需求而超过 1 小时。该页建议把高优先级留给紧急恢复。

标准优先级进行中时，可以把优先级上调为 High；不能从 High 降回 Standard。账户级限制：高优先级再水化每小时最多 10 GiB，按存储账户而不是单个 Blob 计算，吞吐在同一账户内所有正在再水化的 Blob 之间共享，超出可能被节流或延长。该页提醒：不要按单个 Blob 的“最多 15 小时”做线性外推去估批量任务。

若高优先级请求针对小于 10 GB 的已归档 Blob、且耗时超过 5 小时，Azure Storage 不按高优先级取回费率收费，仍适用标准取回费率。Set Blob Tier 再水化会收取数据读取事务和取回量；Copy Blob 还会对目标收取写入事务，但因为源仍留在 Archive，不收取提前删除费。Set Blob Tier 一旦开始不能取消；再水化期间访问层仍显示 Archive，完成后才变为新层。Set Blob Tier 不更新最后修改时间，如果账户有生命周期策略，再水化后可能因最后修改时间仍超过阈值而被重新归档。避免办法是给 `tierToArchive` 加上 `daysAfterLastTierChangeGreaterThan` 条件，或者改用复制，让新 Blob 带有新的最后修改时间。复制时，在完成前不要删除源 Blob。跨存储账户复制再水化需要同一区域，且服务版本 2021-02-12 及之后才支持。

### 生命周期管理

[Azure Blob Storage lifecycle management overview](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)（页上未见更新日期）写明，策略可以：在当前版本、早期版本或快照一段时间内未被访问或修改后，转到更冷的层；在被访问时立即从 Cool 转回 Hot；在生命周期结束时删除当前版本、早期版本或快照；规则可作用于整个存储账户、选定容器，或用名称前缀、Blob 索引标签筛选。支持通用 v2、高级块 Blob 和 Blob Storage 账户中的块 Blob 和追加 Blob。生命周期不影响 `$logs` 或 `$web` 等系统容器。

添加或编辑规则后，更改生效并开始第一次执行最多可能需要 24 小时。活动策略定期处理对象；如果修改策略，正在进行的运行会被中断。删除策略后不再调度新运行，但已经在进行的运行会继续到完成，你要为完成这次运行所需的操作付费。该页建议：先禁用策略，等 24 小时，再删除策略。生命周期管理策略本身免费；客户为 Set Blob Tier API 调用支付标准操作费用；删除操作免费。不过其他服务（页上举例 Microsoft Defender for Storage）可能对由生命周期策略管理的操作收费。每次更新 Blob 的最后访问时间归入 other operations，每个对象每 24 小时最多按一次“other transaction”计费，即使一天被访问数千次，并且与读事务费用分开。

已知限制包括：高级块 Blob 账户尚不支持分层；除这类账户外，分层只允许块 Blob，不包括追加 Blob 和页 Blob；不能用生命周期把已归档 Blob 再水化到在线层；策略必须整份读写，不支持部分更新；每条规则最多 10 个区分大小写的前缀和最多 10 个 Blob 索引标签条件；不能把使用 encryption scope 的 Blob 用生命周期改到 Archive；不可变容器中的 Blob 不会被删除动作删除；已经处于软删除状态的 Blob，删除动作也不会再生效——带软删除的账户里，生命周期删除会把对象放入软删除并保留到软删除期限结束；若 Azure Storage Action 试图删除同一 Blob，删除动作可能失败。

[Azure Blob Storage lifecycle management policy structure](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)（页上未见更新日期）写明：一个策略最多 100 条规则，规则名最多 256 个字母或数字，区分大小写，策略内唯一。过滤器若定义多个，之间是逻辑 AND；过滤器只能指定包含，不能指定排除。`prefixMatch` 必须以容器名开头，不支持通配符，`*` 和 `?` 按字面量处理。动作包括 `TierToCool`、`TierToCold`、`TierToArchive`、`enableAutoTierToHotFromCool` 和 `Delete`。对同一 Blob 定义多个动作时，生命周期应用费用最低的那个，页上的例子是删除比 `tierToArchive` 更便宜，`tierToArchive` 比 `tierToCool` 更便宜。

`TierToArchive` 不支持追加 Blob、页 Blob、高级块 Blob 账户，也不支持使用 encryption scope 的 Blob，以及配置了 ZRS、GZRS / RA-GZRS 的账户。`enableAutoTierToHotFromCool` 必须与 `daysAfterLastAccessTimeGreaterThan` 一起用，对启用该动作之前就已经在 Cool 层的 Blob 没有效果，而且每 30 天只会把 Blob 从 Cool 转到 Hot 一次，目的是避免多次提前删除罚金；该动作不支持早期版本或快照。删除动作不支持页 Blob 或不可变容器中的 Blob。若 Blob 有早期版本或快照，删除当前版本前必须先删除这些版本和快照，策略里需要同时包含它们。

运行条件都是基于时间的：`daysAfterModificationGreaterThan`、`daysAfterCreationGreaterThan`、`daysAfterLastAccessTimeGreaterThan`、`daysAfterLastTierChangeGreaterThan`（仅用于 `tierToArchive`，表示再水化后在在线层至少再停留的天数）。当前版本用该版本的最后修改时间或最后访问时间；早期版本用该版本首次写入的创建时间，不是它变成早期版本的时间；快照用快照创建时间。开启访问时间跟踪后，Get Blob 和 Put Blob 会更新 `LastAccessTime`，但 Get Blob Properties、Get Blob Metadata、Get Blob Tags 不会。为降低读延迟，最近 24 小时内只有第一次读取会更新最后访问时间。如果规则用了 `daysAfterLastAccessTimeGreaterThan` 却没开启跟踪，或 `LastAccessTime` 为空，则改用启用跟踪的日期。

未完成分片上传或未提交块是否可用生命周期清理，这次打开的生命周期概述和策略结构页都没有写，这里记为这次没读到，不补步骤。

## Google Cloud

以下来自 docs.cloud.google.com 与 cloud.google.com。

### 存储类型

[Storage classes](https://docs.cloud.google.com/storage/docs/storage-classes)（页上未见更新日期）写明，每个对象都有存储类型元数据，影响可用性和定价。创建桶时可指定默认存储类型，未指定时默认为 Standard storage；之后改桶的默认类型，不会改变已经在桶里的对象。可以重写对象或用 Object Lifecycle Management 改变现有对象的存储类型，也可以启用 Autoclass 让服务自动管理转换。

该页对照表：

- Rapid storage（`RAPID`）：无最短存储时长，无取回费，仅用于 Rapid Bucket 的 zonal bucket。
- Standard storage（`STANDARD`）：无最短存储时长，无取回费，适合频繁访问或只存很短时间的数据。
- Nearline storage（`NEARLINE`）：最短 30 天，有取回费，适合平均每月读取或修改一次或更少的数据。
- Coldline storage（`COLDLINE`）：最短 90 天，有取回费，适合最多每季度一次的访问。
- Archive storage（`ARCHIVE`）：最短 365 天，有取回费。该页明确写：与其他云厂商最冷的存储服务不同，你的数据在毫秒级可用，而不是数小时或数天。适合计划一年不到一次的访问，例如法规存档或灾难恢复。

所有主要存储类型共同的特征包括：无最小对象大小；全球可访问；低延迟且无离线数据取回；年持久性 99.999999999%。页面还列出若干旧类型（Multi-Regional、Regional、Durable Reduced Availability），并建议除非已经在用，否则改用 Standard storage。这次不把旧类型当作新桶的推荐选项。

[Cloud Storage pricing](https://cloud.google.com/storage/pricing)（页上未见更新日期）再次列出最短时长：Standard 与 Rapid 为 None，Nearline 30 天，Coldline 90 天，Archive 365 天。若在最短时长前删除、替换或移动对象，仍按最短时长计费。该页列出的例外是：对象位于 Standard 或 Rapid；由 Object Lifecycle Management 变更存储类型；对象位于已启用 Autoclass 的桶。提前删除费用在对象被删除时计；在启用对象版本控制后变为非当前版本，或在启用软删除后变为软删除，不会立刻触发，要到符合永久删除条件时才计。作为当前、非当前和/或软删除对象存在的时间，都计入最短存储时长。

该页取回费表在撰写时列出的目录价是：Rapid 与 Standard 为 $0，Nearline 为 $0.01 / 1 gibibyte，Coldline 为 $0.02 / 1 gibibyte，Archive 为 $0.05 / 1 gibibyte。这是打开该页时看到的美元目录价，若以其他货币支付，以 Cloud Platform SKUs 上的价格为准，实际账单以当时定价页为准。取回费在读取、复制、移动或重写 Nearline、Coldline 或 Archive 的对象数据或元数据时产生，并叠加在操作费和网络费之上。已启用 Autoclass 的桶、恢复软删除对象、从 Rapid Cache 读取，不收取取回费。取回费和数据传输费按实际访问的数据量计，而不是整个对象大小，例如只请求 100 MB 对象的前 8 MB，就按 8 MB 计。

该页还写：用生命周期或手动把对象改到目标存储类型时，按目标类型的 Class A 操作费率计费；当很小的对象占比很高、尤其是大量对象小于 1 MB 时，改存储类型可能对账单影响很大。页面同时列出 Always Free 用量上限，例如 Standard storage 5 GB-months，以及指定类别操作和数据传输的月度免费额度，且写明适用于 US-WEST1、US-CENTRAL1 和 US-EAST1，不适用于 Rapid storage，额度可能变更。是否仍有效，以该定价页当前说明为准，这里不把它写成长期承诺。

### 对象生命周期

[Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle)（页上未见更新日期）写明：配置必须设在桶上，包含一组规则，每条规则一个动作和一条或多条条件，对象要满足该规则的全部条件才会执行。多条规则包含同一动作时，对象满足其中任一规则就会执行。若多条规则同时满足，只执行其中一条：`Delete` 优先于任何 `SetStorageClass`；多个 `SetStorageClass` 则选择静态存储单价最低的目标类型。配置变更最多可能 24 小时才生效，在此期间仍可能按旧配置执行。该页举例：把 `age` 从 10 天改成 20 天，一个已 11 天的对象仍可能在最多 24 小时内按旧规则被删除。

动作为以下之一：`Delete`、`SetStorageClass`、`AbortIncompleteMultipartUpload`。

- `Delete`：默认情况下，删除实时对象会使其进入软删除，并保留七天，可在保留期内恢复。若桶启用了 Object Versioning，删除实时版本会让它变成非当前版本；删除非当前版本才是从桶中删掉该版本。对象仍有 hold，或尚未满足保留策略时，Delete 不会生效，条件持续满足则在 hold 解除且保留期满后执行。
- `SetStorageClass`：改变存储类型，并在条件满足时更新对象的修改时间。支持的转换包括：Standard / Multi-Regional / Regional → Nearline、Coldline 或 Archive；Nearline → Coldline 或 Archive；Coldline → Archive。Cloud Storage 不会校验转换是否在上表中，你可以写出表中没有的转换，但实际不会发生。不能在已启用 Autoclass 的桶上设置 `SetStorageClass`。
- `AbortIncompleteMultipartUpload`：中止未完成的分段上传并删除相关分片。只能与 `age`、`matchesPrefix`、`matchesSuffix` 一起使用，和其他条件组合会报错。

条件包括 `age`、`createdBefore`、`customTimeBefore`、`daysSinceCustomTime`、`daysSinceNoncurrentTime`、`isLive`、`matchesStorageClass`、`matchesPrefix` 与 `matchesSuffix`、`noncurrentTimeBefore`、`numNewerVersions`、`sizeAboveBytes` 与 `sizeBelowBytes`。全部可选，但至少要有一个。`age` 从资源创建时间起算，单位是天；对对象而言，创建时间是成功写入桶、例如上传完成的时间，对象变为非当前版本不会改变年龄；对分段上传，创建时间是上传发起的时间。`age` 为 0 时，条件在对象创建后的 UTC 午夜满足。

### Autoclass

[Autoclass](https://docs.cloud.google.com/storage/docs/autoclass)（页上未见更新日期）写明：启用后，加入桶的对象都从 Standard storage 开始，即使请求里指定了其他类型也会被忽略；桶的默认存储类型保持为 Standard。默认的终点类型是 Nearline，对象转到 Nearline 后一直停留到再次被访问；也可以把终点配成 Archive。小于 128 KiB 的对象不向更冷类型转换，永久放在 Standard；判断大小时只看对象数据，不包括元数据。读取对象数据会把对象转回 Standard；只读或编辑元数据不会。软删除对象在保留期内保持原存储类型；恢复后的新对象从 Standard 开始，不论软删除时是什么类型。

转换节奏：对象数据被访问则转到 Standard；至少 128 KiB 且 30 天未访问转到 Nearline。若终点是 Archive：90 天未访问转到 Coldline（其中至少 30 天在 Standard、60 天在 Nearline）；365 天未访问转到 Archive（再加至少 275 天 Coldline）。最后访问时间只在使用 `get` 对象方法访问时更新，`copy` 和 `rewrite` 不更新。转换是异步的，资格到达和实际转换之间可能有滞后；滞后期间通常仍按转换前的类型计费，因启用 Autoclass 而转到 Standard 的情况除外。

定价例外（该页原文）：使用 Autoclass 会有管理费和启用费用；取回费和提前删除费除作为启用费用的一部分外不另收；所有操作按 Standard 费率计；Autoclass 把对象转到更冷类型不收操作费；从 Nearline 转回 Standard 不收 Class A 操作费；从 Coldline 或 Archive 转到 Standard 或 Nearline 时，每次转换收一次 Class A 操作费。对已有桶启用 Autoclass 时，除软删除对象外全部转到 Standard，并产生一次性启用费用；已在 Standard 的对象会被视为刚刚转入，需要再过 30 天无访问才有资格转 Nearline。禁用后，对象停留在当时的类型，定价结构不再适用，且一天之内不能重新启用。

限制：桶不能同时启用 Autoclass，又在生命周期里使用 `SetStorageClass` 动作或 `matchesStorageClass` 条件，这样的请求会失败。该页还提示：如果多数数据已经明确属于某一类存储的使用场景，或有其他 Google Cloud 服务定期读取该桶（页上举例 Sensitive Data Protection 扫描），则不建议用 Autoclass。

### 软删除与生命周期的关系

[Soft delete](https://docs.cloud.google.com/storage/docs/soft-delete)（页上未见更新日期）写明：对支持该功能的桶默认启用，默认保留 7 天，可在 7 到 90 天之间自定义；把保留时长设为 0 即禁用。改保留时长只影响之后删除的资源。软删除对象和桶默认不能读、改、覆盖，只能列出或恢复。Object Lifecycle Management 规则不影响已经软删除的对象，不能用生命周期去改它们的存储类型或把它们永久删除；被生命周期删除的对象会变成软删除。若同时启用了 Object Versioning，删除实时对象会变成非当前版本，删除非当前版本才变成软删除。XML API 分段上传的分片不受软删除保护。Autoclass 不对软删除对象收管理费，也不改变它们的存储类型；恢复后的对象是 Standard。软删除对象在保留期内继续产生存储费用。该页提醒：主要存放短生命周期临时数据的桶，启用软删除可能显著增加存储费用。

## 检查清单

按这个顺序做。每一步先确认业务还要不要读这些对象，再改类型或设删除规则。规则一旦执行，删除通常不能当没发生过；归档取回也往往要等待，并另计费用。

1. 先盘点桶和前缀里的数据，而不是先抄一条“30 天转低频、90 天转归档”的模板。看对象大小分布、访问频率、是否还要实时读、必须保留多久。私有权限和临时访问链接的核对见 [对象存储私有桶与临时访问链接](object-storage-private-bucket-presigned-url.md)，这里不重复签名设置。
2. 对照账号所属站点的当前文档，确认该地域实际支持哪些类型。阿里云冷归档、深度冷归档并非所有地域都有，且只支持本地冗余；腾讯云多 AZ 类型、智能分层和深度归档都有地域名单，中国香港、新加坡等页上写了要加白；Azure 的 Archive 不支持 ZRS / GZRS / RA-GZRS 账户。国际站没打开的规则，不要用中国站菜单去改国际站桶。
3. 先算最短时长和起算点，再写转换天数。阿里云低频和归档从 Last Modified 起算，冷归档和深度冷归档从转为该类型时起算。腾讯云归档最短是 90 天，不是阿里云的 60 天；Google Cloud 的 Archive 最短是 365 天，而且生命周期改类型本身不收提前删除费，这条不要套到阿里云或腾讯云。AWS 不允许在同一条规则里、在最短时长结束前再转出。
4. 明确冷数据怎么读。若业务可能在数分钟内要下载原文件，不要只看单价就进腾讯云归档、AWS Flexible Retrieval / Deep Archive，或 Azure Archive。阿里云归档若要直读，需确认已开启归档直读并理解直读费用。Google Cloud Archive 是毫秒级可读，但仍有取回费。
5. 评估小对象和请求次数。大量小于 64 KB（阿里云、腾讯云）或小于 128 KB（AWS IA 与 Glacier Instant Retrieval，以及生命周期默认不转换的对象）的文件，计费容量或转换请求可能高于省下的存储费。腾讯云生命周期不会转换小于 64 KB 的对象，这些小文件会留在原类型。Google Cloud 定价页提醒，大量小于 1 MB 的对象改类型时，Class A 操作费可能很显著。
6. 为未完成分片上传单独加一条清理规则，和对象过期删除分开。阿里云可用 AbortMultipartUpload；腾讯云可按碎片创建后天数清理；AWS 建议 AbortIncompleteMultipartUpload，页上写明中止未完成分片不收 early delete charges；Google Cloud 的该动作只能搭配 age 与前缀/后缀。Azure 这一项这次没读到，不在清单里假装已经有官方步骤。
7. 开启版本控制、软删除或对象保留策略的桶，要按文档理解“删除”实际落到哪一层。阿里云在 ObjectWorm 保留期内不会被生命周期 Expiration 删掉。Azure 生命周期删除在启用软删除的账户里会进入软删除保留。Google Cloud 默认软删除 7 天，生命周期删除会变成软删除；软删除对象在保留期内仍计存储费，也会计入最短时长，但提前删除费要到永久删除时才产生。
8. 先在测试桶或用更窄的前缀验证，再应用到生产。阿里云生命周期概述和 Google Cloud 生命周期页都建议先用测试数据或小范围前缀验证。改规则后不要假定立刻生效：阿里云、Azure、Google Cloud 已打开页面都写了最多约 24 小时的加载或生效窗口；腾讯云是次日北京时间 0 点开始扫描。验证时用 HEAD/GET 看对象当前类型和是否仍在，不要只看规则是否已保存。
9. 规则执行后的下一小时、下一天再看账单和监控。生命周期请求费、取回费、不足时长费用可能和存储容量费分项出现，或像腾讯云那样合并在存储容量费用里。账单里还有这一项，先对消费时间，再判断规则是不是已经执行、对象是不是还在。

## 常见坑

- 只比较每 GB 存储单价，没把取回费、解冻费、转换请求费和最短时长补收费算进去。阿里云写过：月均访问高于 1 次时，转到低频或归档后总成本可能更高。AWS 定价页也提醒，用生命周期转入任何存储类型都有 ingest 或转换请求费。
- 把各云的“归档”当成同一套天数。阿里云归档最短 60 天，腾讯云归档最短 90 天，AWS Glacier Instant Retrieval 与 Flexible Retrieval 是 90 天、Deep Archive 是 180 天，Azure Archive 是 180 天，Google Cloud Archive 是 365 天。不要用其中一朵云的天数去配另一朵云的规则。
- 忽略起算点。阿里云把低频、归档的不足时长按 Last Modified 算，冷归档和深度冷归档按转入该类型的时间算；生命周期计费页的例子里，转冷归档后只存 1 天就删除，要补 179 天。AWS 在规则条件满足时就开始按目标类型和最短时长计费，即使物理转换还没完成（转到 Intelligent-Tiering 除外）。
- 腾讯云不会帮你拦截过短的生命周期。概述页写明：即使规则短于 30/90/180 天，也会按你的配置执行并可能收取提前删除费用。未开启版本控制时，上传同名对象覆盖也会触发提前删除。
- 以为归档数据随时能读。腾讯云归档、AWS Flexible Retrieval 与 Deep Archive、Azure Archive 都要先恢复或再水化。Azure 用 Set Blob Tier 再水化不会更新最后修改时间，生命周期可能把刚恢复的 Blob 再归档回去。AWS 恢复后存储类型仍是 Glacier 类，你同时为归档和临时 Standard 副本付费；阿里云和腾讯云的冷归档或深度归档同样会为临时副本计标准存储或临时存储。
- 海量小文件直接进低频或归档。64 KB 或 128 KB 的最小计费规格会放大实际容量；腾讯云生命周期还不会转换小于 64 KB 的对象，它们会一直停在原类型。AWS 默认不转换小于 128 KB 的对象，若手动打开小对象转换，转换请求费可能更高。
- 只清理对象、不清理未完成分片。阿里云、腾讯云、AWS、Google Cloud 已打开页面都写了分片会继续计存储费。Google Cloud 还写明软删除不保护 XML API 分段上传分片，中止或未使用的分片可能产生提前删除费用。
- 用 API 覆盖生命周期配置时只提交新规则。阿里云页写明 PutBucketLifecycle 会完全覆盖当前全部规则。Azure 页写明策略必须整份读写，不支持部分更新。
- 把 Google Cloud 生命周期改类型“不收提前删除费”理解成改类型完全免费。定价页写这条例外只针对提前删除费；改类型仍按目标类型收取 Class A 操作费，小对象很多时这笔费用可能很高。Autoclass 与生命周期里的 `SetStorageClass` 或 `matchesStorageClass` 不能并用。
- 在国际站账号上直接套用中国站文档，或在 AWS 中国区账号上直接套用这次打开的全球文档操作步骤。腾讯云国际站对应页这次是 404；AWS 中国区文档和世纪互联 Azure 都没有打开。

## 参考来源

资料核对日期：2026-11-18。下面只列出这次实际打开并据以写稿的页面。页上能看到更新日期的，写在标题后；看不见的，不补。

阿里云中国站：

- [OSS有哪几种存储类型](https://help.aliyun.com/zh/oss/user-guide/overview-53/)（页上「更新时间」栏未见具体日期）
- [什么是OSS生命周期](https://help.aliyun.com/zh/oss/user-guide/overview-54/)（页上「更新时间」栏未见具体日期）
- [OSS存储费用的计费项](https://help.aliyun.com/zh/oss/storage-fees)（页上「更新时间」栏未见具体日期）
- [通过生命周期规则转换或删除Object是如何计费的](https://help.aliyun.com/zh/oss/user-guide/fees-related-to-lifecycle-rules)（页上「更新时间」栏未见具体日期）
- [转换文件存储类型](https://help.aliyun.com/zh/oss/user-guide/convert-storage-classes)（页上「更新时间」栏未见具体日期）
- [解冻文件](https://help.aliyun.com/zh/oss/user-guide/restore-objects-for-access)（页上「更新时间」栏未见具体日期）
- [删除碎片](https://help.aliyun.com/zh/oss/user-guide/delete-parts)（页上「更新时间」栏未见具体日期）

没有打开中国站的 OSS 产品定价页、生命周期配置元素专页，以及「如何避免产生存储不足规定时长容量费用」的完整专页。可见「更新时间」栏在抓取结果中没有给出具体日期，不以页面源码里的隐藏字段代替“页上能看到的日期”。

阿里云国际站：

- [Storage classes](https://www.alibabacloud.com/help/en/oss/user-guide/overview-53/)（Last Updated: Sep 23, 2026）
- [Lifecycle overview](https://www.alibabacloud.com/help/en/oss/user-guide/overview-54/)（Last Updated: Oct 21, 2026）
- [Storage fees](https://www.alibabacloud.com/help/en/oss/storage-fees)（Last Updated: Oct 20, 2026）
- [Billing for transforming or deleting objects using lifecycle rules](https://www.alibabacloud.com/help/en/oss/user-guide/fees-related-to-lifecycle-rules)（Last Updated: Sep 21, 2026）

国际站解冻专页、删除碎片专页和产品定价页这次没有打开。最短时长和生命周期计费与中国站已打开的对应页一致；解冻优先级、副本有效期和配额只写在中国站解冻页。

腾讯云中国站：

- [存储类型概述](https://cloud.tencent.com/document/product/436/33417)（最近更新时间：2026-10-12 14:51:04）
- [生命周期概述](https://cloud.tencent.com/document/product/436/17028)（最近更新时间：2025-08-04 17:57:32）
- [存储容量费用](https://cloud.tencent.com/document/product/436/53482)（最近更新时间：2026-10-12 14:51:03）
- [数据取回费用](https://cloud.tencent.com/document/product/436/53862)（最近更新时间：2026-08-18 17:21:57）
- [恢复归档对象](https://cloud.tencent.com/document/product/436/32430)（最近更新时间：2026-04-08 15:18:42）
- [删除碎片文件](https://cloud.tencent.com/document/product/436/17313)（最近更新时间：2026-03-13 15:24:42）

没有打开「智能分层存储简介」专页，所以没有写智能分层各层自动转换的天数。产品定价中心的完整价目表没有抄进正文。

腾讯云国际站这次没有打开。`https://www.tencentcloud.com/document/product/436/33417` 与 `https://intl.cloud.tencent.com/document/product/436/33417` 均返回 404。

AWS：

- [Understanding and managing Amazon S3 storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)（页上未见更新日期）
- [Transitioning objects using Amazon S3 Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)（页上未见更新日期）
- [Lifecycle configuration elements](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)（页上未见更新日期）
- [Understanding archive retrieval options](https://docs.aws.amazon.com/AmazonS3/latest/userguide/restoring-objects-retrieval-options.html)（页上未见更新日期）
- [Configuring a bucket lifecycle configuration to delete incomplete multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html)（页上未见更新日期）
- [Amazon S3 定价](https://aws.amazon.com/s3/pricing/)（页上未见更新日期）

没有打开 docs.amazonaws.cn，也没有打开 S3 FAQ 里关于 Glacier 提前删除的单独问答页。过期删除（Expiration）的专门说明页没有按同样深度打开，正文里的过期规则以生命周期配置元素页已写明的部分为准。

Microsoft Azure：

- [Access tiers for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)（页上未见更新日期）
- [Azure Blob Storage lifecycle management overview](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)（页上未见更新日期）
- [Azure Blob Storage lifecycle management policy structure](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-policy-structure)（页上未见更新日期）
- [Blob rehydration from the archive tier](https://learn.microsoft.com/en-us/azure/storage/blobs/archive-rehydrate-overview)（页上未见更新日期）
- [Azure Blob Storage pricing](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/)（页上未见更新日期；单价为占位符）
- [Azure Blob 存储定价](https://azure.microsoft.com/zh-cn/pricing/details/storage/blobs/)（页上未见更新日期；单价为占位符）

没有打开 Smart tier 专页、未提交块或未完成分片的清理页，以及由世纪互联运营的 Azure 文档和定价。Learn 文档的中文版这次没有作为对照页打开。

Google Cloud：

- [Storage classes](https://docs.cloud.google.com/storage/docs/storage-classes)（页上未见更新日期）
- [Object Lifecycle Management](https://docs.cloud.google.com/storage/docs/lifecycle)（页上未见更新日期）
- [Autoclass](https://docs.cloud.google.com/storage/docs/autoclass)（页上未见更新日期）
- [Soft delete](https://docs.cloud.google.com/storage/docs/soft-delete)（页上未见更新日期）
- [Cloud Storage pricing](https://cloud.google.com/storage/pricing)（页上未见更新日期）

如需为阿里云、腾讯云、Google Cloud、AWS、Microsoft Azure 的国际站账号开户或充值，可访问海宝云（CloudCup）官网 <https://www.haibaocloud.com>。官网常见问题当前写明：充值仅支持 USDT-TRC20，单笔最低 50 USDT；其他币种或法币需联系客服定制。具体以官网当前说明为准。Telegram 客服：@cloudcup。
