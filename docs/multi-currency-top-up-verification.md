# 云账号充值与多币种付款核对：币种、汇率、到账与账单对账

一笔付款要核对的是三列，而不是钱包里少了多少：账单用哪种币种出、你实际付出的是哪种币种、这笔钱进了哪个云账号的余额、预付款还是发票。币种可以在创建时被定死，也可以改了要到下个账期才生效。汇率按各家写明的日期锁定。手续费由谁承担，只按官方页面写明的句子理解。

本文不写单价，不把文档示例里的金额当成现行汇率或手续费。控制台路径、能否改币种、欠费后的停机和释放，都来自撰写当天打开的官方页面，并附有链接。打不开或正文没有展开的规则，不补步骤。

云服务器账单里计算、磁盘、公网和快照怎么叠在一起，见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。费用预算、用量封顶到了阈值会不会停服务，各家不一样，见 [CDN 费用配置与防超支](cdn-cost-configuration.md)。这两类不在这里重复。

核对时间：2026-11-12。资料核对日期：2026-11-12。阿里云、腾讯云的中国站和国际站分开写。AWS 使用 docs.aws.amazon.com 与 aws.amazon.com，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 learn.microsoft.com 与 azure.microsoft.com 的全球文档；由世纪互联运营的 Azure 没有单独打开。页面改版后，以你账号所属站点的当前文档为准。

## 付款之前先对上这三列

1. 账单币种。发票、月账单和消费明细上的币种。有的在开户或签署协议时按地址确定，之后不能在原账户上改；有的可以改，但已生成的发票仍用原币种。
2. 付款币种。银行卡扣款、支付宝换汇、银行汇出的币种。它可以和账单币种不同。不同时，先看官方页面把汇率定在哪一天，再看手续费写的是云厂商不收，还是由汇款行或发卡行收取。
3. 钱落在哪里。现金余额、信用额度、预付款（Advance Pay）、赠金或积分，和后付费发票不是同一本账。钱包或网银显示已转出，不等于云控制台已经入账。到账以该云的余额、收支明细、付款记录或发票状态为准。

下面的账号 ID、金额只作占位。不要把示例写成真实账号、银行卡号或链上地址。

## 结算币种怎么确定、能不能改

### 阿里云

中国站和国际站是两套账号和账单。[中国站和国际站的区别](https://help.aliyun.com/zh/account/aliyun-vs-alibaba-cloud) 写明：中国站定价币种和结算币种均为人民币（CNY），价格包含增值税；国际站定价币种是美元（USD），结算币种支持多种其他主流货币，价格不含税，税费按账号所在国家或地区另计。两边账号不能互相变更。

国际站规则来自 www.alibabacloud.com。[管理您的结算币种](https://www.alibabacloud.com/help/zh/user-center/modify-payment-currency)（更新时间 Apr 29, 2026）写明，默认使用美元计费和结算。这一页正文展开的可切换范围是：

| 目标币种 | 该页写明的条件 |
| --- | --- |
| 欧元 | 欧洲经济区用户，且签约主体为 Alibaba (Netherlands) B.V. |
| 新加坡元、港币 | 新加坡或中国香港地区用户，且签约主体为 Alibaba Cloud (Singapore) Private Limited |
| 马来西亚林吉特 | 签约主体为马来西亚的账号，默认使用 MYR，不是从美元自行切换 |

每个自然月只有一个结算币种。在费用与成本控制台改完后，新币种于北京时间次月 1 日 0 点生效。生效后新产生的订单、账单和发票按新币种显示；历史订单、账单和发票金额不变。该页写明，修改后暂不支持跨币种退款，例如降配退款或退订退款。曾经用过的美元账户和余额不会自动转到新币种；历史待支付的美元账单仍向美元账户汇款核销。

新版控制台路径：费用与成本，账户 > 资金账户，在账户设置里点结算币种旁的修改。同一区域可以查看汇率。旧版在支付方式管理右上角的结算币种管理。切换前确认已绑定的支付方式支持该币种。该页把欧元、新加坡元、港币的汇率来源写成市场公允汇率，更新频率写成月度固定汇率。

同一页后面的汇率转换说明是另一套时间点：系统按第三方市场公允汇率换算；预付费订单用支付当日汇率，后付费账单用结算当日汇率。银行卡或数字钱包支付的后付费账单，在累积金额达到一定阈值后自动结算，该页写通常为等值 1,000 美元，结算时按当日汇率换算，月底再按当日汇率出账。1,000 美元是这句里的“通常”值，不是你账号上写死的阈值。信控用户次月 1 日收到上月账单，同一份账单里不同资源的结算时间不同，可能出现多个汇率。

这一页的摘要写有“泰国用户可申请本币结算”。2026-11-12 打开的正文没有泰国的申请步骤和生效规则，这里不补。

中国站对比页没有写可以改结算币种。中国站充值不要套用国际站的美元、欧元或港币切换。

### 腾讯云

国际站与中国站账号资源隔离，不能转移或合并。[腾讯云账号概述](https://www.tencentcloud.com/zh/document/product/378/80429)（最后更新 2026-06-15）把交易货币写成：国际站美元，中国站人民币。支付方式也不同。国际站充值是银行转账、在线充值和自动托收；中国站充值包括银行转账、在线充值（微信、QQ、网银等）和自动充值。

国际站 [充值流程说明](https://www.tencentcloud.com/zh/document/product/555/11319)（最后更新 2026-11-04）再次写明：账户以美元计价，账单与余额均以 USD 展示。这一页没有“把结算币种改成另一种法币”的开关。汇出币种如果不是美元，换汇发生在银行，见下一节。

国际站发票的结算和显示货币也是美元。见 [线上自助开票](https://www.tencentcloud.com/zh/document/product/555/31993)（最后更新 2026-09-30）。税率按产品、注册主体所在地和渠道变化，以票面为准，这里不抄该页列举的税率。

中国站的交易货币以账号概述里的“人民币”为准。不要把国际站收款账户用于中国站账号，也不要把中国站的人民币账单按美元余额去对。

### AWS

[用非美元支付账单](https://repost.aws/knowledge-center/non-us-currency) 写明，默认所有 AWS 客户以美元出账。付款币种在账单控制台修改，步骤见 [更改支付账单的币种](https://docs.aws.amazon.com/zh_cn/awsaccountbilling/latest/aboutv2/manage-payment-method.html)。中文页注明是机器翻译，与英文不一致时以英文原文为准。

路径：打开账单与成本管理控制台 <https://console.aws.amazon.com/costmanagement/>，导航窗格选择付款首选项（Payment preferences），在默认付款首选项里编辑，于付款币种中选择后保存。登记卖家（seller of record）不止一个时，要按付款资料分别指定币种。

同一篇知识中心写明这些限制：

- 控制台里按首选币种显示的金额，是按美元估算的。最终月度账单用你选择的当地货币生成。
- 汇率每天可能变化。用到发票上的汇率，在发票生成时确定。货币转换由 Amazon Services LLC 提供。
- 当地货币可以随时改。发票一旦最终确定，这张发票必须用该币种支付。
- 非美元发票可以用 Visa 或 MasterCard 支付。其他卡只按美元扣。所选付款币种若不被发卡行支持，可能产生外币转换费，费用由发卡行侧产生，知识中心没有写 AWS 另收一笔转换费。

登记卖家会改变可选币种。该文写明：印度主体默认以印度卢比出账；Amazon AWS Servicos Brasil Ltda.、Amazon Web Services Korea LLC、Amazon Web Services South Africa (Pty) Ltd.、Amazon Web Services Malaysia Sdn. Bhd. 的客户只能用当地货币。欧洲支持哪些付款币种，该文另有链接，这次没有打开那一页，不列币种名单。AWS 当前支持的完整币种表这次也没有打开，不编一份清单。

这是付款币种，不是另开一个可提现的现金钱包。预付款是另一项功能，见下一节。有未使用的 Advance Pay 资金时，[配额与限制](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-limits.html) 写明不能更改首选币种、登记卖家和默认付款方式，资金也不能提现、退款、转移或换成其他币种。

AWS 中国区文档这次没有打开。中国区账号不要按上面的全球商业区页面操作。

### Microsoft Azure

本节核对的是全球 Azure 上的 Microsoft 客户协议（Microsoft Customer Agreement）。企业协议的定价货币不同，不要套用。

[计费帐户入门](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/mca-overview) 写明：计费对象信息（billing profile）继承计费帐户的国家或地区和货币，无法更改。每月发票在月初为每个计费对象信息生成，包含上月订阅费用。

货币由签署时的合法地址决定，对照表在 [Microsoft 客户协议常见问题解答](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/microsoft-customer-agreement/microsoft-customer-agreement-faq)。该页同时写了三句要一起看的话：

- 在 Microsoft 客户协议和 Microsoft 合作伙伴协议下，Azure 在全球以美元定价。企业协议的价格以当地货币表示。
- Azure 用户始终以当地货币付费。定价货币和计费货币不同时，计费时做货币转换。
- 若使用该页列出的其他受支持货币交易，每月费用先按美元计算，付款时再把总额转换成当地货币。

汇率在同一页的写法是：使用上个月末最后一个营业日之前两个营业日的伦敦收盘即期汇率；若那两天碰到主要市场的银行假日，设定日通常再往前一天。该汇率适用于下个月的全部购买和消耗，并出现在下月初发布的发票上。零售价格 API 按你指定的货币给出的是按当前转换率估算的价格，该页写明不能拿来核对发票。从门户下载价目表时，Microsoft 客户协议和合作伙伴协议下的 Azure 第一方服务价格显示为美元；企业协议的第一方服务价格为当地货币。

[定价常见问题](https://azure.microsoft.com/zh-cn/pricing/faq/) 另写：协议若在阿根廷、香港特别行政区、印度尼西亚、墨西哥、马来西亚、沙特阿拉伯、土耳其或南非签署，关联用量须以美元支付，支付系统不再支持这些区域的本地货币。具体仍以客户协议常见问题解答的地址表和你门户里计费对象信息上的发票货币为准。成本分析里切换显示货币不会改发票币种；这次打开的页面没有提供“在已有计费对象信息上改发票货币”的步骤。

协议类型在门户里看：搜索“成本管理 + 计费”。只有一个计费范围时，打开设置 > 属性，看计费帐户类型是不是 Microsoft 客户协议。通过合作伙伴购买的对象信息，货币问题问合作伙伴，不按直签表猜测。

常见问题解答里还有一句：直接从 Azure.com 购买的新客户，其所列国家或地区“均以美元交易”，后面又是按合法地址分币种的长表。两处覆盖范围这次没有逐国对齐。对账用计费对象信息上的发票货币和发票 PDF，不用其中一句代替另一句。

由世纪互联运营的 Azure 这次没有打开，不把它写成与全球文档同一套货币。

### Google Cloud

[Cloud Billing 账号可以使用的货币](https://cloud.google.com/billing/docs/resources/currency?hl=zh-cn) 写明：每个 Cloud Billing 账号只用一种币种，创建后不能改。需要另一种币种时，新建一个 Cloud Billing 账号。币种按账单邮寄地址对应的国家或地区自动分配，创建时不能另选。见 [修改 Cloud Billing 账号](https://docs.cloud.google.com/billing/docs/how-to/modify-billing-account?hl=zh-cn) 和 [创建自助 Cloud Billing 账号](https://docs.cloud.google.com/billing/docs/how-to/create-billing-account?hl=zh-cn)。国家或地区同样不能在原账号上改。子账号的币种与父结算账号一致，见 [billingAccounts 的 `currencyCode`](https://docs.cloud.google.com/billing/docs/reference/rest/v1/billingAccounts)。

按非美元本地货币收费时，该页写明 Google 按主要金融机构发布的汇率，把以美元标出的价格转换成本地货币。汇率在每月月初设置。同一页写，从 2022 年开始在每个日历季度开始时设置巴西雷亚尔（BRL）与美元的汇率，并写部分币种有汇率大幅变化保护，细节指向汇率页。

2026-11-12 打开的 [汇率页](https://cloud.google.com/skus/exchange-rates?hl=zh-cn) 正文是 2020 年、2021 年和 2022 年的限额说明。巴西雷亚尔每季度最多变化 2% 的那段，写明自 2022-01-01 至 2022-12-31。页面没有给出 2026 年仍适用的百分比。这里不把 2% 写成当前保护幅度。页内的历史汇率数字是当年表格，不是本月汇率。

币种页的国家表里没有列出的国家或地区，该页写可能不支持当地货币，Cloud Billing 账号可能会产生美元费用。表中的付款方式只适用于自助在线账号，不适用于按账单付款的账号。需要两步验证、人必须在场才能完成交易的借记卡，不能用于定期自动扣款。

自助在线账号必须在关联的后付 Google 付款资料上保留至少一种有效付款方式，即使用每个结算周期应付金额为 0 也要保留。没有有效付款方式时，Cloud Billing 账号会暂停。见 [添加、移除或更新付款方式](https://docs.cloud.google.com/billing/docs/how-to/payment-methods?hl=zh-cn)。

## 预付余额和后付发票不是同一笔账

预付是先把钱放进云账号的余额、额度或预付款，再被账单扣掉。后付是先使用，再按信用卡、数字钱包或发票付款。优惠券、赠金和云厂商积分会先于或独立于现金被抵扣。只看银行卡扣款，会和账单对不上。

### 阿里云国际站

[资金账户](https://www.alibabacloud.com/help/zh/user-center/fund-account-overview)（更新时间 Sep 20, 2026）把资金账户写成结算用的钱包，里面有现金余额、信控额度、优惠券和代金券。路径：费用与成本，账户 > 资金账户。

直客账户的可用额度，这一页写成：

`可用额度` = `现金余额` + `信控额度` + `信控退款` − `当月未结清` − `历史未结清`

生态账户写成：`可用额度` = `Quota额度` − `Quota消费账本额度` − `未出账金额`。生态账户的订单和按量账单通过 Quota 结算；即使绑定了银行卡或其他支付方式，这些支付方式也不能用于上述支付。先看账户类型，再决定该对现金余额还是 Quota。

[信控支付](https://www.alibabacloud.com/help/zh/user-center/use-and-management-of-the-international-station) 里直客公式少了“当月未结清”一项。对账以资金账户页面上的可用额度为准，不要用另一页的公式重算后认为页面算错。

现金余额是充值、退款或调账进入的现金。收支明细在信控退款或现金余额旁边的收支明细按钮。账单金额和现金余额对不上时，该页写明系统会先用信控额度和优惠券，这些用尽后才扣现金余额。新购不能绕过资金账户另付一笔；要使用其他支付方式，先充进余额再用余额支付。

信控是授予企业用户的信用额度，范围内可以先消费后结算，需要联系客户经理开通。信控用户的对公汇款见 [支付方式介绍](https://www.alibabacloud.com/help/zh/user-center/instruction-of-payment-management/)：专属汇款账号到账后自动转入余额；非专属账号要认领。该页写从发起汇款到处理大约需要 2～4 天，以银行到账时间为准。这是该页的处理时间，不是对所有支付方式的承诺。收款账户以控制台当前展示为准，本文不转抄账号。

银行卡：该页写明阿里云不针对银行卡支付收取手续费。结算币种和交易币种不同时，发卡行可能会收取外币转换费，向发卡行确认。支付宝扣款时，账单不支持 CNY 结算，由支付宝按当天实时汇率换成人民币再扣；该页写的单笔限额是 5 万元人民币，个人支付宝每年跨境转账累计限额为等值 5 万美元。超出后无法完成支付。这是限额说明，不是提高限额的方法。

月账单和明细见 [账单](https://www.alibabacloud.com/help/zh/user-center/bill-view)（更新时间 Sep 22, 2026）。控制台首页右上角是费用 > 账单。类型表写明月账单和消费明细于次月 3 日 12:00 出具，月账单可导出消费汇总 PDF，消费明细在账单详情导出 CSV，单次最多 6 个月。同一页的常见问题又写最终月账单于次月 2 日 12 点后出具。两处日期不一致，对账以你账号账单页上的出账提示为准。分账明细次月 4 日 12:00，不作为结算依据。账单时间按 UTC+8。预付费在支付完成后计入当月账单；后付费按计费周期计入，该页写通常延迟几十分钟至数小时。应付金额的公式在汇总区和明细核对区各写了一套，含不含节省计划抵扣、抹零不一样，按你正在看的那一版账单页计算，不要混用。

### 阿里云中国站

对比页写明结算币种为人民币，支付方式包括支付宝、微信支付、银联和对公汇款。[在线充值](https://help.aliyun.com/zh/user-center/use-alipay-online-banking-to-recharge-online) 列出的充值方式是支付宝、银联在线支付、个人网银、企业网银和对公汇款，实际以充值界面为准。路径是费用与成本控制台首页的充值汇款。充值记录在资金账户看现金余额，或在收支明细把交易类型选为充值。该页写明充值金额是预付款，不能直接开票，消费之后才按消费金额开票。支付宝充值不支持花呗和信用卡；银联、个人网银和企业网银不支持信用卡。对公汇款该页写 1～5 个工作日，开户名须与实名认证一致。这些限制只说明中国站这一页，不要套到国际站。

### 腾讯云国际站

[充值流程说明](https://www.tencentcloud.com/zh/document/product/555/11319) 写明，可用余额为负时要先充值才能购买后付费服务。只有拥有财务权限的创建者和协作者能进入付款页面。

在线充值路径：费用中心 > 资金管理 > 付款，在线支付里输入金额后立即付款。银行卡可以选已绑卡或为这一次新增卡；新增卡只用于这一笔，不会自动变成绑定的付款卡。支付完成后到收支明细查充值记录。该页写银行卡支付 2 小时内到账，在线支付记录通常在 2 小时内生成；已扣款但收支明细没有记录时，先确认是不是充到了另一个账号，不要立刻再付一笔。

银行转账按签约主体和注册地使用不同收款账户，清单在同一页。汇款前到费用中心的账户信息核对签约主体；有商务代表的以合同为准。备注按该页填写云服务费和账号标识。本文不转抄收款账号和 SWIFT。普通银行转账该页写通常 3～5 个工作日，跨境可能更长，到账后要提交工单并附汇款凭证认领。银企直连专属账号在账户信息正确时，收款后自动充进匹配的云账号，适用主体以该页列表为准。

币种和手续费，这一页写明：

- 汇出币种和收款账户的美元不一致时，产生换汇损失和额外手续费。汇率和费用以汇款行成交价为准，腾讯云不参与兑换。
- 非美元汇款的到账金额按银行实际结算汇率折算，可能和汇出时的估算不同。
- 银行转账可能产生额外银行手续费，由汇款方承担。

该页表格里的到账时效和单笔上限是这一页的说明，包括银行卡单笔上限 100,000 USD、部分主体微信支付单笔及单日累计约 4,000 USD、韩国主体在线支付最低 0.1 美元。上限和时效以该页当前表为准。看不到微信支付时，先看签约主体是不是该页写明已开放的主体。

发票：费用中心 > 费用账单 > 发票。每月 6 号开始出上月可开票金额，可开票金额大于 0 才能申请。发票发到抬头里设置的邮箱，也可在发票控制台下载。

账单：费用中心 > 费用账单 > 账单查看。见 [账单查看](https://www.tencentcloud.com/zh/document/product/555/57403)。月度账单每月 1 号生成上月已出账的完整账单，出账完成前的金额不是最终费用。明细账单相对扣费时间约有 20 分钟延迟。账单概览、多维度汇总和资源账单是 T+1，不是实时。

### 腾讯云中国站

交易货币是人民币。在线充值见 [在线充值](https://cloud.tencent.com/document/product/555/7425)：费用中心的充值汇款。该页写只有创建者或管理财务角色的协作者能进入充值页；充值成功不等于新购或续费成功，待支付订单和续费要再去订单或续费管理里支付。收支明细可按交易类型筛选充值并导出。限额表在这一页，本文不抄金额。中国站对公汇款账号和国际站不是同一套，汇款信息以登录后的中国站费用中心为准。

### AWS

日常形态是付款方式和发票，不是阿里云、腾讯云那种先充进现金余额再消费。

[付款](https://docs.aws.amazon.com/zh_cn/awsaccountbilling/latest/aboutv2/view-payment-info.html)：账单与成本管理控制台，导航窗格选择付款（Payments）。

- 到期付款（Payments due）：未结发票。状态包括到期、逾期、已安排、正在处理。
- 未核销资金（Unapplied funds）：未使用的资金，该页写可能包括未核销现金、贷项通知单或 Advance Pay 余额。未核销资金总额不包括 AWS 积分（credits）。
- 交易（Transactions）：发票、付款、退款、贷项通知单和未核销资金。可以下载发票 PDF，表格可以下载 CSV。

[Advance Pay](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/manage-advancepay.html) 是预付用法：登记后把资金存入，发票到期时自动抵扣；余额不够时仍走默认付款方式。功能处于公开预览，只对部分客户开放。登记路径：付款，Advance Pay 选项卡，接受条款后注册。添加资金时，该页写金额须以美元输入。可用范围：登记卖家为 AWS Inc.（仅美元）或 AWS Europe（该页写 USD、EUR 或 GBP）。成员账户不能自己用，只有组织的管理账户（付款账户）可以。GovCloud（US）不可用。看不到这个选项卡时，先在付款首选项的默认付款方式下看服务提供商名称。电汇说明在添加资金后的 Payment summary 里，资金摘要可从 Advance Pay 摘要页下载。配额页写明资金只能支付符合条件的 AWS 费用，例如 AWS Marketplace 发票不在此列，仍用登记时的默认付款方式。

AWS 积分的抵扣细则页这次没有打开到正文。核对时只采用付款页已经写明的一句：未核销资金里不含 AWS 积分。

### Microsoft Azure

[计费概述](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/cost-management-billing-overview) 写明，发票在计费侧生成；额度在发票生成时应用到发票总额。成本管理里的成本分析不包含额度、税费，以及非 Microsoft 客户协议账户上的部分购买（例如支持费用）。对账要同时看发票和成本分析，不能只用一边。

Microsoft 客户协议的付款方式，计费帐户入门页列为：Azure 额度（自动抵扣符合条件的费用）、获准后的电汇（付款说明在发票上）、通过 Azure 网站注册的客户使用的信用卡。

发票：门户搜索“成本管理 + 计费”，打开发票，选择后下载。状态在 [查看和下载发票](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/download-azure-invoice)：应付款表示已生成尚未付；过期未付表示尝试扣款被拒；已付款表示扣款成功。过期未付若仍不付，订阅会被禁用。

用量文件：Microsoft 客户协议可在计费范围概述页底部的快捷方式里选择“下载使用情况和价格”，再准备 Azure 使用情况文件。见 [查看和下载使用情况与费用](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/download-azure-daily-usage)。角色不够时下载按钮不可用，该页按协议类型列出了所需角色。

### Google Cloud

自助在线账号的主体是后付：使用发生在结算周期内，再按阈值或按月自动扣款。部分产品和用户还会有一个预付付款资料，需要先购买抵扣额才能使用对应服务；抵扣额用完后，只停那些由预付资料支付的服务。后付资料如果出现欠款、拒付或付款方式无效，整个 Cloud Billing 账号可能暂停，预付抵扣额也保不住那些服务。见 [拆分结算周期](https://docs.cloud.google.com/billing/docs/how-to/split-charging-cycle)。

交易、付款和文档：Cloud Billing 控制台的 [交易](https://docs.cloud.google.com/billing/docs/how-to/view-history?hl=zh-cn) 页。需要 Billing Account Viewer。默认显示过去 3 个月。可以按费用、付款、调整项（该页写通常是赠送金额）和税费过滤。展开文档行下载对账单 PDF，或进入账单页下载账单。付款收据：交易类型选付款，在说明列打开付款链接。交易表可以下载 CSV。从转销商购买的，不能访问交易页，改用费用表。

账号如果同时有后付和预付，[拆分结算周期](https://docs.cloud.google.com/billing/docs/how-to/split-charging-cycle)（页面最后更新 2026-11-10 UTC）写明：后付交易在 Cloud Billing 控制台看；预付交易、抵扣额余额和购买抵扣额在 Google AI Studio 的结算页。费用表按发票月份对齐，同一个月可能有两张发票，一张后付、一张预付。后付按阈值扣款时，费用涨得很快可能使实际扣款超过阈值；预付抵扣可能延迟，余额在控制台里变成负数后服务会暂停，负数从下一次购买的抵扣额里扣。这些是该页写的系统延迟，不是到账承诺。

和发票对得上的明细是 [费用表](https://docs.cloud.google.com/billing/docs/how-to/cost-table)：结算导航里的 Cost table，工具栏下载 CSV。默认筛选下，费用表合计与发票或对账单合计一致。发票 PDF 本身只有汇总、税费和汇款说明，没有项目级明细。

## 付款之后按这个清单核对

每一笔都做。菜单名称以登录后的站点为准。

1. 账号对上。阿里云国际站看资金账户的账户 ID。腾讯云先确认没有充进另一个 UIN，收支明细里应有这笔充值。AWS 看付款页的交易是不是当前付款账户。Azure 看发票所属的计费对象信息。Google Cloud 看交易页选中的是哪一个 Cloud Billing 账号。代付、企业统一结算或组织管理账户，钱可能不在你打开的那个成员账号上。
2. 币种对上。付款凭证上的币种、云控制台入账币种、账单或发票币种，三处分别记下。阿里云国际站改结算币种后，次月 1 日 0 点（北京时间）才影响新账单，历史美元账单仍用美元。腾讯云国际站账单和余额是美元；你汇出的如果不是美元，入账金额按汇款行汇率折算。AWS 未最终确定前，控制台的首选币种金额是美元估算，以生成后的发票币种为准。Azure 以计费对象信息的发票货币为准，价目表上的美元不是发票金额。Google Cloud 以结算账号的单一币种为准，创建后不能改。
3. 汇率和手续费只记官方写明的承担方。阿里云国际站：银行卡支付阿里云不收手续费，外币转换费向发卡行确认；支付宝按当天汇率换成人民币。欧元、新加坡元、港币切换表是月度固定的市场公允汇率；后付费结算用结算当日汇率，信控月账单里可能有多个汇率。腾讯云国际站：汇款行完成兑换，腾讯云不参与；银行手续费由汇款方承担。AWS：发票生成时锁定汇率，转换由 Amazon Services LLC 提供；发卡行可能另收转换费。Azure：伦敦收盘即期汇率，用于下一个月的购买和消耗。Google Cloud：月初设置汇率。上述页面没有写的手续费比例，不要自己填一个百分数。
4. 在控制台确认入账，而不是只看转出成功。阿里云看现金余额或 Quota，以及收支明细。腾讯云看收支明细；银行卡在线支付该页写记录通常 2 小时内生成，银行转账可能要数个工作日并要认领。AWS 看付款页的交易和未核销资金；Advance Pay 另看 Advance Pay 摘要，并下载资金摘要。Azure 看发票从应付款变为已付款，电汇则按发票上的付款说明。Google Cloud 后付在交易页把类型滤成付款，打开收据；预付抵扣额在 Google AI Studio 的结算页看余额。入账前不要重复支付。
5. 留下凭证。保留付款回单、发票或对账单 PDF、云控制台导出的 CSV，以及银行或卡组织给的授权号。阿里云用月账单 PDF 和账单详情 CSV。腾讯云用发票邮件或发票控制台的 PDF，加上明细账单。AWS 用发票 PDF 和交易 CSV。Azure 用发票下载和用量文件。Google Cloud 用账单或对账单 PDF、交易 CSV 和费用表 CSV。凭证上的账号标识用你自己的 ID，不要把 `<account-id>` 这种占位符汇出去。
6. 和账单明细对账。先对总金额和币种，再下钻到产品或项目。阿里云应付金额按你打开的那一版账单页上的公式算，汇总区和明细核对区对节省计划、抹零的写法不一样。腾讯云月账单在每月 1 号生成完整上月账单之前，页面上的数不是最终数；实时扣费看明细账单。AWS 用成本和使用情况报告做用量级核对：账单与成本管理控制台，旧版页面下的成本和使用情况报告，见 [创建报告](https://docs.aws.amazon.com/zh_cn/cur/latest/userguide/cur-create.html)。该页写通过 Data Exports 创建 CUR 2.0 是推荐方式；旧版入口一个账户最多 10 份报告，开始传到 S3 最多可能 24 小时。Data Exports 新建导出的逐步点击这次没有逐项打开。Azure 发票总额含额度和税，成本分析不含这些项，两边差一口是预期可能出现的，用发票和用量文件一起看。Google Cloud 用费用表对齐发票，不拿发票 PDF 里已经拿掉的明细去对。
7. 余额或额度预警开在会真正收到的联系人上。它只发通知，不会代替停机规则。费用为什么升高，回到 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。预算和用量封顶的阈值、评估周期、到了阈值停不停服务，见 [CDN 费用配置与防超支](cdn-cost-configuration.md)。

## 余额不足或欠费时，停机和释放按哪一页

预警先设上。停机和释放的天数不要在云之间、也不要在中国站和国际站之间混用。产品页如果另有欠费说明，以产品页为准。结清之后已经释放的资源，各页写明不能靠补款恢复。

### 阿里云国际站

[监控账户余额](https://www.alibabacloud.com/help/zh/user-center/monitor-account-balances)（更新时间 Sep 08, 2026）：费用与成本 > 账户 > 资金账户，打开可用额度预警并设置阈值。可用额度低于阈值时，该页写通过邮件和站内信通知，每天 1 次，最多连续 5 天。预警不停止服务，也不停止计费。[账单常见疑问](https://www.alibabacloud.com/help/zh/user-center/bill-solution-troubleshooting-manual-for-common-consumer-bill-questions) 把同一类资金预警写成短信、邮件和站内信。以消息中心里实际开通的渠道为准。该监控页还写明：国际站当月账单正式出账前只含税前金额，次月出账会加上税金。阈值要留出税金，否则出账后可用额度会再降一截。

[延期免停](https://www.alibabacloud.com/help/zh/user-center/grace-period) 写明，直销客户的按量付费欠费后有 15 天延停。延停期间已有的按量服务、资源包和预付费资源可继续用，不能新购；延停期间继续计费。15 天结束仍未结清，运行中的资源停机。云通信按量产品另有延停额度，欠费超过额度会立刻停这些产品，其他产品仍按 15 天。页内的 100 美元、600 美元是例子，不是你的额度。停机后仍可能收到延迟账单，该页写从使用到出账大约 3 小时，特殊情况更晚。资源已经释放的，结清欠费也不能恢复。

资金账户页写明，可以在账户设置里关闭延停。关闭后，欠费时按量资源不再有这段缓冲，可能会被立即停机。

信控支付页的时间表是另一句：可用额度小于 0 后进入欠费，保留 15 天受限制使用期，此期间不能新购也不能自动续费；之后资源停止服务；停服后再过 15 天仍未结清，实例被释放。两页都出现 15 天，但一个写到停机，一个写到停服后再释放。不要合成一句“一律 15 天释放”。ECS 的欠费时间在 [ECS 欠费说明](https://www.alibabacloud.com/help/zh/ecs/overdue-payments)，和其他产品可能不同。账单超预算一文已经引用过国际站按量付费的扣款重试，这里不重复那张扣款表。

### 腾讯云国际站

充值页只写：大部分服务按小时计费，欠费可能导致服务暂停。它没有给出一个适用于全部产品的保护期天数。

[余额预警](https://www.tencentcloud.com/zh/document/product/555/9942)：费用中心 > 资金管理 > 付款，余额预警通知。自定义额度预警在可用额度低于阈值时连续通知 5 天，一个自然日只发一次，5 天后仍低于阈值就停止这一轮。阈值可以是负数。授信客户另有按已用额度占信用额度 70%、80%、90%、100% 分档的自动预警，每个档同样连续 5 天。默认接收人是创建者、全局资源协作者和财务管理员，默认方式是短信。接收人在消息订阅里改。取消勾选“订阅余额预警消息”即关闭自定义预警。预警不代替充值。

云服务器的天数只适用于云服务器。[云服务器欠费说明](https://www.tencentcloud.com/zh/document/product/213/2181)（最后更新 2025-12-31）写明：合作伙伴的客户，欠费处理以与合作伙伴的协议为准。按量计费从余额被扣成负数起，2 小时内仍可使用并继续扣费，2 小时后关机并停止扣费，进入回收站；回收站保留 15 天，此期间把余额充到大于 0 后可以开机。超过 15 天，普通公网 IP、CPU、内存、非弹性云硬盘及数据释放，数据不能找回。按量云硬盘在数据清除前仍按正常标准计费。包年包月到期未续费的，约 48 小时内停服隔离，同样进回收站保留 15 天。余额预警对按量云服务器默认关闭，需要按余额预警页订阅。该页同时写，实际消耗在变，余额预警可能有误差。对象存储和其他产品有各自的欠费说明，不要把这 2 小时和 15 天套过去。

预付费到期停服的通用说明在 [预付费计费说明](https://www.tencentcloud.com/zh/document/product/555/42701)。到期后第 1 天到第 7 天停服，不同产品的到期停服时间不同；停服后仍不续费会被释放，数据清除且不可恢复。

### 腾讯云中国站

[按量计费说明](https://cloud.tencent.com/document/product/555/9617)（最近更新 2026-07-31）是中国站费用中心文档。余额不足会进入欠费，超过约定时间后资源关停并销毁。后付费资源进入欠费保护期：按小时计费的保护期一般为 2 小时，按天计费的一般为 1 天，期间可继续使用但不能开新资源。之后是停服期，一般为 7～15 天，资源强制停服，配置保留到停服期结束。再之后回收销毁，数据清除且不可恢复。该页写明各产品的天数略有不同，以产品文档为准。“一般为”不是每一个产品的保证值。这套天数不要套到国际站。

### AWS

[未付款与服务连续性条款](https://aws.amazon.com/legal/non-payment-and-service-continuity-terms/) 写明：到期未付时，AWS 可以在通知后限制新的或现有资源用量、暂停账户访问，或两者一起做。暂停前备份和取回数据是客户自己的责任。暂停后 30 天内仍未付清，AWS 可以关闭账户。关闭后再有 60 天的关闭后期间可以申请重开；此期间不能访问内容和 AWS 服务，除非重开。重开要联系 AWS Support，并且 AWS 必须在关闭后期间的第 30 天之前收到全部欠款以及发票 ID 和其他要求的信息，用来覆盖转账延迟。关闭后期间结束，账户永久关闭，剩余数据按账户关闭文档删除。条款写明 AWS 没有义务防止因暂停和关闭导致的数据丢失。

付款页把超过到期日仍未付的发票标成逾期。Advance Pay 余额会自动用于发票；没有余额时使用默认付款方式。费用预算的邮件不会代替这条条款，预算怎么建见 CDN 一文。

### Microsoft Azure

[订阅状态](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/manage/subscription-states)（上次更新 2026-06-07）：

- 逾期未付：仍有未结付款。订阅还是活动的，所有操作可用；不付可能变为禁用。
- 警告：逾期未付、用户取消或订阅过期都可能进入。不能新建资源。资源会脱机，订阅恢复活动后可以恢复。该页写此状态不收费。
- 已禁用：不能再创建或管理资源。虚拟机解除分配，临时 IP 释放，存储变为只读，其他服务禁用。原因包括额度过期、达到支出限制、账单过期、信用卡超额或信用卡被禁用或取消。禁用可能持续 1～90 天，视订阅类型而定，之后永久删除。禁用期间仍可以读取或删除资源。

结清逾期余额的门户步骤见 [处理逾期未付余额](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/manage/resolve-past-due-balance)：成本管理 + 计费，发票，对应发票上结清余额，选择付款方式后支付。用发票电汇的，把钱汇到发票底部的说明，而不是随便一个旧账户。该页写，多份订阅共用一张失败的卡时，要一次付清全部未结余额。

### Google Cloud

付款被拒或付款方式无效时，[解决结算问题](https://docs.cloud.google.com/billing/docs/how-to/resolve-issues) 写明服务可能停止，直到付款能被处理。拒付原因在交易页看；该页写 Google 不拒绝付款，拒绝的是银行或卡机构。自助在线账号的后付资料一旦有欠款、拒付或无效付款方式，Cloud Billing 账号可能暂停。暂停后，关联的全部项目里的全部服务都停止，包括用预付抵扣额支付的服务。重新启用付款方式后，后付资料会自动对未结余额发起扣款，处理可能要几小时。暂停期间 BigQuery 结算导出暂停，恢复后不补暂停期间的数据。部分服务要手动重启。

关闭结算账号也会停掉关联项目里的计费服务。见 [关闭或重新开启结算账号](https://docs.cloud.google.com/billing/docs/how-to/close-or-reopen-billing-account)。预算到了阈值不会自动停用结算，这点已写在 CDN 一文。

## 用加密货币经第三方付款时核对什么

有的付款不走云厂商收银台，而是先把 USDT 等加密货币转给第三方，再由对方给云账号充值或授信。这一节只写转账时的核对习惯。它不针对任何代充值服务，不写到账需要多久，也不对任何服务的安全性或是否一定入账作出承诺。云厂商文档里的法币充值时效，不能套用到链上转账。

1. 先确认网络。USDT 常见的 TRC20 走波场（TRON），ERC20 走以太坊。两条网络不能混用。订单或收款页写的是哪一个网络，钱包里就必须选同一个网络。把 TRC20 转到 ERC20 地址，或反过来，不会在另一条网络上自动变成可用余额。只看“都叫 USDT”不够。
2. 地址逐字核对。从订单或收款页复制完整地址，粘贴到钱包后，再逐字对照，不要只看开头和结尾几位。和订单不一致、或事后在聊天里改口的地址，不要转。地址只是占位时写成 `<receive-address>`，不要把真实地址写进文档、工单截图以外的公开页面或代码仓库。
3. 先转一小笔。用你能接受损失的金额，或订单允许的较小金额。在该网络的公开区块浏览器里看到这笔交易之后，再看云控制台的余额、收支明细或订单状态是否出现对应记录。钱包显示已转出，只说明离开了你的钱包。
4. 保存交易哈希（TxID）。和付款回单放在一起。到这条网络的区块浏览器查询状态和确认数。本文不规定多少个确认才算完成。哈希对不上、或浏览器里没有这笔交易时，不要按对方口头说明再转一笔。
5. 不要向任何人提供云账号密码、MFA 验证码、MFA 恢复码、访问密钥、钱包私钥或助记词。对方声称是客服、需要代登录、加快到账或修改风控，也不提供。登录和 MFA 的收紧方式见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。
6. 云厂商要求的实名、3D 验证、KYC 或支付验证，按该云控制台自己的提示完成。本文不提供绕过实名、风控、短信审核或支付限额的方法。

## 参考资料（核对日期：2026-11-12）

规则可能变更。下面是本文核对过的官方页面。

阿里云国际站：

- [管理您的结算币种](https://www.alibabacloud.com/help/zh/user-center/modify-payment-currency)（更新时间 Apr 29, 2026）
- [什么是资金账户](https://www.alibabacloud.com/help/zh/user-center/fund-account-overview)（更新时间 Sep 20, 2026）
- [支付方式介绍](https://www.alibabacloud.com/help/zh/user-center/instruction-of-payment-management/)
- [如何通过信控支付](https://www.alibabacloud.com/help/zh/user-center/use-and-management-of-the-international-station)
- [监控账户余额](https://www.alibabacloud.com/help/zh/user-center/monitor-account-balances)（更新时间 Sep 08, 2026）
- [延期免停](https://www.alibabacloud.com/help/zh/user-center/grace-period)
- [账单](https://www.alibabacloud.com/help/zh/user-center/bill-view)
- [账单常见疑问和排查方法](https://www.alibabacloud.com/help/zh/user-center/bill-solution-troubleshooting-manual-for-common-consumer-bill-questions)
- [ECS 欠费说明](https://www.alibabacloud.com/help/zh/ecs/overdue-payments)

阿里云中国站：

- [中国站和国际站的区别](https://help.aliyun.com/zh/account/aliyun-vs-alibaba-cloud)
- [在线充值](https://help.aliyun.com/zh/user-center/use-alipay-online-banking-to-recharge-online)

腾讯云国际站：

- [腾讯云账号概述](https://www.tencentcloud.com/zh/document/product/378/80429)
- [充值流程说明](https://www.tencentcloud.com/zh/document/product/555/11319)（最后更新 2026-11-04）
- [余额预警指引](https://www.tencentcloud.com/zh/document/product/555/9942)
- [账单查看](https://www.tencentcloud.com/zh/document/product/555/57403)
- [线上自助开票功能说明](https://www.tencentcloud.com/zh/document/product/555/31993)
- [云服务器欠费说明](https://www.tencentcloud.com/zh/document/product/213/2181)
- [预付费计费说明](https://www.tencentcloud.com/zh/document/product/555/42701)

腾讯云中国站：

- [按量计费说明](https://cloud.tencent.com/document/product/555/9617)
- [在线充值](https://cloud.tencent.com/document/product/555/7425)

AWS：

- [用非美元支付账单](https://repost.aws/knowledge-center/non-us-currency)
- [更改支付账单的币种](https://docs.aws.amazon.com/zh_cn/awsaccountbilling/latest/aboutv2/manage-payment-method.html)
- [查看未结发票、未核销资金和付款历史记录](https://docs.aws.amazon.com/zh_cn/awsaccountbilling/latest/aboutv2/view-payment-info.html)
- [Using Advance Pay](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/manage-advancepay.html)
- [配额与限制](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-limits.html)
- [创建成本和使用情况报告](https://docs.aws.amazon.com/zh_cn/cur/latest/userguide/cur-create.html)
- [未付款与服务连续性条款](https://aws.amazon.com/legal/non-payment-and-service-continuity-terms/)

这次没有写入正文的 AWS 材料：欧洲付款币种的单独知识中心页、当前支持币种的完整清单、AWS 积分抵扣细则页（打开后没有得到正文）、Data Exports 新建导出的逐步点击、AWS 中国区文档。

Microsoft Azure：

- [Microsoft 客户协议计费帐户入门](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/mca-overview)
- [Microsoft 客户协议常见问题解答](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/microsoft-customer-agreement/microsoft-customer-agreement-faq)
- [Azure 定价常见问题](https://azure.microsoft.com/zh-cn/pricing/faq/)
- [计费概述](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/cost-management-billing-overview)
- [查看和下载 Azure 发票](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/download-azure-invoice)
- [查看和下载使用情况与费用](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/understand/download-azure-daily-usage)
- [订阅状态](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/manage/subscription-states)
- [处理逾期未付余额](https://learn.microsoft.com/zh-cn/azure/cost-management-billing/manage/resolve-past-due-balance)

由世纪互联运营的 Azure 没有打开。

Google Cloud：

- [Cloud Billing 账号可以使用的货币](https://cloud.google.com/billing/docs/resources/currency?hl=zh-cn)
- [汇率](https://cloud.google.com/skus/exchange-rates?hl=zh-cn)
- [修改 Cloud Billing 账号](https://docs.cloud.google.com/billing/docs/how-to/modify-billing-account?hl=zh-cn)
- [创建自助 Cloud Billing 账号](https://docs.cloud.google.com/billing/docs/how-to/create-billing-account?hl=zh-cn)
- [`billingAccounts`](https://docs.cloud.google.com/billing/docs/reference/rest/v1/billingAccounts)
- [添加、移除或更新付款方式](https://docs.cloud.google.com/billing/docs/how-to/payment-methods?hl=zh-cn)
- [拆分结算周期](https://docs.cloud.google.com/billing/docs/how-to/split-charging-cycle)
- [查看费用和付款记录](https://docs.cloud.google.com/billing/docs/how-to/view-history?hl=zh-cn)
- [费用表](https://docs.cloud.google.com/billing/docs/how-to/cost-table)
- [解决结算问题](https://docs.cloud.google.com/billing/docs/how-to/resolve-issues)
- [关闭或重新开启结算账号](https://docs.cloud.google.com/billing/docs/how-to/close-or-reopen-billing-account)

汇率页上 2020–2022 年的百分比限额没有写成 2026 年的现行幅度。

如需为阿里云、腾讯云、Google Cloud、AWS、Microsoft Azure 的国际站账号开户或充值，可访问海宝云（CloudCup）官网 <https://www.haibaocloud.com>。官网常见问题当前写明：充值仅支持 USDT-TRC20，单笔最低 50 USDT；其他币种或法币需联系客服定制。具体以官网当前说明为准。Telegram 客服：@cloudcup。
