# ✅ 交付前质量核查清单 (Pre-Delivery & Code Review Checklist)

在合并代码、提交 Pull Request 或完成新功能开发前，按照 **P0（阻断）→ P1（严重）→ P2（规范）→ P3（优化）** 四级标准进行逐项核验。出现任何 P0 / P1 问题，**严禁合入生产分支**。

---

## 🛑 P0: 阻断级清单 (架构底线，一票否决)

- [ ] **严禁跨 Feature 互相 import**：检查是否有任何跨 Feature 的引用（如 `from '@/features/users/...'` 出现在 `features/discussions` 中）。跨 Feature 的组装编排必须在 `src/app/` 路由/页面层完成。
- [ ] **严格保持单向依赖流向**：`shared` (`components/hooks/lib/utils/types/config`) 严禁反向 import `features` 或 `app`；`features` 严禁反向 import `app`。
- [ ] **严禁服务端状态镜像存入全局 Store**：严禁在 Zustand/Redux 中设立 `setItems(res.data)` 拷贝服务端返回的数据；所有服务端数据必须直接通过 TanStack Query 管理。
- [ ] **严禁组件内部裸调用网络请求**：禁止在 React 组件内部直接调用 `axios.get/post` 或 `fetch`；所有请求必须收敛至 `features/[feature]/api/` 下封装的三件套。

---

## ⚠️ P1: 严重级清单 (健壮性与可维护性)

- [ ] **API 变更操作必须显式失效关联查询**：在所有 `useMutation` 的 `onSuccess` 回调中，必须调用 `queryClient.invalidateQueries` 刷新被影响的列表或详情缓存。
- [ ] **接口参数与返回必须具备静态与运行时契约**：变更接口入参必须有 Zod Schema 校验与类型推导（`z.infer<...>`），查询返回必须具备明确的 TypeScript 响应实体类型。
- [ ] **禁止巨型组件与内部渲染函数**：单个组件内不得包含超过 2 个由 `function renderXxx()` 组成的嵌套子结构，必须就近拆分成具名小组件。
- [ ] **避免 Props 泛滥**：接收超过 5 个配置性 Props 的组件，应优先考虑使用 `children`、插槽或组合模式（Composition）重构。
- [ ] **全局第三方基础组件完成绝缘封装**：如 Router `Link`、Modal 弹窗、Icon 等通用能力，禁止在业务组件中裸引第三方库，必须通过 `src/components/ui/` 薄层封装。

---

## 📋 P2: 规范级清单 (团队协作与一致性)

- [ ] **全量短横线命名 (kebab-case)**：除 `__tests__` 外，`src` 下所有文件与文件夹必须为 `kebab-case`（如 `discussions-list.tsx`、`use-disclosure.ts`）。
- [ ] **全面使用绝对路径别名 (`@/*`)**：代码中不得出现超过两层的相对路径（如 `../../../../components/button`），统一改写为 `@/components/button`。
- [ ] **避免全量 Barrel 文件 (`index.ts`)**：Feature 根目录下避免一次性重导出所有内部文件，鼓励直接引用目标文件以保护 Vite 的构建速度与 Tree-shaking 效果。
- [ ] **全局环境变量安全**：所有环境变量访问均通过 `src/config/env.ts` 集中导入并经由 Zod 校验，禁止在业务代码深处散落 `process.env` 或 `import.meta.env`。

---

## 🚀 P3: 优化级清单 (极致体验与测试覆盖)

- [ ] **高频路由与交互预加载 (Prefetching)**：利用 TanStack Query 的 `queryOptions` 在路由 Loader 或悬停 (Hover) 时预拉取下一步所需数据，避免白屏与布局抖动。
- [ ] **URL 状态优先原则**：分页参数、筛选条件、活动 Tab 等具备分享与回退价值的状态，优先保存在 URL 查询参数中。
- [ ] **核心业务具备集成测试**：Feature 核心用户路径必须具备基于 Testing Library + MSW 的集成测试用例，确保网络异常或交互变更能够被自动化捕获。
