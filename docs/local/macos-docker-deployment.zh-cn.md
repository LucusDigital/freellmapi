# macOS 本机长期 API 部署

核验日期：2026-10-15。

## 已部署结果

- 源码同步至 upstream/main `5a3abac8b4286cdfdc6dd53f1b99290cc804be2d`，交付分支 `codex/freellmapi-deploy-20261015`。
- Docker 使用官方稳定版 0.14.2 ARM64 镜像，摘要固定在 `compose.local.yaml`；官方镜像源码修订为 `57c0669233bbe0c70a0f9f9927613192be948773`。源码分支比稳定发布更新，部署未声称使用 main 构建。
- 仪表盘：http://127.0.0.1:3001/；OpenAI 兼容 Base URL：http://127.0.0.1:3001/v1。
- 仅绑定本机，重启策略 `unless-stopped`，持久化卷 `freellmapi_freellmapi-data`。
- `.env` 为本机私有配置，权限 0600，已生成加密密钥，未提交 Git。升级时保留此文件及数据卷；仅备份数据库而丢失加密密钥无法恢复提供方密钥。
- Docker Desktop 原已安装且可用（引擎 28.3.3）。登录启动由 `~/Library/LaunchAgents/local.freellmapi.docker-start.plist` 负责，打开 Docker 后容器自动恢复。未执行整机重启验收。
- 已安装桌面版 `/Applications/FreeLLMAPI.app` 从 0.13.6 升至 0.14.2，官方 ZIP SHA-256 为 `7233bca052f76e5297d1b31bb586fc7bd07a0cbaf1febcafff36bcb24935f521`，与官方资产摘要一致，安装后签名及 Apple 公证检查通过。
- 桌面版 API 在 http://127.0.0.1:31415/v1，数据位于 `~/Library/Application Support/FreeLLMAPI`，与 Docker 数据独立。长期客户端请使用 3001；关闭桌面应用不影响 Docker。
- 旧应用备份：`~/Developer/agent-home/scratch/codex/freellmapi/backups/2026-10-15-before-upgrade/FreeLLMAPI-0.13.6.app`。升级前未发现桌面版用户数据目录。

## 首次使用（由用户完成）

1. 打开 http://127.0.0.1:3001/，设置管理账号邮箱和密码。
2. 如要求首次设置码，在本机终端执行以下命令查看，勿发到聊天或公开报告：
   ```sh
   docker logs freellmapi-freellmapi-1 2>&1 | sed -n 's/.*First-run setup code: //p' | tail -1
   ```
   Docker 的连接可能被识别为桥接地址；设置码每次服务启动会变化。
3. 登录后在密钥页添加自己的提供方 API 密钥，再复制统一 API 密钥给客户端。
4. 客户端配置 Base URL 为 `http://127.0.0.1:3001/v1`，模型可选 `auto`。

未创建用户账号、未导入提供方密钥、未发起真实推理或付费请求。服务可用性不代表模型推理已验收。

## 日常运维

在仓库根目录执行。始终带两个 Compose 文件，保持固定版本：

```sh
# 状态
docker compose -f docker-compose.yml -f compose.local.yaml ps
# 启动／恢复
docker compose -f docker-compose.yml -f compose.local.yaml up -d --no-build
# 重启
docker compose -f docker-compose.yml -f compose.local.yaml restart freellmapi
# 停止（主动停止后不会自动恢复，需执行上面的启动命令）
docker compose -f docker-compose.yml -f compose.local.yaml stop freellmapi
# 健康检查
curl --noproxy '*' -fsS http://127.0.0.1:3001/api/ping
```

不使用 `down -v`，该参数会删除数据卷。镜像已固定摘要，更新时先备份数据和 `.env`，再核对新官方版本并修改摘要；未启用 Watchtower。

Docker Desktop 需保持运行。关闭其窗口不等于退出 Docker；退出 Docker 会停止后台 API。登录启动不等于开机未登录即可运行；Mac 休眠或关机会中断服务。

## 本次验收

- 浏览器实际显示中文首次创建账号页面，容器重启后页面仍能加载。
- 容器重启前后 SQLite 数据文件 inode 相同，SQLite `quick_check` 为 `ok`，33 张表。
- Docker 和桌面版 `/api/ping` 均返回 HTTP 200。
- 未认证的 `/v1/models` 均返回 HTTP 401，认证保护存在。
- Docker 为 `healthy`，持久化卷及 `127.0.0.1:3001` 绑定符合配置。
- 登录启动项已加载且执行退出码 0；整机重启后恢复未实测。
