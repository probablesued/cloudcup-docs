# 云账号登录和权限怎么收紧：多因素认证与最小权限

刚开好云账号就用主账号或根用户做日常操作，密码或长期访问密钥再配上过宽的权限，拿到这份凭证的人就可以创建仍会计费的资源。多因素认证（MFA，也叫两步验证）用来挡住“只有密码就能登录控制台”。最小权限用来缩小同一份凭证能操作的资源范围。两件事要一起做：只开 MFA、仍把管理员权限交给一把长期密钥，密钥泄露后 MFA 帮不上忙。

费用怎么累加，见 [云服务器账单为什么超预算](cloud-server-bill-over-budget.md)。本文不重复计算、磁盘、公网和快照的计费表。身份和权限规则会调整。下文的控制台路径、默认值和强制日期，都来自撰写时打开的官方页面，并附有链接。页面改版后，以你账号所属站点的当前文档为准。

核对时间：2026-11-09。阿里云、腾讯云同时存在中国站和国际站，控制台文案、验证方式和部分默认策略不同。两边分开写。国际站账号不要按中国站的点击顺序操作。

## 开户后先做这四件事

1. 主账号、根用户、租户的初始全局管理员只留作开户、账单和文档写明必须由它完成的操作。日常开关服务器，改用单独的子用户或同等身份。
2. 给这个高权限身份开启 MFA，也给每个要登录控制台的人开启 MFA。每人使用自己的身份，不要共用密码。
3. 新建身份先不要授予管理员策略。按产品、资源和操作收窄。职责相同的人放进同一用户组或组，再给组授权。
4. 不要给主账号或根用户创建长期访问密钥。程序访问优先用该云文档推荐的临时凭证、角色、托管标识或附加到资源上的服务身份。必须留下长期密钥时，单独一个身份、最小权限，并且不要写进代码仓库。

## 阿里云

访问控制产品的官方名称是 RAM。中国站规则来自 help.aliyun.com。国际站主账号 MFA 的路径不同，见本节最后一小节。

### 中国站：主账号只在必要时登录

[身份与权限的最佳实践](https://help.aliyun.com/zh/ram/product-overview/best-practices-for-identity-and-access-control) 把主账号比作 Linux 的 root：拥有所有权限，不适合日常使用。文档要求先建一个有管理员权限的 RAM 用户做日常管理和技术操作，再为组织里的每个人单独建 RAM 用户；主账号只在必要时使用，并保管密码和 MFA。

主账号 AccessKey 的风险单独写在 [不使用主账号 AccessKey](https://help.aliyun.com/zh/ram/avoid-using-the-accesskey-pair-of-an-alibaba-cloud-account)：它等同于主账号的完全管理权限，而且不能加访问来源 IP、访问时间这类条件。文档的治理建议是停用主账号 AccessKey，改为权限收窄后的 RAM 用户 AccessKey。从未使用的，直接禁用，禁用 90 天且没有相关问题后再删除。

同一篇最佳实践写明：永久 AccessKey 在手动删除之前一直有效；通过角色扮演拿到的 STS Token 在超过角色最大会话时间（小时级）后自动失效。部署在阿里云上的程序，文档建议用 ECS 实例角色、容器实例角色或函数计算角色扮演，避免把 AccessKey 放进环境。本地调试等必须使用永久 AccessKey 的场景，一个应用的一个环境单独一把，不要共用，不要写进代码，并为这把密钥所在的 RAM 用户授予该场景的最小权限。

[管理 RAM 用户的权限](https://help.aliyun.com/zh/ram/user-guide/grant-permissions-to-the-ram-user) 写明，目前不能给单个 RAM 用户设置独立的消费额度或账单，RAM 用户的消费计入主账号，由主账号统一支付。因此一个权限过大的 RAM 用户，花费记在主账号上。RAM 子用户收不到账户资金类消息，额度预警和扣费提醒默认只发给主账号联系人。

### 中国站：主账号 MFA

[配置账号的 MFA](https://help.aliyun.com/zh/account/configure-account-mfa) 和 [绑定或解绑虚拟 MFA](https://help.aliyun.com/zh/account/manage-vitual-mfa) 把虚拟 MFA 定义为遵循 TOTP（RFC 6238）的应用，每 30 秒生成一个 6 位验证码。推荐应用包括阿里云 App、Google Authenticator 和 Microsoft Authenticator。启用后，登录要先输入账号名和密码，再输入第二重验证。文档写明：即使密码泄露，没有这台设备的人无法登录。

中国站绑定前，这两页要求账号已完成实名认证，并已安装最新版阿里云 App。路径是：登录账号中心，在「我的账号」下选择「账号安全」，单击虚拟 MFA 后面的「绑定」。身份验证方式包括手机号、支付宝和扫脸。然后用阿里云 App 或 Google Authenticator 取得验证码，输入后确认绑定。刷新后状态应为「已绑定」。

[安全指南中的主账号绑定步骤](https://help.aliyun.com/zh/acsg/bind-the-mfa-device-to-the-primary-account) 是另一条入口：控制台右上角头像、安全设置、账号安全、虚拟 MFA。该页写明，主账号绑定 MFA 只对主账号生效，不影响 RAM 用户。更换手机或卸载验证器之前，先在控制台解绑，否则可能无法登录。解绑的现行步骤以 [绑定或解绑虚拟 MFA](https://help.aliyun.com/zh/account/manage-vitual-mfa) 为准。

### 中国站：RAM 用户的 MFA

主账号的 MFA 不会自动保护 RAM 用户。[多因素认证（MFA）](https://help.aliyun.com/zh/ram/user-guide/what-is-multi-factor-authentication) 写明，RAM 用户登录要先输入登录名和密码，再完成第二重验证，例如虚拟 MFA 每 30 秒生成的 6 位码。该页同时写明：密码泄露后，没有 MFA 设备的人无法用这个 RAM 用户登录控制台；已绑定 MFA 时，删除 ECS 实例、创建 AccessKey、修改权限这类高风险操作会要求再次验证。MFA 和授权是两回事。给 RAM 用户授予 `AliyunBSSFullAccess` 等策略，不取决于 MFA 是否已经绑定。

该页列出的方式是：通行密钥（安全级别写为最高）、虚拟 MFA（适合作为首选）、安全手机、安全邮箱。安全邮箱只能用于操作认证，不能用于登录认证。文档建议每个 RAM 用户至少绑定两种不同类型的设备，例如通行密钥加安全手机。原 U2F 已升级为通行密钥。一个安全手机或安全邮箱最多绑定 5 个 RAM 用户。

[管理员为 RAM 用户绑定 MFA 设备](https://help.aliyun.com/zh/ram/user-guide/admin-binds-an-mfa-device-to-a-ram-user) 写明，自 2024 年 8 月 20 日起陆续为所有 RAM 用户开启登录时的强制 MFA。管理员不能代绑通行密钥，通行密钥要 RAM 用户自己绑定。绑定虚拟 MFA 的控制台路径是：主账号或拥有 `AliyunRAMFullAccess` 的 RAM 用户登录 RAM 控制台，身份管理 > 用户，打开目标用户，在认证管理页签的 MFA 信息里复制「绑定 VMFA」链接，用阿里云 App 或 Google Authenticator 扫码或手动输入密钥，再把动态码填回控制台。安全手机、安全邮箱绑上之后还要用户点开激活链接，链接 24 小时内有效。基本信息里的手机和邮箱只是备注，不能当作这组安全手机或安全邮箱。

全局开关在 [管理 RAM 用户安全设置](https://help.aliyun.com/zh/ram/user-guide/manage-security-settings-of-ram-users)：RAM 控制台左侧「设置」，多因素认证区域。对全部 RAM 用户生效。「登录时必须使用 MFA」的选项是「强制所有用户」「依赖每个用户的独立配置」「仅异常登录时使用」，文档写明初始默认是「强制所有用户」。设为强制所有用户后，敏感操作的二次验证也会对所有 RAM 用户启用。安全邮件只参与敏感操作的二次验证。「记住 MFA 验证状态 7 天」初始默认禁用。

[安全指南里的 RAM 用户绑定步骤](https://help.aliyun.com/zh/acsg/bind-an-mfa-device-to-a-ram-user) 仍写着虚拟 MFA 与 U2F 二选一。现行方式以上面的 RAM 产品文档为准。

同一安全设置页还管理 AccessKey 的闲置时间。AccessKey 最大闲置时间的取值包括 90 天、180 天、365 天、730 天，初始默认 730 天；文档建议设为 90 天。被自动禁用的密钥可以重新启用，文档更建议新建一把做轮转。允许用户自己管理 AccessKey 的开关初始默认禁用；生产环境文档建议保持禁用，由管理员分配和轮转。

### 中国站：只授予必要权限

新创建的 RAM 用户没有任何权限，授权后才能访问资源。见 [管理 RAM 用户的权限](https://help.aliyun.com/zh/ram/user-guide/grant-permissions-to-the-ram-user)。

控制台入口有两处：身份管理 > 用户，在操作列点「新增授权」；或权限管理 > 授权，点「新增授权」。资源范围可以是整个账号，也可以是一个资源组。授予资源组权限后，用户登录控制台还要把顶部资源范围切到那个资源组，才能看到里面的资源。系统会标出高风险系统策略，页面举例包括 `AdministratorAccess` 和 `AliyunRAMFullAccess`。概览页快速开始里，「超级用户」会自动授予 `PowerUserAccess`，「账号管理员」会自动授予 `AdministratorAccess`。文档写明，只在确实需要超级管理员权限时使用这两项。

该页的权限安全建议：

- 管某一台 ECS，不要授予 `AdministratorAccess` 或 `AliyunECSFullAccess`。自定义策略把 `Resource` 写成具体实例 ID。
- `AliyunRAMFullAccess` 只给确实要管 RAM 的人，并审计这些人的操作。
- 短期任务用 RAM 角色和 STS 临时凭证。临时凭证到期失效。
- 职责相同的用户放进同一用户组再授权。用户组上的策略不占用用户个人的策略配额。每个 RAM 用户默认可绑定的系统策略和自定义策略各有上限，文档写的默认值是各 20 条。

[身份与权限的最佳实践](https://help.aliyun.com/zh/ram/product-overview/best-practices-for-identity-and-access-control) 把同一原则写成：非管理员场景不要直接用权限过大的系统策略；自定义策略写明资源、操作和条件；按岗位建用户组，把策略授给用户组。一个 RAM 用户启用一把 AccessKey，另一把只用于轮转。

### 国际站主账号 MFA 不要套用中国站步骤

国际站中文文档 [步骤 3：配置账号的 MFA](https://www.alibabacloud.com/help/zh/account/configure-account-mfa)（页面更新时间 Aug 21, 2026）以 Google Authenticator 为例，没有把阿里云 App 列为推荐应用。路径也不同：登录账号中心，进入安全设置，在「其他设置」里点「账号保护」的设置；在开启账号保护页选择保护场景和验证方式。TOTP 这条路径先用邮箱或手机完成身份验证，邮箱验证码为 6 位、15 分钟内有效，再用 Google Authenticator 扫描二维码。短信验证是另一条路径。账号还没绑定手机时，文档写明先用邮箱完成身份验证，再绑定手机。

国际站这篇没有把中国站的实名认证、支付宝或扫脸写成前置步骤。RAM 用户在国际站的菜单名称和强制 MFA 的生效日期，请打开 www.alibabacloud.com 上的 RAM 文档核对。不要把上文中国站「2024 年 8 月 20 日」和「身份管理 > 用户」直接套到国际站控制台。

## 腾讯云

访问管理产品的官方名称是 CAM。中国站文档在 cloud.tencent.com，国际站文档在 www.tencentcloud.com。子用户的安全设置要由主账号或有管理权限的账号操作。

### 中国站：主账号、子用户和最小权限

[安全实践教程](https://cloud.tencent.com/document/product/598/10592) 要求不要使用主账号身份凭证访问腾讯云，也不要把凭证交给别人。为每个访问腾讯云的人创建子账号并授予相应权限。按职责建用户组，把权限授给组，再把用户放进组。同一子账号不要同时管用户、权限和资源。不要给同一个 CAM 用户既设控制台登录密码、又建 API 访问密钥：只调 API 的只建密钥，只用控制台的只设密码。密码、MFA 和访问密钥都不要共享。

最小权限在同一页的定义是：只授予完成任务所需的权限。例子是，只用 CDN 的用户不需要 COS 读写。

访问密钥和登录密码在该页被写成具有同样的权限。主账号对名下资源有完全控制，因此不要为主账号创建访问密钥，也不要拿它做日常工作。身份安全一节进一步写明：不要为根用户创建访问密钥对，因为根用户对账户内全部服务和资源拥有完全访问权限，包括账单信息。需要调用 API 时，给 CAM 用户创建访问密钥。

CAM 角色没有关联的持久证书。用户担任角色时，系统创建临时证书，再用临时密钥调用 API。主账号只在申请角色时使用持久证书。

该页建议为主账号和子账号都绑定 MFA，并开启登录保护和敏感操作保护。支持邮箱登录或微信登录的账号，文档写明强烈推荐 MFA 二次验证。登录密码和云 API 密钥要定期轮换。不再需要的证书和权限删掉。自定义策略可以用条件限制来源 IP、时间等。操作记录用操作审计查看。

### 中国站：主账号 MFA 和账号保护

[绑定虚拟 MFA 设备](https://cloud.tencent.com/document/product/378/55649) 的路径是：登录账号中心的安全设置，在账号保护里找到虚拟 MFA，单击绑定，按页面完成身份验证。该页举例用微信里的腾讯云助手小程序扫码，并写明也可以使用 Google Authenticator、Microsoft Authenticator 等常见虚拟 MFA。绑定时要设定启用范围：登录保护、操作保护，或两者。更换设备前先解绑。

[账号保护](https://cloud.tencent.com/document/product/378/8392) 说明登录保护和操作保护可以同时开启。登录保护在密码正确之后还要再验证一次。操作保护在修改登录方式、密码、安全手机等敏感操作前验证身份。该页列出的类型包括通行密钥、U2F、微信扫码、MFA 设备和手机验证码，也可以选择不开启。U2F 该页写明目前仅支持存量白名单用户。通行密钥校验该页写明仅支持 PC 端。微信扫码要先绑定微信；MFA 设备校验要先绑定 MFA 设备。

敏感操作校验的有效期是 60 分钟。登录时已经做过二次校验的，登录后 60 分钟内的敏感操作不再触发这次校验。可以同时开启多种方式，优先校验排序靠前的设备，其余作为备选。虚拟 MFA 和登录用的是同一个微信时，用该微信扫码登录会跳过登录保护。用已开启登录保护的通行密钥登录，也会跳过登录保护。可选信任设备，7 天内免登录校验。

关闭登录保护后，日常登录只剩账号和密码。该页同时写明，账号 30 天未登录、登录环境异常、注销账号等部分高危场景，即使用户关闭了账号保护，系统仍会要求身份验证。

### 中国站：子用户 MFA

[为子账号设置安全保护](https://cloud.tencent.com/document/product/598/36383) 的单用户路径是：访问管理控制台，用户 > 用户列表，点进用户，打开「安全」，在 MFA 认证设置后点「管理」，勾选要开启的保护类型。选择虚拟 MFA 设备校验后，子用户下次登录要按页面绑定设备。

给以后新建的子用户统一打开登录保护：用户 > 用户设置，展开安全设置，在身份安全设置里按需开启。

[子账号绑定 MFA 设备](https://cloud.tencent.com/document/product/598/134897) 写了两种弹出绑定页的情况。一种是创建子账号时已经开启 MFA，并设定了登录保护或操作保护，子账号首次登录就会看到绑定页。另一种是创建时没开，后来补开，下一次登录再弹出。该页同样以腾讯云助手小程序为例，并写明支持多个设备扫码绑定同一个 MFA 账号。

### 国际站不要套用微信小程序步骤

国际站 [Binding a Virtual MFA Device](https://www.tencentcloud.com/document/product/378/32528) 的入口同样是安全设置里的 Account Protection、Virtual MFA、Bind，绑定时设置 login protection 和 operation protection。这篇以 Google Authenticator 或 Microsoft Authenticator 扫码为例，输入应用上的 6 位动态码，没有把腾讯云助手小程序写成操作步骤。

国际站子用户的安全保护见 [Setting Security Protection for Sub-Users](https://www.tencentcloud.com/document/product/598/32657)：CAM 控制台 Users > User List，点进用户，Security，在 Identity Security 旁点 Manage，选择保护类型。启用虚拟 MFA 后，子用户下次登录再绑定设备。按钮名称和中国站的「MFA 认证设置」不是同一套文案。该页最近更新时间是 2024-01-23，控制台若已改版，以登录后看到的按钮为准。

上文的最小权限、主账号访问密钥和角色临时证书，引用的是中国站安全实践教程。国际站账号请再打开 www.tencentcloud.com 上的 CAM 文档核对，不要把中国站的微信扫码步骤用于国际站。

## AWS

身份产品的官方名称是 IAM。下文引用 docs.aws.amazon.com 的中文页；根用户 MFA 宽限天数与英文页不一致，两页都列出。

[根用户最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/root-user-best-practices.html) 写明，创建账户时的根用户对账户内全部 AWS 资源有完全访问权限。没有必须使用根用户凭证的任务时，不要登录根用户。日常任务创建管理用户来做。单个账户优先用 IAM 角色：角色没有密码或访问密钥这类长期凭证，代入后得到本次会话的临时安全凭证。IAM 用户则带有密码和访问密钥等长期凭证。多个账户由 AWS Organizations 管理时，文档建议用 IAM Identity Center 的员工用户集中管人和权限。

根用户密码、MFA、访问密钥、CloudFront 密钥对和签名证书不要共享，除非对方有严格的业务理由必须使用根用户。密码长度 8 到 128 个字符，至少包含大写、小写、数字和该页列出的符号这四类中的三类，并且不能与账户名或电子邮件地址相同。

所有账户类型（独立账户、管理账户、成员账户）都要为根用户配置 MFA。最多可以注册 8 台当前支持的 MFA 设备，类型可以组合。文档建议注册多台，避免单台设备丢失后无法登录。中文页写明：尚未启用 MFA 时，用户必须在首次尝试登录管理控制台后 35 天内注册。同一主题的英文 [Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html) 写的是 14 天，管理账户一节也写明要在首次登录尝试期间或 14 天宽限期内完成。两页目前不一致。操作前两页都打开，并以这次登录时控制台的实际提示为准。

[根用户的多重身份验证](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/enable-mfa-for-root.html) 建议尽可能使用通行密钥或安全密钥，因为它们更能抵御网络钓鱼。也支持虚拟身份验证器应用和硬件 TOTP 令牌。虚拟 MFA 被写成等待硬件到位时的做法。为根用户启用 MFA 只影响根用户凭证，不会自动给 IAM 用户打开 MFA。丢失 MFA 时，文档要求账户注册的电话和电子邮件保持可用，以便按恢复流程登录。

根用户虚拟 MFA 的控制台路径见 [为根用户启用虚拟 MFA 设备](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/enable-virt-mfa-for-root.html)：用根用户登录，导航栏右侧点账户名称，打开安全凭证，在多重身份验证部分选择分配 MFA 设备，类型选身份验证器应用程序，扫描二维码或输入私有密钥，连续填入两个间隔约 30 秒的一次性密码。通行密钥和安全密钥的启用步骤，从 [根用户的多重身份验证](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/enable-mfa-for-root.html) 进入对应的控制台说明。

IAM 用户的虚拟 MFA 见 [在控制台中分配虚拟 MFA 设备](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html)：打开 IAM 控制台，用户，点进该用户，安全凭证选项卡，分配 MFA 设备。配置虚拟 MFA 需要拿到那台手机或设备。文档建议把通行密钥或安全密钥放在虚拟 MFA 之前，并建议让用户在获得相应权限后自己管理自己的 MFA 设备。每个根用户和每个 IAM 用户同样最多 8 台。

不要为根用户创建访问密钥。根用户访问密钥覆盖账户内全部服务和资源，包括账单信息。需要编程访问时，文档建议使用带根凭证的 `aws login`，取得临时、自动轮换的凭证，而不是长期访问密钥。见 [根用户最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/root-user-best-practices.html)。

[IAM 安全最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/best-practices.html) 要求人员通过身份提供商联合身份，使用临时凭证；工作负载使用 IAM 角色的临时凭证。在 EC2、Lambda 上运行的应用，由计算资源取得角色的临时凭证，不要把 IAM 用户的长期密钥放进这些工作负载。AWS 之外的工作负载，文档列出的方式包括 IAM Roles Anywhere、`AssumeRoleWithSAML`、`AssumeRoleWithWebIdentity`，以及部分服务自己的集成。账户里如果仍需要 IAM 用户或根用户，再要求 MFA。能用防网络钓鱼的通行密钥和安全密钥时，优先用它们。

最低权限在同一页的写法是：只授予完成任务所需的权限，也就是在特定条件下对特定资源允许特定操作。探索阶段可以从较宽的权限开始，场景稳定后把权限收到最低。AWS 托管策略面向所有客户，不一定等于你这个场景的最低权限，文档建议再写成自己的客户管理型策略。IAM Access Analyzer 可以根据 CloudTrail 里的访问活动生成更细的策略。用上次访问信息删掉不再使用的用户、角色、权限、策略和凭证。

长期访问密钥在手动吊销之前一直有效，见 [Secure access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html)。临时安全凭证由 AWS STS 签发，有效期可以配置为几分钟到几小时，到期后不能再用于 API，见 [IAM 临时安全凭证](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_credentials_temp.html)。[IAM 安全最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/best-practices.html) 仍列出少数需要 IAM 用户长期密钥的场景，例如无法使用 IAM 角色的工作负载、不支持 IAM Identity Center 的第三方客户端，以及部分 CodeCommit、Amazon Keyspaces 的服务专用凭证。这些是例外，不是新账户的默认做法。

## Microsoft Azure

人的登录身份在 Microsoft Entra ID。Azure 资源的授权系统是 Azure RBAC。创建 Microsoft Entra 租户的用户会被自动分配全局管理员角色，见 [安全默认值](https://learn.microsoft.com/zh-cn/entra/fundamentals/security-defaults)。这个角色不要拿来做日常开关虚拟机。[标识管理和访问控制最佳做法](https://learn.microsoft.com/zh-cn/azure/security/fundamentals/identity-management-best-practices) 要求关键管理员使用单独的管理员账户，不要把这些账户用于日常邮件和网页浏览；消费者账户（例如 hotmail.com、live.com、outlook.com）要从关键管理员角色里移除。至少准备两个紧急访问账户，不分配给特定个人，只在普通管理员账户不可用时使用。

### 给人开启 MFA

[强制 MFA 规划](https://learn.microsoft.com/zh-cn/entra/identity/authentication/concept-mandatory-multifactor-authentication) 写明，MFA 可以阻止超过 99.2% 的账户入侵攻击。公共 Azure 云分两阶段要求用户账户完成 MFA，不能选择退出。美国政府云和其他主权云不在这篇所述的强制范围内。

- 阶段 1：自 2024 年 10 月起，登录 Azure 门户、Microsoft Entra 管理中心和 Microsoft Intune 管理中心并执行创建、读取、更新或删除，需要 MFA。自 2025 年 2 月起逐步覆盖 Microsoft 365 管理中心。这一阶段不覆盖 Azure CLI、Azure PowerShell、Azure 移动应用和 IaC 工具。
- 阶段 2：自 2025 年 10 月 1 日起，通过 Azure CLI、Azure PowerShell、Azure 移动应用、IaC 工具、控制平面 REST API 和 Azure SDK 做创建、更新或删除，逐步要求 MFA。读取不要求 MFA。发往 `https://management.azure.com` 的资源管理请求在范围内。Microsoft Graph 通常不在范围内。

工作负载标识，包括托管标识和服务主体，两个阶段都不受这条 MFA 强制的影响。用用户身份跑脚本则受影响。文档要求把这类用户身份迁到工作负载标识。测试租户没有例外。已经要求 MFA 的用户不会因为这条强制而改变登录方式；只对一部分人要求 MFA 的租户，其余用户在访问所列应用时也要使用 MFA。

有 Microsoft Entra ID P1 或 P2 时，用条件访问要求 MFA。没有条件访问时，启用安全默认值。安全默认值的路径见 [为 Microsoft Entra ID 配置安全默认值](https://learn.microsoft.com/zh-cn/entra/fundamentals/security-defaults)：至少以条件访问管理员登录 Microsoft Entra 管理中心，Entra ID > 概述 > 属性 > 管理安全默认值，设为启用并保存。2019 年 10 月 22 日及之后创建的租户可能已经启用；新租户在强制之前有 24 小时宽限。启用后，所有用户要注册 MFA，管理员每次登录都要 MFA，访问 Azure 门户、Microsoft Entra 管理中心、Azure PowerShell 和 Azure CLI 的用户也要 MFA。旧式身份验证不支持 MFA，安全默认值会阻止旧协议。安全默认值不能和自定义条件访问混用。需要自定义策略时，先关掉安全默认值，再立刻启用条件访问，避免中间出现空窗。

该页同时写明，99.9% 以上与标识相关的常见攻击可以通过 MFA 加上阻止旧式身份验证来阻止。这个比例和上面的 99.2% 不是同一句话，引用时不要合成一个数字。

[标识管理和访问控制最佳做法](https://learn.microsoft.com/zh-cn/azure/security/fundamentals/identity-management-best-practices) 把启用方式分成安全默认值、按用户状态启用、条件访问，以及需要 Entra ID P2 的基于风险的条件访问。按用户启用会替代条件访问，不能和后两种同时用。防钓鱼方法（FIDO2 安全密钥、通行密钥、Windows Hello 企业版、基于证书的身份验证）被写成优先项。用户注册安全信息的入口是 [https://myprofile.microsoft.com](https://myprofile.microsoft.com) 的安全信息。

强制日期可以推迟，但规划页写明推迟会增加风险，并建议立即配置 MFA。阶段 2 的推迟上限写到 2026 年 7 月 1 日，须由全局管理员操作。本文不把推迟当作开户后的默认步骤。

### Azure RBAC 只授予必要权限

[Azure RBAC 的最佳做法](https://learn.microsoft.com/zh-cn/azure/role-based-access-control/best-practices) 要求只授予完成工作的最低特权，避免在更宽的范围上分配更大的角色。自定义角色只包含需要的操作，不要用通配符 `*` 表示 `Actions` 和 `DataActions`。主体被入侵时，受影响的资源范围由此缩小。

订阅所有者建议最多保留 3 个，以降低所有者账户被入侵后的影响。必须分配特权管理员角色时，范围优先用资源组或单个资源，而不是管理组或整个订阅。有权创建角色分配的角色，考虑加上条件，限制它能转授的角色。特权账户可以用 Microsoft Entra Privileged Identity Management 做成限时激活，到期自动收回，而不是一直保持有效。

角色分配给组，不要逐个分给用户。脚本和自动化使用角色 ID，不使用会改名的角色显示名。

标识最佳实践页补充：范围可以是订阅、资源组或单个资源；职责分开，用内置角色授予某项工作所需的操作。安全团队如果只做评估，授予安全读取者；需要处理 Defender for Cloud 的策略和警报时，再授予安全管理员。全局管理员可以临时提升为用户访问管理员，以便查看全部订阅和管理组，评估结束后删除这次提升。不能控制资源创建位置的组织，更容易出现超出需要的资源；该页把 Azure Policy 写成和 RBAC 不同的一层，RBAC 决定谁可以创建，策略决定拒绝哪些操作或位置。

## Google Cloud

人用 Google 账号或 Cloud Identity 账号登录。资源授权用 IAM。服务账号是另一类身份，不走用户登录那套两步验证。

### 用户账号的两步验证

[Google Cloud 的多重身份验证要求](https://docs.cloud.google.com/docs/authentication/mfa-requirement?hl=zh-cn) 写明，MFA 也称两步验证：除密码外还要另一种身份证明。密码被盗后，没有第二重验证的人仍然进不了账号。已经启用 MFA 的 Google 账号不用再做额外操作。检查位置是 Google 账号设置的安全性标签页，「您的 Google 账号登录选项」里的两步验证。使用通行密钥的账号仍要启用 MFA 并添加身份验证因素。用第三方 IdP 做单点登录时，可以用该 IdP 的 MFA 满足这项要求。

强制开始时间按账号类型区分：

| 账号类型 | 文档写明的开始时间 |
| --- | --- |
| 个人 Google 账号，包括用作 Google Cloud 主账号的 Gmail | 自 2025 年 5 月 12 日起 |
| 不使用 SSO 的企业 Cloud Identity 账号 | 2025 年第 4 季度期间或之后 |
| 使用 Google Workspace SSO、Cloud Identity SSO 或员工身份联合的企业账号 | 2026 年第 1 季度期间或之后 |
| 转销商网域中的用户账号；转销商的最终用户不受影响 | 2025 年 4 月 28 日当天或之后 |

未启用时，控制台会在强制前至少 90 天显示提醒，并至少提前 90 天发邮件。转销商及其用户的对应提前量是 60 天。到期后，未启用 MFA 就不能登录 Google Cloud 控制台或 Firebase 控制台。这项强制不影响服务账号，也不影响 Google Workspace 邮件和云端硬盘、YouTube，以及云上应用的数据平面。用 Google 账号模拟服务账号时，如果这个用户账号已被要求 MFA，则必须启用 MFA 才能登录控制台。用于 Google Cloud 的 Gmail 账号不能选择不接受这项要求。企业账号和转销商账号的例外要联系 Cloud Customer Care，本文不提供绕过步骤。

开启步骤见 [开启两步验证](https://support.google.com/accounts/answer/185839?hl=zh-Hans)：打开 Google 账号，点「安全性与登录」，在登录选项下选择开启两步验证，按屏幕完成。工作、学校或其他组织发放的账号，如果看不到这个选项，向管理员申请。文档建议在不用通行密钥时，把 Google 提示作为第二步，因为它不依赖短信。短信或语音验证码可用，但该页写明这种方式可能被利用电话号码的攻击盗用。通行密钥和硬件安全密钥用于加强防钓鱼。备用验证码只使用一次，并要另行收好。只在自己常用、不与别人共用的设备上勾选「不再询问」。

Cloud Identity 或 Google Workspace 里的超级管理员账号，还要看组织侧的强制策略。[安全的企业基础控制](https://docs.cloud.google.com/docs/security/security-best-practices-catalog/secure-enterprise-foundations?hl=zh-cn) 把「为超级用户账号启用两步验证」和「对超级用户组织部门强制执行两步验证」列为必需控制。前一条建议超级管理员使用 Titan 安全密钥，不能用安全密钥时改用其他安全密钥。后一条建议为超级用户单独建组织部门，并对该部门强制两步验证。

### IAM 最小权限

[安全地使用 IAM](https://docs.cloud.google.com/iam/docs/using-iam-securely?hl=zh-cn) 写明，基本角色包含针对所有 Google Cloud 服务的数千项权限。生产环境在还有其他角色可用时不要授予基本角色，改为最窄的预定义角色或自定义角色。测试或开发环境可以是例外。子资源继承父资源的允许政策：项目级允许某用户管理 Compute Engine 虚拟机时，该用户可以管理项目里的任何虚拟机，单台虚拟机上的允许政策挡不住这条继承。

角色授在所需的最小范围上。只需要向一个 Pub/Sub 主题发布消息时，就把 Publisher 授在这个主题上，而不是授在整个项目上。职责相同的人放进群组，把角色授给群组。条件角色绑定可以让访问自动过期。

Owner（`roles/owner`）可以访问和修改几乎所有资源，包括修改允许政策。只在确实需要这种通用访问时授予。Project IAM Admin 和 Folder IAM Admin 可以改允许政策，但没有对全部资源的直接读写和管理。Service Account User 可以访问该服务账号所能访问的全部资源，授予前先确认这个服务账号自己的权限。每个需要不同权限的组件使用单独的服务账号，只授予该组件需要的权限。

### 服务账号密钥

[管理服务账号密钥的最佳实践](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys?hl=zh-cn) 写明，有用户账号的两步验证和登录验证时，只拿到密码往往登不进去；服务账号密钥不经过这些额外登录验证，拿到密钥就可能直接以该服务账号调用 API。密钥还可能造成权限提升，并且审计上难以分辨是谁使用了这把密钥。

因此把用户管理的服务账号密钥当作例外。能改用其他身份验证方式时就不要新建密钥。在全网域授权这个场景，该页写明不必使用服务账号密钥：先用附加的服务账号、Workload Identity Federation for GKE 或工作负载身份联合完成服务账号身份验证，再调用 `signJwt`。组织在 2024 年 5 月 3 日或之后创建的，默认会强制执行「停用服务账号密钥创建」和「停用服务账号密钥上传」。Editor（`roles/editor`）包含创建或上传服务账号密钥的权限，密钥可以用来取得服务账号已有的访问范围；需要 Editor 时，用上述组织政策限制条件把创建和上传关掉。

密钥文件不要提交到源代码库，不要留在浏览器的下载目录，不要在用户之间传来传去，不要嵌进程序。不再使用的密钥先停用，确认没有调用后再删除。[安全地使用 IAM](https://docs.cloud.google.com/iam/docs/using-iam-securely?hl=zh-cn) 同时写明：从外部拿到的服务账号密钥要先验证再使用。

英文 [Best practices for using service accounts securely](https://docs.cloud.google.com/iam/docs/best-practices-service-accounts) 给了一条选择顺序：代码跑在 Google Cloud 上时，GKE 使用 Workload Identity Federation for GKE，其他计算资源把服务账号附加到资源；单人开发环境优先使用用户凭证，确需服务账号时用用户凭证模拟服务账号；外部身份提供方支持工作负载身份联合时配置联合；没有可行替代方案时才创建服务账号密钥。持有密钥或拥有 `iam.serviceAccountKeys.create` 的人，可以以该服务账号的身份访问它所能访问的资源。这个权限包含在 Service Account Key Admin 和 Editor 中。

## 开户后的核对清单

每个账号做一遍。菜单名称以你登录的站点为准。

1. 日常登录身份已经不是主账号、根用户或租户的初始全局管理员。这个高权限身份只留给开户、账单和文档写明必须由它完成的操作。
2. 高权限身份已绑定 MFA，并且你还能用另一台设备或另一种方式登录。AWS 中文文档写明最多 8 台。阿里云 RAM 文档建议每个 RAM 用户至少两种不同类型的设备。
3. 每个要进控制台的人有自己的身份，并已开启登录 MFA。阿里云中国站到 RAM 控制台的「设置」确认「登录时必须使用 MFA」。腾讯云中国站到账号保护，以及子用户的安全页，确认登录保护已开。AWS 在根用户和 IAM 用户的安全凭证页查看 MFA。Azure 确认安全默认值或条件访问已经要求 MFA，并用 [强制 MFA 规划](https://learn.microsoft.com/zh-cn/entra/identity/authentication/concept-mandatory-multifactor-authentication) 对照自己的租户处于阶段 1 还是阶段 2。Google 在 Google 账号的安全性标签页确认两步验证已开启。
4. 日常身份没有管理员、所有者或基本角色，除非这项工作在当前文档里必须用它。权限范围落在具体产品、资源组、资源或主题上。阿里云新 RAM 用户默认没有任何权限，授权后再登录试一次该做的操作。
5. 主账号和根用户下没有长期访问密钥。阿里云在账号头像里的 AccessKey 页核对。AWS 在根用户安全凭证页核对。腾讯云不要为主账号创建访问密钥。
6. 程序访问已经改用临时凭证、角色、托管标识或附加服务账号。还不能改的，密钥属于单独身份，权限已收窄，不在代码仓库、下载目录和聊天记录里。Google Cloud 同时看组织政策是否停用了服务账号密钥的创建和上传。
7. 阿里云国际站、腾讯云国际站再打开该站点的文档核对一遍按钮名称。不要把中国站的实名认证步骤、支付宝、扫脸或微信小程序步骤用到国际站。

## 参考来源

规则可能变更。下面是本文核对过的官方页面。

阿里云中国站：

- [身份与权限的最佳实践](https://help.aliyun.com/zh/ram/product-overview/best-practices-for-identity-and-access-control)
- [配置账号的 MFA](https://help.aliyun.com/zh/account/configure-account-mfa)
- [绑定或解绑虚拟 MFA](https://help.aliyun.com/zh/account/manage-vitual-mfa)
- [为阿里云主账号绑定 MFA 设备](https://help.aliyun.com/zh/acsg/bind-the-mfa-device-to-the-primary-account)
- [多因素认证（MFA）](https://help.aliyun.com/zh/ram/user-guide/what-is-multi-factor-authentication)
- [管理员为 RAM 用户绑定 MFA 设备](https://help.aliyun.com/zh/ram/user-guide/admin-binds-an-mfa-device-to-a-ram-user)
- [管理 RAM 用户安全设置](https://help.aliyun.com/zh/ram/user-guide/manage-security-settings-of-ram-users)
- [为 RAM 用户开启多因素认证](https://help.aliyun.com/zh/acsg/bind-an-mfa-device-to-a-ram-user)
- [管理 RAM 用户的权限](https://help.aliyun.com/zh/ram/user-guide/grant-permissions-to-the-ram-user)
- [不使用主账号 AccessKey](https://help.aliyun.com/zh/ram/avoid-using-the-accesskey-pair-of-an-alibaba-cloud-account)

阿里云国际站：

- [步骤 3：配置账号的 MFA](https://www.alibabacloud.com/help/zh/account/configure-account-mfa)

腾讯云中国站：

- [安全实践教程](https://cloud.tencent.com/document/product/598/10592)
- [绑定虚拟 MFA 设备](https://cloud.tencent.com/document/product/378/55649)
- [账号保护](https://cloud.tencent.com/document/product/378/8392)
- [为子账号设置安全保护](https://cloud.tencent.com/document/product/598/36383)
- [子账号绑定 MFA 设备](https://cloud.tencent.com/document/product/598/134897)

腾讯云国际站：

- [Binding a Virtual MFA Device](https://www.tencentcloud.com/document/product/378/32528)
- [Setting Security Protection for Sub-Users](https://www.tencentcloud.com/document/product/598/32657)

AWS：

- [根用户最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/root-user-best-practices.html)
- [Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html)
- [根用户的多重身份验证](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/enable-mfa-for-root.html)
- [为根用户启用虚拟 MFA 设备](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/enable-virt-mfa-for-root.html)
- [在控制台中分配虚拟 MFA 设备](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html)
- [IAM 安全最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/best-practices.html)
- [Secure access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html)
- [IAM 临时安全凭证](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_credentials_temp.html)

Microsoft Azure / Microsoft Entra：

- [Azure RBAC 的最佳做法](https://learn.microsoft.com/zh-cn/azure/role-based-access-control/best-practices)
- [Azure 标识管理和访问控制最佳做法](https://learn.microsoft.com/zh-cn/azure/security/fundamentals/identity-management-best-practices)
- [强制性 Microsoft Entra 多重身份验证规划](https://learn.microsoft.com/zh-cn/entra/identity/authentication/concept-mandatory-multifactor-authentication)
- [为 Microsoft Entra ID 配置安全默认值](https://learn.microsoft.com/zh-cn/entra/fundamentals/security-defaults)

Google Cloud：

- [Google Cloud 的多重身份验证要求](https://docs.cloud.google.com/docs/authentication/mfa-requirement?hl=zh-cn)
- [开启两步验证](https://support.google.com/accounts/answer/185839?hl=zh-Hans)
- [安全的企业基础控制](https://docs.cloud.google.com/docs/security/security-best-practices-catalog/secure-enterprise-foundations?hl=zh-cn)
- [安全地使用 IAM](https://docs.cloud.google.com/iam/docs/using-iam-securely?hl=zh-cn)
- [管理服务账号密钥的最佳实践](https://docs.cloud.google.com/iam/docs/best-practices-for-managing-service-account-keys?hl=zh-cn)
- [Best practices for using service accounts securely](https://docs.cloud.google.com/iam/docs/best-practices-service-accounts)

如需为这些云平台的国际站账号开户或充值，可访问海宝云官网 <https://www.haibaocloud.com>。充值币种和最低金额以官网当前说明为准。
