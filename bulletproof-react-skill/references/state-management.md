# 🧠 状态管理分水岭 (State Management Architecture)

在很多失控的 React 项目中，最常见的灾难就是**滥用全局状态管理库将所有数据（尤其是接口数据）打包存储**。Bulletproof React 确立了一道绝对清晰的状态管理分水岭：**彻底隔离服务端状态与客户端状态**。

---

## 一、状态分类矩阵与技术选型

| 状态类型 | 特征描述 | 典型场景 | 推荐处理方案 |
| :--- | :--- | :--- | :--- |
| **服务端状态 (Server State)** | 异步、归属于服务端、具有过期失效性、需要重拉与乐观更新 | 用户信息、文章列表、评论数据、配置参数 | **TanStack Query** (React Query) |
| **URL 状态 (URL State)** | 用户可见、需要支持前进后退、刷新保持与链接分享 | 翻页 `?page=2`、筛选 `?tab=audit`、搜索关键字 | **React Router** `useSearchParams` |
| **全局客户端状态 (Global Client State)** | 同步、纯前端 UI 交互、跨多个 Feature 共享且不宜放在 URL 中 | 全局消息通知 (Toast)、侧边栏收起、夜间模式 | **Zustand** (集中在 `src/stores/`) |
| **Feature 客户端状态 (Feature Client State)** | 同步、仅在特定 Feature 内部多个复杂子组件间共享 | 多步骤向导表单（Wizard）、画布编辑器交互状态 | **Feature-level Zustand** 或 **React Context** |
| **本地状态 (Local State)** | 单一组件自闭环交互 | 弹窗开关、输入框受控值、下拉菜单高亮 | React 原生 `useState` / `useReducer` |

---

## 二、绝不能越界的铁律：严禁将服务端状态镜像存入全局 Store

### ❌ 典型反模式（架构死线）
在接口请求成功后，通过 `useEffect` 或回调把数据塞进全局 Store：

```tsx
// ❌ 灾难模式：双重状态源与数据同步死循环
const UserProfile = () => {
  const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
  const setUser = useUserStore((state) => state.setUser);

  // 绝不要这么写！破坏单一真实数据源，引入竞态条件与不必要的二次渲染
  useEffect(() => {
    if (data) {
      setUser(data);
    }
  }, [data, setUser]);

  return <div>...</div>;
};
```

### ✅ 正确做法：直接消费 TanStack Query
TanStack Query 本身就是一个自带高效缓存、去重与智能刷新的异步状态容器。需要使用服务端数据时，直接调用对应的 API Query Hook：

```tsx
// ✅ 正确做法：唯一真实数据源来自于服务端缓存
export const UserProfile = () => {
  const { data: user, isLoading } = useUser();

  if (isLoading) return <Spinner />;
  return <div>{user.name}</div>;
};
```

---

## 三、Zustand 全局客户端状态最佳实践

当且仅当数据属于**纯客户端运行时 UI 状态**时，使用轻量级 [Zustand](https://github.com/pmndrs/zustand)。

### 全局通知系统示例 (`src/stores/notifications.ts`)

```typescript
// src/components/ui/notifications/notifications-store.ts
import { create } from 'zustand';

export type Notification = {
  id: string;
  type: 'info' | 'warning' | 'success' | 'error';
  title: string;
  message?: string;
};

type NotificationsStore = {
  notifications: Notification[];
  addNotification: (notification: Omit<Notification, 'id'>) => void;
  dismissNotification: (id: string) => void;
};

export const useNotifications = create<NotificationsStore>((set) => ({
  notifications: [],
  addNotification: (notification) =>
    set((state) => ({
      notifications: [
        ...state.notifications,
        { id: Math.random().toString(36).substring(2, 9), ...notification },
      ],
    })),
  dismissNotification: (id) =>
    set((state) => ({
      notifications: state.notifications.filter((n) => n.id !== id),
    })),
}));
```

**Zustand 的核心优势：**
1. **轻量无样板**：没有 Redux 繁重的 Reducer/Action 类型定义与 Provider 包裹。
2. **可在 React 组件树外直接调用**：例如在 `src/lib/api-client.ts` 的 Axios 拦截器中，直接通过 `useNotifications.getState().addNotification(...)` 发送报错通知，无需注入 Hook。

---

## 四、Feature 内部的局部 Store (Feature-level Store)

若某个复杂业务（例如多步骤发布表单）需要跨 3~5 个子组件共享草稿状态，应将该 Store 就近收敛在 Feature 目录内：

```text
src/features/discussions/
├── stores/
│   └── create-discussion-store.ts   # 仅限 discussions 模块内部子组件引用
```

**注意**：此 Feature Store 严禁导出给外部其他 Feature 使用！一旦外部需要访问，说明这部分状态应当重构为由父级装配或重构为 URL 参数。

---

## 五、URL 即状态 (URL as State)

对于列表筛选、分页、Tab 切换等状态，**优先存储在 URL 查询参数中**（`?page=2&status=active`）：

```tsx
// src/features/discussions/components/discussions-list.tsx
import { useSearchParams } from 'react-router';

export const DiscussionsList = () => {
  const [searchParams, setSearchParams] = useSearchParams();
  const page = Number(searchParams.get('page') || 1);

  const handlePageChange = (newPage: number) => {
    setSearchParams({ page: String(newPage) });
  };

  // ...
};
```

**收益：**
- 用户可以复制当前 URL 分享给他人，状态完全还原。
- 原生支持浏览器的“前进”、“后退”与刷新无缝保真。
