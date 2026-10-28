# 迹刻前端

学生考勤与活动管理系统的 Vue 3 前端，使用 Vite、Element Plus 和 Axios。

完整功能介绍、数据库初始化、账号创建及 Docker 部署步骤见[项目 README](../README.md)。

## 本地开发

需要 Node.js 22.12 或更高版本。先按项目 README 启动后端，再在本目录执行：

```bash
npm ci
npm run dev
```

页面默认位于 `http://localhost:8080`，API 默认请求当前浏览器主机的 `5001` 端口。

如需指定后端，在本目录创建 `.env.local`，然后重启开发服务器：

```dotenv
VITE_API_BASE_URL=http://127.0.0.1:5001
```

## 构建

```bash
npm run build
```

产物输出到 `dist/`。`VITE_API_BASE_URL` 在构建时生效；Docker 镜像默认使用 `/api`，由 Nginx 代理到后端。
