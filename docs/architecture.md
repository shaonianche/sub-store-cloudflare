# 架构说明

Sub-Store Cloudflare 是一个单 Worker 应用：管理界面由 Worker Static Assets 托管，配置 API 和订阅输出由同一个 Worker 处理，结构化配置保存在 D1。

它的产品边界是云端订阅配置器：订阅源、节点处理、组合订阅、规则模板、预览校验和最终下载链接。更完整的范围说明见 [product-scope.md](product-scope.md)。

## 运行边界

```text
Cloudflare Worker
  |
  |-- Static Assets                  Vue 管理界面
  |-- /api/env                       环境信息
  |-- /api/settings                  前端设置
  |-- /api/storage                   备份与恢复
  |-- /api/sources                   订阅源
  |-- /api/collections               组合订阅
  |-- /api/templates                 分流模板
  |-- /api/scripts                   内置脚本元数据
  |-- /api/preview/*                 节点预览
  |-- /api/link/*                    快捷下载链接生成
  |-- /api/source/flow/*             订阅源流量信息
  |-- /api/utils/node-info           节点 IP/地理信息查询
  |-- /api/proxy/parse               一次性节点转换
  |-- /api/rule/parse                一次性规则转换
  |-- /api/shares                    独立下载授权
  |-- /api/recycle-bin               有上限的配置回收站
  |-- /download/source/:id[/:target]   单订阅源输出
  |-- /download/collection/:id[/:target] 组合订阅输出
  |-- /download/collection/:id/ruleset/:provider Clash 规则集转成 sing-box rule set
  |
  |-- D1                             配置 / download_grants / recycle_bin
  |-- Cache API                      可选远程订阅短期缓存
  |-- Worker Secrets                 管理端 token / 下载 token
```

核心路径只需要 Workers、D1 和 Secrets。Cache API 是自动降级的边缘优化；KV、R2、Durable Objects、Queue、Cron 都不是必要组件。

## 数据模型

| 表 | 作用 |
| --- | --- |
| `sources` | 保存远程订阅 URL 或本地节点文本。 |
| `collections` | 保存订阅源组合、过滤器和默认模板。 |
| `templates` | 只保存用户创建的规则模板。内置模板由 Worker 代码维护。 |
| `app_settings` | 保存远程订阅请求参数、主题和必要的前端默认状态。 |
| `download_grants` | 保存独立下载授权的 token hash、资源范围、格式限制和有效期。 |
| `recycle_bin` | 最多保存 50 条被删除配置的快照。 |

## 输出流程

```text
客户端请求 /download/collection/:id[/:target]
  |
  |-- 校验下载 token
  |-- 读取 collection
  |-- 应用请求级临时输入参数 url / content / ua
  |-- 拉取 collection 里的 sources
  |-- 解析节点
  |-- 应用 source filters
  |-- 合并
  |-- 应用 collection filters
  |-- 确保节点名唯一
  |-- 套用 template
  |-- 输出 mihomo / stash / surge / surge-mac / loon / qx / shadowrocket / sing-box / v2ray / uri / json
```

`target` 可省略；省略时 Worker 会根据客户端 User-Agent 自动选择输出格式，无法识别时默认输出 Mihomo。`/download/source/:id[/:target]` 走同一套解析和过滤逻辑，只是不读取 collection。

下载请求可以附加 `url`、`content` 和 `ua`：

- `url`：临时替换当前订阅源的远程订阅地址。
- `content`：临时按本地节点文本解析。
- `ua` / `userAgent`：临时覆盖拉取远程订阅时使用的 User-Agent。

这些参数只影响当前请求，不写入 D1。组合订阅会把临时输入应用到组合里第一个选中的订阅源，然后继续执行组合级过滤器和模板。

远程订阅的 User-Agent 优先级是：订阅源自定义 `ua` / 临时 `ua` 参数 > 透传下载请求的 User-Agent > 全局默认 User-Agent。

## 输入与核心能力

远程订阅最多 8 个 URL，可以按行填写并合并；单个响应上限 2 MiB，合计上限 12 MiB。本地订阅支持单行 URI、Mihomo YAML、JSON/JSON5 代理数组、常见 Surge/Loon/Quantumult X 单行节点和完整 Base64 内容。远程响应可以经过 Workers Cache API 短期缓存，并只透传允许的订阅元数据。管理 API 请求体上限是 4 MiB，流量信息、节点信息和 DoH 响应分别限制在 64 KiB。

这版保留的核心能力是：

- 订阅源管理和组合订阅。
- 节点解析、过滤、重命名、去重、排序、域名解析、旗帜和常用属性设置。
- 构建时打包的 JavaScript Filter / Operator。
- Mihomo 规则模板和自定义模板。
- 原始/处理后节点预览，本地节点校验。
- 下载链接级临时输入和一次性格式转换。
- 工具页的一次性节点/规则转换、独立下载授权和配置回收站。
- 单订阅源自定义 User-Agent 和透传 User-Agent。
- 订阅流量信息、配置备份与恢复。
- Mihomo、Stash、Surge、Surge Mac、Surfboard、Loon、Egern、Shadowrocket、Quantumult X、sing-box、v2ray、URI、JSON 输出。

运行时脚本字符串、远程脚本、文件托管、Gist 同步、公开分享平台、无限归档、定时任务和日志系统不在核心路径里，也不会保留空壳 UI 或兼容接口。

## Filters

过滤器是这版自己的小型 JSON DSL，保存在 D1。前端编辑器会把界面里的动作转换成下面这些结构；Worker 只读取这些结构：

- `include`：按字段和正则保留节点。
- `exclude`：按字段和正则排除节点。
- `rename`：按正则重命名字段，默认字段是 `name`。
- `delete-field`：按正则删除字段里的匹配文本，默认字段是 `name`。
- `dedupe`：按一个或多个字段去重，可以删除重复项，也可以给重复节点重命名。
- `sort`：按节点名排序，也支持随机排序。
- `regex-sort`：按一组正则表达式把节点排到前面。
- `resolve`：请求时用 DoH 把节点域名解析成 IPv4/IPv6，并保留 TLS 节点的原始 SNI。
- `flag`：按节点名识别区域旗帜，或移除已有旗帜。
- `quick`：过滤无效节点，并批量设置 `udp`、`tfo`、`skip-cert-verify`、`vmess aead` 等常用属性。
- `script`：按 code-owned script ID 调用已经随 Worker 编译的 Filter / Operator，D1 只保存 ID、kind 和参数。

示例：

```json
[
  { "type": "include", "field": "name", "pattern": "香港|HK|日本|JP" },
  { "type": "exclude", "field": "name", "pattern": "官网|剩余|倍率" },
  { "type": "delete-field", "field": "name", "patterns": ["倍率\\s*\\d+"] },
  { "type": "dedupe", "fields": ["server", "port"], "action": "rename", "link": "-" },
  { "type": "regex-sort", "expressions": ["香港|HK", "日本|JP", "新加坡|SG"], "direction": "asc" },
  { "type": "flag", "mode": "add" },
  { "type": "sort", "direction": "asc" }
]
```

## Templates

内置模板由 Worker 代码直接维护，因此升级 Worker 后现有部署会立即获得模板修正，不需要重复 seed。用户自定义模板保存在 D1。模板只应用于 Mihomo 和 Stash 这类 YAML 输出。导入接口接受 JSON 或 YAML；常见 Mihomo YAML 键名会归一化成内部配置。

- `mixedPort`
- `mixed-port`
- `allowLan`
- `allow-lan`
- `mode`
- `logLevel`
- `log-level`
- `dns`
- `sniffer`
- `proxyGroups`
- `proxy-groups`
- `ruleProviders`
- `rule-providers`
- `rules`

`proxyGroups[].proxies` 或 `proxy-groups[].proxies` 里可以使用 `$all`，生成时会展开为当前组合订阅里的全部节点。空的或只引用已删除组的 `proxy-groups` 不会写进最终 YAML。

内置模板的规则数据来自 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)：`acl4ssr-mihomo`、`acl4ssr-mihomo-no-emoji` 和 `ai-streaming-mihomo` 使用 `meta` 分支编译好的 `.mrs` 规则集（`format: mrs`，`behavior` 为 `domain` / `ipcidr`），默认从 `cdn.jsdelivr.net` 取；`loyalsoldier-whitelist` 和 `loyalsoldier-blacklist` 仍使用 Loyalsoldier 的 YAML 规则集，因为 `reject`、`direct`、`tld-not-cn` 这类列表在 MetaCubeX 没有等价物。

CDN 主机可以通过设置项 `rulesetCdn` 换成自建镜像或 jsDelivr 的其它节点（只接受 `https` 源，其余值会被忽略）：

```bash
curl -X PATCH https://<admin-domain>/api/settings \
  -H 'authorization: Bearer <admin-token>' -H 'content-type: application/json' \
  -d '{"rulesetCdn":"https://fastly.jsdelivr.net"}'
```

Surge、Surfboard、Loon、Egern、Shadowrocket、Quantumult X、v2ray、URI 和 JSON 输出使用同一套节点解析与过滤结果，但不读取 Mihomo 规则模板；`sing-box` 输出会读取同一份模板的分组与可转换规则，见下一节。

## sing-box 与 Mihomo 模板的对应关系

集合绑定的模板同时驱动 Mihomo 和 sing-box 输出，`sing-box` 输出会尽量贴近 Mihomo 链接的分组：

| Mihomo | sing-box | 说明 |
| --- | --- | --- |
| `select` | `selector` | 保留成员顺序，默认选中第一个成员 |
| `url-test` | `urltest` | `interval`（秒）转成 sing-box 的时长字符串；`tolerance` 单位都是毫秒 |
| `fallback` / `load-balance` | `selector` | sing-box 1.13 起移除了这两种出站类型，降级为手动选择 |
| `$all` / `filter` | 相同 | 只展开当前组合里实际存在、且 sing-box 支持的节点 |
| `DIRECT` / `REJECT` / `REJECT-DROP` | `DIRECT` / `REJECT` | `PASS` 和悬空引用会被丢弃，否则 sing-box 启动时报 `dependency[...] not found` |
| `mixed-port` / `allow-lan` | `mixed` 入站的 `listen_port` / `listen` | |
| `MATCH,<策略>` | `route.final` | 没有 MATCH 时使用第一个分组 |
| `RULE-SET,<MetaCubeX 规则集>,<策略>` | `route.rule_set`（`sing` 分支的 `.srs`） | 见下 |
| `GEOIP,<两位国家码>,<策略>` | `geoip/<国家码>.srs` | 例如 `GEOIP,CN` → `geoip/cn.srs` |

规则按类型转换：`DOMAIN`、`DOMAIN-SUFFIX`、`DOMAIN-KEYWORD`、`DOMAIN-REGEX`、`IP-CIDR`、`IP-CIDR6`、`SRC-IP-CIDR`、`DST-PORT`、`SRC-PORT`、`PROCESS-NAME`、`PROCESS-PATH` 会写成 sing-box 的 route 规则。

`RULE-SET` 规则引用的是 MetaCubeX/meta-rules-dat 规则集时（内置模板都是），会生成对应的远程 rule set：

```json
{ "type": "remote", "tag": "GFW", "format": "binary",
  "url": "https://cdn.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@sing/geo/geosite/gfw.srs",
  "update_interval": "1d", "download_detour": "DIRECT" }
```

同一份规则集只生成一次（例如 `ChinaIP` 和 `GEOIP,CN` 共用 `geoip/cn.srs`），策略是 `REJECT` 时写成 `action: "reject"`，其余写成 `outbound`。`download_detour: "DIRECT"` 是必需的：sing-box 1.12/1.13 没有 `http_clients`，不指定时规则集会走默认出站（也就是代理），代理不可用时整份配置直接启动失败。1.14 起该字段会给出弃用告警，等 1.16 真正移除后再按客户端版本切换到 `http_clients`。

不是 MetaCubeX 的规则集（例如 Loyalsoldier 预设的 `.txt`、自定义 URL）会由 Worker 转换后提供：`/download/collection/<collection-id>/ruleset/<provider>?token=<download-token>` 拉取模板里那个 provider，按 `payload` YAML 或 Surge 风格文本解析，输出 sing-box source 格式（`+.domain` → `domain_suffix`、裸域名 → `domain_suffix`、CIDR → `ip_cidr`、`DOMAIN-KEYWORD,` → `domain_keyword`、`PROCESS-NAME,` → `process_name`、`DST-PORT,` / `SRC-PORT,` → 数字 `port` / `source_port`，区间写法 `1000-2000` → `port_range` 的 `"1000:2000"`），结果按 provider 的 `interval` 缓存在 Cache API。profile 里对应写成：

```json
{ "type": "remote", "tag": "reject", "format": "source",
  "url": "https://<your-domain>/download/collection/<id>/ruleset/reject?token=<download-token>",
  "update_interval": "1d", "download_detour": "DIRECT" }
```

这条路径只在集合下载（URL 里带 download token）时生效；用 `convertSubscriptionContent` 这类没有集合上下文的方式渲染时会跳过这些规则。`GEOSITE`、`IP-ASN`、`SCRIPT` 等规则在任何情况下都会被跳过，命中它们的流量会落到 `route.final` 指向的分组。

没有可用模板时（例如集合未绑定模板），`sing-box` 输出回落到内置的 `PROXY` / `AUTO` 两个分组。

## 为什么只用 D1

这个项目的数据是结构化配置，主要是订阅源、组合关系、过滤器和规则模板。D1 可以直接表达这些关系，也方便迁移和导出。大文件、后台任务和跨请求状态都不是核心路径，因此不默认引入其他 Cloudflare 存储或异步组件。

## 上游关系

完整订阅管理系统请参考 [sub-store-org/Sub-Store](https://github.com/sub-store-org/Sub-Store)。本项目把原版中适合免费 Workers 的解析、转换、私有下载授权和误删恢复工作流收敛到 Cloudflare-native 实现；文件、第三方同步、公开分享、无限归档、运行时脚本和持久日志仍不属于默认范围。
