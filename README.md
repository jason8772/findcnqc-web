# FindCNQC v1.7.0

FindCNQC 面向跨境购物用户，提供商品发现、同一商品链接的历史仓库 QC、新手购物指南和 CNBUY 购买入口。公开目录包含有原始商品链接与展示图的商品；**历史仓库 QC 另外校验，缺少 QC 的商品提供 WhatsApp 咨询入口**。卖家展示图不计为 QC。来源校验不等于质量检查、真伪认证、库存确认或履约保证。

正式站：[www.findcnqc.com](https://www.findcnqc.com)。社区：[FindCNQC Discord](https://discord.gg/fV7uQSbGQ)。首阶段以 WhatsApp **+852 6931 8767** 咨询为主要转化入口，不收款、不创建本站账号、不承诺尚未落实的优惠券。

## 运行与发布检查

Node.js 24 或更高，无第三方运行时依赖。

```sh
npm start
npm test
npm run build
```

默认端口 4173，Render 使用 `PORT`。`/api/health` 返回 `version: "1.7.0"` 和当前公开商品数量；`/api/config` 返回公开设置、平台目录、参考汇率和公开目录总数。数量由实际导入内容和发布规则决定，不在文档或页面中写死。

`npm test` 使用固定离线商品 fixture 验证后端接口，不依赖当前目录数量，也不访问外部 QC 服务。数据更新后仍应检查实际发布商品、图片和链接；代码测试不会证明第三方照片此刻可访问或当前库存有效。

## 保留库与公开库

`data/products.json` 是保留库，包含带来源的历史采集记录，不能把它的条数当成公开上架量。服务启动时由 `lib/catalog.mjs` 的 `publishCatalog()` 派生 `publishedProducts`，不修改原始记录。

`data/settings.json` 的 `catalogPolicy: "catalog-with-qc"` 发布完整目录；`"verified-qc-only"` 保留严格 QC 目录模式。两种模式均拒绝隐藏、排除或已报告售罄记录。

完整目录条件包括：

- `published` 不是 `false`，且 `stockStatus` 不是 `source-reports-sold-out`。
- 市场类型、原商品 ID 与 `originalUrl` 一致；目前支持规范的 Weidian、Taobao、1688、Tmall 商品链接。
- 至少一张 HTTPS 商品展示图。

历史 QC 另外要求：至少一张 `qcPhotos` 有当前支持的明确来源 `repsapi.pl`，`sourceUrl` 精确对应 `https://repsapi.pl/items/{marketplace}/{sourceItemId}`。严格 QC 模式还要求这一项。
- QC 图片使用提供方公开的 HTTPS `/v1/photos/{token}.webp` 或 `.jpg` 路径。卖家图片、其他商品的 QC、来源不明的照片不能充数。

同一商品的重复照片会去重。`qcAlbums` 和 `qcImages` 从可信照片重新清理；已有相册分组被保留，不因为同一天拍摄就合并不同订单。原商品 ID 用于跨来源去重。未知上架日期不能描述成“刚上新”，数据观察日期也不等于首次上架日期。

`/api/catalog`、`/api/featured`、`/api/products/:id`、`/product/:id` 和 sitemap 全部使用同一个公开集合。隐藏商品不能通过直接详情地址绕过；携带隐藏商品 ID 的 WhatsApp／Agent 跳转也返回 404。旧收藏引用到已下架商品时，详情可能不可用。

发布校验检查的是来源和身份的一致性，不能独立证明照片授权、真实性、当前可用性或卖家承诺。导入前仍需人工核查来源、公开使用条件及商品匹配关系。照片文件来自第三方服务器，网站不保证永久可用。

## 精选与接口体积

默认排序为 `curated`：有效的数字 `curatedRank` 按升序优先，其余商品按类别轮选，避免同类低价商品占满首屏。兼容 `newest`、`price-asc`、`price-desc`。`featured` 仅选择有商品图及可信 QC 的商品，每类最多一件、总数最多八件；类别不足时不凑数。

列表及首页精选使用 `catalogCard()`：最多返回三张 QC 预览；有效的 `qcCoverUrl` 必须属于该商品可信 QC 图库，优先作为预览第一张。`qcPhotoCount` 与 `qcAlbumCount` 保留完整数量，卡片不返回庞大的 `qcImages`／`qcAlbums`。详情接口继续返回完整照片及相册，前端不能把列表预览长度当成实际总张数。

卖家展示图 `images` 与历史仓库图 `qcPhotos` 分开标注。历史 QC 属于别人此前购买同一链接商品产生的照片；用户自己的订单仍需在所选代采平台到仓后查看 QC。

有 QC 的卡片以该商品审核过的仓库封面作为主图，详情默认打开 QC 页签；卖家展示图仍在独立页签中。卡片使用 640px QC 图，失败时尝试同一商品的有限备用 QC 图／尺寸；详情支持缩略图切换后的失败恢复。可见图片请求设有超时与明确的失败提示，避免网络故障时永久留白。此处理不保证第三方图片源在所有网络中可达。

2026-11-16 的 RepsAPI 官方 Playground 试用为 12 件原本没有 QC 的已收录商品补充了 261 张照片、66 个相册。原有商品、QC、价格与源头链接保留。该轮是人工审核后导入的样本，不是完整第三方数据库导出；生产服务尚未启用 RepsAPI 密钥或每日自动补全任务。

价格是来源快照。原币价与参考换算价分开；`data/fx.json` 中原有币种使用 ECB 经 Frankfurter 发布的参考汇率；RUB 使用俄罗斯央行 2026-11-21 的快照（1 CNY = 12.2759 RUB），单独记录日期与来源，国际运费、境内运费、服务费及税费另行确认。目录中的平台不代表已建立合作、获得返佣资格或保证平台安全与可下单性。

## 搜索与链接解析

`/api/catalog` 搜索整个公开目录，支持标题、品牌、分类、原始商品 ID、原商品链接、支持的代采平台商品链接；Unicode、重音、常见品牌别名和部分多语言分类词会归一化。组合词匹配全部概念，价格、分类、QC 筛选和分页可组合使用。

`/api/resolve-link?url=...` 只解析允许的原站与平台链接，不请求用户提供的任意 URL。匹配到本站商品时返回该商品；尚未收录时明确提示无结果并提供带原链接的 WhatsApp 咨询。它不是对所有中国电商或第三方 QC 数据库的实时搜索，也不保证任意链接都有 QC。

## CNBUY、俄语与 Telegram

v1.7 只展示 [CNBUY](https://cnbuy.pro/)；其他代采平台出站路由返回 404。CNBUY 登录／注册页为 `https://cnbuy.pro/#/account`。通用源商品到 CNBUY 内部商品 ID 的映射尚未验证，所以详情引导先复制淘宝／微店／1688 原链接，再打开 CNBUY 粘贴查询，不能虚构商品深链。原商品与店铺链接、历史 QC 归属仍保留。

俄语包为 `public/locales-ru.js`，支持俄语浏览器初始识别与手动选择，已有语言／币种偏好优先。RUB 使用独立参考汇率与日期，显示和价格筛选采用一致换算；它是参考展示，不承诺以卢布结算。常见俄语分类关键词可搜索。

Telegram 私聊：[cnbuyjason](https://t.me/cnbuyjason)；频道：[CNBUY Official](https://t.me/cnbuy_pro)。分别配置 `telegramContact`、`telegramChannel`，站内跳转为 `/go/telegram` 与 `/go/telegram-channel`。按钮覆盖首页、帮助区、手机菜单、页尾和商品详情；配置缺失时隐藏。目的链接仅使用已校验配置，访客不能覆盖；邀请码和商品上下文只留在本站统计，不附加至最终 Telegram 地址。

`telegram_click` 与 `telegram_channel_click` 由服务器分别记录，客户端不能伪造同名出站事件。后台显示两类点击、UV、点击率，并按日期、国家、来源和商品分析；点击不代表已发送消息或订阅频道。

## Discord 社区与机器人

当前邀请地址为 [FindCNQC 永久邀请](https://discord.gg/fV7uQSbGQ)，配置在 `data/settings.json` 的顶层 `discordInvite`，通过 `/api/config` 公开。站内入口统一走 `/go/discord?placement=nav` 等路径。 导航、首屏社区条、目录横幅、手机菜单、页尾和浮动按钮共用同一配置；12 种语言同步显示社区说明，删除访客邀请码时同步清理这些链接上的旧来源码。

跳转仅接受配置中的 HTTPS `discord.gg/邀请码` 或 `discord.com/invite/邀请码`。请求参数不能提供或覆盖目的地址；配置缺失或格式不合法时返回 404。点击记录为 `discord_click`，不等于用户已经入群。

社区已安装 **FindQC 官方机器人**。它是第三方服务，提供的功能、查询结果、出站链接、额度及规则由 FindQC 控制；不是本站自建的 QC API，也不代表 FindCNQC 与 FindQC 建立数据合作。**安装成功不保证任意商品查询都有照片**，机器人没有结果也不能用卖家图冒充 QC。

本站公开商品的历史 QC 与群内用户提交的任意商品查询是两种范围：前者须通过本站发布条件，后者可能没有记录或暂时查询失败。不要在群里提交住址、付款信息、订单标签或其他个人资料。站内邀请码、本站平台返佣配置与第三方机器人自身的推广码设置彼此独立。

官方说明：[FindQC Bot](https://findqc.com/robots)、[FindQC 条款](https://findqc.com/terms-of-service)。频道结构、原创欢迎／FAQ／规则与后续自有机器人方案保存在本地 `research/community-v2-plan.md`；该研究文档记录的是方案，实际安装状态以本节和服务器配置为准。

## 导流、邀请码与事件

`/go/whatsapp?product=ID&ref=CODE&placement=product` 生成包含商品名称、本站详情和原始链接的 WhatsApp 预填消息。用户决定是否发送；点击不代表添加联系人、咨询完成或成交。

`/go/agent/AGENT_ID?product=ID` 使用已经观察并记录的商品深链格式。无对应市场映射时使用平台官网；`action=register` 打开已核实的注册页面或官网。只有已发布商品可作为商品上下文。默认不附加其他创作者的邀请码。

访客来源码 `?ref=creator_name` 只接受 1–32 位字母、数字、下划线或连字符，保存在当前浏览器。平台返佣码另外配置在 `data/settings.json` 的 `affiliateCodes`，须逐个平台核实参数与实际归因规则后再启用。仅添加链接参数不产生返佣资格或折扣。

记录 `page_view`、`product_view`、`search`、`referral_saved`、`qc_view`、`whatsapp_click`、`agent_click`、`register_click`、`discord_click`。应用事件字段为时间、事件类型、公开商品 ID、平台 ID、来源码和入口位置；不保存完整 IP、电话号码、聊天内容或自由文本搜索词。托管平台自己的请求日志按其服务规则处理。

统计保存在 `DATA_DIR/analytics.sqlite`，生产环境挂载 `/var/data` 的 1GB Render 持久磁盘。使用 Node 24 内置 SQLite、WAL 与正常同步；启动和每小时生成一致性备份，保留 7 个每日文件。备份仍在同一磁盘，不是异地灾备。原始统计保留 90 天；生产不再追加无限增长的 `events.ndjson`。

后台：[stats.findcnqc.com](https://stats.findcnqc.com)。账号通过 `ADMIN_USERNAME`、`ADMIN_PASSWORD_HASH`（`scrypt$saltHex$hashHex`）、`ADMIN_SESSION_SECRET`、`ADMIN_ORIGIN` 配置，密码不进入源码或网页。后台仅在配置的独立主机名开放；登录限频、Secure HttpOnly SameSite 会话、退出 CSRF 检查，会话 8 小时到期且重启后重新登录。公开主站上的后台接口返回 404。

UV 按匿名第一方浏览器 Cookie 去重（90 天）；会话超过 30 分钟无活动重新计次。PV 包含商品详情虚拟页面。活跃停留仅累计可见页面且最近 60 秒内有交互的时间，每 15 秒上报；实时为最近 5 分钟活动，不能当作精确在线连接数。咨询和 Discord 统计的是点击，不代表发送消息、入群或成交。今日与每日按北京时间，周视图为含今天的近 7 天。数据从部署后开始记录，不回填历史；不记录 IP、完整来源 URL 或自由文本搜索，尊重 DNT/GPC 并排除已知机器人。

验收访问应从 `/?qa=1` 进入，设置 1 小时 QA 标记，统计默认排除；后台可勾选包括测试流量验证。`/api/ready` 返回目录、统计存储和后台配置状态。存储出错时公开目录仍可访问，后台明确提示统计不可用。

收藏、语言、币种与来源码保存在当前浏览器，不是服务端账号，不跨设备同步。优惠活动默认关闭；只有确定出资、适用范围与兑现规则后才启用 `promotion`。

## 国家与具体流量来源

v1.6 增加国家／地区、渠道及来源明细。国家从连接 IP 使用本地 DB-IP Country Lite 范围表估算，支持 IPv4/IPv6；仅保存国家代码，不保存访客 IP、不把 IP 发往定位服务。VPN、代理、移动运营商出口与数据库误差会影响结果。未知和本次功能上线前的记录显示为 `ZZ`／未知，不能补造历史国家。

数据来源为 [DB-IP Country Lite](https://db-ip.com/db/download/ip-to-country-lite)，[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)；看板与隐私说明保留 [IP Geolocation by DB-IP](https://db-ip.com) 署名链接。随应用打包的国家库为月度快照，升级数据库需要重新编译与部署，不会自动联网查询或更新。缺失或损坏时返回未知，公开网站继续运行。

生产仅信任 Render 托管代理追加的 `X-Forwarded-For` 最右值；本地／直接部署忽略转发头。这里的国家用于汇总分析，不用于身份确认或权限判定。Render 说明：[读取真实客户端 IP](https://render.com/articles/how-render-handles-ddos-attacks)。

来源明细展示 UTM source、medium、campaign、content、term、来源域名和邀请码。会话采用首次可识别来源，站内浏览与出站点击沿用；UTM 优先于邀请码、再到来源域名。来源网站只保留域名，不保留完整 URL。`utm_term` 是投放者自行设置的链接标记，不代表读取到搜索引擎关键词。无来源、被隐私设置隐藏或从未标记聊天／App 链接进入时显示直接访问／来源不明。已知社交／搜索引荐未标记付费，不等于可以确定为自然流量。旧事件保留已有 source、medium、campaign，但不猜已经丢失的细分字段。

后台的投放链接生成器仅在浏览器中构造本站 URL，不新增保存接口。建议不同渠道、活动和内容使用不同标记，例如 `?utm_source=tiktok&utm_medium=social&utm_campaign=first_test&utm_content=video_a`。复制该链接到对应渠道后，后续访问和咨询点击才能按同一套标签比较。

## 新数据导入

**日常追加或更新经审核 QC：**

```sh
node scripts/merge-reviewed-qc.mjs path/to/approved-qc.json --incremental --batch=2026-11-17
```

增量前保留旧快照，合并后用 `node scripts/audit-incremental-catalog.mjs --before old-products.json --after data/products.json` 检查旧公开商品、商品 ID 和历史 QC 照片均未减少，并确认排除项未重新公开。

输入为商品数组或 `{ "items": [...] }`。保留库中的已有商品通过 `marketplace:sourceItemId` 合并，沿用原本站商品 ID；新的商品加入保留库。脚本保留已有来源和店铺信息，并应用 `data/curation.json` 中的人工设置。输入须先通过 `publishCatalog()`，无可信 QC 或标记售罄的条目不会被当成审核通过的新品。

必需的内容是：商品 ID、标题、市场、原商品 ID／链接、来源页、商品图片，以及包含正确 `source`、`sourceUrl`、`url` 的真实 `qcPhotos`。价格有值时提供明确的人民币价；其他原币数据需要先完成正确转换。相册使用 `qcAlbums[].sourcePage` 和实际照片 URL。仅提供 `qcImages`、写 `qcStatus: "available"`，或自报照片数量，都不能通过发布条件。

审核文件应保留原始来源标题、可核查日期、规格／相册关系、售罄状态；`displayTitle` 可以简洁，但不得编造材质、性能或认证。封面必须从同一商品已有 QC 里选择。用户上传的照片需取得相应展示许可，不因公开发在 Discord 就自动获得站外转载许可。

`merge-reviewed-qc.mjs` 接受 `--batch=YYYY-MM-DD`，默认 UTC 当天；正式批次显式填写审核日期。日常追加须加 `--incremental`，保留以前审核商品、历史照片和相册。已明确排除或售罄的旧记录不会被追加输入重新公开。不加该参数为整批替换，会撤下相同批次中未出现在输入清单里的条目。脚本不自动下载照片、不自动取得数据授权，也不抓取竞争站点。

**完整重建旧来源保留库：**

```sh
node scripts/import-catalog.mjs path/to/catalog-a.json path/to/catalog-b.json
```

这是重建命令，会重新写入 `data/products.json`，并可能根据研究目录更新平台数据与 logo；**不是增量合并**。必须传入要保留的全部来源，或在重建后重新导入完整审核 QC 清单。无参数运行会使用脚本里的历史默认研究文件列表，不能假设其中包含最新审核资料。脚本仍使用历史快照日期，正式更新前应检查。

`research/data-sources.md`、`research/competitors.json` 等记录来源、已取得范围和访问／复用限制。公开访问不等于完整复制授权；不绕过登录、验证码、付费墙或访问限制。导入结果只在服务重启／重新部署后成为新的启动公开集合。

## 可选 QC 与店铺更新

`REPSAPI_KEY` 仅放在服务端环境变量。配置后，已发布商品的详情请求可通过 [RepsAPI QC API](https://repsapi.pl/docs/qc) 查询更新；会校验商品身份、限制频率、缓存结果，并在失败时保留已有可信 QC。未配置时使用本地已经取得的历史图。**这不是全库持续刷新，也不会让隐藏商品自动上架**；新的审核内容需要导入与重新部署。

原商品链接与店铺链接分开。已有明确店铺链接时展示；缺少微店店铺链接时，后端可在限频和超时范围内从该商品公开页面读取真实店铺信息，不能从商品 ID 猜店铺 ID。访问受限时留空；生产已设置 `SELLER_LOOKUP=false`，避免详情请求等待外站；已有店铺链接照常展示。

## 部署

仓库：[jason8772/findcnqc-web](https://github.com/jason8772/findcnqc-web)。Render 备用地址：[findcnqc.onrender.com](https://findcnqc.onrender.com)。`PUBLIC_ORIGIN=https://www.findcnqc.com` 用于分享链接与 sitemap。

本地源码为标准目录；`npm run build` 生成用于浏览器上传的 `deploy/` 包，由 `bootstrap.mjs` 在 Render 还原并运行服务器。只上传生成的部署文件，不上传 `.env`、事件日志、研究原始页面或浏览器资料。

当前服务使用 `npm install --omit=dev && npm run build` 构建、`npm start` 启动、`/api/health` 检查。上传 GitHub 后，应核对 Render 部署的实际提交；不要假定自动部署已启用。当前使用已确认的 0.5c-512mb 常驻实例（$7/月）；免费实例的闲置休眠等待已消除。统计域名新增 $0.25/月，1GB 磁盘 $0.25/月，其他用量另计。挂载磁盘后 Render 不支持零停机部署，更新/重启存在短暂停机窗口；当前是单实例，不是高可用架构。

发布前按实际数据运行测试与构建；发布后核对 health 版本和公开数量、完整目录与仅有 QC 筛选各自计数正确、无 QC 商品卡片可咨询 WhatsApp、隐藏商品详情 404、详情完整相册、Discord／WhatsApp／Agent 跳转、原商品及店铺链接。商品、QC 和精选数量取运行时结果，不沿用上一版的统计。标记 `placement=production_check`、`qa`、`launch` 的测试点击应从投放分析中剔除。
