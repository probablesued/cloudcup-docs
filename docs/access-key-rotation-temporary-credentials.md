# 长期访问密钥与临时凭证：轮换、禁用，不要把密钥写进云服务器

控制台登录靠的是用户名、密码和多因素认证；程序调用 API 靠的是另一套凭证：长期访问密钥，或者有时效的临时凭证。长期密钥一旦写进代码仓库、镜像、客户端或聊天记录，拿到它的人往往不必再过登录验证，就能以该身份调用接口。这篇只写程序访问用的长期密钥和临时凭证：各云已打开页面怎么创建、禁用、轮换和删除，什么时候该改用实例角色、托管标识或联合身份，以及疑似泄露时这些页面写了哪些处置步骤。登录 MFA、主账号少登录、权限怎么收窄，见 [云账号登录和权限怎么收紧：多因素认证与最小权限](cloud-account-mfa-least-privilege.md)。那篇核对清单里已经写过「主账号和根用户下不要留长期访问密钥」和「程序访问优先用临时凭证」；下面把轮换步骤、条数上限、禁用和删除的差别，以及各云推荐的无长期密钥方案展开，不重复登录绑定 MFA 的操作。Google Cloud 服务账号密钥为什么比用户登录更危险、组织政策如何默认禁止创建或上传密钥，那篇「服务账号密钥」一节已经对照过官方最佳实践，这里只补轮换步骤和工作负载身份联合，不把那一节再写一遍。

核对时间：2026-11-27。资料核对日期：2026-11-27。长期密钥和临时凭证分别是什么、每个身份最多几把密钥、轮换时能不能先建第二把再停旧的、删除后还能不能恢复、云服务器上要不要把密钥写进磁盘，都来自这一天打开的官方页面，并附有链接。打不开、正文没有写明，或同一主题两页说法不同的，下面按读到的原文分开写，不补成一套各云通用的「90 天自动轮换按钮」。页面上的天数只作为该页原文，不写成各云现行统一承诺。

范围：阿里云中国站用 help.aliyun.com 上这次能打开的现行页面。国际站打开了 www.alibabacloud.com 的英文创建 AccessKey、轮转 AccessKey 和使用访问凭据的最佳实践，这三页带有 Last Updated。腾讯云中国站用 cloud.tencent.com。国际站打开了 www.tencentcloud.com 的 Root Account Access Key Management。AWS 使用 docs.aws.amazon.com 的全球文档，没有打开 AWS 中国区 docs.amazonaws.cn。Azure 使用 learn.microsoft.com 的全球文档，这次通读的是中文页；由世纪互联运营的 Azure 没有打开。Google Cloud 使用 docs.cloud.google.com。页面改版后，以你账号所属站点的当前文档为准。

## 先看这几件事

1. 能不用长期密钥，就不要创建。阿里云把 STS Token 写成 RAM 角色的临时访问凭证，可以自定义有效期，到期自动失效，不必定期轮换。应用程序如果跑在 ECS 上，最佳实践页建议用实例 RAM 角色，从实例元数据取临时凭证并周期性更新。腾讯云「管理实例角色」写：把载体包含 `cvm.qcloud.com` 的 CAM 角色绑到专有网络 VPC 中的实例后，可在实例内用 STS 临时密钥访问其他云产品 API，临时密钥会周期性更新，无需在实例里保存 SecretKey。AWS 的 Secure access keys 页写：跑在 EC2 上的应用不要直接使用、传递、嵌入 access keys，也不要从某个来源读取它们，应给实例关联 IAM 角色，让应用取得临时安全凭证；SDK 和 CLI 可以自动从角色获取。Azure 托管标识概述写：应用程序可以用托管标识向 Microsoft Entra 获取令牌，无需管理凭据，使用托管标识不产生额外费用。Google Cloud 的密钥轮换页把用户管理的服务账号密钥当成例外，能改用更安全的方式时就不要新建密钥。

2. 不要给主账号或根用户创建长期密钥。阿里云「不使用主账号 AccessKey」写：主账号 AccessKey 等同完全管理权限，而且无法做来源 IP、访问时间等条件限制，一旦泄露风险极大。腾讯云「定期轮转访问密钥的安全建议」写：不要创建或使用主账号密钥，应通过 CAM 子用户和角色授权做日常操作和 API 调用；主账号密钥拥有账户所有资源的完全控制权，包括财务、用户权限和数据管理，一旦泄露可能造成全局性、灾难性事件，且权限无法快速收回。AWS 根用户最佳实践有专门一节 Don't create access keys for the root user；管理 IAM 用户访问密钥页也写不要用账户的根凭证创建 access keys。这和登录 MFA 是两件事：即使控制台已经开了 MFA，主账号或根用户的长期密钥仍然可以走 API。

3. 必须保留长期密钥时，先看清条数上限，再用「第二把替换第一把」的方式轮换。阿里云中国站创建页和国际站轮转页（Last Updated: Jul 10, 2026）都写：每个 RAM 用户最多两个 AccessKey，一个正常使用，另一个留给轮转。腾讯云国际站根账号密钥页写一个根账号最多创建 2 个 API 密钥；中国站删除访问密钥接口的错误码也有「每个账号最多支持两个 AccessKey」。AWS 管理访问密钥页写 You can have a maximum of two access keys per user。常见步骤是：创建第二把，把所有调用方改到新密钥，确认旧密钥不再被使用后禁用，观察业务，再删除旧密钥。禁用之后如果还有调用失败，阿里云轮转页和 AWS 更新访问密钥页都写可以先重新启用旧密钥，再继续把遗漏的调用方改过来。

4. 密钥的 Secret 通常只在创建时显示一次。阿里云创建 AccessKey 页、AWS 管理访问密钥页、Azure「添加客户端机密」以及腾讯云国际站根账号页（写明自 2023 年 11 月 30 日起）都是这个规则。丢失 Secret 不能「再查看一次」，只能作废后新建。不要把密钥写进代码、示例、文档或客户端，也不要提交到仓库；阿里云「使用访问凭据访问阿里云 OpenAPI 最佳实践」把硬编码、客户端内嵌、文档和示例里夹带密钥，列为常见泄露案例。系统环境变量比写进代码好，但仍然是长期密钥，不是实例角色那种会过期、会自动刷新的临时凭证。

5. 禁用和删除不是同一步。禁用之后，用这把密钥发出的请求会失败，但密钥往往还能再启用。删除的后果各云不一样：阿里云 AccessKey 常见问题写，删除 RAM 用户 AccessKey 时会先移入回收站，回收站保存期限为 30 天，超期系统会自动彻底删除，也可以在回收站里手动彻底删除，彻底删除后无法恢复。腾讯云国际站根账号密钥页写：a deleted API key cannot be restored。不要把阿里云的 30 天回收站套到腾讯云或 AWS。还在使用中的密钥，先完成切换再禁用，不要直接删除。

6. 人员身份和程序身份要分开。腾讯云轮转建议页写得最直接：人员子账号不创建 AccessKey，只允许登录控制台；程序子账号只使用 AccessKey，不允许登录控制台；子账号的密钥至少 90 天一轮转，用于敏感业务时用更严格的周期。阿里云创建页建议针对每一个独立业务应用分配一个独立的 RAM 用户和 AccessKey。Google Cloud 管理服务账号密钥的最佳实践写，每个应用使用专用服务账号。权限怎么授予、怎么避免管理员权限过大，见 MFA 与最小权限一文，这里不重复授权步骤。

7. 疑似泄露时，先让这把密钥失效，再查调用记录，不要只换一把同类长期密钥就结束。阿里云「AccessKey 泄露处理方案」写：不在使用中的可以直接禁用并删除；在使用中的先创建新 AccessKey，替换并验证后再禁用、删除。禁用 RAM 用户 AccessKey 时系统会要求完成 MFA；若该密钥最近 60 天内有调用记录，系统会提示禁用可能造成业务中断。该页还写：阿里云若根据外部线索掌握到 AccessKey 已被公开，会通过预留联系方式通知，并可能对该密钥做限制性保护、禁止调用部分高危 API；但按安全责任共担模型，AccessKey 属于客户侧的身份凭证，相关安全责任仍由你承担。AWS 管理访问密钥页建议用 CloudTrail 监控使用情况。腾讯云轮转建议页建议用操作审计，以及 API 密钥管理里的访问记录，确认旧密钥已经没有调用。这些页面没有写「多少分钟内一定能止损」，下面也不编造处置时限。

## 对照

下表只填这次打开的页面写明的规则。单元格写「这次没读到」，是因为对应页面没有打开，或打开了但正文没有这句。天数和上限以各章原文为准，不合成一个各云通用的按钮。

| 云 | 长期密钥 | 每个身份的上限 | 更推荐的临时或无密钥方案 | 轮换时页面怎么写 | 删除后这次读到的恢复规则 |
| --- | --- | --- | --- | --- | --- |
| 阿里云中国站 | AccessKey ID 和 AccessKey Secret | 每个 RAM 用户最多 2 个。创建按钮置灰时，需先删除现有密钥 | STS Token；ECS 实例 RAM 角色；ACK 的 RRSA；函数计算角色。不适用以上方案时，最佳实践把环境变量列为下策 | 使用超过 90 天建议轮转：建第二把，更新调用方，禁用旧的，验证后再删除；失败可再激活旧密钥 | FAQ：RAM 用户 AccessKey 先入回收站，保存 30 天，彻底删除后无法恢复 |
| 阿里云国际站 | 同上 | 轮转页写 Each RAM user can have up to two AccessKeys。Last Updated: Jul 10, 2026 | 最佳实践页（Last Updated: Sep 30, 2026）同样推荐 RRSA、实例 RAM 角色、函数角色；否则用环境变量 | 与中国站同方向，并写每 90 天或更短重复一次 | 这次没有单独打开国际站删除或回收站专页，不套用中国站 30 天 |
| 腾讯云中国站 | SecretId 和 SecretKey，文档里也称作 AccessKey | 删除接口错误码写每个账号最多支持两个 AccessKey | 给 VPC 中的 CVM 绑定载体包含 `cvm.qcloud.com` 的 CAM 角色，从实例元数据取 STS 临时密钥。一台实例一次只能授一个角色，绑定、修改、删除实例角色页写不产生额外费用 | 子账号密钥至少 90 天一轮转：先建新的，用操作审计和访问记录确认业务已切走，再禁用，观察无异常后删除 | 这次打开的轮转建议页没有写回收站。不要套用阿里云 30 天 |
| 腾讯云国际站 | 根账号页：SecretId 前缀为 IKID，加 SecretKey | 一个根账号最多 2 个 API 密钥。Last updated: 2025-04-29 10:29:09 | 这次没有打开国际站实例角色专页，不把中国站的元数据地址套过去 | 根账号页建议定期更换，并在新建后删除旧密钥。该页同时写最好不要为根账号创建日常使用的访问密钥 | 根账号页写删除后不能恢复 |
| AWS | access key ID 和 secret access key | 管理访问密钥页写每个用户最多 2 个 | IAM 角色和临时安全凭证。EC2 使用角色，不要在应用里放长期密钥 | 需要时更新，例如员工离职：建第二把，更新所有调用方，只用新密钥确认后再删除旧的；若仍有调用失败，可再激活旧密钥 | 根用户删除页：停用后可再启用；删除是永久删掉。这次没有读到回收站 |
| Microsoft Azure | 应用注册中的客户端密码；另有证书和联合凭据 | 这次打开的「添加和管理应用凭据」页没有写最多几把客户端密码。客户端密码生存期最长 24 个月，建议设为少于 12 个月 | 托管标识：向 Entra 获取令牌，无需管理凭据。该页写生产环境应使用证书或联合凭据，不应使用客户端密码 | 轮换客户端密码需新建、更新调用方，再去掉旧密码。托管标识没有需要你轮换的客户端密码 | 客户端密码值离开创建页后不再显示。这次没有打开「删除后能否找回」的专页 |
| Google Cloud | 用户管理的服务账号密钥（私钥） | 这次打开的轮换页没有在正文写最多几把。组织政策可能禁止创建或上传，见 MFA 一文已引用的最佳实践 | 工作负载身份联合；代码跑在 Google Cloud 上时，把服务账号附加到资源，或使用 Workload Identity Federation for GKE。轮换页不建议用 Secret Manager 存放并轮换服务账号密钥 | 建议至少每 90 天轮换：识别、建新、全量替换、禁用并观察、再删除。怀疑泄露则立即轮换 | 轮换页把删除被替换的密钥作为最后一步。这次没有另开到一份「删除后可恢复多久」的专页 |

## 阿里云

### 主账号 AccessKey 不要留，RAM 用户最多两把

[创建 AccessKey](https://help.aliyun.com/zh/ram/user-guide/create-an-accesskey-pair) 写明：AccessKey 是长期访问凭证，由 AccessKey ID 和 AccessKey Secret 组成，用于 CLI、SDK、Terraform 等程序性调用，不能用于控制台登录。AccessKey Secret 只在创建时显示一次，后续无法查看。主账号 AccessKey 默认拥有当前账号下的全部操作权限，该页把创建主账号 AccessKey 写成「不推荐」，并强烈建议不创建、不使用。RAM 用户 AccessKey 继承该用户的权限，可以做最小化授权；建议每个独立业务应用使用独立的 RAM 用户和 AccessKey。

每个 RAM 用户最多允许创建 2 个 AccessKey。创建按钮置灰，表示已经达到上限，需要先删除一个现有 AccessKey，按钮才会恢复。删除须先禁用，再删除并移入回收站。国际站 [Create an AccessKey pair](https://www.alibabacloud.com/help/en/ram/user-guide/create-an-accesskey-pair)（Last Updated: Sep 11, 2026）对长期凭证、Secret 只显示一次、主账号 AccessKey 不推荐，以及应给每个应用单独的 RAM 用户和 AccessKey，与中国站同方向。

[不使用主账号 AccessKey](https://help.aliyun.com/zh/ram/avoid-using-the-accesskey-pair-of-an-alibaba-cloud-account) 把这件事标成高风险，并按使用情况给了治理建议：从未使用的，直接禁用，禁用 90 天后如果没有相关问题再删除；超过 90 天未使用的，先确认程序里是否还在用，确认不用再按同样方式禁用并等待；确认仍在使用的，先创建有适当权限的 RAM 用户 AccessKey，完成替换后再处理主账号那一把。等待 90 天再删除，只属于这一页对主账号 AccessKey 的原文，不要套到其他云，也不要理解成「禁用满 90 天就自动删除」。

### 超过 90 天要轮转：先建第二把，再停旧的

[定期轮转 RAM 用户 AccessKey](https://help.aliyun.com/zh/ram/regularly-rotate-accesskey-pairs-of-ram-users) 写：当 RAM 用户的 AccessKey 已经使用超过 90 天，建议及时轮转。最佳实践仍是优先用 STS Token；只能使用固定 AccessKey 时，再定期轮转。治理上可以按 [轮转 RAM 用户的 AccessKey](https://help.aliyun.com/zh/ram/user-guide/rotate-accesskey-pairs-of-ram-users) 手工轮转，或使用密钥管理服务 KMS 的凭据管家做自动轮转。那一页把手工步骤写成：

1. 创建用于轮转的第二个 AccessKey。
2. 在所有使用它的应用程序或系统中，改成新创建的第二把。可在用户详情页查看最后使用时间，初步判断新密钥是否已被使用、旧密钥是否已经不用。
3. 禁用原来的 AccessKey。
4. 验证业务。运行正常再删除原来的 AccessKey；运行异常则暂时重新激活原来的 AccessKey，重复更新、禁用和验证，直到成功。
5. 删除原来的 AccessKey。

国际站 [Rotate an AccessKey for a RAM user](https://www.alibabacloud.com/help/en/ram/user-guide/rotate-accesskey-pairs-of-ram-users)（Last Updated: Jul 10, 2026）步骤同方向，并写 Repeat this rotation every 90 days or sooner。

[AccessKey 常见问题](https://help.aliyun.com/zh/ram/support/faq-about-accesskey-pairs) 写：删除 RAM 用户 AccessKey 时会先移入回收站，回收站中的 AccessKey 支持恢复，保存期限为 30 天；超过期限系统会自动彻底删除，也可以在回收站中手动彻底删除，彻底删除后无法恢复。该页同时警告：使用中的 AccessKey 一旦删除，可能造成应用系统故障。

### 能用 STS 和实例角色时，不要把 AccessKey 放进服务器

[临时访问凭证 STS 概念](https://help.aliyun.com/zh/ram/user-guide/what-is-sts) 写：RAM 角色没有登录密码或访问密钥，只能由可信实体扮演后，通过 STS 取得可以自定义时效和权限的 STS Token。产品优势写成：减少长期 AccessKey 泄露风险；到期自动失效，无需定期轮换；还可以为 Token 绑定自定义权限策略。

[使用访问凭据访问阿里云 OpenAPI 最佳实践](https://help.aliyun.com/zh/ram/use-cases/best-practices-for-using-access-credentials-to-access-alibaba-cloud-openapi) 按部署位置给了方案，不要混用：

- 应用程序在 ACK 上：用 RRSA，让各应用扮演独立 RAM 角色，取得临时凭证。该页写 RRSA 目前支持 1.22 及以上版本的 ACK 托管、Serverless 和 ACK Edge Pro 等所列集群，并需使用阿里云 SDK V2.0。
- 应用程序在 ECS 上：为实例授予 RAM 角色，应用访问实例元数据获取 STS 临时凭证，凭证周期性更新。
- 员工做运维或调试：建议通过角色 SSO 扮演对应职能的 RAM 角色，获取 STS Token，而不是长期使用自己的 AccessKey。
- 函数计算中的应用访问其他云资源：把函数和 RAM 角色关联，使用 STS Token。
- 以上都不适用时，再考虑把凭据放到系统环境变量，而不是写进代码。

国际站 [Best practices for accessing Alibaba Cloud OpenAPI](https://www.alibabacloud.com/help/en/ram/use-cases/best-practices-for-using-access-credentials-to-access-alibaba-cloud-openapi)（Last Updated: Sep 30, 2026）方案表与中国站同方向。这次没有单独把「实例 RAM 角色」操作专页再打开一遍，绑定角色的菜单以 ECS 当前文档为准。

### 疑似泄露：先失效，再轮转，不要只靠平台通知

[AccessKey 泄露处理方案](https://help.aliyun.com/zh/ram/user-guide/solution-to-accesskey-leakage) 写：AccessKey 泄露会威胁该账号下资源安全，可能带来非预期费用。阿里云若根据外部线索掌握到 AccessKey 已被公开，会通过预留联系方式通知，并可能对该密钥做限制性保护；但该页同时写，阿里云无法掌握全部 AccessKey 的使用安全情况，按安全责任共担，AccessKey 的安全仍由你负责。务必关注短信、邮箱和站内消息，并查看账号下的云资源异动。

手动处理按是否仍在使用来分：

- 主账号或 RAM 用户的 AccessKey 确认不在使用中：直接禁用并删除。
- 确认仍在使用中：先创建新 AccessKey 并妥善保管 Secret，替换后再禁用、删除旧的。
- 禁用 RAM 用户 AccessKey 需要完成 MFA。若最近 60 天内有调用记录，系统会提示可能造成业务中断。
- 仅禁用可能不足以解除因泄露触发的登录限制，该页写必须完成删除或轮转；若 RAM 用户仍无法登录，再提交工单申请解除风控限制。页面没有保证提交工单后多久一定恢复，这里也不写时限。

## 腾讯云

### 中国站：人员不持有密钥，程序密钥至少 90 天一轮转

[定期轮转访问密钥（AccessKey）的安全建议](https://cloud.tencent.com/document/product/598/123925) 建议先在访问管理概览页下载用户凭证报告，盘点主账号下全部密钥的状态和使用情况。该页强烈建议不要创建或使用主账号密钥；日常操作和 API 调用改用 CAM 子用户和角色授权。对子账号，它写了三条：权限符合最小权限；人员子账号与程序子账号分离，人员子账号不创建 AccessKey、只允许登录控制台，程序子账号只使用 AccessKey、不允许登录控制台；子账号密钥至少 90 天一轮转，敏感业务用更严格的周期。

轮转步骤该页写成：在 API 密钥管理生成新的子用户密钥并更新到应用；禁用旧密钥前，用操作审计，以及访问管理 > API 密钥管理 > 更多访问记录，确认业务已切换；禁用后观察运行状态，确认无异常再删除。具体禁用、删除入口以该页链接的子账号密钥管理文档为准。

[查看访问密钥活跃时间](https://cloud.tencent.com/document/product/598/120122) 写：腾讯云于 2025 年 6 月 30 日起分批次对所有用户默认启用了自动禁用超过 90 天不活跃的 AccessKey 功能，详情见该页链接的公告。这是中国站这一页的原文，国际站这次没有打开对应公告，不要直接套过去。该页还写：控制台显示的最近访问时间与「更多访问记录」可能不一致，两者都可能有 3 到 5 小时延时；访问记录包含调用任何云服务 API，无论成功或失败。若密钥需要保留，该页写在每 90 天内主动用该密钥发起一次 API 调用以刷新活跃时间，或按推荐方式创建新密钥并轮换、在确认废弃后立即禁用旧密钥。自动禁用不是轮换本身，也不等于已经替你删除旧密钥。

这次打开的中国站页面里，删除接口文档的错误码有 OperationDenied.AccessKeyOverLimit：每个账号最多支持两个 AccessKey。创建第二把用来轮转前，先看是不是已经占满。

### 中国站：CVM 用实例角色拿临时密钥，不要把 SecretKey 留在盘上

[管理实例角色](https://cloud.tencent.com/document/product/213/47668) 写：把 CAM 角色关联到 CVM 后，可在实例内部基于 STS 临时密钥访问其他云产品 API，临时密钥周期性更新；相比直接用 SecretKey，不必自行在实例中保存 SecretKey，改角色授权即可变更权限。使用限制该页写明：实例只支持绑定角色载体包含 `cvm.qcloud.com` 的角色；实例网络类型须为专有网络 VPC；一台实例一次仅支持授予一个 CAM 角色；绑定、修改及删除实例角色不产生额外费用。

获取临时密钥的方式，该页写在实例内访问元数据：

`curl http://metadata.tencentyun.com/latest/meta-data/cam/security-credentials/<角色名称>`

返回值包含 TmpSecretId、TmpSecretKey、Token、ExpiredTime 和 Expiration。角色名称可在控制台实例详情页的基本信息里查看。该地址属于中国站这一页的原文，不要默认用到国际站或其他云。SDK 或 CLI 使用临时密钥时，需要同时带上 Token，只替换 SecretId 和 SecretKey 不够。

### 国际站这次只打开了根账号密钥管理

[Root Account Access Key Management](https://www.tencentcloud.com/document/product/598/34228)（Last updated: 2025-04-29 10:29:09）写：访问密钥用于 API 调用；根账号对资源有完全控制权，最好不要为根账号创建访问密钥并用于日常任务。一个根账号最多创建 2 个 API 密钥。SecretId 用来标识调用方，前缀为 IKID；SecretKey 用来验证身份。该页写，自 2023 年 11 月 30 日起，创建时只会提供 SecretKey，之后无法再查询，须自行妥善保管。禁用后，使用该密钥的请求会被阻止，还可以再启用。删除前要先禁用；删除后不能恢复。该页的访问记录说明写：More Access Records 展示过去 3 个月最新 20 条，包含成功和失败，可能有约 1 小时延迟。

这次没有打开国际站的子用户密钥轮转建议页、90 天自动禁用公告，以及实例角色专页。中国站的「人员子账号不创建密钥」「90 天不活跃自动禁用」和元数据地址，不要直接当作国际站现行规则。

## AWS

### 不要给根用户建访问密钥，IAM 用户最多两把

[Manage access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html) 把使用临时安全凭证（例如 IAM 角色）写成最佳实践，优先于创建长期 access keys。Access keys 由 access key ID 和 secret access key 组成，用于给 CLI 或 API 请求签名。Secret 只能在创建时查看，丢失后必须删除再新建。该页写 You can have a maximum of two access keys per user。它同时写：不要用账户的根凭证创建 access keys；不要把 access keys 或凭证信息放进应用程序文件，也不要把含有这些信息的文件放进项目目录；共享凭证文件中的信息是明文存储的。创建之后，该页建议用 CloudTrail 监控使用情况，按需要定期审查、更新和删除。

[Root user best practices for your AWS account](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html) 有专门一节 Don't create access keys for the root user。该页还写：只要有可行方案，应依赖临时凭证，而不是创建带有长期密码和 access keys 的 IAM 用户。根用户只留给必须由它完成的任务；日常管理应使用其他管理身份。登录 MFA 的绑法见 MFA 与最小权限一文，这里不重复。

[Delete access keys for the root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user_manage_delete-key.html) 写：这些步骤必须用根用户登录，不能以 IAM 用户或角色来做。可以先停用再删除。停用后，用该密钥访问 API 会失败并返回 access denied，但以后还能再启用，不必改密钥 ID 或 secret；删除则是永久删掉。删除对话框里需要先选择 Deactivate，再输入 access key ID 确认。

这次没有打开 docs.amazonaws.cn，也不把上述全球文档里的控制台路径保证为 AWS 中国区的同样菜单。

### 更新密钥：先切新的，再停旧的；EC2 用角色，不要内嵌密钥

[Update access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id-credentials-access-keys-update.html) 写：作为安全最佳实践，在需要时更新 IAM 用户的 access keys，例如员工离职。控制台、CLI 和 API 的无中断更新步骤同方向：

1. 创建第二把 access key。
2. 更新所有应用程序和工具，改用新密钥。
3. 只用新密钥确认应用工作正常。若仍有工具在用原来的密钥，它们会在旧密钥被停用后无法访问；这时可以把第一把重新激活，回到更新调用方这一步。
4. 再等待一段时间，确认所有调用方都已更新后，删除第一把。

该页没有在这次读到的正文里写死「必须每 90 天轮换一次」。不要把阿里云或腾讯云中国站的 90 天原文套到 AWS。

[Secure access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html) 对 EC2 上的应用写得很明确：不要在应用中直接使用 access keys，不要把密钥传给应用，不要嵌入应用，也不要让应用从任何来源读取它们。应定义具有适当权限的 IAM 角色，并在启动实例时关联该角色，或把角色附加到已有实例；应用由此取得临时安全凭证，SDK 和 CLI 可以自动获取。跨账户访问用 IAM 角色建立信任，而不是把长期密钥拷到另一个账户。移动应用不要内嵌 access keys，即便加密存储也不要；该页把 Amazon Cognito 列为管理应用内用户身份的做法。这次没有再单独打开「IAM roles for Amazon EC2」逐步操作专页，实例配置文件和控制台勾选位置以 EC2 当前文档为准。

## Microsoft Azure

### 在 Azure 上跑的应用，优先用托管标识

[Azure 资源的托管标识](https://learn.microsoft.com/zh-cn/entra/identity/managed-identities-azure-resources/overview) 写：手动管理机密和证书是已知的安全问题和故障来源；托管标识让开发人员无需管理这些凭据。应用程序可以使用托管标识从 Microsoft Entra 获取令牌，无需管理任何凭据。该页还写：你甚至可能都无法访问这些凭据；托管标识可以用来向支持 Microsoft Entra 身份验证的资源进行身份验证；使用托管标识不产生额外费用。

托管标识有两种，不要混：

- 系统分配：在虚拟机等资源上直接启用。Entra ID 中会为该资源创建特殊类型的服务主体，和该 Azure 资源的生命周期绑定；删除父资源时，Azure 会自动删除这个服务主体。按设计，只有该资源能用它向 Entra ID 请求令牌。
- 用户分配：作为独立的 Azure 资源创建，可以赋给一个或多个计算资源。该页把用户分配写成 Microsoft 服务建议的托管标识类型。它有独立生命周期，删除计算资源时不会自动删掉这个标识，需要你显式删除。

使用步骤该页概括为：创建并分配托管标识，用 Azure RBAC 授予它对目标服务的权限，然后在代码里用 Azure.Identity 或 MSAL 获取令牌。令牌获取不需要机密，并按代码运行的环境自动进行身份验证。该页还写，可以把托管标识作为 Entra ID 应用的联合标识凭据（FIC）使用；在 Entra ID 应用中将托管标识用作 FIC 时，限制为 20 个 FIC。支持哪些服务，以该页链接的「支持 Azure 资源托管标识的服务」名单为准，这里不抄一份可能过期的清单。

由世纪互联运营的 Azure 这次没有打开，菜单和限制以那一侧当前文档为准。

### 必须用应用凭据时：生产环境不要用客户端密码

如果工作负载不在 Azure 里，或仍然需要应用注册，[在 Microsoft Entra ID 中添加和管理应用凭据](https://learn.microsoft.com/zh-cn/entra/identity-platform/how-to-add-credentials) 把三种凭据分开写：证书、客户端密码、联合凭据。该页写：Microsoft 建议在应用程序进入生产环境之前使用证书而不是客户端密码；客户端密码的安全性低于证书或联合凭据，因此不应在生产环境中使用，虽然它们可能方便本地开发。生产环境中运行的应用程序必须使用证书或联合凭据。

添加客户端密码的路径，该页写：在 Microsoft Entra 管理中心的「应用注册」中打开应用，进入「证书和机密」>「客户端密码」>「新建客户端密码」，填写说明并选择过期时间。客户端机密生存期限制为两年（24 个月）或更短，不能指定超过 24 个月的自定义生存期；Microsoft 建议将过期时间设为少于 12 个月。添加后立即记录「值」，退出该页后此机密值永不再显示。12 个月和 24 个月只属于这一页的原文，不要套到其他云的 AccessKey。

联合凭据用于 GitHub Actions、Kubernetes 或 Azure 外部的计算平台等场景，让工作负载在不必自己管理机密的情况下获取应用程序令牌。轮换客户端密码时，这次打开的页面写的是如何添加新密码，以及 Azure DevOps 自动创建的服务连接要到 Azure DevOps 门户更新，而不是直接在 Entra 里改。它没有写成阿里云那样的「最多两把、先禁用再删」五步流程。稳妥做法仍是：先添加新密码并更新所有调用方，确认旧密码已无流量后，再删除旧密码。不要在只看到新密码后立刻删掉旧的。

## Google Cloud

### 能不用用户管理的密钥，就不要创建；必须留下时至少每 90 天轮换

服务账号密钥一旦落到不该存放的地方，持有私钥的人就可以该服务账号的身份调用 API，而且不一定再经过用户账号的两步验证。威胁、组织政策默认禁止创建或上传、以及「不要把密钥提交到仓库或留在下载目录」，见 [云账号登录和权限怎么收紧：多因素认证与最小权限](cloud-account-mfa-least-privilege.md) 里「服务账号密钥」一节，以及那里引用的 [管理服务账号密钥的最佳实践](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys)。下面只写这次另外打开的轮换页和联合身份页。

[Key rotation](https://docs.cloud.google.com/iam/docs/key-rotation) 写：应定期轮换你管理的全部密钥，包括服务账号密钥；建议至少每 90 天轮换一次，以降低泄露密钥仍有效的风险。如果认为密钥已经泄露，应立即轮换。该页再次强调：只要有更安全的替代方案，就不要用服务账号密钥；如果无法创建密钥，可能是组织关闭了这项能力。从外部来源拿到的密钥，使用前必须先验证。

轮换步骤该页写成：

1. 识别需要轮换的服务账号密钥。
2. 为同一批服务账号创建新密钥。
3. 在所有应用程序中用新密钥替换旧密钥。
4. 禁用被替换的密钥，并观察应用程序是否按预期工作。
5. 删除被替换的服务账号密钥。

该页提到可以用集中的密钥管理服务做自动轮换，但明确不建议用 Google Cloud 的 Secret Manager 来存放和轮换服务账号密钥：要读取 Secret Manager 里的密钥，应用本身就需要一个 Google Cloud 能够识别的身份；如果应用已经有这种身份，它本可以直接用该身份访问 Google Cloud，而不必再使用服务账号密钥。对其他云上的密钥管理服务，该页也写了同样的思路。不要把「密钥放进保管箱」理解成已经改用了临时凭证。

### 外部工作负载用联合身份，而不是下载一把长期密钥

[Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation) 写：它让本地或多云工作负载使用联合身份访问 Google Cloud 资源，而不使用服务账号密钥，从而去掉密钥保管和轮换负担。可以配置给使用 X.509 客户端证书的服务、跑在 AWS 或 Azure 上的工作负载、连接本地 Active Directory 的系统、GitHub 和 GitLab 等流水线，以及任何支持 OIDC 或 SAML 2.0 的外部身份提供方。配置后，把外部身份提供方的凭证交给 Security Token Service，换来联合令牌。权限可以直接授给工作负载身份池中的主体，也可以先模拟服务账号。该页建议：为每个需要访问 Google Cloud 的非 Google Cloud 环境（例如开发、预发、生产）创建单独的工作负载身份池。代码如果已经跑在 Google Cloud 上，仍应优先把服务账号附加到资源，或使用 GKE 的工作负载身份联合，而不是先下载用户管理的密钥。

## 检查清单

每个账号做一遍。菜单名称以你登录的站点为准。

1. 主账号、根用户下没有仍启用的长期访问密钥。阿里云在账号头像的 AccessKey 页核对。腾讯云中国站在访问管理的 API 密钥管理和用户凭证报告里核对。AWS 用根用户打开安全凭证页核对。还在使用的，先改成最小权限的子身份或角色，再禁用旧密钥。
2. 控制台登录已经启用 MFA，且日常登录不是主账号或根用户。这一条的做法见 [云账号登录和权限怎么收紧：多因素认证与最小权限](cloud-account-mfa-least-privilege.md)，不要只轮换密钥却不收紧登录。
3. 跑在云服务器、容器或函数里的程序，已经改用实例角色、托管标识、附加服务账号或联合身份，实例磁盘、镜像和代码仓库里没有长期密钥文件。还不能改的，密钥属于单独的程序身份，权限已收窄，并且不和人员登录账号混用。
4. 仍在使用的长期密钥没有超过各页建议的使用时间：阿里云 RAM 用户 AccessKey 使用超过 90 天建议轮转；腾讯云中国站写子账号密钥至少 90 天一轮转，并核对该站是否已启用 90 天不活跃自动禁用；Google Cloud 建议服务账号密钥至少每 90 天轮换。AWS 这次打开的更新页写的是在需要时更新，例如员工离职，不要把 90 天套过去。Azure 客户端密码核对其过期时间是否仍在该页建议的范围内。
5. 轮换时先创建第二把，更新全部调用方，再禁用旧的并观察，最后再删除。确认没有漏掉定时任务、CI/CD 变量、监控探针和另一台机器上的配置。阿里云删除 RAM 用户 AccessKey 后，还要记住回收站会再留 30 天。
6. 人员账号没有创建长期访问密钥；程序账号不能登录控制台，或至少没有把同一把密钥发给多个互不相关的应用。
7. 知道疑似泄露时要先禁用或轮转哪一把密钥，并从操作审计、CloudTrail 或密钥访问记录里核对应急措施是否生效。阿里云还要查看短信、邮箱和站内消息里是否有限制性保护通知。
8. 阿里云国际站、腾讯云国际站再打开该站点文档核对按钮名称和限制，不要把中国站的 90 天自动禁用、元数据地址或回收站天数用到国际站。

## 常见坑

- 以为控制台开了 MFA，主账号或根用户的长期密钥就不再能调 API。登录保护和程序访问凭证是两套机制。
- 把 AccessKey、SecretKey 或服务账号 JSON 写进代码、示例、镜像或客户端，提交到仓库后即使后来轮换了生产配置，历史提交里的旧密钥仍可能有效，直到你禁用或删除它。
- 两把密钥都占满了，不先处理掉其中一把，就无法创建用于轮换的新密钥。禁用往往仍占用名额，阿里云创建页写的是达到上限后需先删除。
- 只改了生产环境的环境变量，漏掉 CI/CD、定时任务、另一地域的实例或本地开发机，一禁用旧密钥就有任务失败。这时应先按阿里云和 AWS 的页面重新启用旧密钥，补齐调用方后再禁。
- 把阿里云 RAM 用户 AccessKey 的 30 天回收站，或腾讯云中国站「90 天不活跃自动禁用」，套到其他云或其他站点。
- 在 ECS 或 CVM 里写入长期密钥，却以为「机器在我的账号下」就等于已经用了实例角色。实例角色的临时凭证来自元数据，并且会过期刷新；写在磁盘上的 Secret 不会。
- 用 Secret Manager 一类产品存放 Google Cloud 服务账号密钥，以为这样就不用轮换、也不会泄露。Google Cloud 的轮换页写，能读取保管箱的身份往往已经可以被 Google Cloud 识别，这时更应该直接用该身份，而不是再包一层长期密钥。
- 给 Azure 虚拟机启用了系统分配的托管标识，删除虚拟机后这个标识会随资源一起删除；用户分配的标识则不会自动消失，权限可能还留在目标资源上。
- 轮换后把旧密钥直接删除，而不是先禁用观察。腾讯云国际站根账号页写删除后不能恢复；AWS 根用户密钥删除也是永久的。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。页上能看到的更新日期已注明；中国站部分页面在这次抓取里只有「更新时间」栏、没有具体日期的，按页上未见更新日期处理。

阿里云中国站：

- [创建 AccessKey](https://help.aliyun.com/zh/ram/user-guide/create-an-accesskey-pair)（页上未见具体更新日期）
- [定期轮转 RAM 用户 AccessKey](https://help.aliyun.com/zh/ram/regularly-rotate-accesskey-pairs-of-ram-users)（页上未见具体更新日期）
- [轮转 RAM 用户的 AccessKey](https://help.aliyun.com/zh/ram/user-guide/rotate-accesskey-pairs-of-ram-users)（页上未见具体更新日期）
- [不使用主账号 AccessKey](https://help.aliyun.com/zh/ram/avoid-using-the-accesskey-pair-of-an-alibaba-cloud-account)
- [AccessKey 常见问题](https://help.aliyun.com/zh/ram/support/faq-about-accesskey-pairs)
- [临时访问凭证 STS 概念](https://help.aliyun.com/zh/ram/user-guide/what-is-sts)
- [使用访问凭据访问阿里云 OpenAPI 最佳实践](https://help.aliyun.com/zh/ram/use-cases/best-practices-for-using-access-credentials-to-access-alibaba-cloud-openapi)
- [AccessKey 泄露处理方案](https://help.aliyun.com/zh/ram/user-guide/solution-to-accesskey-leakage)（页上未见具体更新日期）

没有单独打开「实例 RAM 角色」操作专页、禁用 AccessKey 逐步页，以及 KMS 凭据管家自动轮转的完整配置页。ECS 上授予角色的菜单以实例 RAM 角色当前文档为准。

阿里云国际站：

- [Create an AccessKey pair](https://www.alibabacloud.com/help/en/ram/user-guide/create-an-accesskey-pair)（Last Updated: Sep 11, 2026）
- [Rotate an AccessKey for a RAM user](https://www.alibabacloud.com/help/en/ram/user-guide/rotate-accesskey-pairs-of-ram-users)（Last Updated: Jul 10, 2026）
- [Best practices for accessing Alibaba Cloud OpenAPI](https://www.alibabacloud.com/help/en/ram/use-cases/best-practices-for-using-access-credentials-to-access-alibaba-cloud-openapi)（Last Updated: Sep 30, 2026）

没有打开国际站的删除、回收站和泄露处理专页。

腾讯云中国站：

- [定期轮转访问密钥（AccessKey）的安全建议](https://cloud.tencent.com/document/product/598/123925)（页上未见更新日期）
- [查看访问密钥活跃时间](https://cloud.tencent.com/document/product/598/120122)（页上未见更新日期）
- [管理实例角色](https://cloud.tencent.com/document/product/213/47668)（页上未见更新日期）

没有单独打开「子账号访问密钥管理」「禁用子账号 API 密钥」「删除子账号 API 密钥」逐步页，以及 2025-06-30 自动禁用公告正文；90 天不活跃自动禁用这一事实来自「查看访问密钥活跃时间」页上的原文和它给出的公告链接。每个账号最多两个 AccessKey，来自这次打开的删除访问密钥接口错误码说明。

腾讯云国际站：

- [Root Account Access Key Management](https://www.tencentcloud.com/document/product/598/34228)（Last updated: 2025-04-29 10:29:09）

没有打开国际站的子用户密钥轮转建议、自动禁用公告和实例角色专页。

AWS：

- [Manage access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)（页上未见更新日期）
- [Update access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id-credentials-access-keys-update.html)（页上未见更新日期）
- [Secure access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html)（页上未见更新日期）
- [Root user best practices for your AWS account](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)（页上未见更新日期）
- [Delete access keys for the root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user_manage_delete-key.html)（页上未见更新日期）

没有打开 docs.amazonaws.cn，以及 IAM roles for Amazon EC2 的逐步操作专页。

Microsoft Azure：

- [Azure 资源的托管标识](https://learn.microsoft.com/zh-cn/entra/identity/managed-identities-azure-resources/overview)（页上未见更新日期）
- [在 Microsoft Entra ID 中添加和管理应用凭据](https://learn.microsoft.com/zh-cn/entra/identity-platform/how-to-add-credentials)（页上未见更新日期）

没有打开由世纪互联运营的 Azure 文档，以及「支持 Azure 资源托管标识的服务」完整名单页。

Google Cloud：

- [Key rotation](https://docs.cloud.google.com/iam/docs/key-rotation)（页上未见更新日期）
- [Best practices for managing service account keys](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys)（页上未见更新日期；与 MFA 一文交叉引用，本次为核对轮换相关表述再次打开）
- [Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation)（页上未见更新日期）

如需为阿里云、腾讯云、AWS、谷歌云开户或使用 USDT 充值，可访问自助站 <https://www.cloudcup.online>。海宝云官网：<https://www.haibaocloud.com>。Telegram：@cloudcup。充值币种、网络和最低金额以各站点当前说明为准。
