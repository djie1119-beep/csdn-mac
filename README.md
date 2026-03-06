# CSDN Unlocker

基于 FastAPI + Playwright 的自助解锁平台。  
支持 Token 配额、历史记录、后台批量导出链接。

## 1. 目录结构

```text
csdn-unlocker/
├── main.py                # FastAPI 入口（API + admin 后台）
├── unlocker.py            # Playwright 抓取与内容清理
├── auth.py                # Token 校验/扣减逻辑
├── token_service.py       # Token 持久化（create/disable/delete）
├── requirements.txt       # 依赖清单
└── cache/
    ├── tokens.json        # Token 数据
    ├── meta.json          # 历史记录
    └── html/              # 解锁后的 HTML 缓存
```

## 2. 环境要求

- Python 3.10+
- Google Chrome
- macOS / Linux

## 3. 安装依赖

```bash
cd csdn-unlocker
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m playwright install
```

## 4. 启动步骤

> 使用仓库根目录的 `./start.sh` 启动时，默认开启“改代码自动重启 uvicorn”（后台 watcher 监听 `csdn-unlocker/*.py`）。
> 如需关闭自动重启：`UVICORN_AUTO_RESTART=0 ./start.sh`
> 如需改成 uvicorn 自带热更新：`UVICORN_RELOAD=1 ./start.sh`

### 4.1 启动 Chrome 调试模式

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --user-data-dir="$HOME/CSDN/csdn-profile" \
  --remote-debugging-port=9222
```

> 保持这个 Chrome 窗口开启。

### 4.2 启动后端

```bash
cd csdn-unlocker
uvicorn main:app --host 127.0.0.1 --port 8000
```

- API 文档：`http://127.0.0.1:8000/docs`
- 管理后台：`http://127.0.0.1:8000/admin`

### 4.3 本地打开前端页面（开发调试）

在仓库根目录执行：

```bash
./start-client.sh
```

Windows PowerShell 可执行：

```powershell
.\start-client.ps1
```

默认访问地址：`http://127.0.0.1:8080/index.html`

说明：

- 当前端运行在 `localhost/127.0.0.1` 的非 `8000` 端口时，会自动请求本地后端 `http://127.0.0.1:8000/api`
- 停止服务可执行：`./stop.sh`（已包含 frontend 进程）

## 5. 常用接口

### 5.1 解锁文章

`POST /api/unlock`

Header:

- `X-Token: <token>`

Body:

```json
{"url": "https://blog.csdn.net/xxx/article/details/xxxx"}
```

示例：

```bash
curl -X POST http://127.0.0.1:8000/api/unlock \
  -H "X-Token: <token>" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://blog.csdn.net/xxx/article/details/xxxx"}'
```

成功返回：

```json
{"message":"解锁成功"}
```

### 5.4 下载资源（真实浏览器点击）

当链接属于 `download.csdn.net`，或文库页面检测到“VIP下载/下载”按钮时，后端会通过 Playwright 连接你已登录的 Chrome（CDP 9222）自动点击触发下载，等待下载完成后把文件移动到 `csdn-unlocker/cache/files/`。

`POST /api/unlock` 成功返回示例：

```json
{
  "message": "下载完成",
  "kind": "file",
  "filename": "<uuid>__xxx.zip",
  "download_url": "/api/files/<uuid>__xxx.zip?token=..."
}
```

下载文件：`GET /api/files/{name}?token=...`

可选环境变量：

- `CSDN_DOWNLOAD_DIR`：指定 Chrome 下载目录（默认 `~/Downloads`）。

### 5.2 查询额度

`GET /api/token_info`

- 支持 `X-Token` header 或 `?token=`
- 返回：`expired_at`、`daily_limit`、`remain`

### 5.3 查询当前 token 历史

`GET /api/history`

Header:

- `X-Token: <token>`

## 6. 管理后台功能

后台不需要 `admin_key`，直接可用：

- 单个生成 token（天数 + 每日次数）
- 禁用/删除 token（同页操作，不跳转）
- 查看全历史
- 批量导出 token 链接（按日/周/月/年 + 数量）
  - 导出前缀固定为：`http://download.dingjie.site/?code=`
  - 支持一键下载和一键复制

## 7. Token 规则

- 只有 `enabled = false` 才视为禁用
- 每日次数在解锁成功后才扣减（失败不扣）
- 超限返回：`429 Daily unlock limit reached`

## 8. 部署建议

- 本地后端监听 `127.0.0.1:8000`
- 公网可通过 Nginx/NPS 转发
- 建议忽略缓存 HTML：

```gitignore
csdn-unlocker/cache/html/
```

## 9. 快速排查

- `403 Invalid token`：token 错误或不存在
- `403 Token disabled`：token 被禁用
- `403 Token expired`：token 已过期
- `429 Daily unlock limit reached`：当日配额已用完

## 10. 免责声明

仅供学习与技术交流，请遵守目标站点规则与相关法律法规。
