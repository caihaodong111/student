# 迹刻 · 学生考勤与活动管理系统

面向学生组织、院系和社团的 Web 考勤系统，提供学号登录、打卡提交、管理员审批、活动发布和权限委托。采用 Vue 3 + Flask + MySQL，支持本地开发与 Docker 部署。

## 功能

- **学生端**：学号登录、提交打卡、查看记录与审批消息、催办申请、浏览活动、修改个人资料与密码、申请修改手机号。
- **管理员端**：按部门查看成员与记录，批量通过或驳回打卡，处理手机号变更，提交活动供审核。
- **超级管理员端**：审批管理员申请与活动，管理临时超级管理员授权及委托申请，通过开发者管理页面维护用户资料与角色。
- **会话与时间**：使用 Bearer Token 鉴权；会话有效期为 30 天，默认在上次登录超过 7 天后要求重新登录；业务时间统一为北京时间（UTC+8）。

公开注册已关闭，账号由维护者创建或导入。系统允许一天多次打卡，两次打卡至少间隔 10 秒；提交后进入待审批状态。

## 技术栈

| 层级 | 技术 |
| --- | --- |
| 前端 | Vue 3、Vite 7、Element Plus、Axios |
| 后端 | Python、Flask、Flask-CORS、Werkzeug |
| 数据库 | MySQL、PyMySQL、自建连接池 |
| 部署 | Docker Compose、Nginx、Gunicorn |

## 项目结构

```text
.
├── README.md
├── docker-compose.yml           # 前后端容器编排，不含 MySQL
├── punch_system/                # Flask 后端
│   ├── app.py                   # API、鉴权与业务逻辑
│   ├── db_env.py                # 环境配置与数据库连接池
│   ├── .env.example             # 后端配置示例，不含真实凭据
│   ├── setup_database.py        # 建库、建表与结构补齐
│   ├── db_migrations_minimal/    # 分表 SQL 脚本
│   ├── import_users_from_excel.py
│   ├── requirements.txt
│   └── Dockerfile
└── punch_system_vue/            # Vue 前端
    ├── src/components/          # 学生端与管理端界面
    ├── src/lib/                 # API 请求、活动数据及工具函数
    ├── nginx/default.conf       # 静态资源与 /api 反向代理
    ├── package.json
    └── Dockerfile
```

以下命令均以本 README 所在的仓库根目录为起点。终端示例适用于 macOS / Linux。

## 本地运行

### 1. 准备环境

- Python 3.11（与后端 Docker 镜像一致）
- Node.js 22.12 或更高版本、npm
- 可访问的 MySQL 8 数据库服务

### 2. 配置后端

```bash
cd punch_system
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cp .env.example .env
```

编辑 `.env`，填写 MySQL 地址、端口、用户名和密码。配置文件由程序自动读取，系统环境变量优先于文件中的同名配置。

### 3. 初始化数据库

在上述终端中运行：

```bash
python setup_database.py
```

初始化脚本会创建数据库及业务表，并尝试补齐已有表的字段。数据库账号需要相应的建库、建表、修改表结构和读写权限。请确认输出包含“数据库设置完成”；脚本捕获异常后可能仍以退出码 0 结束。

新安装优先使用该脚本。`db_migrations_minimal/` 提供独立 SQL，当前业务还依赖手机号申请、活动和权限委托等表，仅执行前四份 SQL 不足以覆盖完整功能。对已有数据库执行结构变更前请先备份。

### 4. 创建首个账号

初始化脚本不会创建默认账号。可在已激活虚拟环境的 `punch_system/` 目录运行下面的交互式示例，创建超级管理员；密码通过终端输入，不写入命令或源码。

```bash
python -c '
from getpass import getpass
from werkzeug.security import generate_password_hash
from db_env import get_db_connection

username = input("管理员用户名：").strip()
student_no = input("登录学号（唯一）：").strip()
department = input("所属部门：").strip()
password = getpass("初始密码（8–64 位，含大写、小写字母和数字）：")
if not username or not student_no:
    raise SystemExit("用户名和学号不能为空")
if not (8 <= len(password) <= 64 and any("A" <= c <= "Z" for c in password)
        and any("a" <= c <= "z" for c in password) and any("0" <= c <= "9" for c in password)):
    raise SystemExit("密码不符合要求")

conn = get_db_connection()
try:
    with conn.cursor() as cursor:
        cursor.execute(
            """INSERT INTO users
               (username, nickname, student_no, department, password, role, is_admin)
               VALUES (%s, %s, %s, %s, %s, %s, %s)""",
            (username, username, student_no, department,
             generate_password_hash(password), "super_admin", 1),
        )
    conn.commit()
    print("超级管理员已创建")
finally:
    conn.close()
'
```

学生端使用学号与密码登录；独立管理端登录接口使用用户名与密码。创建普通成员时，将示例中的角色改为 `user`、`is_admin` 改为 `0`；创建部门管理员时使用 `admin`、`1`，并填写正确的部门。

### 5. 启动后端

继续在 `punch_system/` 中运行：

```bash
python app.py
```

示例配置将服务启动在 `http://127.0.0.1:5001`。可在另一个终端检查：

```bash
curl http://127.0.0.1:5001/test
```

该接口仅确认 HTTP 服务可访问，不检查数据库连接。

### 6. 启动前端

另开终端，从仓库根目录运行：

```bash
cd punch_system_vue
npm ci
npm run dev
```

打开 `http://localhost:8080`。前端默认请求当前浏览器主机的 `5001` 端口，因此本地后端端口需与之保持一致。

如需连接其他后端，在 `punch_system_vue/.env.local` 中设置后重启 Vite：

```dotenv
VITE_API_BASE_URL=http://127.0.0.1:5001
```

本地直接连接 Flask 时不加 `/api` 前缀；Docker 部署通过 Nginx 转发，使用 `/api`。通过局域网访问时，请将 API 地址设为浏览器可访问的后端地址。

## Docker 部署

需要 Docker Engine / Docker Desktop、Compose，以及已启动的外部 MySQL。现有 Compose 使用本地镜像标签，没有 `build` 配置，也没有包含数据库服务。

1. 复制 `punch_system/.env.example` 为 `punch_system/.env`，填写数据库配置。
2. 将 `DB_HOST` 改为后端容器可访问的数据库地址。容器内的 `127.0.0.1` 指向容器自身；Docker Desktop 连接宿主机数据库时可使用 `host.docker.internal`。Linux 请按部署网络配置宿主机地址或主机映射。
3. 从仓库根目录构建镜像并启动：

```bash
docker build -t punch_system:0.1 ./punch_system
docker build -t punch_system_vue:0.1 ./punch_system_vue
docker compose up -d
docker compose logs -f backend
```

后端容器默认在启动时执行数据库初始化。确认初始化成功后，打开 `http://localhost:8080`；可用 `curl http://localhost:8080/api/test` 检查代理路径。

首次创建账号时，执行 `docker compose exec backend sh` 进入容器，再运行上文的 Python 账号创建示例。容器内已有依赖，无需创建虚拟环境。

前端镜像默认把 `VITE_API_BASE_URL` 编译为 `/api`，由 Nginx 将请求转发至 `backend:5001`。该变量在构建时生效，修改后需要重新构建镜像。默认活动种子来自前端源码；后端独立镜像不包含该目录，首次部署可能没有预置活动，可在管理端创建并审批。

常用运维命令：

```bash
docker compose ps
docker compose logs --tail=100 backend
docker compose down
```

## 配置参考

### 后端

| 变量 | 代码默认值 | 说明 |
| --- | --- | --- |
| `DB_HOST` | `127.0.0.1` | MySQL 地址 |
| `DB_PORT` | `3306` | MySQL 端口 |
| `DB_USER` | `root` | 数据库用户名，部署时使用专用账号 |
| `DB_PASSWORD` | 空 | 数据库密码 |
| `DB_NAME` | `student` | 数据库名 |
| `DB_CHARSET` | `utf8mb4` | 连接字符集 |
| `DB_POOL_MAX_CONNECTIONS` | `100` | 每个进程的连接池上限 |
| `DB_POOL_MIN_CACHED` | `3` | 每个进程预建连接数 |
| `DB_POOL_ACQUIRE_TIMEOUT` | `10` | 等待连接的秒数 |
| `PORT` | `5000` | 直接运行 `app.py` 时的端口；示例与 Compose 使用 `5001` |
| `FLASK_HOST` | `0.0.0.0` | 开发服务监听地址 |
| `FLASK_DEBUG` | `1` | 开发调试开关；示例与 Compose 均设为 `0` |
| `RELOGIN_AFTER_DAYS` | `7` | 超过上次登录多少天后要求重新登录 |
| `APP_LOG_LEVEL` | `INFO` | 应用日志级别 |
| `REQUEST_LOG_ENABLED` | `1` | 是否输出请求日志 |
| `RUN_DB_SETUP` | `1` | 容器启动时是否初始化数据库 |
| `GUNICORN_WORKERS` | `2` | 容器中的 Gunicorn 工作进程数 |
| `GUNICORN_THREADS` | `8` | 每个工作进程的线程数 |
| `GUNICORN_TIMEOUT` | `60` | Gunicorn 请求超时秒数 |

数据库会话时区由程序设为 `+08:00`。连接池按进程独立创建，调整 Gunicorn 进程数时需同时考虑数据库连接总数。

### 前端

| 变量 | 说明 |
| --- | --- |
| `VITE_API_BASE_URL` | 显式 API 基础地址，优先使用；Docker 构建默认 `/api` |
| `VITE_API_PORT` | 未设置基础地址时的后端端口，默认 `5001` |

构建前端：

```bash
cd punch_system_vue
npm run build
```

产物位于 `punch_system_vue/dist/`。直接部署静态文件时需同时配置 API 地址或反向代理。

## Excel 导入

`punch_system/import_users_from_excel.py` 支持从工作簿的当前活动工作表导入账号，第一行为表头，后续六列依次为：

| 姓名 | 班级 | 学号 | 部门 | 手机号 | 职级 |
| --- | --- | --- | --- | --- | --- |
| 示例同学 | 示例班级 | 202600000001 | 示例部门 | 13800000000 | 部员 |

职级映射为：`部员 → user`、`部长 → admin`、`主席 → super_admin`。姓名作为唯一用户名，学号也必须唯一；所有六个字段都需填写。

**该脚本会先清空用户、会话、打卡、活动及各类申请、授权表，再执行导入。仅用于已备份并明确需要重置的数据库。** 脚本还包含原项目的学号修正规则，复用前需按自己的学号体系调整。

在后端虚拟环境中额外安装 `openpyxl`（当前未包含在 `requirements.txt` 中），再执行：

```bash
cd punch_system
python -m pip install openpyxl
python import_users_from_excel.py /path/to/users.xlsx --default-password 'ReplaceWithYourOwn123'
```

请替换示例密码并在首次登录后修改。导入脚本会向终端输出统一初始密码，勿将相关日志公开。

## 常见问题

- **前端无法连接后端**：检查后端是否监听 `5001`、`VITE_API_BASE_URL` 是否正确，以及直连 Flask 时是否误加了 `/api`；Docker 部署检查 `backend` 容器日志。
- **服务正常但登录失败**：`/test` 不校验数据库。确认初始化成功且已创建账号；学生端填学号，独立管理端填用户名。
- **提示需要重新登录**：除 Token 到期外，系统默认在上次登录超过 7 天后清理会话，可通过 `RELOGIN_AFTER_DAYS` 调整。
- **管理员看不到其他部门记录**：普通管理员受部门范围限制，请检查账号及成员的 `department` 是否一致。
- **MySQL 认证提示缺少 `cryptography`**：在运行后端的同一 Python 环境中重新安装 `requirements.txt` 中的依赖。

## 开源发布说明

当前仓库尚未包含 `LICENSE`，发布前请由维护者选定许可证并补充许可证文件。

现有代码中还需处理以下发布事项：

- `import_users_from_excel.py` 的 `STUDENT_NO_OVERRIDES` 含原项目个人信息，应移除或匿名化；真实成员表格、数据库备份和 `.env` 不应公开。
- `punch_system_vue/nginx/default.conf` 中的 `server_name` 包含原部署 IP，应改为自己的域名或通用配置。
- 前端 `src/lib/api.js` 中部分调试日志会输出 Token、请求头和用户数据，正式部署前应移除；后端当前允许所有来源跨域，应按实际部署域名收紧。
- `.gitignore` 已排除 `.env`、`.env.*`（保留 `.env.example`）、虚拟环境及镜像归档；Docker 构建上下文也排除了实际环境文件和归档。发布时请确认这些忽略规则仍然生效。

## 参与贡献

欢迎通过 Issue 描述问题或提交 Pull Request。反馈时请提供复现步骤、运行环境及脱敏后的日志；涉及页面改动可附截图。修改前端后运行 `npm run build`，涉及后端或数据库时请在独立测试数据库验证登录、权限和相关业务流程。
