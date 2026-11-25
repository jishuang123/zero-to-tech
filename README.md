# zero-to-tech

一个学习全栈开发的练习项目，使用 Next.js、React 和 FastAPI，包含个人主页与中文文字实验室。分析结果通过 SQLite 保存，并用 Cookie 区分不同浏览器会话的历史记录。

## 已实现的功能

- 个人主页从后端获取简介和作品信息，请求失败时使用本地默认内容。
- 输入中文，生成带声调的拼音。
- 使用 SnowNLP 计算情感分数，并展示情感判断。
- 自动保存分析结果，点击“历史记录”查看当前浏览器会话最近的分析。
- 历史弹窗支持关闭按钮、Esc 键和点击遮罩关闭，记录时间按浏览器本地时区显示。

情感分数范围为 `0～1`，越接近 `1` 表示越积极。当前代码将分数保留两位小数后分类，`>= 0.6` 为“偏积极”，`<= 0.4` 为“偏消极”，其余为“中性”。结果取决于 SnowNLP 的模型和输入内容。

## 技术栈

| 部分 | 使用的技术 |
| --- | --- |
| 前端 | Next.js 15、React 19、CSS、Anime.js |
| 后端 | Python、FastAPI、Uvicorn、Pydantic |
| 中文处理 | pypinyin、SnowNLP |
| 数据存储 | SQLite |
| 本地配置 | 前端 `.env.local`、后端 `.env` |

## 本地运行

### 1. 准备环境并获取项目

需要安装 Git、Node.js 和 Python。项目当前使用 Node.js 24、Python 3.11 进行开发。

```bash
git clone --depth 1 https://github.com/jishuang123/zero-to-tech.git
cd zero-to-tech
```

这里使用浅克隆获取最新提交，减少下载量。需要完整提交历史时，可在项目目录执行 `git fetch --unshallow`。

### 2. 配置环境变量

仓库中的 `.env.example` 和 `backend/.env.example` 提供字段示意，其中的地址占位内容需要替换。请在本地创建以下两个文件。

项目根目录的 `.env.local` 填写以下内容。

```dotenv
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

`backend/.env` 填写以下内容。

```dotenv
ALLOWED_ORIGINS=http://localhost:3000
```

| 变量 | 用途 |
| --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | 浏览器请求后端接口时使用的基础地址，末尾不加 `/` |
| `ALLOWED_ORIGINS` | 后端允许访问接口的前端来源，包含协议、域名和端口 |

多个前端来源用英文逗号分隔，逗号前后不要加空格，例如 `http://localhost:3000,http://127.0.0.1:3000`。后端当前直接读取这个变量，启动前必须配置。

修改前端环境变量后，需要重启开发服务。以 `NEXT_PUBLIC_` 开头的变量会提供给浏览器，应只放可公开的配置。生产构建会将其写入前端文件，修改地址后需要重新构建。[Next.js 环境变量说明](https://nextjs.org/docs/15/app/guides/environment-variables)

### 3. 启动后端

从项目根目录进入 `backend`，创建虚拟环境、安装依赖并启动开发服务。以下命令直接使用虚拟环境中的 Python，无需先激活环境。

Windows PowerShell 下执行以下命令。

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Linux / macOS 下执行以下命令，需先安装 Python 3.11 及其虚拟环境支持。

```bash
cd backend
python3.11 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

启动后可访问 [接口文档](http://127.0.0.1:8000/docs)。保持这个终端运行，按 `Ctrl+C` 停止服务。

`--reload` 用于本地开发，代码变更时会自动重启后端。请始终在 `backend` 目录启动，数据库会保存为该目录下的 `history.db`。

### 4. 启动前端

另开一个终端，在项目根目录执行以下命令。

```bash
npm ci
npm run dev
```

打开 [个人主页](http://localhost:3000) 或 [文字实验室](http://localhost:3000/text-lab)。本地配置使用 `localhost:3000` 作为前端来源，浏览器地址与后端允许的来源应保持一致。

输入一句中文，点击“开始分析”，看到拼音和情感结果后，再点击“历史记录”查看保存的内容。前后端各保留一套运行中的服务即可。

## 后端接口

| 方法 | 路径 | 功能 |
| --- | --- | --- |
| `GET` | `/api/profile` | 获取个人主页数据 |
| `POST` | `/api/analyze` | 分析文本并保存记录 |
| `GET` | `/api/history?limit=10` | 获取当前会话最近的历史记录，默认 10 条 |

`POST /api/analyze` 使用 JSON 请求体。

```json
{
  "text": "今天很开心"
}
```

响应包含 `text`、`pinyin`、`score`、`label` 和 `created_at`。时间以 UTC 保存，前端展示时转换为本地时间。

分析和历史接口会读取或设置名为 `session_id` 的 Cookie，有效期为 30 天。前端使用 `credentials: "include"` 携带 Cookie，后端通过 `allow_credentials=True` 配合指定的 CORS 来源。

历史记录跟随当前浏览器的 Cookie。清除 Cookie、切换浏览器或使用新的无痕会话后，会以新的会话查询历史。当前项目提供匿名会话历史，没有跨设备账号同步。

## 项目结构

```text
zero-to-tech/
├── app/
│   ├── layout.jsx            # 全站布局和样式入口
│   ├── page.jsx              # 个人主页 /
│   └── text-lab/page.jsx     # 文字实验室 /text-lab
├── components/
│   ├── HomeView.jsx          # 主页数据加载和展示
│   ├── TextLabView.jsx       # 分析结果、历史记录和弹窗状态
│   ├── InputCard.jsx         # 文本输入和分析请求
│   ├── ResultCard.jsx        # 分析结果展示
│   └── HistoryModal.jsx      # 历史记录弹窗
├── backend/
│   ├── main.py               # FastAPI 路由、CORS、会话和文本分析
│   ├── storage.py            # SQLite 初始化、写入和查询
│   ├── requirements.txt      # Python 依赖
│   ├── .env.example          # 后端配置字段示例
│   └── handmade.py           # 早期手写 HTTP 服务练习
├── css/                      # 页面与组件样式
├── data/site.js              # 页面默认内容
├── .env.example              # 前端配置字段示例
├── next.config.mjs           # Next.js 静态导出配置
├── package.json
└── package-lock.json
```

`.venv`、`.env.local`、`backend/.env`、`history.db` 和构建产物由各自运行环境生成。

## 部署到服务器

前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx。

**1. 拉取代码**

```bash
cd ~/zero-to-tech
git pull
```

**2. 前端：装依赖、写配置、构建**

```bash
npm install
cp .env.example .env.production   # 按下面「配置说明」填好（线上留空）
npm run build                     # 产物进 out/，由 Nginx 提供服务
```

**3. 后端：建环境、装依赖、写配置**

```bash
cd backend
python3 -m venv --prompt=zero-to-tech .venv   # 首次部署才需要
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # 按下面「配置说明」填好
```

**4. 后端：委托给 systemd**（首次部署才需要）

新建 `/etc/systemd/system/zero-backend.service`：

```ini
[Unit]
Description=zero-to-tech backend

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/zero-to-tech/backend
ExecStart=/home/ubuntu/zero-to-tech/backend/.venv/bin/fastapi run --host 127.0.0.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start zero-backend     # 启动
sudo systemctl enable zero-backend    # 开机自启
sudo systemctl status zero-backend    # 看到 active (running) 就对了
```

**5. Nginx：静态文件 ＋ `/api/` 反向代理**（首次部署才需要）

编辑 `/etc/nginx/sites-enabled/default`：

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /home/ubuntu/zero-to-tech/out;
    index index.html;

    location / {
        try_files $uri $uri.html $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;     # 末尾不要加斜杠

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

**6. 安全组 / 防火墙**

只放行 **22（SSH）** 和 **80（HTTP）**。**8000 不要放行**——后端只监听 `127.0.0.1`，由 Nginx 转发进去。

**7. 验证**

```bash
curl localhost/api/profile        # 服务器本机，注意不带端口
```

浏览器访问 `http://服务器IP`，做一次分析、看一眼历史记录。换个无痕窗口再试，两边的历史应该互相看不到。

## 日常发布

```bash
cd ~/zero-to-tech
git pull

npm run build                          # 前端改了才需要
sudo systemctl restart zero-backend    # 后端改了才需要
```

## 常用运维命令

```bash
sudo systemctl status zero-backend     # 看状态
sudo systemctl restart zero-backend    # 重启
sudo systemctl stop zero-backend       # 停掉
journalctl -u zero-backend -f          # 实时看日志（Ctrl+C 退出）
journalctl -u zero-backend -n 50       # 看最后 50 行

sudo nginx -t                          # 改完 Nginx 配置先验语法
sudo systemctl reload nginx            # 再让它重读
```

## 提交到 GitHub

提交源代码、`package-lock.json`、`backend/requirements.txt` 和两份 `.env.example`。本地环境文件、虚拟环境、数据库与构建产物已有忽略规则。

提交前使用 `git status` 和 `git diff --cached` 核对暂存内容。配置示例中只保留字段和示意值，实际密码、密钥应放在各环境的私有配置中。
