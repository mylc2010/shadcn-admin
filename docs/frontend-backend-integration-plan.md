# shadcn-admin × MeetWise-server 前后端接入执行计划

> 交付形式：**执行计划文档**（不含实际接入代码）。
> 目标读者：前端 / 后端 / 联调负责人。
> 结论定位：本计划为「决策完备（decision-complete）」——实施者按本文即可落地，无需再做架构性决策。

---

## 1. 目标与范围

将当前 **纯前端 mock** 的 `shadcn-admin`（React 19 + Vite 8 + TanStack Router/Query/Table + Zustand + react-hook-form/zod）接入 **MeetWise-server**（Spring Boot 3.5 + Sa-Token + MyBatis-Plus，RuoYi-Vue-Plus 衍生架构），实现：

1. 真实登录 / 登出 / 会话保持（替换 `sleep + mock token`）。
2. 统一 HTTP 客户端与响应解包（`R<T>` / `TableDataInfo<T>`）。
3. 鉴权路由守卫与用户信息 / 权限 / 角色加载。
4. 数据表格（users / tasks 等）由后端分页接口驱动。
5. 错误处理、Token 失效重定向、验证码、多租户、`clientid` 头等契约对齐。
6. 国际化（已完成）在错误提示、后端消息展示上的延续。

**不在本次范围**：具体业务模块（meeting / guest / workflow 等）的深度对接，仅在 §9 给出扩展范式。

---

## 2. 现状盘点（前端）

| 关注点 | 现状 | 文件 |
|---|---|---|
| 登录逻辑 | `toast.promise(sleep(2000))` 后写入 mock user + `'mock-access-token'` | `src/features/auth/sign-in/components/user-auth-form.tsx` |
| 会话存储 | Zustand + cookie，键 `thisisjustarandomstring` | `src/stores/auth-store.ts` |
| HTTP 客户端 | **无**集中式实例；`axios` 已在依赖中 | `src/lib/handle-server-error.ts`（仅用到 `AxiosError`） |
| 服务端错误处理 | 读取 `error.response.data.title`（**与后端 `R.msg` 不匹配**） | `src/lib/handle-server-error.ts` |
| 环境变量 | 仅 `VITE_CLERK_PUBLISHABLE_KEY`（Clerk 未实际使用） | `.env.example` |
| 数据表格 | 本地静态数组（`data/*.ts`），无服务端分页 | `src/features/{tasks,users}/data/*` |
| 路由守卫 | 已有 `_authenticated` 布局，基于 auth-store token 判断 | `src/routes/_authenticated/*` |
| 数据请求 | 已依赖 TanStack Query，但当前多为 mock | 全局 |

> **关键差距**：无 axios 实例、无 API 层、错误解包字段不对、登录为 mock、表格非服务端分页。

---

## 3. 后端 API 契约（已核对源码）

### 3.1 基础约定
- **Base URL**：`http://<host>:6039/`（`server.port: 6039`，`context-path: /`）。
- **统一响应** `R<T>`：`{ code: number, msg: string, data: T }`；`code === 200` 成功，`500` 失败（另有 `601` 警告等）。
- **分页响应** `TableDataInfo<T>`：`{ total: number, rows: T[], code: number, msg: string }`。
- **鉴权头**：`Authorization: Bearer <access_token>`（Sa-Token `token-name: Authorization`，`is-concurrent: true`，`is-share: false`，JWT）。
- **客户端头（强制）**：**每个请求**需带 `clientid: <clientId>`（`LoginHelper.CLIENT_KEY = "clientid"`），其值须与登录时 `clientId` 及 token 内扩展一致，否则鉴权失败。
- **接口文档**：`springdoc` 开启，聚合文档可访问（用于联调比对）。

### 3.2 认证相关端点
| 方法 | 路径 | 说明 | 请求体 / 响应 |
|---|---|---|---|
| GET | `/auth/code` | 获取图形验证码 | `R<CaptchaVo>`；`CaptchaVo = { captchaEnabled: boolean, uuid?: string, img?: string(base64) }`。`captchaEnabled=false` 时登录无需验证码 |
| POST | `/auth/login` | 登录（`@ApiEncrypt`） | body 为 **JSON 字符串**：`PasswordLoginBody = { clientId, grantType:'password', tenantId?, username, password, code?, uuid? }`；响应 `R<LoginVo>`，`LoginVo = { access_token, expire_in, refresh_expire_in }` |
| POST | `/auth/logout` | 登出 | `R<Void>` |
| POST | `/auth/register` | 注册（`@ApiEncrypt`） | `RegisterBody`；`R<Void>` |
| GET | `/auth/tenant/list` | 租户列表（登录页选择租户） | `R<LoginTenantVo>` |
| GET | `/system/user/getInfo` | 当前登录用户信息 | `R<UserInfoVo>`；`UserInfoVo = { user: SysUserVo, roles: string[], permissions: string[] }` |

### 3.3 接口加密（`@ApiEncrypt`）——关键决策点
- 全局开关 `api-decrypt.enabled: false`（**默认关闭**）。
- **决策（推荐）**：联调与一期上线保持 `api-decrypt.enabled=false`。此时 `@ApiEncrypt` 注解不触发解密过滤器，`/auth/login` 可**直接提交明文 JSON 字符串**（仍需 `Content-Type: application/json`，body 为字符串化的 JSON）。
- **备选（开启加密时）**：前端需实现「AES 随机密钥加密 body + RSA 公钥加密 AES 密钥」，并将加密后的 AES 密钥放入请求头 `encrypt-key`（`api-decrypt.headerFlag: encrypt-key`），公钥取自后端 `api-decrypt.publicKey`。该分支单独排期，不阻塞一期。

> 结论：**一期按明文 JSON 接入**；把「加密适配层」设计为可插拔（见 §5 拦截器），后续开启仅需替换 `transformRequest`。

---

## 4. 目标接入架构（分层）

```
.env(.development/.production)     ← VITE_API_BASE_URL / VITE_CLIENT_ID / VITE_TENANT_ID
        │
        ▼
src/lib/http/client.ts             ← axios 实例：baseURL、请求拦截(注入 Authorization+clientid)、
        │                             响应拦截(解包 R/TableDataInfo、401/601 处理、错误 toast)
        ▼
src/lib/http/types.ts              ← R<T> / TableDataInfo<T> / PageQuery / LoginVo / UserInfoVo 等类型
        │
        ▼
src/features/*/api/*.ts            ← 端点封装（authApi、userApi、taskApi...），仅做「路径+参数」映射
        │
        ▼
src/features/*/hooks/use-*.ts      ← TanStack Query 封装（useLogin/useUserInfo/useUserTable...）
        │
        ▼
组件 / 路由守卫 / store            ← 消费 hooks；auth-store 存 token+user+permissions
```

**设计原则**
- 组件不直接 `import axios`，只调用 Query hooks。
- 响应拦截器**统一解包**：成功返回 `res.data.data`（或整个 `TableDataInfo`），失败抛出规范化错误。
- 所有后端文案（`msg`）走 toast；前端已有 i18n 用于**静态 UI**，后端动态消息按原样透传（可选做 i18n key 映射，见 §10）。

---

## 5. 关键实现决策

### 5.1 环境变量（新增 `.env.development` / `.env.production`，更新 `.env.example`）
```
VITE_API_BASE_URL=http://localhost:6039      # 生产为网关/域名
VITE_CLIENT_ID=<向后端索取的 PC 端 clientId>  # 对应 sys_client 表某条记录
VITE_GRANT_TYPE=password
VITE_TENANT_ID=000000                         # 若未启用多租户可留空/默认
VITE_API_ENCRYPT=false                        # 与后端 api-decrypt.enabled 对齐
```
- `clientId` 必须由后端在 `sys_client` 表中登记（`grantType` 含 `password`，`status=0` 正常），否则登录返回 `auth.grant.type.error`。

### 5.2 axios 实例（`src/lib/http/client.ts`）
- `baseURL = import.meta.env.VITE_API_BASE_URL`，`timeout`，`headers['Content-Type']='application/json'`。
- **请求拦截器**：
  - 注入 `Authorization: Bearer <token>`（从 auth-store/持久化读取）。
  - 注入 `clientid: <VITE_CLIENT_ID>`（**所有请求**）。
  - 若 `VITE_API_ENCRYPT=true` 且命中加密端点 → 走加密 `transformRequest`（可插拔）。
- **响应拦截器**：
  - HTTP 2xx 且 `body.code === 200` → 返回 `body`（或 `body.data`，二选一，全项目统一）。**决策：返回整个 `R`/`TableDataInfo`**，由 api 层决定取 `data` 还是 `rows`，避免分页信息丢失。
  - `body.code === 401`（未登录/token 失效）→ 清理会话、跳转登录（携带 `redirectTo`）、toast。
  - `body.code === 601`（警告）/ `500`（失败）→ `toast.error(body.msg)` 并 `reject`。
  - 网络/超时错误 → 复用 `handle-server-error.ts`（需按 §5.3 改造字段）。
- **登录 body 序列化**：`/auth/login` 的 body 必须是 `JSON.stringify(payload)` 字符串（后端 `@RequestBody String body` 再 `JsonUtils.parseObject`）。用 `data: JSON.stringify(payload)` 且 `Content-Type: application/json`。

### 5.3 错误处理改造（`src/lib/handle-server-error.ts`）
- 现读取 `error.response.data.title` → 改为优先读取 `error.response.data.msg`（后端 `R.msg`）。
- 保留 204 / 网络异常分支；错误文案接入 i18n 的 `common:toast.*`（如 `somethingWentWrong`）作为兜底。

### 5.4 会话存储与持久化
- 复用 `auth-store.ts`，但：
  - `accessToken` 存后端返回的 `access_token`。
  - `user` 结构对齐 `SysUserVo`（见 §7）。
  - 新增 `roles: string[]`、`permissions: string[]`（来自 `getInfo`）。
  - **持久化介质决策**：Token 建议存 `localStorage`（或保留 cookie，但需 `Secure/SameSite` 策略）；避免 XSS 风险由现有 `xss` 过滤 + CSP 兜底。
- 新增 `hasPermission(code)` / `hasRole(role)` 选择器，供按钮级鉴权。

### 5.5 登录流程（替换 mock）
```
1. (可选) GET /auth/tenant/list  → 渲染租户选择（多租户开启时）
2. GET /auth/code               → captchaEnabled 为 true 时渲染验证码图片+输入框，保存 uuid
3. 用户提交 → POST /auth/login  { clientId, grantType:'password', tenantId, username, password, code, uuid }
4. 成功 → 存 access_token；调用 GET /system/user/getInfo → 存 user/roles/permissions
5. navigate(redirectTo || '/')
6. 失败 → toast.error(msg)；刷新验证码
```
- **表单字段调整**：现登录表单用 `email`，后端用 `username`（登录名）。**决策**：将 sign-in 表单字段由 `email` 改为 `username`（或保留邮箱登录则需后端支持 `grantType`/邮箱登录策略）。i18n 键 `auth:common.email*` 相应调整为 `username*`。
- **验证码 UI**：新增验证码输入组件（受 `captchaEnabled` 控制显隐），i18n 补充 `auth:captcha.*` 文案（中/英）。

### 5.6 登出
- `sign-out-dialog` 确认 → `POST /auth/logout` → 无论成功失败均 `auth.reset()` 并跳登录。

### 5.7 路由守卫
- `_authenticated` 布局：进入前校验 token 是否存在且未过期（`expire_in`）。
- 无 token → 重定向 `/sign-in?redirectTo=<当前路径>`。
- 首次进入且有 token 但 store 无 user → 触发 `getInfo`（Query，`staleTime` 较长）。
- 可选：基于 `permissions` 做菜单/路由级过滤（与后端菜单权限对齐）。

### 5.8 数据表格服务端分页（users/tasks 等）
- 后端列表接口约定入参：`pageNum`、`pageSize`、`orderByColumn`、`isAsc` + 业务过滤字段；返回 `TableDataInfo<T>`。
- TanStack Table 改为 **manual pagination / sorting / filtering**：
  - `getCoreRowModel` 保留，去掉 `getPaginationRowModel` 的客户端分页。
  - `onPaginationChange` / `onSortingChange` → 更新 Query 的 `pageNum/pageSize/orderBy` → 触发请求。
  - `rowCount = total`，`data = rows`。
- 现有共享分页组件（已 i18n 的 `dataTable` 命名空间）无需改文案，仅改数据来源。

---

## 6. 分阶段执行计划

### Phase 0 — 准备与对齐（0.5 天）
- [ ] 向后端索取：PC 端 `clientId`、是否启用多租户、是否启用验证码、是否启用接口加密、测试账号。
- [ ] 确认后端 CORS 策略（允许前端 dev origin，允许 `Authorization`/`clientid` 头）。**若未配置，需后端补充 CORS 或前端走 Vite proxy（§8）**。
- [ ] 启动后端 `meet-admin`（profile `dev`，端口 6039）确认可访问 `/auth/code`。

### Phase 1 — HTTP 基础设施（1 天）
- [ ] 新增 `.env.development` / `.env.production`，更新 `.env.example`。
- [ ] 新增 `src/lib/http/types.ts`（`R<T>`、`TableDataInfo<T>`、`PageQuery`、`LoginVo`、`UserInfoVo`、`SysUserVo`、`CaptchaVo`）。
- [ ] 新增 `src/lib/http/client.ts`（实例 + 请求/响应拦截器）。
- [ ] 改造 `src/lib/handle-server-error.ts`（`msg` 字段）。
- [ ] 预留加密适配层占位（`transformRequest` hook，一期不实现）。

### Phase 2 — 认证接入（1.5 天）
- [ ] 新增 `src/features/auth/api/auth-api.ts`：`getCode`、`login`、`logout`、`getInfo`、`tenantList`。
- [ ] 新增 hooks：`useCaptcha`、`useLogin`、`useLogout`、`useUserInfo`。
- [ ] 改造 `user-auth-form.tsx`：`email`→`username`、接入 `useLogin`、去 mock、错误 toast、`redirectTo`。
- [ ] 新增验证码输入组件（受 `captchaEnabled` 控制）。
- [ ] 改造 `auth-store.ts`：存真实 token/user/roles/permissions，持久化。
- [ ] 改造 `sign-out-dialog`：调用 `useLogout`。
- [ ] 改造 `_authenticated` 路由守卫：token 校验 + `getInfo` 拉取。
- [ ] i18n：新增/调整 `auth:captcha.*`、`auth:common.username*`、`common:toast.*` 相关键（中/英同步）。

### Phase 3 — 用户/任务表格服务端化（2 天，可按模块拆分）
- [ ] `userApi`：`listUsers(PageQuery)`、`addUser`、`updateUser`、`deleteUser`、`resetPwd` 等（对齐 `SysUserController`）。
- [ ] `useUserTable`（Query + 分页/排序/过滤参数）。
- [ ] 改造 `users-table.tsx` / `users-columns.tsx`：manual pagination，字段映射（§7）。
- [ ] 改造 users 各弹窗（action/invite/delete/multi-delete）：由 `showSubmittedData` mock 改为真实 mutation + 成功后 `queryClient.invalidateQueries`。
- [ ] tasks 模块同法接入（若无对应后端接口，则保留 mock 并标注 TODO）。

### Phase 4 — 权限与体验（1 天）
- [ ] 按钮级鉴权：`hasPermission` 指令/hook 包裹操作按钮。
- [ ] 菜单级过滤（可选，与后端菜单权限对齐）。
- [ ] 401 全局拦截跳转验证；token 过期刷新策略（若有 refresh_token）。
- [ ] SSE 通知接入（后端 `sse.enabled: true`，`/resource/sse`）——可选，单独排期。

### Phase 5 — 联调与验收（1 天）
- [ ] 走查 §11 验收清单；修复字段/契约偏差。
- [ ] 生产环境变量与网关地址确认；构建产物验证。

> **总工期估算**：约 7 人日（单人），可与后端 CORS/clientId 准备并行压缩至 5 天。

---

## 7. 数据模型映射

### 7.1 用户（前端 `User` schema ↔ 后端 `SysUserVo`）
| 前端字段 | 后端字段 | 说明 |
|---|---|---|
| `id` | `userId` | 主键，注意 Long 精度（>2^53 用字符串接收） |
| `username` | `userName` | 登录名 |
| `firstName`/`lastName` | `nickName`（或拆分） | 后端通常为单一 `nickName`；**决策**：前端合并展示 `nickName`，或后端补字段 |
| `email` | `email` | |
| `phoneNumber` | `phonenumber` | 注意后端字段拼写 |
| `status` | `status` | 后端 `'0'正常/'1'停用` ↔ 前端 `active/inactive`（需枚举映射） |
| `role` | `roles[]`（角色对象数组） | 后端为多角色；前端当前单角色 → **决策**：取首个或改造为多选 |
| `createdAt`/`updatedAt` | `createTime`/`updateTime` | 时间格式统一（后端 `yyyy-MM-dd HH:mm:ss`） |

> **注意**：Long 型 ID 精度问题——后端 Jackson 通常已配置 Long→String 序列化；前端类型用 `string` 接收 ID，避免 JS 精度丢失。

### 7.2 状态/角色枚举映射
- 前端 `users/data/data.ts` 的 `statuses`/`roles`（已 i18n）保留为**展示层**；新增 `mapBackendStatus(code)` / `mapBackendRole(roleKey)` 做后端码值→前端 key 的映射，再经 i18n 渲染 label。

---

## 8. 跨域与代理（CORS）

**两种方案（择一，决策见下）**：

- **方案 A（推荐 dev）：Vite dev proxy**
  ```ts
  // vite.config.ts
  server: { proxy: { '/api': { target: 'http://localhost:6039', changeOrigin: true, rewrite: p => p.replace(/^\/api/, '') } } }
  ```
  前端 `VITE_API_BASE_URL=/api`，规避浏览器 CORS。
- **方案 B：后端 CORS 放行**（生产推荐，由网关/Nginx 统一处理）。

**决策**：开发用 A（零后端改动即可联调）；生产用 B（网关同源或显式 CORS）。两者对 axios 层透明（仅 `baseURL` 不同）。

---

## 9. 业务模块扩展范式（meeting / guest / workflow 等）

后端已具备大量 `R<Vo>` + `TableDataInfo<Vo>` 风格接口（如 `MeetingController`、`GuestController`、`TaskController`，路径多含 `/meetings/{meetingId}/...`）。扩展统一遵循：
1. `features/<mod>/api/<mod>-api.ts` 封装端点；
2. `hooks/use-<mod>*.ts` 封装 Query/Mutation；
3. 组件消费 hooks；
4. 新增 i18n 命名空间（`config.ts` 的 `namespaces` + `zh-CN.ts`/`en-US.ts` 同步补齐）。

> 该范式与 §5/§6 一致，后续模块按此复制即可，无需再设计。

---

## 10. 国际化与后端消息的协同

- **静态 UI 文案**：已全量 i18n（zh-CN 默认 / en-US），命名空间含 `common/layout/auth/dashboard/errors/settings/dataTable/tasks/users/chats/apps`。
- **后端动态消息**（`R.msg`）：一期**原样透传**（多为中文）。
- **可选增强**：为常见后端错误码建立 `code → i18n key` 映射表（如 `auth.grant.type.error`），在响应拦截器中翻译；需后端提供**稳定的业务错误码清单**。列为二期。
- **新增 i18n 键**（接入期需补，中英同步）：`auth:captcha.*`、`auth:common.username*`、`common:toast.sessionExpired`（已存在，复用）、`common:toast.unauthorized` 等。

---

## 11. 验收标准（联调 Checklist）

- [ ] 登录页可获取验证码（若启用），输入正确凭据登录成功，写入真实 `access_token`。
- [ ] 登录后 `getInfo` 正确加载 user/roles/permissions，页面展示当前用户。
- [ ] 刷新页面会话保持；token 失效时自动跳转登录并带 `redirectTo`。
- [ ] 登出清理会话并跳转登录。
- [ ] 每个请求头含 `Authorization` 与 `clientid`。
- [ ] users 表格服务端分页/排序/过滤生效，`total` 正确。
- [ ] users 增/改/删/邀请走真实接口，成功后列表自动刷新。
- [ ] 后端错误（500/601/401）以 toast 展示 `msg`，无未处理异常。
- [ ] `pnpm build`（`tsc -b && vite build`）通过；`pnpm test` 现有用例通过（登录表单字段调整后需同步更新对应测试断言）。
- [ ] 中英双语下，接入新增文案均正确显示。

---

## 12. 风险与回退

| 风险 | 影响 | 缓解 |
|---|---|---|
| `clientId` 未登记 / grantType 不匹配 | 登录必失败 | Phase 0 强制向后端索取并验证 |
| CORS 未放行 | dev 无法联调 | 采用 Vite proxy（方案 A） |
| Long ID 精度丢失 | 数据错乱 | ID 一律 `string` 接收 |
| 登录字段 email→username 改动 | 现有 sign-in 测试失败 | 同步更新测试与 i18n 键 |
| 接口加密若被开启 | 明文登录 400 | 加密适配层可插拔，按需实现 |
| 多租户 / 验证码开关差异 | 流程分支多 | 以 `/auth/code` 的 `captchaEnabled`、后端租户开关为准做条件渲染 |
| 后端字段命名（`phonenumber`/`userName`） | 映射易错 | §7 映射表为准，集中在一处转换 |

**回退策略**：接入层（http/api/hooks）与 UI 解耦，若后端未就绪，可用 MSW 或保留 mock adapter 通过同一 axios 实例返回假数据，UI 不感知，实现平滑切换。

---

## 13. 交付物清单（实施后应产出）

- 配置：`.env.development`、`.env.production`、更新 `.env.example`、`vite.config.ts`（proxy）。
- 基础设施：`src/lib/http/client.ts`、`src/lib/http/types.ts`、改造 `handle-server-error.ts`。
- 认证：`src/features/auth/api/auth-api.ts`、`hooks/use-{login,logout,captcha,user-info}.ts`、改造 sign-in 表单/验证码组件/auth-store/路由守卫/sign-out。
- 业务：`src/features/users/api/*` + `hooks/*` + 表格/弹窗改造（tasks 视后端接口而定）。
- i18n：新增键（zh-CN/en-US 同步）。
- 文档：本计划的落地记录与后端错误码清单（二期）。

---

*本计划基于对 `MeetWise-server`（端口 6039、Sa-Token `Authorization`、`clientid` 头、`R<T>`/`TableDataInfo<T>`、`/auth/login` `/auth/code` `/auth/logout` `/system/user/getInfo`、`api-decrypt.enabled=false`）与 `shadcn-admin`（mock 登录、Zustand auth-store、无 axios 实例）源码的实际核对编写，可直接据此排期实施。*
