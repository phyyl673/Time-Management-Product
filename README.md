# Time-Management-Product ---Daybit

**Daybit** — 你的随身小本本：学生/个人日常规划与时间管理单页应用，支持登录与云端同步，换设备、换浏览器数据不丢失。

---

## 一、Motivation（项目动机）

- **痛点**：日程、课表、待办、Due、与求职相关的任务、求职进度等散落在不同地方，换设备或清缓存后容易丢失，缺少一个「以我为中心」的随身本。
- **目标**：做一个**单页、免安装**的规划工具，把日历、课表、今日待办、Due 时间线、求职、购物、心情日记和 AI 助手收进一个页面；通过**账号登录 + 云端同步**，在手机、平板、电脑上都能看到同一份数据。
- **定位**：轻量、专注「今日」与「时间线」的日规划产品，不替代日历 App，而是把「今天要做什么」「哪些 Due 快到了」「怎么对长线任务进行规划」和「零散清单」统一在一起。

---

## 二、功能概览

（p.s.以下功能均从自己出发，目前该网站暂未考虑更庞大的用户群体，但是计划后期如果有迭代将会修改部分模块为更通用的模式。）

| 模块 | 说明 |
|------|------|
| **日历** | 月视图、今日高亮、按日期查看日程；支持课表一键导入（目前只设置了剑桥课表 URL 一键同步）。 |
| **课表 / 日程** | 左侧「今天 · x月x日」列表展示当日课程与任务；可手动添加课程（时间、名称、地点），仅登录用户可见与同步。 |
| **今日聚焦** | 当日待办列表，支持勾选完成、归档、子任务；与 Due 子任务、求职「今日必做/本周内」可同步展示。 |
| **Due 时间线** | 按课程/分类管理的截止任务，紧急程度与截止日展示；进度 100% 自动归档。 |
| **找工作** | 求职任务列表与分类（实习、全职、技术准备等），状态可与今日聚焦同步。 |
| **购物清单** | 待购项列表，顶栏显示待购数量。 |
| **心情日记** | 顶栏快捷心情 + 更多记录，按日保存；仅登录用户云端同步。 |
| **AI 助手** | 右侧面板对话，可生成今日计划、与任务联动；对话历史仅登录用户云端同步。 |
| **登录 / 注册** | 用户名 + 密码，注册需确认密码；未注册用户登录时提示「该用户名未注册，请先注册」，密码错误时提示「用户名或密码错误」。 |
| **云同步** | 登录后：任务、Due、求职、购物、心情、头像、AI 对话等按模块写入 Supabase；换设备登录同一账号即从云端恢复。 |
| **多端适配** | 手机端支持上下、左右滑动查看整页；小屏下 body 可滚动，中间主区域保持最小宽度以横向滑动查看三栏。 |

**未登录与已登录的差异**：未登录时不加载、不展示任何用户数据（课表、日程、今日聚焦、Due 等均为空）；仅登录后从云端加载并展示该用户的数据，或本地首次使用后随写随同步。

---

## 三、前端设计

- **形态**：单页应用，一个 `daybit-planner.html` 内包含 HTML + CSS + JavaScript，无构建步骤，浏览器直接打开即可使用。
- **布局**：顶栏（Logo、日期、今日课数/Due 数/待办数、心情快捷、头像/登录）+ 三栏主体：
  - **左栏**：日历月视图、今日日程列表、添加课程（仅登录显示）、同步课表（仅登录显示）；
  - **中栏**：Tab 切换 — 今日聚焦 / Due 时间线 / 找工作 / 购物清单；
  - **右栏**：AI 助手对话。
  - **下栏**：心情&灵感随心记；点击即上弹出最近持续的心情记录（仅保留过往日期的表情以确保隐私性，点击可查看具体心情内容）并显示日期，右侧可添加新的随笔心情日记。
- **状态与持久化**：全局变量（如 `dayTasks`、`dayFocusTasks`、`dueTasks`、`courseList`、`aiConversationHistory` 等）驱动 UI；未登录时仅用内存（且不读本地缓存中的用户数据），登录后本地 `saveToStorage()` 写 localStorage，并通过封装后的 `syncModule` / `syncModuleDebounced` 按模块同步到 Supabase。
- **技术栈**：原生 JS、CSS（含媒体查询做手机端滚动）、Supabase JS 客户端（认证与数据库）；字体与图标使用 Google Fonts、OpenMoji 等 CDN。

---

## 四、后端设计（Supabase）

- **用途**：用户身份 + 结构化数据存储，不写自建后端，仅用 Supabase 的 Auth 与 Database。
- **表设计**：
  - **users**：存注册用户，字段含 `id`（uuid）、`user_name`、`password_hash`（前端 SHA-256 后写入）；登录时按 `user_name` + `password_hash` 校验。
  - **daybit_data**：按用户存各模块数据，字段如 `user_id`（关联 users.id）、`module`（模块名）、`payload`（JSON 字符串）、`updated_at`；唯一约束 `(user_id, module)`，用 upsert 覆盖更新。
- **模块（module）与 payload**：
  - `tasks`：`dayTasks`、`dayFocusTasks`、`todayTasks`、`courseList`、`customCategories`
  - `due`：`dueItems`
  - `job`：`jobItems`、`jobCategories`
  - `shop`：`shopItems`
  - `mood`：心情/日记记录数组
  - `avatar`：`{ dataUrl }` 头像 base64
  - `ai`：`aiConversationHistory`
- **同步逻辑**：前端在 `saveToStorage`、心情保存、头像上传等时机调用 `syncModule(moduleName, payload)` 或防抖的 `syncModuleDebounced`；登录后 `loadFromCloud()` 拉取该 `user_id` 下全部 `daybit_data`，按 `module` 解析 `payload` 写回内存并刷新界面。
- **安全**：依赖 Supabase 的 RLS（Row Level Security），建议配置为：用户仅能读写自己的 `users` 行与 `daybit_data` 行（如 `auth.uid()` 或与前端传入的 `user_id` 一致，视实际采用的 Auth 方式而定）。当前为前端用 `password_hash` 查 users 表，未使用 Supabase Auth，部署时需在 Supabase 中配置好 RLS 与表结构。

---

## 五、使用与部署

- **部署**：将仓库部署到 Netlify / Vercel / GitHub Pages 等静态托管，部署目录指向仓库根目录，访问生成的 URL 即可使用；云同步需在 Supabase 中创建好 `users` 与 `daybit_data` 表并配置 RLS。

---

## 六、目录与文件

| 文件 | 说明 |
|------|------|
| `daybit-planner.html` | Daybit 单页应用：上述全部功能 + 登录与 Supabase 云同步 |
| `_redirects` | Netlify 重定向配置：访问根路径 `/` 时直接返回 `daybit-planner.html`（200 改写），这样打开站点首页就是规划页 |
| `README.md` | 本说明（动机、功能、前后端设计、使用与部署） |

---
