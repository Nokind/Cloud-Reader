# Cloud Reader · 私人云端阅读器
一个部署在 Cloudflare Pages 上的多用户电子书阅读器。每位用户拥有独立书库、阅读进度、书签、笔记和本地缓存，适合个人与家庭自建阅读空间。
<img width="1849" height="912" alt="image" src="https://github.com/user-attachments/assets/34cec399-cb86-4539-8330-ac0ba20fa663" />


## 功能

- **多用户独立空间**：管理员创建用户、管理账号和容量配额；书籍与阅读记录按账号隔离。
- **多格式阅读**：TXT、EPUB、PDF、Markdown、HTML、DOCX、CBZ。
- **阅读记录同步**：同一站点账号同步进度、书签、笔记和阅读设置。
- **按需加载与离线缓存**：TXT 支持分段加载，可在阅读设置中开启整本下载；本地阅读无需额外离线密码。
- **个性化排版**：12 种字体选项、10 套配色，以及自定义文字/背景颜色、字重、字间距等。未安装的字体会使用后备字体。
- **云端备份与恢复**：支持 OneDrive、Google Drive、WebDAV；百度网盘、阿里云盘通过 WebDAV 挂载服务接入。

> 云服务功能是应用格式的备份与合并恢复。它不是网盘文件浏览器，也不提供浏览器关闭后的常驻双向同步。

## 部署前准备

准备一个 Cloudflare 账号，以及本项目的 **v1.2.2 部署包**。首次启用 R2 时按 Cloudflare 控制台提示完成开通。

| 资源               | 用途          | 建议资源名称               | 程序要求的绑定名称 |
| ---------------- | ----------- | -------------------- | --------- |
| Cloudflare Pages | 运行网站和后端     | `cloud-reader`       | —         |
| D1 数据库           | 用户、书目、阅读记录等 | `cloud-reader-db`    | `DB`      |
| R2 存储桶           | 书籍原文件       | `cloud-reader-books` | `BOOKS`   |

资源名称可以自定义，**绑定名称必须区分大小写，准确填写 `DB` 和 `BOOKS`**。R2 存储桶保持私有，无需开启公开访问，也不需要为本部署方式申请 R2 API 密钥。

部署包已经包含前端与后端，不需要运行 `npm install` 或构建命令。仅使用 GitHub Pages 无法运行此项目所需的后端。

## 方式一：控制台直接上传（适合首次部署）

### 1. 创建数据库和存储桶

在 Cloudflare 控制台中：

1. 找到 **D1**，创建数据库，例如 `cloud-reader-db`。
2. 找到 **R2 Object Storage**，创建存储桶，例如 `cloud-reader-books`。
3. 保持存储桶私有。

使用全新空数据库即可，程序首次访问初始化接口时会自动建表。**无需手动导入 SQL 文件。**

### 2. 创建 Pages 项目并上传

进入 **Workers & Pages → 创建应用 → Pages → 上传资产 / Direct Upload**，设置项目名称，上传发布包 ZIP 或解压后的部署文件夹。控制台文案可能随版本调整，请确认创建的是 Pages 项目。

部署根目录应直接包含：

```text
_worker.js
_routes.json
index.html
```

不能多套一层 `Cloud-Reader.v1.2.2/` 文件夹；不要上传整个开发工作区或 `node_modules`。

本项目采用 Pages Advanced Mode，后端入口为 `_worker.js`。Cloudflare 支持通过控制台直接上传这一形式的 Worker。参见 [Direct Upload 官方说明](https://developers.cloudflare.com/pages/get-started/direct-upload/) 和 [Advanced Mode 官方说明](https://developers.cloudflare.com/pages/functions/advanced-mode/)。

第一次部署后暂时看到配置提示是正常的，继续完成下面的绑定。

### 3. 添加 D1 和 R2 绑定

进入刚创建的 **Pages 项目 → Settings → Bindings → Add**，在生产环境添加：

| 绑定类型        | Variable name | 选择的资源       |
| ----------- | ------------- | ----------- |
| D1 database | `DB`          | 第 1 步创建的数据库 |
| R2 bucket   | `BOOKS`       | 第 1 步创建的存储桶 |

`DB`、`BOOKS` 是资源绑定，不能用同名普通文本变量代替。绑定配置与重新部署要求可参见 [Pages Bindings 官方说明](https://developers.cloudflare.com/pages/functions/bindings/)。

### 4. 设置环境变量和密钥

进入 **Settings → Variables and Secrets**，为生产环境添加：

| 名称                | 类型     | 是否必填        | 说明                           |
| ----------------- | ------ | ----------- | ---------------------------- |
| `ADMIN_USERNAME`  | 文本变量   | 新站点必填       | 管理员用户名，3–32 位英文字母、数字、下划线或连字符 |
| `ADMIN_PASSWORD`  | Secret | 新站点必填       | 管理员登录密码，8–256 位              |
| `DEPLOY_KEY`      | Secret | 是           | 独立随机长字符串，作为服务端密钥长期保留         |
| `PASSWORD_PEPPER` | Secret | 否，建议首次部署时设置 | 独立随机长字符串，用于密码哈希；设置后应长期保留     |
| `SITE_QUOTA_GB`   | 文本变量   | 否           | 站点应用容量上限，默认 `8`；可设置为正数       |

建议使用密码管理器分别生成至少 32 位随机字符串。不要将实际密钥提交到 GitHub。`DEPLOY_KEY` 由你自己设置，不是 Cloudflare API Token，也不是管理员登录密码。

**账号创建后不要随意更改这两个密钥。** 未显式设置 `PASSWORD_PEPPER` 时，程序会从 `DEPLOY_KEY` 派生密码 pepper；此时修改 `DEPLOY_KEY` 或补填不同的 `PASSWORD_PEPPER` 都会影响已有账号的密码验证。

`SITE_QUOTA_GB` 是程序限制，不代表 Cloudflare 套餐赠送的容量，也不会替你调整平台计费或配额。

### 5. 重新部署

保存绑定和变量后，回到项目部署页面，再次上传同一份部署包，创建新部署，使配置生效。

等待部署成功，打开 Cloudflare 分配的 HTTPS 地址：

```text
https://你的项目名.pages.dev/
```

### 6. 登录管理员

保存 `ADMIN_USERNAME` 和 `ADMIN_PASSWORD` 并重新部署后，打开网站。程序会自动创建管理员，无需在网页输入初始化密钥。使用这两个变量中的账号密码直接登录，然后导入一本 TXT 或 EPUB 检查阅读功能。

如需改名或重置管理员密码，在 Pages 修改对应变量并重新部署；下一次访问登录状态接口或登录时会应用配置，旧管理员会话随凭据变更失效。管理员用户 ID、书籍和阅读记录不变。如果用户名已被其他用户使用，会提示配置冲突，不会将该用户提升为管理员。

升级旧站点时配置这两个变量，会接管原始管理员账号；不配置则保留旧账号登录方式。新站点必须同时配置两项，网页初始化接口已关闭。

### 7. 创建其他用户

管理员登录后进入 **用户与独立空间 / 用户管理**，创建账号并设置配额。用户使用各自账号登录，每个账号拥有独立的书库和阅读记录。

不要让不同使用者共用同一个账号，否则他们会共享该账号的空间。

## 方式二：从 GitHub 自动部署

适合希望通过推送 GitHub 仓库更新网站的用户。新建 Pages 项目时选择 **连接 Git / Import an existing Git repository**，授权并选择自己的仓库。

推荐仓库组织：

```text
README.md
Cloud-Reader.v1.2.2/
  _worker.js
  _routes.json
  index.html
```

构建配置：

| 配置项    | 填写内容                  |
| ------ | --------------------- |
| 生产分支   | 实际使用的分支，例如 `main`     |
| 框架预设   | `None`                |
| 根目录    | 仓库根目录                 |
| 构建命令   | 留空                    |
| 构建输出目录 | `Cloud-Reader.v1.2.2` |

如果将三个部署文件直接放在仓库根目录，构建输出目录使用 `.`。

这里部署的是已经生成的发布文件，不需要在 Cloudflare 中执行开发工作区的补丁构建脚本。创建项目后，仍需按上文添加 `DB`、`BOOKS` 和 `DEPLOY_KEY`，然后重新部署并直接登录管理员。

之后推送生产分支即可触发部署。升级到新的版本目录时，记得同步修改构建输出目录。详情参见 [Cloudflare Git 集成说明](https://developers.cloudflare.com/pages/get-started/git-integration/)。

直接上传与 Git 集成应在创建项目时选定。不要假定已有直接上传项目可以直接切换为 Git 集成；如需改用新项目，应同时检查数据库、存储绑定和域名配置。

## 部署后检查

- 首页能进入登录页面。
- 管理员可登录、导入书籍并打开阅读。
- 刷新网页后，书籍仍存在。
- 第二个账号看不到第一个账号的书库。
- 访问 `/api/v1/version`，应看到 `version` 为 `cloud-reader-v1.2.2`。

版本接口仅确认部署版本，不能代替数据库、登录和上传功能检查。

## 可选：配置外部云服务

基础阅读功能不依赖外部网盘，可以先完成站点部署，再按需配置。

入口：**设置与数据 → 云服务**，或直接访问 `/cloud`。

| 服务                            | Pages 项目中需配置                                  | 接入说明                          |
| ----------------------------- | --------------------------------------------- | ----------------------------- |
| OneDrive                      | `ONEDRIVE_CLIENT_ID`、`ONEDRIVE_CLIENT_SECRET` | 注册 Microsoft OAuth 应用后授权      |
| Google Drive                  | `GOOGLE_CLIENT_ID`、`GOOGLE_CLIENT_SECRET`     | 启用 Drive API，创建 Web OAuth 客户端 |
| WebDAV / Nextcloud / ownCloud | `WEBDAV_ALLOWED_HOSTS`                        | 在网页填写 HTTPS WebDAV 地址与凭据      |
| 百度网盘 / 阿里云盘                   | `WEBDAV_ALLOWED_HOSTS`                        | 先准备挂载对应网盘的可写 WebDAV 服务        |

OAuth 客户端密钥应保存为 Secret，配置更改后重新部署。回调地址必须与实际站点域名完全一致：

```text
https://你的域名/cloud/oauth/onedrive/callback
https://你的域名/cloud/oauth/google/callback
```

OneDrive 使用 `Files.ReadWrite.AppFolder` 权限及离线授权；Google Drive 使用 `https://www.googleapis.com/auth/drive.appdata` 范围。Google 应用处于测试状态时，需要将使用者加入测试用户列表。

WebDAV 配置示例：

```text
WEBDAV_ALLOWED_HOSTS=dav.example.com,cloud.example.com
```

这里只填写允许访问的域名，不填协议、路径或通配符。网页中的目录地址则填写完整 URL，例如 `https://dav.example.com/CloudReader/`；预先创建此目录。当前支持标准 HTTPS 443 端口，请使用最终地址，服务不能依赖重定向。

百度网盘、阿里云盘当前没有原生 OAuth/API 适配器。页面需要的是 WebDAV 用户名和应用密码，不是网盘登录密码。

云备份包含书籍、书目和阅读记录；合并恢复导入缺失书籍并合并较新记录。备份绑定同一站点的用户 ID，不提供跨站点账号映射。自动备份仅在云服务页面打开且联网时每 15 分钟运行。

外部云服务已做模拟协议测试，仍需部署者使用自己的真实账号完成联调。

## 升级已有站点

1. 备份重要数据，并记录现有资源绑定和密钥配置。
2. 将新版部署文件上传到**现有 Pages 项目**，或更新 Git 集成项目的部署目录。
3. 保留原 D1、R2、`DEPLOY_KEY` 和 `PASSWORD_PEPPER`，不要重建数据库或执行旧版迁移 SQL。
4. 部署成功后刷新网页，并通过版本接口确认更新。

从 v1.2.0 / v1.2.1 升级到 v1.2.2 不需要清除浏览器缓存、删除书籍或重置阅读记录。

从需要离线口令的旧版升级时，已同步的数据登录后可重新载入。如果有未同步的旧本地数据，可在设置中的“导入旧版未同步的本地数据”中输入原口令完成一次迁移；原缓存会保留。

## 常见问题

### 页面提示“Pages 后端尚未运行”

检查部署根目录是否直接包含 `_worker.js`，是否误传了外层文件夹，以及是否创建了 Pages 项目。单独双击 `index.html` 或部署到 GitHub Pages 无法运行后端。

### 页面提示缺少 DB、BOOKS 或 DEPLOY_KEY

检查绑定名称、资源类型和生产环境变量是否正确，确认保存后已经重新部署。生产环境与预览环境的配置分别生效。

### 数据库初始化失败

确认 `DB` 绑定到可用的 D1 数据库。新部署使用空数据库即可；旧站点保留原数据库。查看 Cloudflare 部署及运行日志，不要通过清空数据库解决问题。

### 管理员账号或密码未配置

在 Pages 的生产环境设置 `ADMIN_USERNAME` 和 Secret `ADMIN_PASSWORD`，保存并重新部署。密码按原值使用，不会删除前后空格。`DEPLOY_KEY` 仍须保留，但不再是网页初始化凭据。

### TXT 打开白屏

确认部署的是 v1.2.2 或包含同一缓存修复的后续版本，然后刷新页面。v1.2.2 统一了整套静态资源的缓存地址，修复了旧模块混用导致的白屏。如果仍失败，请记录书籍格式、大小及浏览器报错以便排查。

### 关闭整本下载后，EPUB / PDF 仍然下载文件

这些格式需要完整文件才能解析。“打开时下载完整书籍”选项主要控制 TXT 的按需分段阅读。

### 离线时无法打开某本书

先联网登录并完整缓存该书。分段阅读过部分正文不代表整本书已经可离线使用。浏览器清理站点数据会影响本地缓存。

### 修改密钥后原密码无法登录

密码验证依赖创建账号时使用的 pepper。优先恢复原有密钥配置；不要删除数据库或重新初始化站点。

### 云服务无法连接

检查客户端 ID、Secret、OAuth 回调地址、测试用户列表，或 WebDAV 域名白名单、目录权限与最终 URL。修改 Pages 配置后重新部署。

## 数据存储与容量

账号和阅读元数据保存在 D1，书籍文件保存在私有 R2，本地缓存按账号保存在浏览器。站点配额默认 8 GB，单本上传上限 200 MB；实际可用容量和请求额度还受到 Cloudflare 平台限制。

管理员应同时备份 D1 和 R2：只有数据库或只有书籍文件，都不足以完整恢复站点。云备份不替代站点数据库备份。不要把真实密钥、个人书库或用户数据提交到公开仓库。
