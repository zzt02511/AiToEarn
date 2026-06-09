# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AiToEarn — OPC（一人公司）的 AI 内容营销智能体平台。支持内容创作、多平台发布、互动运营、内容变现四大核心能力。

支持渠道：抖音、小红书、快手、B站、视频号、TikTok、YouTube、Facebook、Instagram、Threads、Twitter/X、Pinterest、LinkedIn

## Repository Layout

```
aitoearn/
├── project/
│   ├── aitoearn-backend/     # Nx + NestJS 后端（pnpm monorepo）
│   ├── aitoearn-web/         # Next.js 14 前端（pnpm）
│   └── aitoearn-electron/    # Electron 桌面端（npm）
├── docker-compose.yml        # 一键部署
├── nginx/                    # Nginx 配置
├── scripts/                  # 部署脚本
├── presentation/             # 展示素材
├── AGENTS.md                 # AI 工作规则（必须先读）
├── CONTRIBUTING.md
└── DOCKER_DEPLOYMENT_CN.md / DOCKER_DEPLOYMENT_EN.md
```

**重要：根目录没有统一的 package.json，不要在此执行 install/build。**

## Development Commands

### Backend (project/aitoearn-backend)

Nx workspace with two NestJS apps and shared libs.

```bash
cd project/aitoearn-backend
# 安装依赖
pnpm install
# 配置本地开发
cp apps/aitoearn-ai/config/config.js apps/aitoearn-ai/config/local.config.js
cp apps/aitoearn-server/config/config.js apps/aitoearn-server/config/local.config.js
# 启动服务（分别开两个终端）
pnpm nx serve aitoearn-ai       # AI 服务
pnpm nx serve aitoearn-server   # 主服务
# 构建
pnpm nx build <project>
pnpm nx run-many --target=build --all
# 测试
pnpm nx run <project>:test
# Lint
pnpm nx run-many --target=lint
```

**后端子项目结构**：
- `apps/aitoearn-server/` — 主服务（内容发布、账号管理、交易市场等）
- `apps/aitoearn-ai/` — AI 服务（Agent 编排、内容生成）
- `libs/` — 共享库（common、mongodb、redis、auth、queue 等 18 个）

### Web Frontend (project/aitoearn-web)

```bash
cd project/aitoearn-web
pnpm install
pnpm run dev              # 开发（端口 6061）
pnpm run dev:local        # 连接本地后端
# 类型检查（不要用 npm run build 做类型检查）
pnpm run type-check       # tsc --noEmit
# 构建
pnpm run build
# Lint
pnpm run lint
# E2E 测试（Playwright）
pnpm run test:agent       # 运行 Agent 测试
pnpm run test:agent:quick # 快速单提示词测试
```

### Electron Desktop (project/aitoearn-electron)

```bash
cd project/aitoearn-electron
npm install
npm run rebuild           # 编译 sqlite
npm run dev               # 启动开发
```

### Docker Deployment

```bash
docker compose up -d      # 一键部署（含 MongoDB、Redis）
# 配置 Relay（在 docker-compose.yml 中设置环境变量）
RELAY_SERVER_URL: https://aitoearn.ai/api
RELAY_API_KEY: 你的API-Key
RELAY_CALLBACK_URL: http://localhost:8080/api/plat/relay-callback
```

## Backend Architecture (project/aitoearn-backend)

### Technology Stack
- **Runtime**: Node.js 20.18+, pnpm 10, Nx 22
- **Framework**: NestJS 11, Express
- **Database**: MongoDB (Mongoose), Redis
- **Validation**: Zod 4 (DTO schema → `createZodDto`)
- **Testing**: Vitest, coverage v8

### Layered Architecture
```
Controller (路由/参数绑定/VO 转换) → Service (业务编排/权限过滤) → Repository (数据访问)
```
- Controller 只处理路由和 VO 转换，不能注入 Repository
- Service 处理业务逻辑和权限过滤
- Repository 只做数据访问，不含业务逻辑

### Key Patterns
- **DTO**: Zod schema → `createZodDto(schema, 'Name')`，分页用 `PaginationDtoSchema`
- **VO**: Zod schema → `createZodDto` / `createPaginationVo`，Controller 中转换输出
- **Error Handling**: 业务错误用 `AppException + ResponseCode`，协议错误用 `HttpException`
- **Response Format**: 统一 `{ data, code: number, message }` 格式
- **Repository Methods**: 命名必须 `getBy`/`listBy`/`create`/`update`/`delete`/`count` 前缀

### Critical Rules
- 禁止 mutation，必须使用 immutable 模式
- 禁止 `console`，使用注入的 Logger 实例
- 禁止在业务逻辑中用 try-catch，由框架统一处理
- 禁止 `as any` 绕过类型检查
- 过滤/聚合必须在数据库层完成，禁止在应用层 forEach/filter/reduce
- 方法重命名后必须 `pnpm nx build` 验证所有引用

## Web Frontend Architecture (project/aitoearn-web)

### Technology Stack
- **Framework**: Next.js 14 (App Router)，React 18
- **Styling**: Tailwind CSS v4 + shadcn/ui + SCSS Modules
- **State**: Zustand 5 (全局 + 局部 store)
- **i18n**: i18next 24 + react-i18next 15 + locize
- **Testing**: Playwright (E2E)
- **Components**: shadcn/ui + Radix UI + Ant Design 5

### Directory Structure
```
src/
├── app/
│   ├── [lng]/             # 国际化路由
│   │   ├── accounts/      # 账号管理
│   │   ├── agent-assets/  # Agent 素材
│   │   ├── ai-social/     # AI 社交
│   │   ├── auth/          # 认证
│   │   ├── chat/          # 聊天/Agent 交互
│   │   ├── draft-box/     # 草稿箱
│   │   ├── tasks-history/ # 任务历史
│   │   └── websit/        # 网站管理
│   ├── api/               # API 请求方法
│   ├── components/        # 公共组件
│   ├── hooks/             # 自定义 hooks
│   ├── lib/               # 工具库（i18n, store, utils）
│   ├── store/             # 全局状态管理
│   ├── types/             # TypeScript 类型
│   ├── utils/             # 工具函数
│   ├── config/            # 业务常量（平台配置、发布类型等）
│   └── middleware.ts      # 路由守卫
```

### Key Patterns
- **API**: 方法在 `src/api/`，类型在 `src/api/types/`，响应必须检查 `code === 0`
- **Store**: Zustand + `combine`，多属性取值必须用 `useShallow`
- **Page State**: 页面级状态用局部 zustand store，不用 props 层层传递
- **Components**: 每个组件独立文件夹，`index.tsx` 导出
- **i18n**: 所有文本国际化，新增 key 需同步翻译文件
- **Currency**: 用 `appCurrencySymbol` / `appCurrency`（`src/utils/currency.ts`），禁止硬编码
- **Number Input**: 用 `NumberInput` 组件，禁止 `<Input type="number">`
- **Routing**: 使用不带语言前缀的 `<Link>`，middleware 自动处理
- **New Code**: 写前先查 `src/utils/README.md` + `src/lib/README.md` + `src/hooks/` + `src/components/README.md`

### Critical Rules
- 必须先检查已有实现再写新代码（见 `src/utils/README.md` 等 README 文件）
- 禁止 `as any`，必须修复源头类型
- 禁止硬编码颜色，使用 shadcn/ui 语义化变量
- 按钮 loading: `Loader2` 图标 + `disabled`，不改变文字
- Tailwind v4: 使用 `bg-(--xxx)` 语法而非 `bg-[var(--xxx)]`

## Environment Rules

- **中国版 (cn)**: `aitoearn.cn` 域名，cn API Key
- **国际版 (ai)**: `aitoearn.ai` 域名，ai API Key
- **MCP**: CMCP `/api/unified/mcp`，SSE `/api/unified/sse`
- **Relay**: `RELAY_API_KEY` 来源决定 `RELAY_SERVER_URL`
- 环境与 Key 不匹配导致 401

## Multi-Project Sync Rules

- 文档类改动（README、DOCKER_DEPLOYMENT）：默认三语同步
- 用户可见能力/安装/MCP/API Key 相关：检查中/英/日 README
- Docker 部署说明：同步检查中/英文两版
- Electron 项目使用 npm（非 pnpm），注意区分
