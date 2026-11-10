# 对象存储私有桶与临时访问链接：预签名 URL 和签名 URL

对象要给别人看时，不必把桶改成公共读。公共读的地址谁拿到都能下，外网流出和请求费记在桶的拥有者账上。私有桶配合有有效期的临时链接，只把这一次的下载或上传交给拿到链接的人。链接本身就是凭证：有效期内可以反复使用，过期、签名密钥停用，或者文档写明的策略被撤销之后，才不能再当通行证。

各云的叫法不同。阿里云 OSS、腾讯云 COS、AWS S3、Google Cloud Storage 文档里的预签名 URL 或 Signed URL，是把签名放进查询字符串。Azure Blob Storage 的对应物是共享访问签名（SAS）。它们都不是匿名访问。身份和长期密钥怎么收，见 [云账号登录和权限怎么收紧](cloud-account-mfa-least-privilege.md)。云服务器那几类费用不在这里重复，见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。

核对时间：2026-11-10。资料核对日期：2026-11-10。控制台路径、有效期上限、默认值和命令参数都来自这一天打开的官方页面，并附有链接。阿里云、腾讯云同时有中国站和国际站。AWS 全球商业区和 AWS 中国区、Azure 全球云和由世纪互联运营的 Azure，账号与文档站点也不通用。下文只把这次打开过的页面写进去；某条规则若只在一个站点核对过，会标明。页面改版后，以你账号所属站点的当前文档为准。

## 先做这三件事

1. 桶保持私有，并打开该云用来挡住匿名公共访问的开关。对象需要外发时，发临时链接，不把 ACL 改成公共读。
2. 签名用的身份只保留这一次操作需要的权限。下载用读，上传用写。不要用主账号、根用户或整个账号的管理员密钥来签名。
3. 有效期按这次传输需要的时间来设。链接发出去就收不回“这一条”，只能让它到期，或按该云文档停用签它的密钥、委派密钥或存储访问策略。

## 阿里云 OSS

中国站规则来自 help.aliyun.com。国际站同一主题的有效期表来自 www.alibabacloud.com 的英文用户指南，数字与中国站一致，见本节有效期。

### 私有，并打开阻止公共访问

[Bucket ACL](https://help.aliyun.com/zh/oss/user-guide/oss-bucket-acl) 的默认值是 `private`：只有拥有者或被授权用户能读写。`public-read` 允许任何人读取，`public-read-write` 允许任何人读写。该页写明，除特殊场景外不要用公共读写；公共读也可能造成数据暴露和费用上升。

阻止公共访问是另一层。开启后，已有的公共 ACL 和 Bucket Policy 会被忽略，也不能新建公共访问权限。优先级是 OSS 全局，高于单个 Bucket，高于接入点，高于对象 FC 接入点。全局开着时，下层无法单独放行。见 [阻止公共访问](https://help.aliyun.com/zh/oss/user-guide/block-public-access)。

控制台：

- 账号下全部 Bucket：OSS 管理控制台，左侧「数据服务 > 阻止公共访问」。
- 单个 Bucket：Bucket 列表，进入目标 Bucket，「权限控制 > 阻止公共访问」。
- 查看或修改 ACL：同一 Bucket 的「权限控制 > 读写权限」。创建时如果阻止公共访问开着，ACL 只能是 private；要改成公共读或公共读写，必须先关掉阻止公共访问。

默认值不要混成一句。[Bucket ACL](https://help.aliyun.com/zh/oss/user-guide/oss-bucket-acl) 写：创建 Bucket 时默认开启阻止公共访问。[创建 Bucket 的 API 说明](https://help.aliyun.com/zh/oss/developer-reference/put-bucket) 写的是另一句：北京时间 2025-10-13 10:00:00 起，通过 API、SDK、ossutil 创建 Bucket 时逐步默认开启，各地域何时生效见公告。本文没有逐地域打开那份公告。[PutBucketPublicAccessBlock](https://help.aliyun.com/zh/oss/developer-reference/putbucketpublicaccessblock) 里 `BlockPublicAccess` 字段的缺省描述是 `false`，这是该接口元素不传时的缺省，不是“旧 Bucket 已经关掉”。已经建好的 Bucket，以控制台「权限控制 > 阻止公共访问」的当前状态为准。

阻止公共访问挡的是匿名公共访问。私有对象访问失败时，[0003-00000005](https://help.aliyun.com/zh/oss/user-guide/0003-00000005) 的处理建议之一就是由有权限的账号签发 URL，而不是把 ACL 改成公共读。

ossutil 2.0 打开 Bucket 级开关的示例使用占位桶名。见 [阻止公共访问](https://help.aliyun.com/zh/oss/user-guide/block-public-access)：

```bash
ossutil api put-bucket-public-access-block --bucket <bucket-name> --public-access-block-configuration "{\"BlockPublicAccess\":\"true\"}"
```

### 生成预签名 URL

文件默认私有。拥有者可以生成带有效期的下载或预览链接。见 [使用预签名 URL 下载或预览文件](https://help.aliyun.com/zh/oss/user-guide/how-to-obtain-the-url-of-a-single-object-or-the-urls-of-multiple-objects)。

控制台有不止一个入口，默认秒数不一样，都写在同一页：

- 进入 Bucket 的文件列表，点开文件，在右侧详情里复制文件 URL。这一条的默认有效期是 300 秒。
- 详情面板里的过期时间，用来改这条链接的有效期。生成之后不能再改，过期前可以多次访问。
- 批量导出 URL 的参数表把过期时间写成 60 到 32400 秒。不要把 60 秒当成上面那个 300 秒入口的默认值。

ossbrowser 2.0：文件列表里对目标对象选「获取地址」，弹窗默认 3600 秒，最大 32400 秒。ossbrowser 2.0 只支持 V4，1.0 只支持 V1。

命令行用 ossutil 2.0 的 `presign`。不带时间参数时，默认 15 分钟。见该用户指南，以及 [presign](https://help.aliyun.com/zh/oss/developer-reference/presign-generate-presigned-url)：

```bash
ossutil presign oss://<bucket-name>/EXAMPLE.txt --expires-duration 1h
```

`--expires-duration` 的取值范围是 1 秒到 1 周，默认 `15m0s`。ossutil 1.0 的命令名是 `sign`，默认签名版本是 V1；2.0 的 `presign` 默认 V4。见 [ossutil 1.0 和 2.0 的功能差异](https://help.aliyun.com/zh/oss/developer-reference/1-0-and-2-0-functional-differences)。

上传用 PUT 的预签名 URL。SDK 生成的上传链接，该页写最大 7 天；用 STS Token 时最大 43200 秒。有效期内可以多次上传，多次上传会覆盖同一对象。见 [使用预签名 URL 上传文件](https://help.aliyun.com/zh/oss/user-guide/upload-files-using-presigned-urls)。

### 有效期上限

同一篇下载文档把上限按签名版本和工具拆开。推荐 V4。V4 用“签名时的 UTC 时间加一段秒数”，最大 604800 秒（7 天）。V1 用 Unix 时间戳，文档写的上限是时间戳能表示的最大时间，没有写成某一个天数，并且写明 V1 不再维护。

| 工具 | 文档写明的最大有效期 |
| --- | --- |
| OSS SDK 2.0、ossutil 2.0 | V4 为 604800 秒，超出报错。默认 V4。V1 仍可指定，上限是时间戳上限 |
| OSS SDK 1.0、ossutil 1.0 | V4 同样写 604800 秒，但超出不报错。默认 V1 |
| 控制台、ossbrowser 1.0 和 2.0 | 32400 秒（9 小时） |

SDK 1.0 或 ossutil 1.0 在 V4 下超出 7 天却不报错，页面没有写“不报错就等于链接能用过 7 天”。V4 的上限仍按 604800 秒理解。

用 STS Token 签名时，链接不能长过 Token 本身，文档写的上限是 43200 秒（12 小时）。这是这条路径的上限，不是每一张 Token 都会发满 12 小时。

国际站英文用户指南的同一张表，V4、V1、STS、控制台和 ossbrowser 的数字与上表相同。见 [Using presigned URLs to download or preview files](https://www.alibabacloud.com/help/en/oss/user-guide/how-to-obtain-the-url-of-a-single-object-or-the-urls-of-multiple-objects)。国际站 V4 参数页还写了一条时间窗口：OSS 收到请求的时间 T 要满足 `x-oss-date` 前 15 分钟到 `x-oss-date` 加 `x-oss-expires` 之间；用 AccessKey 时 `x-oss-expires` 为 1 到 604800 秒，用 STS 时为 1 到 43200 秒。见 [V4 signatures in presigned URLs](https://www.alibabacloud.com/help/en/oss/developer-reference/add-signatures-to-urls)。中国站用户指南的有效期表没有把这 15 分钟写成另一条上限。

地域不同，Endpoint 不同。同地域的 ECS 访问应使用内网 Endpoint。这是地域差异，不是中国站和国际站两套有效期。

### 用最小权限签名

下载或预览要成功，签名身份需要 `oss:GetObject`。上传要成功，需要 `oss:PutObject`。见上面的下载文档和上传文档。上传文档同时写明：预签名 URL 本身可以在没有权限的情况下生成，第三方能否传上去，取决于签名身份有没有 `oss:PutObject`。

[权限与访问控制概述](https://help.aliyun.com/zh/oss/user-guide/permissions-and-access-control-overview) 写明，签名请求先验证签名，再评估 Control Policy、RAM Policy、Bucket Policy 和 ACL。RAM 用户默认没有任何权限，授权后再签。不要用主账号 AccessKey 签名。权限收窄的做法见上一篇账号文章，这里不重复。

### 链接泄露之后

有效期在生成时写死，之后不能改短某一条已经发出的链接。能做的是让签出它的凭证不再被接受。

请求带签名，但 `AccessKeyId` 已禁用时，返回 403，错误码 `0002-00000903`。该页的示例写在 `Authorization` 头里，处理建议同时指向 Header 签名和 URL 签名。见 [0002-00000903](https://help.aliyun.com/en/oss/user-guide/0002-00000903)。签名 FAQ 另写：`AccessKeyId` 不存在或为 Inactive 时，同样是 403，错误码 `InvalidAccessKeyId`。见 [Object URL 签名 FAQ](https://help.aliyun.com/zh/oss/issues-related-to-the-signed-url-of-an-object)。两页都没有提供“只作废一条 URL、同一把密钥签出的其他 URL 继续有效”的开关。停用或删除的是这把 AccessKey。

STS 签出的链接，到期时间不会超过 Token。Token 到期后，这条路径上的链接不能再使用。本文打开的预签名页面没有写一套提前吊销单张 STS Token 的控制台步骤。

### 防盗链挡不住“伪造来源”，但会检查签名 URL

[防盗链](https://help.aliyun.com/zh/oss/user-guide/hotlink-protection) 的控制台路径是：目标 Bucket，「数据安全 > 防盗链」。可以配白名单、黑名单和是否允许空 Referer。白名单加黑名单的总大小不超过 20 KB。规则在整个 Bucket 生效，不能按单个对象分开。匹配不忽略协议，`http://` 和 `https://` 要分别写。

触发范围写得很窄：只有匿名访问和签名 URL 会做防盗链检查。请求头里带 `Authorization` 的 API 调用不检查。STS 签出的 URL 没有这枚请求头，因此仍要过防盗链。空 Referer 被拒绝时，错误码是 `0003-00000501`。见 [0003-00000501](https://help.aliyun.com/zh/oss/user-guide/0003-00000501)。

[权限概述](https://help.aliyun.com/zh/oss/user-guide/permissions-and-access-control-overview) 同时写明 Referer 可以伪造，更高的安全性要用签名 URL。防盗链不能代替私有桶。CDN 如果缓存了对象，边缘命中可能绕过 OSS 上的 Referer 规则，要在 CDN 上再配一层。见防盗链页的生产环境说明。

## 腾讯云 COS

中国站文档在 cloud.tencent.com，国际站同一功能的英文页在 www.tencentcloud.com。这次两边都打开了预签名说明，有效期数字一致。国际站文档是英文。控制台若显示英文按钮，按该站文档里的 Details、temporary link 对照，不要只按中文菜单找。

### 默认私有，没有同名的“阻止公共访问”开关

[访问控制基本概念](https://cloud.tencent.com/document/product/436/30749) 写明，存储桶和对象默认私有，匿名用户不能凭对象 URL 直接访问。[ACL](https://cloud.tencent.com/document/product/436/30752) 里存储桶预设 `private` 是默认：创建者拥有 `FULL_CONTROL`，其他人没有权限。`public-read` 给匿名用户组读权限，`public-read-write` 给匿名用户组完全控制。文档不建议把写、写 ACL 或完全控制授给匿名用户。

控制台路径：对象存储控制台，存储桶列表，进入存储桶，「权限管理 > 存储桶访问权限」，公共权限选私有读写，保存。创建存储桶的弹窗里也可以设。见同一篇 ACL 文档。公有读或公有读写时，该页建议打开告警。

这次在中国站 COS 文档里检索，没有找到与 S3 Block Public Access 或 OSS 阻止公共访问同名、会忽略已有公共策略的总开关。私有 ACL 也不等于策略里没有匿名允许。访问控制基本概念里的存储桶策略示例，可以把 `cos:GetObject` 授给匿名身份，效果类似公有读。确认没有公共访问时，要同时看公共权限和 Policy。

对象 ACL 还有一个范围限制：对象上的访问权限只在通过默认域名访问时有效；走 CDN 加速域名或自定义域名时，以存储桶权限为准。见 [ACL](https://cloud.tencent.com/document/product/436/30752)。

[授权与身份认证](https://cloud.tencent.com/document/product/436/68279) 把公有读写成有风险的操作：链接一旦公开，不经身份认证就能下载，并可能被盗刷流量。

### 生成预签名 URL

[使用预签名 URL 访问 COS](https://cloud.tencent.com/document/product/436/68284) 把签名嵌进对象 URL。拿到链接的人可以下载；上传同理，拿到有效上传链接的人都可以传。

控制台：存储桶的文件列表，打开对象详情，复制临时链接。这一条的有效期是 1 小时。

COSBrowser：使用主账号密钥时最长 2 小时，使用子账号密钥时最长 1.5 天。见同一页。国际站 [Accessing COS Using a Pre-Signed URL](https://www.tencentcloud.com/document/product/436/45243) 的控制台和 COSBrowser 数字相同。

要自己定时长，用 SDK。Python SDK 的 `get_presigned_url` 里，`Expired` 单位是秒，默认 300。见 [生成预签名 URL](https://cloud.tencent.com/document/product/436/35153)。下面只保留时长和占位名称，密钥不要写进命令或仓库：

```python
url = client.get_presigned_url(
    Method='GET',
    Bucket='<bucket-name>',
    Key='EXAMPLE.txt',
    Expired=900
)
```

该页还写明：获取签名时默认把 Host 算进签名。`SignHost` 默认 `True`。设为 `False` 后使用者可以改域名，页面注明这有安全风险。敏感的请求头和参数应算进签名，避免使用者改掉。

2024-01-01 之后创建的桶，不支持用默认域名在浏览器里预览，文档建议改用自定义域名。见同一篇 Python 文档的注意事项。

### 有效期上限

永久密钥和临时密钥不是同一个上限。

永久密钥不过期。中国站预签名说明写：链接有效期取决于你设置的签名有效期，并用“任意时长”描述 SDK 这条路径。国际站对应小标题是 any validity period，正文同样写有效期取决于你设置的签名时长。两页都没有给出永久密钥签名的数字上限，这里不补一个天数。

临时密钥会把链接再截短。实际有效时间是签名时长和临时密钥时长的较小值。[权限管理问题](https://cloud.tencent.com/document/product/436/56637) 写明：主账号临时密钥最长 7200 秒（2 小时），子账号最长 129600 秒（36 小时），默认 1800 秒（30 分钟）。国际站 [COS Authorization and Identity Verification Process](https://www.tencentcloud.com/document/product/436/45228) 的主账号 7200 秒、子账号 129600 秒、默认 1800 秒与此相同。预签名说明里的“临时密钥最长 36 小时”是这段范围的上沿，主账号拿不到 36 小时。

签名过期后请求会中断。重新申请签名后要重做失败的请求，不支持断点续传。见预签名说明里的注意事项。

公有读对象如果仍使用带签名的链接，签名过期后会被拒绝，不会退回成匿名可读。见 [权限管理问题](https://cloud.tencent.com/document/product/436/56637)。

### 用最小权限签名，泄露后停用这把密钥

预签名说明要求：如果仍用永久密钥签名，权限只留上传或下载，时长只用完成本次传输所需的最短时间。临时密钥在申请时用 policy 收窄到目标桶和对象。见 [生成预签名 URL](https://cloud.tencent.com/document/product/436/35153) 和国际站 [Generating a Pre-Signed URL](https://www.tencentcloud.com/document/product/436/54312)。不要为主账号创建访问密钥，见上一篇账号文章。

[防盗刷指引](https://cloud.tencent.com/document/product/436/97973) 写明，访问日志里的 `userSecretKeyId` 能对应到哪一把密钥。确认不是自己的业务在用，就到 CAM 控制台禁用这把密钥。文档没有写“只作废一条 URL”。禁用的是密钥。临时密钥则等到它自己到期；链接不会长过这张临时密钥。

### 防盗链不检查带签名的请求

[设置防盗链](https://cloud.tencent.com/document/product/436/13319) 的路径是：存储桶，「安全管理 > 防盗链设置」。黑名单拒绝名单内的域名或 IP，白名单只允许名单内的来源，其他返回 403。Referer 最多 30 行，支持 `*`。空 Referer 要单独选。通过 CDN 域名访问时，先执行 CDN 的防盗链，再执行 COS 的规则。

同一页写明：访问对象时只要带签名，无论签名在 URL 还是 Header，都不做防盗链验证。因此防盗链挡不住已经泄露的预签名 URL。它用来限制匿名访问的来源。防盗刷指引建议没有公共读需求时保持私有读写；必须限制浏览器来源时，再配防盗链，并把空 Referer 设为拒绝。空 Referer 拒绝后，浏览器地址栏直接打开也会失败。见 [防盗链问题](https://cloud.tencent.com/document/product/436/56651)。

## AWS S3

本节引用 docs.aws.amazon.com 的中文用户指南，也就是全球商业区文档。AWS 中国区是另一套分区，文档在 docs.amazonaws.cn，账号和终端节点不能混用。这次没有打开中国区页面，没有核对 7 天和 12 小时是否在中国区文档里被改写。中国区账号请再打开 docs.amazonaws.cn。

### 屏蔽公共访问权限

新的通用存储桶默认不允许公共访问。创建时，屏蔽公共访问权限的四项都会打开，对象所有权默认是“存储桶拥有者强制”，ACL 被禁用。见 [通用存储桶概述](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/UsingBucket.html) 和 [授予对数据的公共访问权限](https://docs.aws.amazon.com/AmazonS3/latest/userguide/granting-public-access.html)。[AWS Prescriptive Guidance 的 ACCT.08](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-08.html) 把“新建即打开”写成 2023-04-28 及之后创建的存储桶。更早的存储桶要自己打开。

四项设置彼此独立，可以只开其中几项。账户、存储桶、接入点取最严格的组合。组织级策略一旦启用，四项一起生效，不能在组织级只挑一项。见 [阻止公共访问](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)。

| 设置 | 文档中的作用 |
| --- | --- |
| `BlockPublicAcls` | 拒绝新的公共 ACL。已有 ACL 不改写 |
| `IgnorePublicAcls` | 忽略已有公共 ACL |
| `BlockPublicPolicy` | 拒绝新的公共存储桶策略或接入点策略。已有策略不改写 |
| `RestrictPublicBuckets` | 已有公共策略不再对账户外的匿名或跨账户访问生效 |

关掉这些设置后，原来的公共 ACL 或策略会重新生效。该项设置不按单个对象配置。账户级设置作用于该分区内的全部区域，生效可能不是各区域同一时刻。见同一页。

用户指南建议账户级再打开全部四项。账户级不是“每个新账户都已经打开”的同义词。由组织策略管着的账户，不能在账户级改这组设置。

控制台：

- 账户：打开 S3 控制台，导航窗格里选「账户和组织设置」，在此账户的屏蔽公共访问权限设置下编辑。保存时按提示输入 `confirm`。见 [为账户配置屏蔽公共访问权限](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/configuring-block-public-access-account.html)。中文界面写“屏蔽公共访问权限”，英文界面写 Block Public Access。
- 存储桶：进入存储桶，打开「权限」，编辑「屏蔽公共访问权限（存储桶设置）」，保存时同样输入 `confirm`。见 [为存储桶配置屏蔽公共访问权限](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/configuring-block-public-access-bucket.html)。存储桶级关掉，账户级或组织级仍开着时，公共访问还是被挡住。

这次打开的屏蔽公共访问权限页面，定义的是 ACL 和存储桶策略怎样算“公共”。页面没有写“预签名 URL 会被这项设置拒绝”。预签名 URL 使用的是签名凭证，不是把对象授给 `AllUsers`。链接打不开时，先查签名身份还有没有对应操作的权限，以及凭证是否已被撤销。

### 生成预签名 URL

对象默认私有。控制台可以不写代码就生成下载用的预签名 URL：S3 控制台，左侧「通用存储桶」，进入桶，在对象的操作菜单里选「使用预签名 URL 共享」，填有效时间，创建。URL 会复制到剪贴板。控制台这条路径的过期时间是 1 分钟到 12 小时。见 [使用预签名 URL 共享对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)。

CLI 的 `--expires-in` 单位是秒，默认 3600，最大 604800。见 [presign](https://docs.aws.amazon.com/cli/latest/reference/s3/presign.html)。示例用 15 分钟，不把最大值写进日常命令：

```bash
aws s3 presign s3://<bucket-name>/EXAMPLE.txt --expires-in 900
```

2019-03-20 之后推出的区域，共享对象那一页要求在请求里带上区域和 `endpoint-url`。该页的示例是 `af-south-1`。你的桶在哪个区域，就用该区域的终端节点，不要抄示例区域。

上传可以把 HTTP 方法签成 `PUT`。若同一个键已有对象，S3 会用上传内容替换它。见 [使用预签名 URL 下载和上传对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/using-presigned-url.html)。

### 有效期上限

签名版本 4 的 `X-Amz-Expires` 最小 1 秒，最大 604800 秒（7 天）。签名计算用的签名密钥最长 7 天，所以预签名 URL 不能更长。见 [使用查询字符串参数的签名版本 4](https://docs.aws.amazon.com/AmazonS3/latest/developerguide/sigv4-query-string-auth.html)。

这个 7 天，只在使用 IAM 用户的长期访问密钥、并且走签名版本 4 时成立。临时凭证签出的 URL，先到期的是凭证，不是你填写的那个更长秒数。同一篇下载文档写明：

- IAM 角色：角色会话结束，URL 结束。`AssumeRole` 的会话默认 1 小时。
- EC2 实例配置文件：元数据凭证定期轮换，最长大约 6 小时。
- ECS 任务或容器：角色凭证通常每 1 到 6 小时轮换。

控制台最长 12 小时，CLI 和 SDK 最长可以到 7 天。见 [使用预签名 URL 下载和上传对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/using-presigned-url.html)。

为了凑满 7 天而给根用户或权限很宽的 IAM 用户新建长期密钥，和最小权限是相反的。短时间的下载或上传，用角色的临时凭证即可，接受它先于 7 天到期。

S3 在收到 HTTP 请求时检查到期时间。大文件若在到期前已经开始下载，下载过程中跨过到期点仍会继续。连接断开后，到期再重试会失败。见同一页。

### 权限在使用时再评一次，撤销凭证会使链接失效

有凭证的人都能生成预签名 URL。对方要访问成功，签名身份在执行这项操作时必须有权限，例如下载所需的 `s3:GetObject`。见下载文档。

[Prescriptive Guidance 的概述和 FAQ](https://docs.aws.amazon.com/prescriptive-guidance/latest/presigned-url-best-practices/faq.html) 写得更明确：请求被处理时，签名身份当时的身份策略、资源策略、会话策略和服务控制策略仍然生效。权限若已经收回，即使 URL 还没到你填写的到期时间，也会被拒绝。签名过期、临时凭证过期，或签名凭证被撤销，请求都会失败。

用户指南的原话是：用来签名的凭证被撤销、删除或停用时，预签名 URL 到期，即使填写的到期时间更晚。见下载文档。没有“只作废一条 URL”的开关。作废的是这组凭证，以及它在请求当时已经不再拥有的权限。

### Referer 不能当作访问控制

存储桶策略可以用 `aws:Referer` 限制引用来源。示例页写明，不要用它阻止别人直接访问 S3：Referer 可以由改造过的客户端改成任意值。它只适合降低内容被别的网站引用的可能。见 [存储桶策略示例](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/example-bucket-policies.html)。私有对象的外发，用预签名 URL。不要为了配 Referer 而关掉屏蔽公共访问权限。

## Microsoft Azure Blob Storage

本节引用 learn.microsoft.com 的全球 Azure 文档。由世纪互联运营的 Azure 使用另一套文档和终结点。这次没有打开那一套页面，没有核对用户委派 SAS 的 7 天上限是否被改写。

### 账户禁止匿名，容器默认专用

匿名读取要同时满足两层，缺一不可。见 [修正匿名读取访问](https://learn.microsoft.com/zh-cn/azure/storage/blobs/anonymous-read-access-prevent)：

- 存储账户的 `AllowBlobPublicAccess`。Azure 资源管理器存储账户的默认配置是禁止匿名，用户不能再把容器改成匿名；已经改成匿名的容器也不再接受匿名请求。该页覆盖常规用途 v2、高级块 Blob、高级文件共享和 Blob Storage 账户。
- 容器的匿名访问级别。默认是专用（门户里的 Private）：容器和 Blob 都要经过授权。只有账户允许匿名之后，才能把容器改成 Blob（匿名可读 Blob，但不能列出容器）或 Container（容器和 Blob 都可匿名读，并可列出 Blob）。不能给单个 Blob 单独设匿名级别。见 [配置匿名读取访问](https://learn.microsoft.com/en-us/azure/storage/blobs/anonymous-read-access-configure)。

关掉账户匿名的门户路径：存储账户，「设置」下的「配置」，把“允许 Blob 匿名访问”设为已禁用。见修正匿名访问一文。CLI 创建时可以显式加上 `--allow-blob-public-access false`。该页的示例要求 Azure CLI 2.9.0 或更高版本。

账户禁止匿名之后，授权请求仍然可用，包括 SAS。匿名开关挡的不是 SAS。

### 生成 SAS

Microsoft 的建议是优先用户委派 SAS：用 Microsoft Entra 凭据签名，应用程序不必保存存储账户密钥。服务 SAS 和账户 SAS 用账户密钥签名。见 [SAS 概述](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-sas-overview)。

门户这条路径签的是服务 SAS，不是用户委派 SAS。步骤见 [使用门户管理容器](https://learn.microsoft.com/zh-cn/azure/storage/blobs/blob-containers-portal)：进入容器列表，选中容器，更多操作里选生成 SAS，签名方法选账户密钥，再选一把签名密钥。存储访问策略可以选“无”。权限按需要勾选，并填写开始和到期时间。允许的协议默认 HTTPS。生成的令牌和 URL 只显示一次。门户返回的令牌不含问号，拼到资源 URL 上时要自己加上 `?`。

用户委派 SAS 用 Azure CLI。`--auth-mode login` 和 `--as-user` 都要带。到期时间必须落在开始时间起的 7 天内；写得更长，密钥到期后 SAS 也只在这 7 天内有效。见 [使用 Azure CLI 创建用户委派 SAS](https://learn.microsoft.com/zh-cn/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli)。下载只授予读：

```bash
az storage blob generate-sas \
    --account-name <storage-account> \
    --container-name <container> \
    --name EXAMPLE.txt \
    --permissions r \
    --expiry <date-time> \
    --auth-mode login \
    --as-user \
    --full-uri
```

`--full-uri` 会返回带 SAS 的 Blob URI。不带这个参数时，返回的令牌同样不含 `?`。官方示例里的权限字符串可以写成多个字母的组合；只发下载链接时用 `r`。用户委派 SAS 不能绑定存储访问策略。

创建用户委派密钥的身份，需要带有 `Microsoft.Storage/storageAccounts/blobServices/generateUserDelegationKey` 的角色。读取数据再授予存储 Blob 数据读取者这类数据角色。存储 Blob 委派者角色用来获取委派密钥。见 [使用 Microsoft Entra ID 授权 Blob 访问](https://learn.microsoft.com/zh-cn/azure/storage/blobs/authorize-access-azure-active-directory)。账户密钥和 `listkeys` 权限不要发给只负责发下载链接的人。

### 有效期上限不是一种 SAS 一个数字

用户委派密钥最长 7 天。用户委派 SAS 因此最长 7 天，和你是否另外配置了 SAS 过期策略无关。过期策略若大于 7 天，对用户委派 SAS 没有放宽作用。见 [SAS 过期策略](https://learn.microsoft.com/zh-cn/azure/storage/common/sas-expiration-policy) 和 CLI 一文。

用账户密钥签名的服务 SAS、账户 SAS，这次打开的概述没有写成与 7 天相同的硬上限。概述反而提醒：不引用存储访问策略的临时 SAS 应使用较短到期时间；引用存储访问策略时，策略上的到期时间可以设在很远的将来。使用 2012-02-12 之前的 REST 版本、并且不引用存储访问策略时，SAS 最长 1 小时，更长会失败。这是旧客户端的限制，不是当前用户委派 SAS 的限制。见 [SAS 概述](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-sas-overview)。

SAS 过期策略是账户上的建议上限，不是默认存在的 7 天硬顶。默认操作是 Log：超出建议区间的 SAS 仍可使用，并在诊断日志的 `SasExpiryStatus` 里留下记录。改成 Block 后，超出的请求会被拒绝。这项 Block 不覆盖通过 HDFS 终结点使用的用户委派 SAS，也不覆盖带存储访问策略的服务 SAS。见 [SAS 过期策略](https://learn.microsoft.com/zh-cn/azure/storage/common/sas-expiration-policy)。

开始时间如果写成“现在”，前几分钟可能因为时钟偏差失败。概述建议把开始时间设到至少 15 分钟之前，或者不设开始时间。到期时间同样可能看到最多 15 分钟的偏差。

### 泄露后怎样失效

三种 SAS 的失效办法不同。

- 用户委派 SAS：调用撤销用户委派密钥。CLI 是 `az storage account revoke-delegation-keys`，作用于该存储账户上的全部用户委派密钥，不能只点一条 SAS。密钥和角色分配会被缓存，CLI 一文写明从发起撤销到现有 SAS 失效可能有延迟。见 [使用 Azure CLI 创建用户委派 SAS](https://learn.microsoft.com/zh-cn/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli)。[保护 Blob 存储](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-blobs) 则写成撤销用户委派密钥会立即使由其签发的 SAS 失效。两页目前不一致。操作时以你这次调用后、用旧 SAS 实际访问的结果为准，不要假定点击之后的下一秒一定已经失效。也可以改掉或删除签名主体的 RBAC 角色；同一篇 REST 说明写明，角色同样会被缓存，也可能延迟。见 [创建用户委派 SAS](https://learn.microsoft.com/en-us/rest/api/storageservices/create-user-delegation-sas)。
- 带存储访问策略的服务 SAS：修改或删除该策略会立刻作用于所有引用它的 SAS。做法包括删除策略、改掉签名标识符，或把到期时间改到过去。每个容器最多 5 条存储访问策略。用户委派 SAS 和账户 SAS 不能使用这种策略。见 [定义存储访问策略](https://learn.microsoft.com/zh-cn/rest/api/storageservices/define-stored-access-policy)。
- 不带存储访问策略的服务 SAS 或账户 SAS：轮换签出它的那把账户密钥。存储账户有两把密钥，轮换一把只影响用这把签出的 SAS。用户委派 SAS 不使用账户密钥，不受这次轮换影响。轮换后，还在用旧密钥的应用程序要改用另一把或新密钥。见 [管理账户访问密钥](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-account-keys-manage)。

### 这次没有找到容器级 Referer 黑白名单

匿名访问和 SAS 这几页没有 OSS、COS 那种按 Referer 黑白名单拒绝请求的容器设置。已经发出的 SAS，不会因为 Referer 对不上而被这几页描述的功能拦住。要限制的是 SAS 的权限、到期时间和上文的撤销办法。分发时使用 HTTPS。概述写明，SAS 若经 HTTP 被截获，截获者可以按 SAS 上的权限使用它。

## Google Cloud Storage

本节引用 cloud.google.com 和 docs.cloud.google.com。这次没有看到与阿里云、腾讯云对应的“中国站 / 国际站”两套 Signed URL 上限。没有加入组织的项目不能使用组织策略，公共访问防护要设在存储桶上。见 [Public access prevention](https://docs.cloud.google.com/storage/docs/public-access-prevention)。

### 公共访问防护，以及统一存储桶级访问权限

公共访问防护打开后，不能再通过 IAM 或 ACL 把数据授给 `allUsers` 或 `allAuthenticatedUsers`。已有的这类授权还在策略里，但会被覆盖。可以设在单个存储桶上；存储桶属于组织时，也可以用组织策略约束条件 `storage.publicAccessPrevention`，作用在项目、文件夹或组织。见 [Public access prevention](https://docs.cloud.google.com/storage/docs/public-access-prevention)。

控制台：Cloud Storage 的存储桶列表，进入存储桶，打开 Permissions，在 Public access 卡片里选择 Prevent public access，再确认。见 [Use public access prevention](https://docs.cloud.google.com/storage/docs/using-public-access-prevention)。读取是强一致的，强制执行最多可能要 10 分钟才生效。生效前已经被公开缓存的对象，还可能按 `Cache-Control` 留在互联网缓存里。该页举例：`Cache-Control:max-age` 若是默认的 3600 秒，缓存可能再保留这么久。

Signed URL 不受公共访问防护影响。该页写明，Signed URL 仍把有时限、范围很窄的访问交给持有链接的人。私有桶经 Cloud Load Balancing 或 Cloud CDN 的访问也不受这项设置影响。

统一存储桶级访问权限启用后，ACL 失效，只剩 IAM。连续启用满 90 天后不能再关掉。它不是所有新存储桶的全局默认值。组织约束条件 `constraints/storage.uniformBucketLevelAccess` 可以要求新建存储桶启用它；有的组织默认就启用了这条约束，要问组织管理员。见 [Uniform bucket-level access](https://docs.cloud.google.com/storage/docs/uniform-bucket-level-access) 和 [组织策略约束条件](https://docs.cloud.google.com/storage/docs/org-policy-constraints)。

### 生成 Signed URL

Signed URL 把签名放在查询字符串里。持有链接的人在有效期内可以做签名所允许的操作，不需要自己有 Google 账号。只能访问 XML API 终端节点。见 [Signed URLs](https://docs.cloud.google.com/storage/docs/access-control/signed-urls)。

这次打开的签名页面给出的生成方式是 gcloud 和客户端库，没有写在控制台里点一个按钮生成。下载示例见 [V4 signing with Cloud Storage tools](https://docs.cloud.google.com/storage/docs/access-control/signing-urls-with-helpers)：

```bash
gcloud storage sign-url gs://<bucket-name>/EXAMPLE.txt \
    --impersonate-service-account=<signer@example-project.iam.gserviceaccount.com> \
    --duration=10m
```

`gcloud storage sign-url` 的 `--duration` 默认 1 小时。使用系统管理的密钥时最长 12 小时，原因是这把密钥在 12 小时之后可能不再有效。改用 `--private-key-file`，或先用 `gcloud auth activate-service-account` 登录服务账号再签名，最长 7 天。见 [gcloud storage sign-url](https://docs.cloud.google.com/sdk/gcloud/reference/storage/sign-url)。能用模拟服务账号时，不必为了凑 7 天去新建用户管理的服务账号密钥。服务账号密钥的取舍见上一篇账号文章。

上传示例在同一篇工具文档里：`--http-verb=PUT`，并用 `--duration` 和 `--headers` 把这次上传要带的 `content-type` 签进去。使用链接的人必须带上签过名的头，值也必须一致。

### 有效期上限

V4 的 `X-Goog-Expires` 从 `X-Goog-Date` 起算，最长 604800 秒（7 天）。见 [Signed URLs](https://docs.cloud.google.com/storage/docs/access-control/signed-urls) 和 [Canonical requests](https://docs.cloud.google.com/storage/docs/authentication/canonical-requests)。

这是签名格式的上限。gcloud 走系统管理密钥时，命令本身只允许到 12 小时，到不了 7 天。两句话都要保留。

签名从 `X-Goog-Date` 前 15 分钟起就可以使用，直到你指定的到期时间。`X-Goog-Date` 必须和凭证范围里的日期是同一天。见 [Signatures](https://docs.cloud.google.com/storage/docs/authentication/signatures)。

### 签名身份只给这次操作的权限

生成时必须指定一个有能力完成该请求的账号。读对象，这个账号自己要能读该对象。多数情况用服务账号。自己实现签名算法时，也可以使用带 HMAC 密钥的用户账号。见 [Signed URLs](https://docs.cloud.google.com/storage/docs/access-control/signed-urls)。

不要为了签名把 `roles/storage.admin` 或基本角色授给这个服务账号。权限收在它要签的那个存储桶或对象前缀上。基本角色过宽的原因见上一篇账号文章。

### 到期，或轮换用来签名的密钥

Signed URL 页面写明，知道链接的人可以访问，直到到期，或者用来签名的密钥被轮换。这次打开的该页没有写“删掉服务账号的 IAM 绑定之后，已经发出的链接会立刻失效”。公共访问防护页也写明 Signed URL 不受该设置影响。因此，泄露之后按文档能依赖的是：等它到期，或轮换签出它的那把密钥。用户管理的私钥若已经分发出去，轮换包括停用并删除这把密钥，再让新的签名使用新密钥。

## 防盗链和流量：公开地址会把流出记到桶上

临时链接解决的是“谁可以发起这次传输”。传输一旦发生，费用仍记在资源所属的账户上。下面只说明哪些方向会计费。单价、免费额度和资源包规格会变，打开各页当前表格核对，本文不抄单价。云服务器的公网带宽不套用到对象存储。

- 阿里云 OSS：[流量费用](https://help.aliyun.com/zh/oss/traffic-fees) 写明，外网 Endpoint 或传输加速 Endpoint 上的 GetObject、预览和图片处理产生外网流出，这项计费。外网上传产生的流入不计流量费。同地域内网 Endpoint 的流入和流出不计流量费。CDN 回源流出、跨区域复制流量另计。返回码为 4xx、5xx 的请求不计流量费。同一篇下载仍会同时产生请求费，因为请求按 API 调用另计。预签名 URL 使用外网域名下载，走的就是外网流出。
- 腾讯云 COS：[流量费用](https://cloud.tencent.com/document/product/436/53863) 写明，外网下行计费，外网上行不计费，同地域内网上下行不计费。CDN 或 EdgeOne 回源、跨地域复制、全球加速另计。[计费计量问题](https://cloud.tencent.com/document/product/436/30747) 写明，用控制台、工具、API、SDK、浏览器或对象地址下载，只要走的是公网，就产生外网下行，和你用不用预签名无关。请求按发送次数计，成功或失败都计。见 [计费概述](https://cloud.tencent.com/document/product/436/16871)。
- AWS S3：预签名下载仍是 `GetObject`。数据传出和请求如何计费，以 [S3 定价](https://aws.amazon.com/cn/s3/pricing/) 的当前表为准。全球商业区“传入免费、传出按量、每月合并计算的免费出站额度”已写在账单那篇，这里不重复。到期前已经开始的下载，跨过 URL 到期时间仍会传完，这段字节仍是一次传出。
- Azure：SAS 概述写明，通过 SAS 的每一次使用都记在账户上。只读 SAS 可以被反复下载，带写权限的 SAS 可以被用来上传。传入免费、传出另计的带宽规则见 [带宽定价](https://azure.microsoft.com/zh-cn/pricing/details/bandwidth/)，本文不抄表。
- Google Cloud Storage：公共访问防护页写明，强制执行后，原先匿名、不进数据访问日志的读取，可能变成有身份的访问并增加日志量，日志可能产生 Cloud Audit Logs 费用。对象经公网下载的网络费以 [Cloud Storage 价格](https://cloud.google.com/storage/pricing?hl=zh-cn) 的当前网络部分为准。

公共读加上一个固定 URL，第三方网页可以长期引用，流出由桶的拥有者承担。私有桶加上短时链接，把“谁能发起”收窄到持有链接的人，但持有者在有效期内仍能反复下载。OSS 的 Referer 会检查签名 URL，COS 的防盗链不检查带签名的请求，S3 的 `aws:Referer` 可以被伪造，Azure 和 Cloud Storage 这次打开的页面没有同等的桶级 Referer 名单。防盗链不能代替私有和短有效期。

## 发出链接之前

每个桶做一遍。菜单以你登录的站点为准。

1. 匿名访问是关的。OSS 看「权限控制 > 阻止公共访问」和 ACL 是否为 private，并记住 2025-10-13 之后 API 创建才逐步默认打开，旧桶以页面现状为准。COS 看公共权限是私有读写，同时看存储桶策略没有把 `GetObject` 授给匿名身份；这次没有找到同名的阻止公共访问总开关。S3 看存储桶和账户的四项屏蔽公共访问权限，2023-04-28 之前的桶不要假设已经打开。Azure 看账户的“允许 Blob 匿名访问”为已禁用，容器为专用。Cloud Storage 看 Public access 为 Prevent public access，并记住强制执行可能要到 10 分钟，缓存还可能按 `Cache-Control` 再留一段时间。
2. 链接有效期只覆盖这一次传输。OSS 控制台最长 32400 秒，V4 最长 604800 秒，STS 不超过 Token 且文档上限 43200 秒。COS 控制台这条临时链接是 1 小时；永久密钥没有在文档里写成一个数字上限，临时密钥取两者较小值，主账号临时密钥最长 7200 秒。S3 控制台 1 分钟到 12 小时，CLI 默认 3600 秒、最大 604800 秒，临时凭证会更短。Azure 用户委派 SAS 最长 7 天；账户密钥 SAS 不要照搬这 7 天。Cloud Storage 的 V4 最长 604800 秒，gcloud 使用系统管理密钥时最长 12 小时。
3. 签名身份不是主账号、根用户或整个账号的管理员。OSS 下载只需要 `oss:GetObject`，上传只需要 `oss:PutObject`。S3 下载需要 `s3:GetObject`。Azure 发用户委派 SAS 的人有委派密钥权限和对应的数据权限，SAS 上的权限是 `r` 而不是读写删。Cloud Storage 的服务账号只能做链接所允许的那一项。
4. 知道链接泄露后的失效粒度。OSS 停用的是 AccessKey，不是单条 URL。COS 在 CAM 禁用对应密钥。S3 撤销、删除或停用签名凭证，权限变更在请求时生效。Azure 用户委派密钥用撤销命令，但是否立即生效，两篇文档不一致；服务 SAS 优先绑存储访问策略，以便删除策略即失效；账户密钥要轮换签出 SAS 的那一把。Cloud Storage 按文档依赖到期或轮换签名密钥，不要指望只删 IAM 绑定或只开公共访问防护。
5. 防盗链的预期和产品一致。OSS 的签名 URL 会检查 Referer，空 Referer 可能被拒绝。COS 带签名的请求不检查防盗链。S3 不要用 Referer 代替预签名 URL。CDN 要单独配规则，否则边缘缓存可能绕过源站。
6. 接受流量仍会计费。外网下载、重复打开同一条链接、PUT 覆盖，都会产生该产品文档里写明的请求和流出。单价回到上一节的官方页面，不要用云服务器的带宽价代替。
7. 国际站、AWS 中国区、世纪互联运营的 Azure，再打开该站点的文档看按钮名称和终端节点。本文已经核对并且两边一致的，只有阿里云预签名有效期表，以及腾讯云预签名的控制台 1 小时、COSBrowser 时长和临时密钥上限。其余站点的差异不要靠推测补齐。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。资料核对日期：2026-11-10。

阿里云中国站：

- [Bucket ACL](https://help.aliyun.com/zh/oss/user-guide/oss-bucket-acl)
- [阻止公共访问](https://help.aliyun.com/zh/oss/user-guide/block-public-access)
- [PutBucketPublicAccessBlock](https://help.aliyun.com/zh/oss/developer-reference/putbucketpublicaccessblock)
- [put-bucket](https://help.aliyun.com/zh/oss/developer-reference/put-bucket)
- [使用预签名 URL 下载或预览文件](https://help.aliyun.com/zh/oss/user-guide/how-to-obtain-the-url-of-a-single-object-or-the-urls-of-multiple-objects)
- [使用预签名 URL 上传文件](https://help.aliyun.com/zh/oss/user-guide/upload-files-using-presigned-urls)
- [presign（生成预签名 URL）](https://help.aliyun.com/zh/oss/developer-reference/presign-generate-presigned-url)
- [ossutil 1.0 和 2.0 的功能差异](https://help.aliyun.com/zh/oss/developer-reference/1-0-and-2-0-functional-differences)
- [权限与访问控制概述](https://help.aliyun.com/zh/oss/user-guide/permissions-and-access-control-overview)
- [0003-00000005](https://help.aliyun.com/zh/oss/user-guide/0003-00000005)
- [0002-00000903](https://help.aliyun.com/en/oss/user-guide/0002-00000903)
- [Object URL 签名 FAQ](https://help.aliyun.com/zh/oss/issues-related-to-the-signed-url-of-an-object)
- [防盗链](https://help.aliyun.com/zh/oss/user-guide/hotlink-protection)
- [0003-00000501](https://help.aliyun.com/zh/oss/user-guide/0003-00000501)
- [流量费用](https://help.aliyun.com/zh/oss/traffic-fees)

阿里云国际站：

- [Using presigned URLs to download or preview files](https://www.alibabacloud.com/help/en/oss/user-guide/how-to-obtain-the-url-of-a-single-object-or-the-urls-of-multiple-objects)
- [V4 signatures in presigned URLs](https://www.alibabacloud.com/help/en/oss/developer-reference/add-signatures-to-urls)

腾讯云中国站：

- [访问控制基本概念](https://cloud.tencent.com/document/product/436/30749)
- [ACL](https://cloud.tencent.com/document/product/436/30752)
- [COS 授权与身份认证流程](https://cloud.tencent.com/document/product/436/68279)
- [使用预签名 URL 访问 COS](https://cloud.tencent.com/document/product/436/68284)
- [生成预签名 URL](https://cloud.tencent.com/document/product/436/35153)
- [权限管理问题](https://cloud.tencent.com/document/product/436/56637)
- [防盗刷指引](https://cloud.tencent.com/document/product/436/97973)
- [设置防盗链](https://cloud.tencent.com/document/product/436/13319)
- [防盗链问题](https://cloud.tencent.com/document/product/436/56651)
- [流量费用](https://cloud.tencent.com/document/product/436/53863)
- [计费计量问题](https://cloud.tencent.com/document/product/436/30747)
- [计费概述](https://cloud.tencent.com/document/product/436/16871)

腾讯云国际站：

- [Accessing COS Using a Pre-Signed URL](https://www.tencentcloud.com/document/product/436/45243)
- [COS Authorization and Identity Verification Process](https://www.tencentcloud.com/document/product/436/45228)
- [Generating a Pre-Signed URL](https://www.tencentcloud.com/document/product/436/54312)

AWS：

- [通用存储桶概述](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/UsingBucket.html)
- [授予对 Amazon S3 数据的公共访问权限](https://docs.aws.amazon.com/AmazonS3/latest/userguide/granting-public-access.html)
- [ACCT.08 防止对私有存储桶的公共访问](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-08.html)
- [阻止公共访问](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [为账户配置屏蔽公共访问权限](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/configuring-block-public-access-account.html)
- [为存储桶配置屏蔽公共访问权限](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/configuring-block-public-access-bucket.html)
- [使用预签名 URL 共享对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [使用预签名 URL 下载和上传对象](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/using-presigned-url.html)
- [presign](https://docs.aws.amazon.com/cli/latest/reference/s3/presign.html)
- [使用查询字符串参数的签名版本 4](https://docs.aws.amazon.com/AmazonS3/latest/developerguide/sigv4-query-string-auth.html)
- [预签名 URL 的常见问题](https://docs.aws.amazon.com/prescriptive-guidance/latest/presigned-url-best-practices/faq.html)
- [存储桶策略示例](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/example-bucket-policies.html)
- [S3 定价](https://aws.amazon.com/cn/s3/pricing/)

Microsoft Azure：

- [修正 Blob 数据的匿名读取访问](https://learn.microsoft.com/zh-cn/azure/storage/blobs/anonymous-read-access-prevent)
- [配置容器和 Blob 的匿名读取访问](https://learn.microsoft.com/zh-cn/azure/storage/blobs/anonymous-read-access-configure)
- [使用门户管理 Blob 容器](https://learn.microsoft.com/zh-cn/azure/storage/blobs/blob-containers-portal)
- [使用 SAS 授予有限访问权限](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-sas-overview)
- [使用 Azure CLI 创建用户委派 SAS](https://learn.microsoft.com/zh-cn/azure/storage/blobs/storage-blob-user-delegation-sas-create-cli)
- [使用 Microsoft Entra ID 授权 Blob 访问](https://learn.microsoft.com/zh-cn/azure/storage/blobs/authorize-access-azure-active-directory)
- [SAS 过期策略](https://learn.microsoft.com/zh-cn/azure/storage/common/sas-expiration-policy)
- [创建用户委派 SAS](https://learn.microsoft.com/en-us/rest/api/storageservices/create-user-delegation-sas)
- [定义存储访问策略](https://learn.microsoft.com/zh-cn/rest/api/storageservices/define-stored-access-policy)
- [管理账户访问密钥](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-account-keys-manage)
- [保护 Azure Blob 存储](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-blobs)
- [带宽定价](https://azure.microsoft.com/zh-cn/pricing/details/bandwidth/)

Google Cloud：

- [Public access prevention](https://docs.cloud.google.com/storage/docs/public-access-prevention)
- [Use public access prevention](https://docs.cloud.google.com/storage/docs/using-public-access-prevention)
- [Uniform bucket-level access](https://docs.cloud.google.com/storage/docs/uniform-bucket-level-access)
- [组织策略约束条件](https://docs.cloud.google.com/storage/docs/org-policy-constraints)
- [Signed URLs](https://docs.cloud.google.com/storage/docs/access-control/signed-urls)
- [V4 signing with Cloud Storage tools](https://docs.cloud.google.com/storage/docs/access-control/signing-urls-with-helpers)
- [gcloud storage sign-url](https://docs.cloud.google.com/sdk/gcloud/reference/storage/sign-url)
- [Canonical requests](https://docs.cloud.google.com/storage/docs/authentication/canonical-requests)
- [Signatures](https://docs.cloud.google.com/storage/docs/authentication/signatures)
- [Cloud Storage 价格](https://cloud.google.com/storage/pricing?hl=zh-cn)

如需为这些云平台的国际站账号开户或充值，可访问海宝云官网 <https://www.haibaocloud.com>。充值币种和最低金额以官网当前说明为准。Telegram 客服：@cloudcup。
