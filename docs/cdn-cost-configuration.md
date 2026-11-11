# CDN 费用配置与防超支：流量、请求、回源与用量上限

CDN 账单超支，通常不是“开了加速就固定一个月费”。边缘把内容发给用户、请求次数、可选的 HTTPS 或 QUIC、以及源站自己的出方向，会落在不同计费项上。缓存命中率高，减少的是回源，不一定把边缘下行也变成零。用量封顶、限速和预算告警也不是同一件事：有的会把域名停掉，有的只发通知，有的只把单个连接放慢。

云服务器的计算、磁盘、公网和快照不在这里重复，见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。对象存储被 CDN 回源或被公共地址引用时怎么计流出，见 [对象存储私有桶与临时访问链接](object-storage-private-bucket-presigned-url.md)。本文不写单价。阶梯价、分区价和套餐月费以各页当前表格为准。

核对时间：2026-11-11。资料核对日期：2026-11-11。控制台路径、默认值和限制都来自这一天打开的官方页面，并附有链接。阿里云中国站在 help.aliyun.com，国际站计费概述在 www.alibabacloud.com。腾讯云 CDN 的计费、封顶和缓存来自中国站 cloud.tencent.com；国际站 CDN 文档这次没有打开。AWS 使用 aws.amazon.com 与 docs.aws.amazon.com，没有打开 AWS 中国区 docs.amazonaws.cn。Azure Front Door 计费使用 learn.microsoft.com 的全球文档中文页；由世纪互联运营的 Azure 没有单独打开产品计费页。页面改版后，以你账号所属站点的当前文档为准。

## 先把 CDN 账单拆开

按这个顺序看，比先记一个“每 GB 单价”有用：

1. 确认产品是不是这篇写的 CDN。阿里云在 CDN 控制台里业务类型为全站加速的域名，按全站加速价格计费，和 CDN 域名不会同时存在，也不重复计费。腾讯云全站加速 ECDN、安全加速 SCDN 有各自的计费文档，不要套用下面的 CDN 表。
2. 基础服务先分清是按流量还是按带宽。两种通常只能取一种。流量包往往只能抵按流量的账。
3. 再看可选计费项有没有打开：HTTPS 请求、QUIC、实时日志、WAF、刷新失效。没开的功能，对应文档写明不产生该项费用。
4. 回源单独看。多家 CDN 文档写明 CDN 侧不计回源流量，源站（云服务器、对象存储或另一家云）仍可能为出方向计费。
5. 最后看有没有“到阈值就停服务”的开关。预算邮件和云监控告警，文档写的是通知；域名下线才是停加速。

出账有延迟。阿里云 CDN 按小时的账通常在计费周期结束后 3～4 小时。腾讯云按小时的消耗在其后 2～4 小时内计算并扣费。看监控比等账单更快，但监控触发下线本身也有延迟，见后面的封顶一节。

## 计费项

下面只列打开的官方页面写出的计费项。单价、阶梯金额和示例演算不抄。

### 阿里云 CDN

中国站 [计费概述](https://help.aliyun.com/zh/cdn/product-overview/billing-overview) 写明，计费按账户 UID 汇总。没有特别说明时，按流量、按带宽峰值和月结 95 带宽峰值，都指 L1 节点的下行流量或带宽。

基础服务三选一，默认按流量：

| 方式 | 文档写明的计量 | 出账 |
| --- | --- | --- |
| 按流量（默认） | 每月从节点流出的实际流量，阶梯计费。可用资源包抵扣，超出部分仍按量后付费 | 按小时。场景说明是流量波动大、全天带宽利用率小于 30%。30% 是选型说明，不是计费阈值 |
| 按带宽峰值 | 每 5 分钟一个点，一天 288 个点，取最大值 | 按日。每日凌晨 4 点左右出前一日账单。开通条件见下一节 |
| 月结 95 带宽峰值 | 每 5 分钟一个点，按月去掉最高的约 5% 之后取值，再乘有效因子 | 下个自然月 1 日凌晨 4 点左右。有效因子是当月有效天数除以当月总天数 |

下行流量是用户从边缘下载产生的流量，计入基础服务。回源流量不计入 CDN 计费；源站仍可能产生出方向费用。计费区域按访问用户 IP 判断，中国站概述列出中国内地、北美、欧洲、亚太 1/2/3、中东/非洲、南美。日志里的流量小于计费流量，因为计费包含 TCP/IP 包头和重传。见 [基础服务计费](https://help.aliyun.com/zh/cdn/product-overview/billing-rules-of-basic-services)。

增值服务单独计，见 [增值服务计费](https://help.aliyun.com/zh/cdn/billing-of-value-added-services-1)：

- 静态 HTTPS 请求数：开启 HTTPS 后产生，按量或资源包，按小时出账，延迟 3～4 小时。CDN 默认不产生动态 HTTP/HTTPS 请求账单；动态请求出现在使用全站加速且存在动态请求时。
- 静态 QUIC 请求数：开启 QUIC 后按量、按小时。
- CDN WAF 请求数：只支持中国内地加速节点。按开启了 WAF 的域名、在账户维度汇总请求次数。
- 实时日志投递：成功投递后按条数、按小时。

[请求数计费常见问题](https://help.aliyun.com/zh/cdn/product-overview/faq-about-the-billing-of-requests) 写明，请求数按应用层访问日志统计，获取失败的访问也会计入。浏览器再次请求、节点返回 304 时，仍可能产生请求数费用和很少的流量费用。同一页写明，自 2024 年 2 月 1 日起，静态 HTTPS 请求数每月前 500 万次免费，超出后计费。这是免费额度，不是单价。被刷请求数会产生费用；开了 IP 黑白名单并封禁 IP，该页写可以减少流量费，请求数是否仍会计入，以该页“获取失败仍统计”的原则和实际账单为准。

国际站 [计费概述](https://www.alibabacloud.com/help/zh/cdn/product-overview/billing-overview)（页面更新时间 Oct 19, 2026）同样把默认写成按流量，回源不计入 CDN、源站可能另计，出账延迟同样是 3～4 小时。这一页的基础服务只写按流量和按带宽峰值两种，计费周期表也只有小时和日，没有月结 95。中国站概述把月结 95 写成第三种基础服务。国际站账号不要按中国站的三种方式理解；这次没有打开国际站的变更计费方式页，不把中国站的 5 Gbps、月消费金额条件套过去。

CDN 与全站加速可共享下行流量包和静态 HTTPS 请求数包，这句来自中国站概述。国际站这一页没有同样的句子。

### 腾讯云 CDN

以下来自中国站。[计费概述](https://cloud.tencent.com/document/product/228/2949)（最近更新 2026-02-11）写明，基础服务是流量或带宽二选一，CDN 侧不收取回源流量费用。开通时各计费项的结算周期默认是小时结；要月结需联系商务。

| 计费项 | 文档写明的范围 |
| --- | --- |
| 按流量 | 从节点流出的实际流量，月度阶梯累进，可用流量包抵扣。小时结：前 1 小时的消耗在后 2～4 小时内扣费。月结流量是另一档，见基础服务计费 |
| 按峰值带宽 | 每 5 分钟一个点，一天 288 个点，按计费策略取值。不能用流量包抵扣。带宽值按该页所链的基础服务计费：5 分钟流量换算，进制写明 1 GB = 1000 MB |
| QUIC 访问请求数 | 未使用则不产生 |
| HTTPS 请求数 | 2023 年 1 月 5 日起，开启 HTTPS 后按请求数单独计费，小时结 |
| APK 动态打包请求数 | 使用了该功能且收到计费通知才计 |
| 实时日志投递 | 按投递日志量。该概述写功能处于内测，需联系开通 |

计费区域分中国境内和中国境外。境内用统一的中国大陆价格，不再按省拆。境外按节点所在服务地区分成 8 个大区：亚太 1、亚太 2、亚太 3、中东、欧洲、北美、南美、非洲。全球加速同时产生两边用量时，费用相加。中国香港、中国澳门在亚太 1，中国台湾在亚太 2，不要和“中国境内”混成一个价。

[基础服务计费](https://cloud.tencent.com/document/product/228/75562) 把月结写成需月消费金额大于 10 万元或预期超过 10 万元，且无历史欠费，再联系商务。月结带宽有两种算法：日峰取月均，以及月 95。月度有效天的定义是自然月某一天用量大于 1000 bps。小时结用户不要把月 95 的算法套到自己的日账单上。

[明细账单](https://cloud.tencent.com/document/product/228/6071) 写明，CDN 按账号计费，不按项目或地域分摊。账单里项目统一写成默认项目，可用区写成其他。标签分账目前该页写只支持中国站，且要月结，计费方式为流量、带宽或 HTTPS；其他增值项暂不支持。带宽按标签分出的“计费带宽”是按流量比例反算的分摊值，不是该域名的真实峰值。

### Amazon CloudFront

中文 [定价页](https://aws.amazon.com/cn/cloudfront/pricing/) 当前主体是固定费率方案：把 CDN 与该页列出的 WAF、DDoS 防护、DNS、日志摄取、TLS 证书、无服务器边缘计算以及一份 Amazon S3 存储抵扣金放进按月的方案，并写明不收取超额费用。方案名称是免费、Pro、业务、高级和自定义定价。页面列出各方案的每月请求额度、数据传输额度、WAF 规则条数和包含的 S3 存储额度，月费以该页当前数字为准，这里不抄。

固定费率有两点和“封顶”不是一回事：

- 页面写即使流量高峰或遭受攻击也不产生超额费用。它没有写用量达到额度后分配会自动停止服务。
- 被拦截的 DDoS 以及被 AWS WAF 拦截的请求，不计入使用额度。没有被拦截、仍然由 CloudFront 处理的流量，仍计入额度。
- 该页常见问题写明，固定费率方案不得与其他优惠、促销或折扣叠加。创建分配时可以选择该方案，也可以把已有分配从即用即付改过来。

即用即付的账单行，以 [账单与用量报告解读](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/billing-and-usage-interpreting.html) 列出的收费名为准。地区用两位缩写，例如 US、EU、AP、JP。主要行是：

- `region-DataTransfer-Out-Bytes`：边缘响应 `GET`、`HEAD` 的字节。用量类型再按 HTTP/HTTPS、以及 TTL 是否达到 3600 秒分成静态和动态。`DELETE`、`OPTIONS`、`PATCH`、`POST`、`PUT` 的响应字节另记为 Proxy，包含 gRPC。
- `region-DataTransfer-Out-OBytes`：上述方法从边缘传到源站或边缘函数的字节，包含 WebSocket 从客户端到服务端的数据。
- `region-Requests-Tier1`：HTTP `GET` 和 `HEAD`。
- `region-Requests-Tier2-HTTPS`：HTTPS `GET` 和 `HEAD`。
- HTTP/HTTPS Proxy 请求：转发到源站或边缘函数的 `DELETE`、`OPTIONS`、`PATCH`、`POST`、`PUT`，以及带 `Upgrade: websocket` 的 `GET`。HTTPS Proxy 还包含 gRPC。字段级加密另有 `Requests-HTTPS-Proxy-FLE`。
- Origin Shield：`region-Requests-OriginShield` 是作为增量层到达 Origin Shield 的请求数。动态、不可缓存且回源的请求始终算增量层；可缓存请求有时算。还有源与区域边缘缓存之间的字节行。估算公式在该页指向的 Origin Shield 文档，本文不另补没有在计费表里的单价。
- `Invalidations`：失效路径。每月每个账户、所有分配合计，前 1000 条路径不收费，超出后按路径计。`/images/*` 这种通配路径算 1 条，即使它让很多文件失效。缓存标签失效和路径失效共用这 1000 条。见 [失效路径如何计费](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PayingForInvalidation.html)。
- `SSL-Cert-Custom`：使用自定义 SSL 证书且选择传统客户端（专用 IP）时的费用。[分配设置](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesGeneral.html) 写明这项会产生额外月费，并建议能用 SNI 时不要选传统客户端。
- 实时日志行数、CloudFront Functions 调用次数、Lambda@Edge 的调用次数和 GB-秒、KeyValueStore 的读取与 API 调用。

同一篇解读写明，表内不包含从 Amazon S3 存储桶传到 CloudFront 边缘的费用；如果有，出现在账单的 AWS Data Transfer 部分。固定费率常见问题另写，经 CloudFront 提供流量时，CloudFront 与 AWS 源之间的数据传输费用免除。两处覆盖的计费模式不同。对账时看账单上的实际行，不要把固定费率那句直接套到即用即付的 Data Transfer 行。

标准日志：分配设置写明开启日志本身不另收 CloudFront 费用，存放和读取日志文件可能产生存储费用。

价格等级不是用量上限。[分配设置](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesGeneral.html) 写明，默认从全部 CloudFront 区域的边缘节点提供对象。选更低的价格等级后，只使用该等级里延迟最低的边缘节点，等级外的观众可能更慢。CLI 与 CloudFormation 允许的取值包括 `PriceClass_100`、`PriceClass_200`、`PriceClass_All`，CLI 还列出 `None`。该字段只适用于标准分配，多租户分配不能设。这次打开的中文定价页没有展开价格等级和区域的对照表，区域名单以定价页当前说明为准。

### Azure Front Door 与 Azure CDN

这次按全球文档核对的现行计费对象是 Azure Front Door 标准版和高级版，见 [Azure Front Door 计费](https://learn.microsoft.com/zh-cn/azure/frontdoor/billing)。经典 Front Door 的价格该页指向定价页，本文不另写一套经典单价。

[产品比较](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-cdn-comparison) 写明：Front Door（经典）于 2027 年 3 月 31 日停用；Azure CDN Standard from Microsoft（经典）于 2027 年 9 月 30 日停用。页首时间线写，自 2025 年 8 月 15 日起，这两档都不再支持新建配置文件、新域名和托管证书。同页停用表又把“停止创建新资源”的截止日期写成 Front Door（经典）2025 年 3 月 31 日、CDN Standard from Microsoft（经典）2025 年 8 月 15 日。两处日期不一致。新建前以当前门户是否还允许创建为准。新配置使用 Front Door 标准版或高级版。由 Akamai、Verizon 提供的历史 Azure CDN 档位，这次没有打开对应的现行计费页，不写它们的计费项。

标准版和高级版的计费组成：

- 每个配置文件按小时收基本费用，不足 1 小时按 1 小时。一个配置文件里多个终结点不按终结点另收基本费。费率随层级变化，数字在定价页。
- 从客户端到边缘的请求数。用请求的 `Host` 头对应到配置文件。标准版和高级版价格不同，并随处理请求的边缘位置所在地理区域变化。
- 从边缘发往源站的字节。缓存命中、没有向源站发请求时，这项不收。流量越大，每 GB 的价格档越低。源站在哪个区域不改变这项的区域归属，区域看的是处理请求的边缘。
- 从源站发回 Front Door 的数据，Front Door 不计。源站在 Azure 上时，该页写明不收从 Azure 源站到 Front Door 的出站数据传输费；源站那项 Azure 服务仍可能为处理请求收费。源站在 Azure 外时，其他网络商可能收费。
- 从边缘发回客户端的字节。响应如果压缩了，只对压缩后的数据收费。
- 该页写明，流量加速、响应缓存、响应压缩、规则引擎、Front Door 自带的 DDoS 防护、自定义 WAF 规则不另收费。高级版使用托管 WAF 规则集或专用链接源，也不另收费。专用链接流量相对公网源不另收一笔传输费。
- 规则集在边缘执行的操作，中文计费页另有“调用”和“超额执行时间”两个计量。示例换算以该页为准。

[常见问题](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-faq) 写明，Front Door 资源不能停用，只能删除。没有流量时，数据传输和请求这些随用量变化的计量不收；基本费用在配置文件删除前一直收，包括没有任何流量的时候。经典版的 WAF 策略和规则即使禁用仍计费。标准版和高级版的自定义 WAF 是否另收费，以计费页“自定义 WAF 规则不另收费”那句为准，不要和经典版的 WAF 计费混用。

WAF 拦截请求时，请求不到源站，但 Front Door 仍对请求和返回给客户端的响应计费。计费页的例子是自定义错误页 1 KB 仍计入边缘到客户端的传输。

### Google Cloud CDN

[Cloud CDN 价格](https://cloud.google.com/cdn/pricing?hl=zh-cn) 按三种情况列收费项。非美元账户以 Cloud Platform SKU 上的本币价格为准。

| 情况 | 收费项 |
| --- | --- |
| 可缓存内容，缓存命中 | 缓存查询 + 缓存数据传出。`GET` 和 `HEAD` 会引发缓存查询，`POST` 和 `PUT` 不会 |
| 可缓存内容，缓存未命中 | 上面两项，再加缓存填充，以及适用的 Cloud Load Balancing 数据处理或 Cloud Storage 操作费。这些费用取代直接从 Compute Engine 或 Cloud Storage 提供服务时的数据传输费。外部后端的缓存填充，对外部后端的请求按互联网数据传出费率计；边缘和来源之间仍走 Google 骨干网 |
| 无法缓存的内容 | 缓存查询 + 适用的负载均衡数据处理 + 标准 Compute Engine 或 Cloud Storage 互联网数据传出。外部后端还要对发往外部后端的请求字节按互联网传出计费 |

因此，后端开着 Cloud CDN、但响应不可缓存时，仍然会产生缓存查询。缓存命中不再叠加缓存填充。该页写，传送热门内容的典型工作负载里，缓存填充通常少于总数据传输 GiB 的 10%。这是页面上的典型情况说明，不是承诺。

缓存数据传出按项目、按流量离开 Google 网络的目的地、按月分档。目的地包括亚太（含香港）、中国、欧洲、北美洲（含夏威夷）、大洋洲、南美洲和其他目的地。进入中国大陆的流量由中国大陆以外的网点送出，该页注明性能和可靠性可能不同于大陆境内网点。缓存填充分三种：北美或欧洲境内、亚太/南美/中东/非洲/大洋洲各自境内、跨地区。选择性加入网络区域并使用 BYOIP 时，从雅加达、多哈、达曼传出还有附加费，表在同一页。

缓存数据传出和缓存填充的计费字节包含 HTTP 响应标头（压缩后）、响应正文和尾部。费用按天累计，在结算周期结束时收取。未结算用量在 Google API 控制台的项目结算页查看。

图片优化该页写目前是预览，预览期间免费，正式发布后开始收费。失效（cache invalidation）是否另收费，这一页没有列出单独的失效计费项，这里不补。

价格计算器：<https://cloud.google.com/products/calculator?hl=zh-cn>。

## 计费方式怎么切换

### 阿里云

中国站自 2021 年 1 月 8 日起默认按流量。[变更计费方式](https://help.aliyun.com/zh/cdn/product-overview/change-the-metering-method) 写明：

- 已购买下行流量包时，计费方式必须是按流量，否则不能抵扣。
- 提交变更后，生效前不能取消，也不能再次变更。生效前仍按原方式扣费。
- 从按带宽峰值或月结 95 改回按流量：CDN 控制台概览页右侧查看当前方式，点变更，确认生效时间后，在变配页勾选专属条款并立即购买。
- 其他方向，例如从按流量改到月结 95，要联系客户经理。同意后才会打开控制台选项。改到按带宽峰值，业务需近 30 天带宽峰值超过 5 Gbps。改到月结 95，需月消费金额大于 10 万元。这是中国站页面上的条件。国际站是否同一门槛，这次没有打开国际站的变更页。
- 生效时间：按流量改出、以及按日的带宽峰值改出，都是次日 0 点。月结 95 改出是次月 1 日 0 点。

月消费不足 10 万元，或流量波形不符合按月模式（月 95 不能用于异常突发），基础服务计费页写明阿里云保留把月计费改回按流量的权利。

### 腾讯云

[变更计费方式](https://cloud.tencent.com/document/product/228/3077)（最近更新 2026-02-10）写明，自 2020 年 12 月 7 日 21:30 起开通的 CDN，基础服务默认是按流量小时计费。按流量小时计费不能自行改成按带宽峰值；签约大客户要改带宽结算，联系商务经理。

当天还是带宽计费、尚未产生消耗时改成流量，次日结算按流量，前提是中途不再改。当天已经产生消耗再改成流量，次日仍按带宽结算；若不再改，第三天结算第二天的消耗时按流量。改成带宽计费后，有效流量包不能再抵扣。

月结不是控制台上的自助开关，见上一节的 10 万元和欠费条件。

### CloudFront、Front Door、Cloud CDN

这三家这次打开的页面没有“按流量 / 按带宽峰值”这种二选一的切换。CloudFront 是固定费率方案和即用即付之间的选择，在创建或修改分配时决定。价格等级改变的是使用哪些边缘区域，不改变请求是否计费。Front Door 的基本费跟配置文件层级走，用量跟请求和传输走，没有带宽峰值月结这一档。Cloud CDN 的缓存传出、缓存填充和缓存查询是同时存在的项，不是三选一。

## 缓存和回源费用

命中缓存，少打到源站。CDN 账单上的边缘下行通常还在：用户仍然从边缘把字节下载走。各家对“回源那一段谁收费”的写法不同。

### 阿里云

[配置缓存过期时间](https://help.aliyun.com/zh/cdn/user-guide/configure-the-cdn-cache-expiration-time) 的决策顺序是：源站若返回 `Pragma: no-cache` 或 `Cache-Control: no-cache`、`no-store`、`max-age=0`，节点不缓存；否则控制台规则优先；没有命中控制台规则时再看源站头，优先级是 `Cache-Control`（先 `s-maxage` 再 `max-age`）、`Expires`、`Last-Modified`、`ETag`。`Last-Modified` 的启发式缓存是时间差的 0.1，并夹在 10 秒到 3600 秒。只有 `ETag` 时默认缓存 10 秒。控制台规则和源站头都没有，则不缓存。

只对源站状态码 200、203、206、300、301、308、410 执行这套缓存。404 等要到「缓存配置 > 状态码过期时间」另配。

控制台路径：域名管理，目标域名的管理，缓存配置 > 缓存过期时间，添加。类型是目录或文件后缀。目录以 `/` 开头，一条规则一个目录，`/` 匹配全部路径。后缀用英文逗号分隔，区分大小写，不带点。过期时间 0 秒到 3 年。权重 1～99，大的优先；权重相同则先创建的优先。

几个默认开关：优先遵循源站缓存策略默认关闭；忽略源站不缓存标头默认关闭；客户端跟随 CDN 缓存策略默认关闭；过期时间为 0 时，强制内容重新验证默认关闭，关闭等于不缓存，开启等于每次回源验证（304）。创建域名时选的业务类型不会预设固定缓存天数。

规则改完只影响新缓存。旧对象要立刻换策略，得刷新。访问不热的对象可能在过期前就被挤出节点，响应头 `X-Swift-CacheTime` 是上限，不是保证。`X-Cache: HIT` 表示命中，`MISS` 表示回源。过短的过期时间会提高回源，源站出方向费用上去；CDN 下行仍按 L1 实际流出计。

[功能概述](https://help.aliyun.com/zh/cdn/features) 把忽略参数的默认值写成关闭。打开后，回源去掉 `?` 后面的参数，用来提高命中率。查询参数本身有业务含义时不要直接开。

### 腾讯云

[节点缓存过期配置](https://cloud.tencent.com/document/product/228/47672)（最近更新 2026-01-21）：域名管理，管理，缓存配置，新增规则。类型可以是全部文件、文件后缀、目录、全路径、首页。缓存选项是遵循源站、缓存、不缓存。单个域名最多 100 条规则，一条规则里最多 100 组内容，用分号分隔。优先级是列表底部高于顶部。只缓存 `GET` 和 `HEAD`。

未配置规则或没有命中规则时，遵循源站 `Cache-Control`；没有该字段则默认缓存 600 秒。同一页的 WordPress 示例又把“`php;jsp;asp;aspx` 不缓存、其余文件缓存 30 天”称为 CDN 默认缓存规则。两处不是同一句话。对账和改配置时打开这一页，看账号里实际已经存在的规则，不要把 600 秒和 30 天写成一个默认值。

遵循源站时，`no-cache`、`no-store`、`private` 不缓存；同时有 `max-age` 和 `s-maxage` 时用 `s-maxage`。没有 `Cache-Control` 和 `Expires` 时，启发式缓存默认关闭则缓存 0 秒；打开后，有 `Last-Modified` 则取时间差的 0.1，没有则 600 秒。自定义缓存且关闭强制缓存时，源站的 `no-cache`、`no-store`、`private` 仍然不缓存；打开强制缓存才忽略源站 `Cache-Control`。不缓存则每次回源。

访问很少的资源可能未到过期就被删掉。源站更新后节点不会自动实时替换，要刷新，或换文件名。缓存过短会增加源站带宽；CDN 侧仍不计回源流量，源站带宽是源站的账。

### CloudFront

账单解读把 TTL 是否达到 3600 秒分成静态和动态两种用量类型。TTL 短于 3600 秒的 `GET`/`HEAD` 走动态行。Origin Shield 对动态请求始终按增量层计请求费，用来换更少的源站并发；可缓存请求只有走到另一区域的区域边缘缓存时才计这笔 Origin Shield 请求费。源站在 AWS 上时，固定费率说明写数据传输免除；即用即付是否在 Data Transfer 里另有一行，以账单为准。

缓存行为按路径分别设置源站、是否缓存、是否要求 HTTPS 或签名 URL。中文定价页的功能表里，自定义缓存规则从业务方案起才包含，免费和 Pro 标成没有。[固定费率方案](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html) 把缓存行为条数写成免费 5、Pro 10、业务 50、高级 100。以你创建分配时控制台显示的方案说明为准。

### Azure Front Door

[缓存](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching) 写明，每个边缘站点有自己的缓存，所以即使命中率看起来不低，源站仍可能看到回源。适合缓存的是图片、CSS、JavaScript；带身份的 API 不应缓存。建议静态和动态分成不同路由，动态路由关掉缓存。

没有 `Cache-Control` 时，默认在 1 到 3 天之间随机取一个缓存时间。过期顺序是 `s-maxage`、`max-age`、`Expires`。缓存时间不能超过 366 天。`private`、`no-cache`、`no-store` 不缓存；该页写即使规则引擎覆盖缓存行为，这些值仍然生效。规则引擎里的 Override always 只作用于“可以缓存”的响应。

标准版和高级版的缓存行为和时长在规则引擎里配，规则引擎覆盖路由上的配置。缓存可以关。关了以后，不论源站头怎么写都不缓存，每次都到源站，边缘到源的请求字节和源站侧费用会上去；边缘到客户端的传输和请求数仍然计。

校验只用 `Last-Modified`，不支持 `ETag`。大于 8 MB 的内容如果各副本的 `Last-Modified` 不一致，可能出现校验失败和 5xx。带 `Authorization` 的请求默认不缓存，除非响应里有 `must-revalidate`、`public` 或 `s-maxage`。可缓存响应在发给客户端前会去掉 `Set-Cookie`。

`X-Cache` 看的是响应前 8 MB：`TCP_HIT` 或 `TCP_REMOTE_HIT` 是命中，`TCP_MISS` 是回源。访问日志里的 `HIT`、`REMOTE_HIT`、`MISS`、`PARTIAL_HIT` 是另一组状态。`N/A` 表示签名 URL 或 WAF 拒绝了请求。

缓存命中时不计边缘到源的传输，到客户端的传输和请求数仍计。源站在 Azure 时，源站到 Front Door 的传输 Front Door 不计，所以“提高命中率”主要减少的是边缘到源的请求字节，以及源站自己的计算。它不会取消客户端下载的那一笔。

清除缓存不区分查询字符串，一条 URL 会清掉所有查询变体，且不区分大小写。通配域名不能直接整域清除，要写具体子域。不常访问的对象可能在到期前就被丢掉。

### Google Cloud CDN

[缓存概览](https://docs.cloud.google.com/cdn/docs/caching?hl=zh-cn) 的三种模式：

| 模式 | 行为 |
| --- | --- |
| `CACHE_ALL_STATIC` | 自动缓存成功的静态内容；源站给出有效缓存指令的响应也会缓存。用 gcloud 或 REST API 创建并启用 Cloud CDN 的后端，这是默认 |
| `USE_ORIGIN_HEADERS` | 源站必须给出有效缓存指令，否则不在边缘缓存，每次回源 |
| `FORCE_CACHE_ALL` | 成功响应都缓存，忽略源站的 `private`、`no-store`。该页写这不适合带用户身份的动态内容。私有桶要缓存私有内容时，又必须用这个模式 |

没有新鲜度指令时，`CACHE_ALL_STATIC` 仍可缓存静态内容类型。HTML 和 JSON 默认不按成功响应缓存。Cloud Storage 对公开对象、又没有指定元数据时，会送出 `Cache-Control: public, max-age=3600`。过期时间超过 30 天（2592000 秒）时，Cloud CDN 按 2592000 秒处理，下游客户端仍看到原来的 `max-age`。

后端存储桶的默认缓存键不含协议和主机，默认只把 Cloud Storage 认识的查询参数（例如 generation）放进缓存键。后端服务默认用完整请求 URI。查询字符串不同会拆成不同缓存项，命中率下降，缓存填充增加。

带 `Authorization` 的回源，在 `USE_ORIGIN_HEADERS` 和 `CACHE_ALL_STATIC` 下，只有响应包含 `public`、`must-revalidate` 或 `s-maxage` 才缓存。`FORCE_CACHE_ALL` 没有这条限制。

请求折叠默认开启，用来把并发的缓存填充合成少量回源。该页建议对延迟极敏感、又不在乎源站负载的场景再关掉。

签名 URL 的响应，不论源站 `Cache-Control` 如何都会缓存。`--signed-url-cache-max-age` 默认 1 小时（3600 秒）。`FORCE_CACHE_ALL` 会覆盖这个上限。

## 防盗链、URL 鉴权和限速

这三样都可能把请求挡在源站之外。挡在 CDN 节点上的请求，阿里云写明仍可能产生少量流量费；走 HTTPS 时还有 HTTPS 请求费。不要把“返回 403”理解成这笔请求免费。

Referer 可以被客户端改写。对象存储那篇已经写过源站防盗链和 CDN 缓存的关系：边缘命中可能不再经过源站的 Referer 规则，所以 CDN 上要另配。

### 阿里云

Referer 黑白名单默认不启用，任何网站都能访问。[配置 Referer 黑/白名单](https://help.aliyun.com/zh/cdn/user-guide/configure-a-referer-whitelist-or-blacklist-to-enable-hotlink-protection) 的路径是：域名管理，管理，访问控制，Referer 黑/白名单，修改配置。黑名单和白名单只能选一种。规则总长度不超过 60 KB，不写协议头。填写 `example.com` 且未勾选精确匹配时，会带上子域名。精确匹配默认不勾。忽略 Scheme 默认不勾；不勾时 Referer 必须带 `http://` 或 `https://`。允许通过浏览器地址栏直接访问默认不勾，不勾则空 Referer 会被拒绝。拒绝时返回 403，日志仍记录。可选填以 `http://` 或 `https://` 开头的重定向 URL，拦截后返回 302。

URL 鉴权默认未开启。分发的内容在未开鉴权时，拿到 URL 就能访问。[配置 URL 鉴权](https://help.aliyun.com/zh/cdn/user-guide/configure-url-signing) 的路径是同一页访问控制里的 URL 鉴权。类型有 A、B、C、F。主 KEY 或备 KEY 至少填一个，6～128 个字符，只支持大小写字母和数字。有效时长 1～31536000 秒，默认 1800 秒。鉴权失败返回 403，仍有少量流量费，HTTPS 另计请求数。回源前会去掉鉴权参数，用原始 URL 做缓存键。刷新和预热要提交去掉鉴权参数的原始 URL。关掉 CDN 鉴权但客户端仍带鉴权参数时，节点无法还原原始 URL，请求都会回源，源站流量上去。要停用，源站和 CDN 一起关。密钥不要写进仓库或文档，控制台和签算程序各放一份即可。

[单请求限速](https://help.aliyun.com/zh/cdn/user-guide/configuration-order-request-speed-limit) 不把域名下线。它限制单个请求的下行速率，用来压带宽峰值，最低 100 KB/s。不限速大小默认 0 字节，先发送这么多再限速。限速时段按北京时间，开始时间默认 0，结束时间默认 24。每个 Range 请求（206）重新计算不限速大小，分片下载时实际速率可能高于预期。文件小于每秒限速值时，限速不起作用。路径：流量限制，单请求限速。

IP 黑白名单、UA 黑白名单默认未开启。用量封顶不能按单个 IP 设。按 IP 做频次拦截，用量封顶常见问题指向的是边缘安全加速的 WAF，不是 CDN 用量封顶本身。

### 腾讯云

[防盗链配置](https://cloud.tencent.com/document/product/228/41454) 默认关闭。路径：域名管理，管理，访问控制。黑名单命中返回 403；白名单不匹配返回 403。空 Referer 要单独勾选拒绝或允许。匹配是前缀匹配，支持 `*`。全球加速可以加境内、境外不同配置；区域特殊配置加上后该页写暂时不能直接删除，只能关掉开关。开关关掉后，下面原有配置不会在现网生效。

[鉴权配置说明](https://cloud.tencent.com/document/product/228/41622) 写明默认关闭。打开后，客户端要按所选方式计算签名，节点校验通过才放行。方式为 TypeA、TypeB、TypeC、TypeD。全球加速同样可以加区域特殊配置，加上后暂时不能直接删除。这次没有把各 Type 的签名字段默认名写成操作步骤；算法以该页所链的说明为准。密钥用占位符，不要提交到仓库。

[下行限速](https://cloud.tencent.com/document/product/228/45076) 默认关闭，限制单连接下行速度，用来压峰值带宽，不减少已经产生的流量。全网用户生效。最多 10 条规则，单位 KB/s，正整数 1～1000000。类型为全部文件、文件类型、目录、指定路径，不支持正则。底部规则优先级高于顶部。

[攻击风险高额账单](https://cloud.tencent.com/document/product/228/51813)（最近更新 2026-07-14）还列出 IP 黑白名单、IP 访问限频、UA 黑白名单和流量防盗刷。这些在该页被写成免费的访问控制。IP 限频的具体次数上限，这次没有打开单独的配置页，不写一个默认 QPS。

### CloudFront

签名 URL 在缓存行为里把“限制查看器访问”设为是，并指定可信签名者。签名用的查询参数会在回源前去掉，不应放进缓存键，否则每个签名都变成一条缓存。具体参数名见查询字符串文档。本文不提供签名算法实现。

地理限制是分配上的限制项，用来按国家允许或拒绝，不是按字节封顶。AWS WAF 可以挂到分配上，在请求到达源站前拦截。固定费率方案自带 WAF 规则条数；即用即付的 WAF 费用不在 CloudFront 那张解读表里，要另看 WAF 的账单。

分配状态设为 Disabled 后，CloudFront 不接受使用该分配域名的终端用户请求，直到改回 Enabled。可以反复切换。配置传播通常几分钟，负载高时更长。见分配设置。这是人工停分配，不是用量到阈值自动停。

### Azure Front Door

这次打开的计费页、缓存页和常见问题没有“用量到顶自动下线终结点”的开关。拒绝请求用 WAF 或规则引擎。访问日志里缓存状态 `N/A` 表示签名 URL 或规则引擎拒绝了该请求。签名 URL 的配置步骤这次没有逐页打开，不写参数名和默认有效期。

WAF 速率限制是自定义规则，不是规则引擎里的缓存设置。见 [WAF 速率限制](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-rate-limit)。默认按套接字 IP 分别计数，时间窗口是 1 分钟或 5 分钟，阈值由你填写。高级版还可以按国家/地区、ASN 或 JA4 分组，或把匹配请求合在一个计数里；这些额外分组该页标为预览。阈值设得很低时（该页举例大约每分钟 200 次以下），请求可能打到还没同步计数的另一台 Front Door 服务器，从而超过阈值仍放行。拦截成功的请求仍按计费页的例子收取请求费和返回给客户端的字节。

### Google Cloud CDN

[签名网址](https://docs.cloud.google.com/cdn/docs/using-signed-urls?hl=zh-cn) 写明：Cloud CDN 不拦截没有 `Signature` 参数或 `Cloud-CDN-Cookie` 的请求，只拒绝签名参数不合法的请求。没有签名的请求如果缓存命中，会从缓存提供。签名响应和未签名响应分开缓存。源站必须自己校验签名；无效签名应返回 HTTP 403，403 不可缓存。如果对无效签名返回了可缓存的状态码，以后的合法请求可能被错误拒绝。

因此，只在 Cloud CDN 上配签名、源站仍接受不带签名的直接访问，挡不住绕过 CDN 的人，也挡不住已经进入未签名缓存的对象。源站校验是文档要求的另一半。

这次打开的 Cloud CDN 价格、缓存和预算页面，没有阿里云那种“域名用量到阈值就停止边缘服务”的开关。

## 用量封顶、告警和账单

### 阿里云：用量封顶会下线域名

[配置用量封顶](https://help.aliyun.com/zh/cdn/user-guide/configure-usage-cap) 按单个加速域名设置带宽、流量或 HTTPS 请求数。达到阈值后域名下线，到解封时间再自动恢复。功能本身不另收费。没有账户级或域名组总封顶，每个域名单独配。也不能按单个 IP 封顶。

| 类型 | 统计 | 阈值范围 | 解封时间 |
| --- | --- | --- | --- |
| 流量 | 每 5 分钟、每小时、当天 24 点前、当月 | 1 MB～10000 TB | 该页写从触发时刻起算，到达设定时间后自动上线。可选枚举写在带宽封顶一节 |
| 带宽 | 超过阈值则下线 | 1 Mbps～1 Tbps | 5 分钟、1 小时、1 天、1 月、永不解封 |
| HTTPS 请求数 | 周期与流量相同 | 1 百万次～100 亿次 | 与流量相同，该页没有把带宽那一组枚举再列一遍 |

路径：域名管理，管理，流量限制，用量封顶，修改配置。没有通用默认阈值。建议先在左侧用量查询看近 7 天。

监控大约延迟 10 分钟。达到阈值后大约 10 分钟才下线，这段时间的流量、带宽、请求正常计费。[流量限制常见问题](https://help.aliyun.com/zh/cdn/user-guide/usage-cap-faq) 的例子是：带宽封顶 10 Gbps，下线前峰值可以已经到 25 Gbps，按带宽峰值的当日账单按 25 Gbps 算；按流量时，延迟窗口里的流量计入该小时。下线后该域名的请求都不能再通过 CDN，网站打不开。DNS 可能指到 `offline.*.kunlun*.com` 或返回 614，控制台状态为已停止。

解封时间到了会自动上线，即使你中途手动上线过。想保持下线，要删除这条封顶配置，只关开关不够：配置记录还在就可能继续生效。提前恢复用域名管理里的启用。

资源包用完不会自动停服务，超出部分按量后付费。资源包预警可能迟到，收到时超额费用可能已经产生。自动复购会在剩余量或到期时再买一包，盗刷时可能连续复购；该常见问题建议和用量封顶一起用。自动复购的新包默认继承复购配置。

云监控告警不停止服务。[设置报警](https://help.aliyun.com/zh/cdn/user-guide/set-an-alert-rule) 的入口：CDN 控制台域名管理右上角「监控告警设置」，或云监控里云资源监控 > 云产品监控 > CDN，创建报警规则。资源范围可以是全部域名、应用分组或单个域名。通道沉默周期最短 5 分钟，最长 24 小时。功能概述列出的监控项包括带宽峰值、4xx/5xx 占比、命中率、公网下行流量和 QPS。示例里的 500 Mbytes 是文档示例，不是默认阈值。

[预算管理](https://help.aliyun.com/zh/user-center/how-to-manage-a-budget) 在费用与成本控制台。首次要免费开通，开通后等 24 小时且账号已有消费才能用。费用预算按年、季、月，范围可以包含产品。触发阈值时发邮件、短信、站内信。该页写明不会限制实际的资源使用。费用预算创建后 T+2 才能看实际值和预测值；超过阈值后 T+1 发一次通知。新增预警在 T+1 日 12:00（UTC+8）后生效。每条预算最多 5 组预警。CDN 的小时级突增可能在预算通知之前就已经出账。

[功能概述](https://help.aliyun.com/zh/cdn/features) 里的“配置带宽封顶”仍写着超出后域名下线、所有请求返回源站，默认值为关闭。用量封顶页和常见问题写的是下线后无法通过 CDN 访问，并可能解析到无效地址。两套说法都在中国站文档里。操作和排障以用量封顶页、常见问题为准。下线后不要假设流量会自动改走源站，也不要假设这段访问不再计 CDN 费。

### 腾讯云：用量封顶默认关闭，超出后返回 404

[用量封顶配置](https://cloud.tencent.com/document/product/228/41733)（最近更新 2026-02-10）默认关闭。路径：域名管理，管理，高级配置。统计周期内超过告警阈值会推送消息；超过访问阈值并选择了对应动作后，关闭该域名的 CDN，请求返回 404。

瞬间用量按每 5 分钟统计流量、带宽或 HTTPS 请求数。累计用量按每小时、当天 24 点前或自然月，只支持流量和 HTTPS 请求数，不支持累计带宽。瞬间流量和瞬间带宽只能配一条。HTTPS 请求数封顶只支持加速类型为网页小文件、下载大文件、音视频点播的域名。

解封可以是 60 分钟、12 小时、24 小时、3 天，或永不解封。永不解封要自己到域名管理重新开启。告警阈值是访问阈值的 10%～90%，只能填 10 的倍数。扫描粒度 5 分钟，用量涨得快时，可能一次扫描就从“还没到告警百分比”跳到访问阈值，然后两条通知都会发。

生效延迟大约 10 分钟，期间消耗正常计费。检测到超出后，返回 404 还要在全网节点逐步下发。统计起点是配置时间往前对齐到 5 分钟。选“每小时”时，第一个周期可能不足 1 小时。

攻击风险页把同一功能概括成“5 分钟用量超出就关闭并返回 404”。配置页比这句多了累计周期和告警百分比。以配置页的可选项为准。该页同时写明，恶意攻击或盗刷产生的高额账单不能免除或退款。

流量计费用户可以在流量包管理设余额比例告警。带宽和流量曲线还可以在腾讯云可观测平台按域名或项目做监控，通知渠道该页写的是短信、邮件和微信。

账单：费用中心，费用账单 > 账单查看，明细账单，产品选内容分发网络 CDN。扩展字段 1 是计费大区，字段 3 是用量（带宽 Mbps，流量 GB）。境内和境外是不同账单。

预算：费用中心，成本管理 > 预算管理。周期可以是日、月、季、年。费用范围可以选产品。每条预算最多 3 组阈值提醒、3 组波动提醒。月度费用预算模板的阈值是实际支出达到 80% 和 100%。零支出预算的金额是 0.01，实际支出达到 0.01 即通知。通知渠道包括邮件、短信、站内信、企业微信。同一条提醒在一个重置周期内只触发一次。这次打开的预算页没有写达到阈值后自动关闭 CDN。

### AWS：预算在账单控制台，停服务要改分配状态

即用即付没有本次打开的“流量到顶自动禁用分配”。固定费率写的是不收超额费，不是到额度就停。

预算入口：[创建预算](https://docs.aws.amazon.com/zh_cn/cost-management/latest/userguide/budgets-create.html)。打开账单与成本管理控制台 <https://console.aws.amazon.com/cost-management/>，导航窗格选择预算。该页写可以跟踪成本和使用情况并执行操作。中文页注明是机器翻译，与英文不一致时以英文为准。这次没有打开一份把“预算操作的内置动作”写成直接禁用指定 CloudFront 分配的页面，不把预算告警说成自动关停分配。

账单行名见前面的解读页。失效路径按月、按账户合计前 1000 条不收费。

### Azure：预算不停止资源，配置文件要删除才停基本费

[创建和管理预算](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/costs/tutorial-acm-create-budgets) 写明，超出阈值会通知，不会影响资源，也不会停止使用。成本和用量数据通常 8～24 小时后可用，预算每 24 小时评估一次，达到阈值后通常在评估后 1 小时内发邮件。新订阅最多 48 小时才能使用成本管理。

入口：门户里打开订阅、资源组或管理组，菜单选预算。重置周期有月、季、年；部分订阅还可以按计费月。至少一个阈值和邮箱，最多五个阈值、五个邮箱。阈值范围 0.01%～1000%。订阅和资源组范围还可以挂操作组。操作组能发通知或触发自动化；教程没有把操作组的默认动作写成删除 Front Door 配置文件。

中国的世纪互联等主权云，该教程写预算按计费货币评估，不按美元。这只说明预算评估货币。Front Door 标准版/高级版的计费项是否与全球文档逐项相同，这次没有打开世纪互联的产品页。

要停掉 Front Door 的基本费，常见问题的写法是删除配置文件，不能只点停用。删除前确认 DNS 已经不再指向它。缓存清除不停止计费。

### Google Cloud：预算默认不封顶

[预算和提醒](https://docs.cloud.google.com/billing/docs/how-to/budgets?hl=zh-cn) 写明，预算不会自动设置用量或支出上限，到阈值时不自动停止使用或停止结算。默认阈值是实际支出的 50%、90% 和 100%。范围可以是结算账号、组织、文件夹、项目、服务或标签。日历周期从太平洋时间 0 点算。

有结算账号权限时，在控制台的预算和提醒页创建。只有项目权限时，先选项目，再进结算，费用管理里的预算和提醒。

同一页写，用预算的编程通知停用项目的 Cloud Billing，是自动控制支出的一种做法。那是停整个项目的结算，不是只停 Cloud CDN，也会影响项目里的其他服务。步骤以该页所链的说明为准。未结算的 CDN 用量在项目结算页看。

## 流量被刷时

先分清三件事：告警只是通知；限速让单连接变慢，流量还在累计；封顶或禁用才会停止后续加速。延迟窗口里已经发生的用量照计。被拦截的 HTTPS 请求在阿里云仍可能计请求数。腾讯云写明盗刷账单不能免除或退款。

1. 看监控，不要只等账单。阿里云用云监控的下行流量、带宽、QPS、4xx/5xx。腾讯云用可观测平台和用量封顶的百分比告警。CloudFront 看账单行突然升高之前，先看分配的请求和字节。Front Door 看访问日志里的缓存状态、WAF 和 `Pop`。Cloud CDN 看缓存填充是否相对传出异常变大，以及是不是不可缓存流量在付缓存查询费。
2. 阿里云、腾讯云如果已经配了用量封顶，确认阈值不是高到帮不上忙，也确认解封策略不是几分钟后自动恢复。阿里云只关开关不够，配置还在就会再生效。腾讯云盗刷期间若设了定时解封，域名会自己回来。
3. 还没封顶、流量仍在涨：阿里云可以在流量限制里把该域名的封顶阈值改到当前业务能接受的水平，或在域名管理停用。腾讯云把超出阈值的动作设为返回 404，或在域名管理下线。CloudFront 把分配状态改为 Disabled。Front Door 不能停用配置文件，要停止基本费就删除；在此之前用 WAF 拦截只能减少到源站的流量，请求和错误页字节仍计。Cloud CDN 没有同等的自动下线；预算通知不会自己停。要停边缘缓存，需在对应后端关掉 Cloud CDN。这次没有打开一份“一键关停所有区域缓存”的单独应急页。
4. 能识别来源时再收紧访问，而不是先把桶改成公共读。阿里云 Referer、URL 鉴权、IP 名单；腾讯云防盗链、鉴权、IP 名单和限频。这些拦截本身可能仍有请求费。对象存储若被直接访问，按对象存储那篇处理，不要只改 CDN。
5. 阿里云资源包自动复购在盗刷时会继续买包。先处理封顶，再决定是否暂停复购。腾讯云流量包告警同样不会停服务。
6. 源站出方向可能仍在涨。回源风暴时看云服务器公网或对象存储外网下行，规则在前两篇，不在 CDN 的“回源不计费”那一句里。
7. 恢复前先看日志里的 Referer、URL 和客户端分布，再上线。阿里云自动解封到点会再上线；若原因还在，删掉封顶配置才能保持停止，确认后再按业务重新加上封顶。

## 上线前检查清单

每个加速域名或分配做一遍。菜单以登录的站点为准。

1. 产品选对。阿里云全站加速域名不要按 CDN 价估算。腾讯云 ECDN、SCDN 分开看。新的 Azure 配置用 Front Door 标准版或高级版；经典 CDN 和经典 Front Door 按比较页的停用日期安排，不新建在即将停用的档上。
2. 计费方式已知。阿里云中国站默认按流量；改带宽峰值或月结 95 不是控制台上随时可点的方向，还有 5 Gbps 和月消费条件。国际站概述这次只看到按流量和按带宽峰值。腾讯云 2020-12-07 21:30 之后开通的默认按流量小时结，不能自行改成带宽峰值；带宽计费时流量包不抵扣。CloudFront 分清固定费率和即用即付。固定费率的额度不是自动停服。Front Door 记住没有流量也收配置文件基本费。
3. 缓存规则覆盖静态资源，动态路径和登录路径不缓存。阿里云业务类型不会预设天数；过期时间 0 且未开强制验证才是不缓存。腾讯云未命中规则时的 600 秒，和示例里的 30 天默认规则，以控制台现有规则为准。Front Door 没有 `Cache-Control` 时会缓存 1～3 天。Cloud CDN 用 API 创建时默认 `CACHE_ALL_STATIC`，`FORCE_CACHE_ALL` 会缓存本不该公开的响应。
4. 查询字符串不会把同一文件拆成大量缓存项，除非业务需要。阿里云忽略参数默认关。Cloud CDN 后端服务默认用完整 URI。
5. 公开资源才考虑只配 Referer。下载、音视频和安装包用 URL 鉴权或签名，并让源站同步校验。阿里云鉴权有效时长默认 1800 秒。Cloud CDN 不拦截完全没带签名的请求。关掉鉴权时，客户端不能继续带鉴权参数，否则阿里云会全部回源。
6. 阿里云、腾讯云的用量封顶按域名设好，解封时间在盗刷场景下不要选最短档。阿里云没有账户级总封顶。封顶有大约 10 分钟延迟，延迟内的用量会计费。CloudFront、Front Door、Cloud CDN 不要假设存在同样的自动下线。
7. 告警接到人。阿里云云监控和费用预算，预算 T+1 才通知，且不限额。腾讯云用量封顶的百分比告警、流量包余额、费用中心预算。AWS 在成本管理控制台建预算。Azure 预算每 24 小时评估，不停止资源；Front Door 基本费要靠删除配置文件才停。Google Cloud 预算默认不封顶，默认阈值 50%、90%、100%。
8. HTTPS、QUIC、WAF、实时日志、专用 IP 证书、失效路径这些可选项，只开需要的。阿里云静态 HTTPS 自 2024-02-01 起每月前 500 万次免费，超出后按请求数计，失败的请求也会计。CloudFront 每月合计前 1000 条失效路径不收费，通配符算 1 条。
9. 源站出方向有预算。CDN 不计回源，不等于源站不计。云服务器和对象存储的规则见前两篇。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。资料核对日期：2026-11-11。

阿里云中国站：

- [计费概述](https://help.aliyun.com/zh/cdn/product-overview/billing-overview)
- [基础服务计费](https://help.aliyun.com/zh/cdn/product-overview/billing-rules-of-basic-services)
- [增值服务计费](https://help.aliyun.com/zh/cdn/billing-of-value-added-services-1)
- [变更计费方式](https://help.aliyun.com/zh/cdn/product-overview/change-the-metering-method)
- [关于请求数计费的常见问题](https://help.aliyun.com/zh/cdn/product-overview/faq-about-the-billing-of-requests)
- [配置用量封顶](https://help.aliyun.com/zh/cdn/user-guide/configure-usage-cap)
- [流量限制常见问题](https://help.aliyun.com/zh/cdn/user-guide/usage-cap-faq)
- [配置单请求限速](https://help.aliyun.com/zh/cdn/user-guide/configuration-order-request-speed-limit)
- [配置缓存过期时间](https://help.aliyun.com/zh/cdn/user-guide/configure-the-cdn-cache-expiration-time)
- [配置 Referer 黑/白名单](https://help.aliyun.com/zh/cdn/user-guide/configure-a-referer-whitelist-or-blacklist-to-enable-hotlink-protection)
- [配置 URL 鉴权](https://help.aliyun.com/zh/cdn/user-guide/configure-url-signing)
- [设置报警](https://help.aliyun.com/zh/cdn/user-guide/set-an-alert-rule)
- [预算管理](https://help.aliyun.com/zh/user-center/how-to-manage-a-budget)
- [功能概述](https://help.aliyun.com/zh/cdn/features)

阿里云国际站：

- [计费概述](https://www.alibabacloud.com/help/zh/cdn/product-overview/billing-overview)（页面更新时间 Oct 19, 2026）

腾讯云中国站：

- [计费概述](https://cloud.tencent.com/document/product/228/2949)
- [基础服务计费](https://cloud.tencent.com/document/product/228/75562)
- [变更计费方式](https://cloud.tencent.com/document/product/228/3077)
- [用量封顶配置](https://cloud.tencent.com/document/product/228/41733)
- [攻击风险高额账单](https://cloud.tencent.com/document/product/228/51813)
- [明细账单查询](https://cloud.tencent.com/document/product/228/6071)
- [节点缓存过期配置](https://cloud.tencent.com/document/product/228/47672)
- [防盗链配置](https://cloud.tencent.com/document/product/228/41454)
- [鉴权配置说明](https://cloud.tencent.com/document/product/228/41622)
- [下行限速配置](https://cloud.tencent.com/document/product/228/45076)
- [预算管理](https://cloud.tencent.com/document/product/555/65784)

AWS：

- [CloudFront 定价](https://aws.amazon.com/cn/cloudfront/pricing/)
- [解读 CloudFront 账单和用量报告](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/billing-and-usage-interpreting.html)
- [失效路径如何计费](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PayingForInvalidation.html)
- [分配设置](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesGeneral.html)
- [固定费率方案](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html)
- [创建预算](https://docs.aws.amazon.com/zh_cn/cost-management/latest/userguide/budgets-create.html)

Microsoft Azure：

- [Azure Front Door 计费](https://learn.microsoft.com/zh-cn/azure/frontdoor/billing)
- [Azure Front Door 与 Azure CDN 的比较](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-cdn-comparison)
- [Azure Front Door 缓存](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching)
- [Azure Front Door 常见问题](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-faq)
- [WAF 速率限制](https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-rate-limit)
- [创建和管理预算](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/costs/tutorial-acm-create-budgets)

Google Cloud：

- [Cloud CDN 价格](https://cloud.google.com/cdn/pricing?hl=zh-cn)
- [缓存概览](https://docs.cloud.google.com/cdn/docs/caching?hl=zh-cn)
- [使用签名网址](https://docs.cloud.google.com/cdn/docs/using-signed-urls?hl=zh-cn)
- [创建、修改或删除预算和预算提醒](https://docs.cloud.google.com/billing/docs/how-to/budgets?hl=zh-cn)

如需为这些云平台的国际站账号开户或充值，可访问海宝云官网 <https://www.haibaocloud.com>。充值币种和最低金额以官网当前说明为准。Telegram 客服：@cloudcup。
