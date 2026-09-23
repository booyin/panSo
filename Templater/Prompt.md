项目 Prompt

```markdown
## 环境信息（已确定，直接使用，不要问）

GITHUB_REPO:        https://github.com/booyin/panSo.git
GITHUB_BRANCH:      main
CF_DOMAIN:          panSo.L.cd              # 绑 CF 的主域名
CF_PAGES_PROJECT:   pan-search             # CF Pages 项目名
CF_WORKER_NAME:     pan-search             # Worker 名字
CF_KV_NAMESPACE:    PAN_SEARCH_KV          # KV 命名空间名
CF_D1_DATABASE:     pan-stats              # D1 数据库名
LINK_SALT:          你的随机盐（32位以上）  # 不要提交到仓库
ADMIN_TOKEN:        你的管理密码（16位以上）# 保护 /admin

Secrets（只引用，不写入代码）：
CLOUDFLARE_API_TOKEN、CLOUDFLARE_ACCOUNT_ID、LINK_SALT、ADMIN_TOKEN

```

# 项目：网盘资料搜索站
## 目标

Obsidian 写笔记（Templater 生成 uid），Obsidian-git push 
   ↓  
Actions 解析（正文链接 + 下架名单offline.md）
   ↓  
生成索引 + 写cloudflare- KV 
   ↓  
Hugo 构建 生成静态站 
   ↓  
部署到 Cloudflare Pages + GitHub Pages 
   ↓  
Worker 短链跳转 + D1 统计 
   ↓  
下架删 KV + 改 offline。

用户搜索资料，点按钮跳转网盘。真实链接存 Cloudflare KV，前端只拿短链。  


## 技术栈
- 源数据：Obsidian Markdown，obsidian-git 同步
- 构建：GitHub Actions + Node.js 20（ESM）
- 索引：前端 Fuse.js，静态 search-index.json
- 静态站：Hugo
- 部署：Cloudflare Pages + GitHub Pages，同一份 public/
- 后端：Cloudflare Worker + KV + D1

## 目录结构
repo/
├─ archetypes/             
├─ Templater/
│  ├─ kv.html              # kv列表带删除功能+删除确认及备注 ，页面设计参照## web
│  └─ pan.md               # Templater插件调用新建笔记的模板md，
├─ content/
│  ├─ list/                 # 公开列表，Hugo 渲染页面
│  ├─ \_hidden/              # 隐性索引，Hugo 忽略，只进 JSON
│  └─ \_offline.md           # 下架名单，Hugo 忽略，Actions 读
├─ web/
│  ├─ index.html            # 搜索页，Hugo 首页复用
│  └─ fuse.min.js           # 本地化，不用 CDN
├─ worker/
│  ├─ index.js              # 短链跳转 + 统计
│  └─ stats.js              # /admin 统计 API
├─ scripts/
│  ├─ parse-links.mjs       # 正文链接解析
│  ├─ parse-links.test.mjs  # 单测
│  ├─ build-index.mjs       # 生成 search-index.json
│  ├─ sync-kv.mjs           # 真实链接写 KV
│  └─ load-offline.mjs      # 读下架名单
├─ migrations/
│  └─ 0001\_init.sql         # D1 表结构
├─ layouts/                 # Hugo 模板
├─ static/                  # Hugo 静态资源（索引写这里）
├─ .github/workflows/build.yml
├─ package.json
├─ wrangler.toml
├─ hugo.toml
├─ README.md
├─ COMPLIANCE.md
└─ TAKEDOWN.md

## pan.md 模板笔记规范

```markdown
---
uid: <%* tR += tp.date.now("YYYYMMDDHHmmss") + Math.floor(Math.random()*0xffff).toString(16).padStart(4,'0') %>
title: <%* tR += tp.file.title %>
tags: [教程, PDF]
categories: []    # hugo网站用得到
return_status:""  # 如果有报错信息
---
<!-- 正文用 H2 区块写网盘链接： -->
## 网盘链接
- 百度网盘：              提取码：    
- 夸克网盘：             
- 阿里云盘：              提取码：    

<!-- 只解析 ## 网盘链接 这个 H2 段落，正文其他链接不抓。 -->

### 网盘资料介绍及使用方法
……

免费分享，失效请留言 <微信公众号客服>

```

## web
> 本地kv.html和hugo网页都参照这个web设计，kv.html每个列表卡片右上角多个删除x按钮。
```html
<header> <H1> <tab栏:电脑端横向摆，移动端变成汉堡菜单> <搜索🔍框><双色样式切换></header>
<左栏> <content> <右栏> 
<content>列表样式：网盘资料分享卡片左右分布，列表左边一个浅色图框默认显示网盘平台名字，右栏上部显示标题，右栏下部显示网盘平台名、日期、大小、tags、热度数量。所以列表卡片是两行字的高度。第一行列表是轮播效果。
左右栏手机端没有。

```


## 核心流程

1. Obsidian 新建笔记，Templater 生成 uid
2. 填网盘链接，obsidian-git push
3. Actions 触发：  
	a. npm ci  
	b. parse-links.test.mjs  
	c. build-index.mjs --check  
	d. build-index.mjs（读 content/，生成 static/search-index.json）  
	e. sync-kv.mjs（真实 url/code 写 KV）  
	f. hugo --minify（读 static/，产出 public/）  
	g. 部署 gh-pages  
	h. 部署 CF Pages
4. 用户访问 Pages，搜索，点按钮 /go/{pan}/{shortCode}
5. Worker 302 到真实网盘链接，同时记 D1。**一定让用户端去网盘平台/拉新渠道商的链接是原生的网盘分享链接，切不可有CF-KV的迹象**

## 各脚本职责

### parse-links.mjs

导出 extractLinks(content)，返回 \[{ pan, url, code, domain }\]。


- 没有则解析 ## 网盘链接 H2 列表
- 平台名从域名映射：
export const PAN_MAP = {
  'pan.baidu.com':        { key: 'baidu',  name: '百度网盘' },
  'pan.quark.cn':         { key: 'quark',  name: '夸克网盘' },
  'pan.quark.com':        { key: 'quark',  name: '夸克网盘' },
  'www.aliyundrive.com':  { key: 'ali',    name: '阿里云盘' },
  'www.alipan.com':       { key: 'ali',    name: '阿里云盘' },
  'pan.xunlei.com':       { key: 'xunlei', name: '迅雷网盘' },
  'cloud.189.cn':         { key: 'tianyi', name: '天翼云盘' },
  'www.123pan.com':       { key: '123',    name: '123网盘' },
  '123pan.com':           { key: '123',    name: '123网盘' },
  'pan.115.com':          { key: '115',    name: '115网盘' },
  'cowtransfer.com':      { key: 'cow',    name: '奶牛快传' },
};
- 提取码支持「提取码：xxx」「密码：xxx」「code: xxx」
- 解析失败 throw，不静默返回空
- 配单测覆盖：代码块、列表、无提取码、无链接

### build-index.mjs

- 读 content/list 和 content/\_hidden
- 读 content/\_offline.md 下架名单
- 对每条笔记：
	- 校验 uid 格式 /^\\d{14}\[0-9a-f\]{4}$/
		- 查重 uid，冲突 exit(1)
		- 若 uid 在下架名单，跳过
		- 解析正文链接
		- 若所有链接都被下架，跳过
		- shortCode = uid.slice(8, 14) + uid.slice(14);
		- 生成 { uid, shortCode, title, tags, size, date, desc,  
		links: \[{pan, panName, linkId}\], listVisible, text }  
		linkId = `${pan}_${shortCode}`
- 写 static/search-index.json
- 日志输出：总数、隐藏数、下架数
- \--check 模式只校验不写文件

### sync-kv.mjs

- 读 content/list 和 content/\_hidden
- 解析真实 url 和 code
- 对每个链接：KV key = `link:{pan}_{shortCode}`，value = {url, code}
- 用 wrangler kv:key put 批量写入
- 跳过下架名单里的 uid
- 日志不打印 url 和 code

### load-offline.mjs

- 读 content/\_offline.md
- 格式：
	# offline\_list
	- 202609231430000001
		- 20260923143000000a
- 正则 /^\\s\*\[-\*+\]\\s+(\[A-Za-z0-9\_-\]+)\\s\*$/
- 返回 Set
- 兼容 frontmatter

## Worker

worker/index.js 路由：

- GET /go/:pan/:shortCode → 302 跳转
- GET /code/:shortCode → 返回 { code }
- GET /health → ok

/go 逻辑：

1. Referer 校验，空 referer 也放行但记录
2. 解析 uid：cookie uid 优先，其次 IP+UA+SALT 的 SHA-256 哈希
3. 写 D1 clicks（ts, uid, short\_code, pan, link\_id, channel, ref, country, ua）
4. channel 从 referer 域名推断：[github.com](https://github.com/)→gh，你的域名→cf，  
	[t.me](https://t.me/)→tg，[weibo.com](https://weibo.com/)→wb，其他→mirror
5. 查 KV link:{pan}\_{shortCode}
6. 查不到返回 410
7. 302 到真实 url

限流：单 IP 每分钟 30 次，KV 计数，超限 429。  
不存 IP 明文。

worker/stats.js：

- GET /admin/channels?days=30 → 渠道榜
- GET /admin/users?days=30 → 忠实用户榜
- GET /admin/links?days=30 → 热资料榜
- GET /admin/searches?days=30 → 热搜词榜
- 用 Bearer Token 保护
- 全部 SQL 聚合

wrangler.toml：  
name = "pan-search"  
main = "worker/index.js"  
compatibility\_date = "2026-01-01"

\[\[kv\_namespaces\]\]  
binding = "KV"  
id = "你的KV\_ID"

\[\[d1\_databases\]\]  
binding = "DB"  
database\_name = "pan-stats"  
database\_id = "你的D1\_ID"

## D1 表结构

migrations/0001\_init.sql：

CREATE TABLE clicks (  
id INTEGER PRIMARY KEY AUTOINCREMENT,  
ts TEXT NOT NULL,  
uid TEXT NOT NULL,  
short\_code TEXT,  
pan TEXT,  
link\_id TEXT,  
channel TEXT,  
ref TEXT,  
country TEXT,  
ua TEXT  
);  
CREATE INDEX idx\_uid ON clicks(uid);  
CREATE INDEX idx\_short\_code ON clicks(short\_code);  
CREATE INDEX idx\_channel ON clicks(channel);  
CREATE INDEX idx\_ts ON clicks(ts);

CREATE TABLE searches (  
id INTEGER PRIMARY KEY AUTOINCREMENT,  
ts TEXT,  
uid TEXT,  
q TEXT,  
result\_count INTEGER  
);

## 前端 web/index.html

单文件 H5，移动优先，深色主题，原生 JS，不用框架。

- fetch ./search-index.json（加时间戳防缓存）
- Fuse.js keys：title 0.5、tags 0.25、desc 0.15、pan 0.05、text 0.05
- 无关键词只显示 listVisible: true
- 有关键词全量搜，含隐藏条目
- 标签过滤，点击切换
- 支持 ?q= 和 ?tag= 深链，URL 同步
- 卡片：标题、平台、大小、日期、介绍、多平台按钮
- 按钮文案「打开百度网盘」，href="/go/baidu/{shortCode}"
- 提取码按钮点击后 /code/{shortCode} 拉取
- 底部：最后更新、失效反馈、隐私说明
- 所有输出转义防 XSS

## Hugo

hugo.toml：  
baseURL = 'https://你的域名/'  
languageCode = 'zh-cn'  
title = '网盘资料库'  
disableKinds = \['taxonomy', 'term'\]

layouts/index.html 复用 web/index.html 的搜索页。  
layouts/\_default/single.html 渲染详情页。  
content/\_hidden 因下划线被 Hugo 忽略，但 build-index.mjs 能读到。

## Actions

.github/workflows/build.yml：

on:  
push:  
paths: \['content/**', 'scripts/**', 'web/**', 'hugo.toml', 'layouts/**'\]  
schedule:

- cron: '0 3 \* \* \*'  
	workflow\_dispatch:

concurrency:  
group: build-${{ github.ref }}  
cancel-in-progress: true

permissions:  
contents: write

jobs:  
build:  
runs-on: ubuntu-latest  
steps:

- uses: actions/checkout@v4
- uses: actions/setup-node@v4  
	with: { node-version: 20 }
- name: 暂缓，等待更多 push  
	run: sleep 180
- run: npm ci
- run: node scripts/parse-links.test.mjs
- run: node scripts/build-index.mjs --check
- run: node scripts/build-index.mjs
- name: 同步 KV  
	run: node scripts/sync-kv.mjs  
	env:  
	CLOUDFLARE\_API\_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }} CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE\_ACCOUNT\_ID }}  
	LINK\_SALT: ${{ [secrets.LINK](https://secrets.link/)\_SALT }}
- run: hugo --minify
- uses: peaceiris/actions-gh-pages@v4  
	with:  
	github\_token: ${{ secrets.GITHUB\_TOKEN }}  
	publish\_dir: ./public
- uses: cloudflare/wrangler-action@v3  
	with:  
	apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }} accountId: ${{ secrets.CLOUDFLARE\_ACCOUNT\_ID }}  
	command: pages deploy public --project-name=pan-search

Secrets：CLOUDFLARE\_API\_TOKEN、CLOUDFLARE\_ACCOUNT\_ID、LINK\_SALT

## 下架机制

收到投诉：

1. 在本地kv.html里面搜到该uid点击该卡片右上角删除按钮>弹窗确认。
   批量删 KV： 【Templater/放一个 kv.html本地双链文件链接CF-kv网页搜索删除】
	wrangler kv:key delete "link:baidu\_{shortCode}"  
	wrangler kv:key delete "link:quark\_{shortCode}"  
	...
2. content/\_offline.md 加一行 uid
3. git push
4. Actions 重建，该笔记不进索引
5. 记录到 TAKEDOWN.md

## README

- 项目简介
- 在线搜索徽章按钮
- 快捷入口：搜教程、搜 PDF、按标签
- \<details> 折叠静态列表，Actions 自动更新
- 失效反馈方式
- 免责声明：不存储文件，仅分享链接
- 隐私说明：匿名统计

注意：GitHub README 过滤 script/iframe/form，不能内嵌搜索，  
只能跳转。

## 合规

COMPLIANCE.md 包含：

- 不存 IP 明文，只存加盐哈希
- 提供 no-track cookie 退出选项
- 隐私说明和投诉下架入口
- 只分享合法授权内容
- 网盘官方拉新渠道
- 多分享链接轮换
- 短链 ID 定期轮换
- 渠道水印
- 二次替换识别

## 要求

- 代码简洁，注释中文
- 所有脚本 ESM
- 本地能跑：node scripts/build-index.mjs
- CI 能跑，校验失败要 fail
- 日志不打印 url 和 code
- 输出完整代码 + 验证命令 + 改动文件清单



## 闭环流程

1. 填环境变量清单
2. 粘 prompt + 规格给 AI
3. AI 生成代码 + 自检结果
4. 你用抽查表核对 AI 输出
5. 手动做 7 件事（KV/D1/Secrets/部署/绑定）
6. 本地 node scripts/build-index.mjs 验证
7. git push 触发 Actions
8. 打开 https://2ff.cc.cd 验证搜索
9. 点按钮，验证 /go/baidu/xxx 能 302 到网盘
10. 查 D1：wrangler d1 execute pan-stats --command="SELECT COUNT(*) FROM clicks" 



## 自检约束

生成代码后，必须逐项检查并输出结果：

### A. 文件完整性
A1. 目录结构里列出的每个文件都生成了吗？逐文件列出 ✅/❌
A2. package.json 的 dependencies 和代码里 import 的包一致吗？
A3. .gitignore 是否排除了 node_modules/、dist/、public/、.env、kv-bulk.json？

### B. uid 与短链
B1. Templater 模板生成的 uid 是否匹配 /^\d{14}[0-9a-z]{4}$/？
B2. build-index.mjs 是否包含 uid 格式校验 + 查重 + 冲突 exit(1)？
B3. shortCode 是否 = sha256(uid).slice(0, 8)？是否全局唯一？
B4. linkId 是否 = `${pan}_${shortCode}`？KV key 是否 = `link:${linkId}`？
B5. 前端短链是否 = `/go/${pan}/${shortCode}`？

### C. 链接解析
C1. parse-links.mjs 是否优先代码块、回退 H2 列表、都没有则 throw？
C2. 是否只解析 ## 网盘链接 段落，正文其他链接不抓？
C3. 平台映射是否独立文件（scripts/pans.mjs）？
C4. 是否支持「提取码：」「密码：」「code:」三种写法？
C5. URL 的 query 参数是否原样保留、不截断？

### D. Worker 与 KV
D1. Worker 路由是否包含 /go/:pan/:shortCode、/code/:pan/:shortCode、/health？
D2. /go 是否 302 跳转、是否原样转发完整 URL（含 query）？
D3. 是否只做 302，没有 JS 跳转、meta 跳转、反代？
D4. KV 读取失败是否返回 410 页面（HTML，不是空响应）？
D5. 限流 key 是否带 expirationTtl？
D6. 是否不存 IP 明文，uid 用 SHA-256 + SALT？
D7. channel 推断是否独立文件，是否覆盖 gh/cf/tg/wb/mirror？

### E. 索引与前端
E1. search-index.json 是否不含真实 url 和 code？
E2. 前端 fetch 是否用根路径 /search-index.json（不是 ./）？
E3. Fuse.js keys 权重是否 = title 0.5 / tags 0.25 / desc 0.15 / pan 0.05 / text 0.05？
E4. 无关键词时是否只显示 listVisible: true？
E5. 有关键词时是否全量搜（含隐藏条目）？
E6. 所有输出是否转义（防 XSS）？
E7. 提取码是否走 /code/:pan/:shortCode，不写在 JSON 里？

### F. Hugo
F1. hugo.toml 的 baseURL 是否 = https://2ff.cc.cd/？
F2. content/_hidden 是否因下划线被 Hugo 忽略？
F3. static/search-index.json 是否会被 Hugo 复制到 public/？
F4. layouts/index.html 是否复用搜索页？

### G. Actions
G1. 是否包含：npm ci → 单测 → build --check → build → sync-kv → hugo → 部署 Worker → 部署 Pages？
G2. 是否去掉了 sleep 180，只用 concurrency 防抖？
G3. Worker 部署步骤是否存在（wrangler deploy）？
G4. 是否用 Secrets 引用 CLOUDFLARE_API_TOKEN / CLOUDFLARE_ACCOUNT_ID / LINK_SALT？
G5. 日志是否不打印 url 和 code？
G6. 校验失败是否 exit(1) 阻止部署？

### H. 下架机制
H1. content/_offline.md 格式是否统一（顶格 - uid）？
H2. load-offline.mjs 正则是否 = /^\s*[-*+]\s+([A-Za-z0-9_-]+)\s*$/？
H3. scripts/offline.mjs 是否能一条命令删所有平台的 KV？
H4. build-index.mjs 是否跳过下架名单里的 uid？

### I. 合规
I1. COMPLIANCE.md 是否包含：不存 IP 明文、no-track 退出、投诉入口、只分享合法内容？
I2. TAKEDOWN.md 是否有下架流程？
I3. 前端底部是否有隐私说明和失效反馈入口？

### J. 外部依赖核对（AI 无法验证，必须提醒用户）
J1. GitHub Secrets 是否配置了：CLOUDFLARE_API_TOKEN、CLOUDFLARE_ACCOUNT_ID、LINK_SALT、ADMIN_TOKEN？
J2. CF KV 命名空间是否创建？ID 是否填入 wrangler.toml？
J3. CF D1 数据库是否创建？ID 是否填入 wrangler.toml？
J4. D1 表是否已执行 migrations/0001_init.sql？
J5. CF Pages 项目是否创建？名称是否 = pan-search？
J6. Worker 是否首次部署？路由是否绑定到 CF_DOMAIN？
J7. 自定义域名 2ff.cc.cd 是否已绑定到 CF Pages？

## 输出格式

1. 文件树
2. 每个文件完整代码（用 ``` 包裹，标明路径）
3. 自检结果表（A1-J7 逐项 ✅/❌）
4. 用户需手动执行的命令清单（按顺序）
5. 本地验证命令 + 预期输出

## 禁止

- 禁止留 TODO、占位符（除 Secrets 外）
- 禁止说「请自行补充」
- 禁止省略代码
- 禁止修改已定的规格（uid 格式、短链格式、KV key 格式等）
- 禁止把真实链接写进 search-index.json