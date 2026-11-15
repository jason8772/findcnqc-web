# FindCNQC

面向跨境购物用户的商品发现、历史 QC 浏览、新手指南和代采平台目录。首版以 WhatsApp `+852 6931 8767` 咨询为主要转化入口，不收款、不创建账号、不承诺尚未落实的优惠券。

## 运行

Node.js 22 或更高，无第三方运行时依赖。

```sh
npm start
npm test
```

默认端口 4173；Render 使用 `PORT`。健康检查 `/api/health`。

## 数据与真实性

- `data/products.json`：带来源和商品原始链接的快照。商品原始 ID 用于跨站去重；最新抓取时间不等于商品首次上架时间。
- 卖家展示图 `images` 与真实历史仓库图 `qcPhotos` / `qcAlbums` 分开保存。没有 QC 的商品不会显示“已验货”。历史 QC 不是当前订单实拍或真伪认证。
- `research/data-sources.md` 和 `research/competitors.json` 记录公开接口、实际采集范围、访问限制和复用限制。未声称已复制所有站点。
- `data/fx.json` 使用 ECB 经 Frankfurter 发布的参考汇率。原始人民币价与估算显示价分开；国际运费、服务费、税费另计。
- `data/platforms.json` 是平台目录，不表示商业合作、平台安全或当前可下单保证。需要复核的平台保留状态。

## 导流与邀请码

`/go/whatsapp?product=ID&ref=CODE&placement=product` 生成带商品上下文的 WhatsApp 链接。

`/go/agent/AGENT_ID?product=ID` 使用公开观察到的商品链接格式。无商品映射的平台打开官网；`action=register` 打开已核实注册页面或官网。首次上线不附带任何第三方创作者邀请码。

访客来源码：`?ref=creator_name`，只接受 1–32 位字母、数字、下划线、连字符；保存在浏览器用于后续链接。平台返佣码独立配置在 `data/settings.json` 的 `affiliateCodes`：

```json
{"hipobuy":{"enabled":false,"parameter":"inviteCode","value":"YOUR_CONFIRMED_CODE"}}
```

启用前核实平台实际参数与归因规则。设置链接参数不等于已经获得返佣资格。

## 流量测试

记录 `page_view`、`product_view`、`qc_view`、`whatsapp_click`、`agent_click` 等事件。事件只含日期、事件类型、商品、平台、来源码和入口位置；不保存完整 IP、电话号码、聊天内容或搜索词。WhatsApp 点击不是已添加联系人，也不是已成交。

事件写入 stdout（Render Logs 可见）和 `runtime/events.ndjson`。**Render Free 文件系统与内存不持久，重启会丢失本地文件/计数**；正式长期投放需连接持久日志/数据库或持久盘。环境变量 `DATA_DIR` 可指定已挂载持久目录。`ANALYTICS_TOKEN` 配置后才能用 Bearer token 读取 `/api/admin/stats`，否则返回 401。不要把 token 放在网页或 URL。

## 新数据导入

```sh
node scripts/import-catalog.mjs path/to/authorized-catalog.json
```

输入包含 `id,title,sourceUrl,source,marketplace,sourceItemId,originalUrl,priceCny,images,category,addedAt,qcAlbums`。QC 必须有明确来源，不可使用卖家照片冒充。公开访问不自动等于获得完整复制授权。

## 部署

部署仓库：`https://github.com/jason8772/findcnqc-web`（只含本次网站的公共部署文件）。Render 独立服务，域名 `www.findcnqc.com`。最初创建的私有 `findcnqc` 仓库为空，未放入业务资料。

本地源码为标准目录。浏览器上传用 `npm run build` 生成 `deploy/`，包含版本化应用数据包和 `bootstrap.mjs`；Render 启动时还原相同文件后启动 `server.mjs`。只上传 `deploy/` 中生成文件，不上传 `.env`、事件日志、研究原始页面或浏览器资料。

Render：Node runtime，build `npm install --omit=dev`，start `npm start`，health `/api/health`。Free 可能休眠并出现首次唤醒延迟。生产升级需按实际测试结果决定。

`PUBLIC_ORIGIN=https://www.findcnqc.com` 用于商品分享和 sitemap。优惠活动默认关闭，确定可兑现规则后才启用 `promotion`。

`REPSAPI_KEY` 可在 Render 环境变量中接入实时 QC 查询（密钥仅服务端使用，未配置时沿用已取得的历史图）。接口会校验商品身份、限制请求频率，并缓存已验证图片；供货平台故障时保留已有 QC。
