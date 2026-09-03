---
name: bulletproof-react-skill
description: "React 企业级架构设计与规范指南。当用户进行 React 项目架构搭建、模块划分、功能开发（Feature-based 模块化）、API 接口层封装（TanStack Query / Axios）、单向依赖治理（防跨 Feature 循环引用与依赖违规）、状态管理（Zustand / Server State 与 Client State 分离）、组件规范设计、目录重构、代码审查（Code Review）或遵循 bulletproof-react 最佳实践时使用。"
argument-hint: "[audit | init | feature <name> | api <name> | component <name> | --quick]"
---

# Bulletproof React 企业级架构规范与操作指南

本 Skill 基于 [alan2207/bulletproof-react](https://github.com/alan2207/bulletproof-react) 生产级架构体系构建，旨在指导现代 React + TypeScript 项目的工程化落地、模块边界治理与长期可维护性。适用于中大型前端应用开发、微前端功能拆分与存量代码重构。

---

## ⚡ 核心铁律 (Iron Laws)

> [!IMPORTANT]
> **STRICT BOUNDARIES & UNIDIRECTIONAL DEPENDENCIES**
> （边界不清的代码即技术负债，严禁破坏模块隔离与单向流动原则）

1. **铁律 1 (单向依赖流)**：代码依赖流向必须且只能单向流动：`shared` (`components/hooks/lib/utils/types/config`) → `features` → `app`。
   - `shared` 严禁反向 import `features` 或 `app`。
   - `features` 严禁反向 import `app`。
2. **铁律 2 (严禁跨 Feature 直接引用)**：任何 Feature 之间禁止互相直接 import（例如 `features/discussions` 严禁直接引用 `features/comments`）。跨 Feature 的状态联动或组件组装，**必须上浮至 `app/` 路由/页面层完成**。
3. **铁律 3 (服务端状态与客户端状态隔离)**：严禁将服务端 API 返回的业务数据镜像存入全局客户端状态库（如 Zustand/Redux）。凡是异步请求数据，统一由 **TanStack Query**（或 SWR/RTK Query）管理缓存、重新验证与失效；客户端 Store 仅维护纯本地 UI 交互状态。
4. **铁律 4 (接口请求标准化三件套)**：严禁在 UI 组件内随手编写裸 `axios/fetch` 调用。每一个 API 请求必须以独立文件封装，包含：**DTO 类型与 Zod 校验 Schema** + **独立 Fetcher 函数** + **Query/Mutation Hook（配套 queryOptions 与精确缓存失效）**。

---

## 🚩 刹车信号 (Red Flags)

在编写或审查代码时，只要出现以下任何一种迹象，**立即强制中断并重构**：

- ❌ 跨 Feature 导入：出现类似 `import { ... } from '@/features/other-feature/...'`。
- ❌ 状态双重维护：定义了 `const [data, setData] = useState()` 或在 Zustand 中专门写一个 action 把 API 响应结果 `setUsers(res.data)`。
- ❌ 内部渲染函数泛滥：在一个大组件中定义 `renderHeader()`, `renderList()`, `renderFooter()`，而不是拆分成独立自解释组件。
- ❌ 滥用垃圾桶 Barrel 文件：在 Feature 根目录下通过 `index.ts` 导出所有内部细节，破坏 Vite Tree-shaking 并诱发循环依赖。
- ❌ 相对路径深度地狱：出现 `../../../../components/button`，未配置使用 `@/*` 绝对路径别名。
- ❌ 随意命名：文件或文件夹出现驼峰命名或下划线命名（未遵循标准 `kebab-case`）。

---

## 🎯 引导注意力的自查问题 (Probing Questions)

在编写、重构或审查每一段代码前，主动带着以下问题进行深度思考：

1. **针对边界**：“这个组件/工具函数是当前 Feature 独有的，还是跨业务通用的？如果只有本业务用，它是否被死死关在 `src/features/[feature-name]` 内部？”
2. **针对依赖**：“如果把当前这个 Feature 文件夹直接物理删除，应用中的其他 Feature 能否正常编译，还是会引起连锁爆红？”
3. **针对状态**：“这段数据是来源于服务端的快照，还是纯粹的本地交互状态？如果重新刷新页面或多端同步，谁是唯一真实数据源（Source of Truth）？”
4. **针对组件**：“这个组件接收的 props 数量是不是超过了 5 个？是否能够通过 `children` 或 Slot 组合模式拆解，避免把所有配置项都作为 prop 传入？”

---

## 🔄 标准工作流 (Standard Workflow)

执行相关任务时，按照以下清单逐步推进，不要跳过关键步骤：

- [ ] **Step 1: 架构定位与模块归属确认** ⛔ `BLOCKING`
  - 确认代码层级归属：`app`（路由/集成）、`features`（领域业务）还是 `shared`（通用底层）。
  - 若为新业务功能，在 `src/features/` 下以 `kebab-case` 建立专属功能目录。

- [ ] **Step 2: 按需加载规范知识库** ⚠️ `REQUIRED`
  - 根据涉及的开发任务，加载对应规范指南：
    - 涉及 目录划分、跨模块依赖、ESLint 边界控制：`Load references/project-structure.md`
    - 涉及 API 封装、Axios 单例、TanStack Query v5、MSW Mock：`Load references/api-layer.md`
    - 涉及 UI 组件分层、组合模式、第三方库封装、Tailwind 样式：`Load references/components-and-styling.md`
    - 涉及 客户端状态、Zustand Store、服务端状态分离：`Load references/state-management.md`
    - 涉及 Vitest、Testing Library 行为驱动测试、Playwright E2E：`Load references/testing-and-quality.md`

- [ ] **Step 3: 编码落地与模式套用** ⚠️ `REQUIRED`
  - 严格贯彻 API 三件套规范、UI 组合模式与单向依赖流。
  - 严格杜绝跨 Feature 直接 import。

- [ ] **Step 4: 交付前对照检查清单** ⛔ `BLOCKING`
  - 执行 `Load references/checklist.md` 进行 P0 ~ P3 逐级自查。
  - 确保无跨模块污染、无状态镜像、无裸 API 调用。

---

## 📚 知识库路由 (References Router)

当需要深入了解具体设计细节与工程代码模板时，按需读取对应模块：

1. [项目结构与依赖治理规范](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/project-structure.md) (`references/project-structure.md`):
   - 核心目录层级拓扑（`app`, `features`, `components`, `lib`, `config` 等）
   - Feature 内部推荐子目录结构（`api`, `components`, `hooks`, `types` 等）
   - ESLint `import/no-restricted-paths` 边界硬约束配置
   - 文件与目录 kebab-case 命名规则与 `@/*` 路径映射

2. [API 层与数据获取范式](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/api-layer.md) (`references/api-layer.md`):
   - 单例 Axios 客户端封装（统一拦截器、Auth、401 重定向与全局通知）
   - API 声明三件套标准模板（DTO Schema + Fetcher + Query/Mutation Hook）
   - TanStack Query v5 规范：`queryOptions`、精确查询缓存失效、数据预拉取（Prefetching）
   - MSW (Mock Service Worker) 模拟 API 与数据工厂

3. [组件设计与样式方案](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/components-and-styling.md) (`references/components-and-styling.md`):
   - 通用 UI 组件 (`components/ui`) vs Feature 独享组件分界
   - 组合模式（Composition）实战：通过 `children` 与 Slots 消除 Props 膨胀
   - 外部第三方 UI 组件隔离适配（封装 Link、Button、Dialog）
   - Tailwind CSS 与 Headless UI / Radix 最佳实践

4. [状态管理分水岭](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/state-management.md) (`references/state-management.md`):
   - 服务端状态（Server State）vs 客户端状态（Client State）权威界限
   - Zustand 最佳实践：纯本地 UI 状态 Store、Feature-level Store
   - 为什么避免在 React 中滥用 Redux 与低效 Context

5. [测试策略与质量保障](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/testing-and-quality.md) (`references/testing-and-quality.md`):
   - 测试哲学：行为驱动（Testing Library）而不是测试内部实现细节
   - Vitest 单元与集成测试（以 Feature 路由为维度的集成测试）
   - Playwright 端到端（E2E）核心冒烟测试
   - Husky + Lint-staged + TypeScript strict 质量门禁

6. [交付前质量检查清单](file:///Users/p/Documents/code/skills/bulletproof-react-skill/references/checklist.md) (`references/checklist.md`):
   - P0 阻断级清单 (跨 Feature 导入、单向流倒置、裸调用、状态镜像)
   - P1 严重级清单 (组件巨大化、缺乏类型推导、缓存失效缺失)
   - P2 规范级清单 (文件命名、路径别名、ESLint 限制)
   - P3 优化级清单 (代码预加载、Tree-shaking、测试覆盖)

---

## 🎛️ 参数指令速查 (Arguments Guide)

用户调用本 Skill 时可传入参数定制执行模式：

- `/bulletproof-react-skill audit`：针对现有 React 代码库执行架构健康度与边界违规检查，输出 P0~P3 审计报告。
- `/bulletproof-react-skill init`：生成标准 Bulletproof React 目录结构脚手架与配置文件模板（ESLint、TSConfig、Vite）。
- `/bulletproof-react-skill feature <name>`：生成一个标准自包含 Feature 模块的完整目录与代码模板。
- `/bulletproof-react-skill api <name>`：基于三件套标准生成特定接口的 Zod Schema、Fetcher 与 TanStack Query Hook。
- `/bulletproof-react-skill component <name>`：基于组合模式与 UI 封装规范生成高质量组件代码。
- `/bulletproof-react-skill --quick`：快速生成精简版项目目录骨架与说明。
