# 📡 API 层与数据获取范式 (API Layer & Data Fetching)

在 Bulletproof React 架构中，API 层承担着前后端数据通信的桥梁。其核心原则是：**单例客户端实例集中配置** + **接口声明三件套标准化封装** + **TanStack Query v5 深度结合**。

---

## 一、全局单例 API 客户端 (`src/lib/api-client.ts`)

应用中所有 HTTP 请求均通过统一的 API 客户端实例发出，集中处理基础 URL、凭证传递、统一拦截器、错误提示及认证失效（401）：

```typescript
// src/lib/api-client.ts
import Axios, { InternalAxiosRequestConfig } from 'axios';
import { env } from '@/config/env';
import { paths } from '@/config/paths';
import { useNotifications } from '@/components/ui/notifications';

function authRequestInterceptor(config: InternalAxiosRequestConfig) {
  if (config.headers) {
    config.headers.Accept = 'application/json';
  }
  // 携带 Cookie 凭证或在此处附加 Bearer Token
  config.withCredentials = true;
  return config;
}

export const api = Axios.create({
  baseURL: env.API_URL,
});

api.interceptors.request.use(authRequestInterceptor);

api.interceptors.response.use(
  (response) => {
    // 自动剥离外层，直接返回数据实体
    return response.data;
  },
  (error) => {
    const message = error.response?.data?.message || error.message;
    
    // 全局错误通知触发
    useNotifications.getState().addNotification({
      type: 'error',
      title: '请求失败',
      message,
    });

    // 401 统一鉴权失效处理：跳转登录并保留重定向来源
    if (error.response?.status === 401) {
      const searchParams = new URLSearchParams();
      const redirectTo =
        searchParams.get('redirectTo') || window.location.pathname;
      window.location.href = paths.auth.login.getHref(redirectTo);
    }

    return Promise.reject(error);
  }
);
```

---

## 二、接口声明三件套标准模板

每一个具体接口均封装为独立文件，存放在对应 Feature 的 `api/` 目录下（如 `src/features/discussions/api/`）。

每个文件包含三部分：
1. **类型与校验契约 (Schema & Types)**：使用 Zod 声明入参 Schema，并推导 TypeScript 类型；
2. **底层请求函数 (Fetcher Function)**：调用单例 `api` 实例完成网络交互；
3. **Query/Mutation Hook**：基于 TanStack Query 封装，暴露 `queryOptions` 或内置缓存自动失效。

### 1. 查询接口模板 (GET / Query)

```typescript
// src/features/discussions/api/get-discussions.ts
import { queryOptions, useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { QueryConfig } from '@/lib/react-query';
import { Discussion, Meta } from '@/types/api';

// 1. 类型定义
export type GetDiscussionsParams = {
  page?: number;
};

export type GetDiscussionsResponse = {
  data: Discussion[];
  meta: Meta;
};

// 2. Fetcher 函数
export const getDiscussions = (
  { page = 1 }: GetDiscussionsParams = {}
): Promise<GetDiscussionsResponse> => {
  return api.get('/discussions', {
    params: { page },
  });
};

// 3. TanStack Query v5 标准：导出 queryOptions 供预拉取与 Hook 共享
export const getDiscussionsQueryOptions = ({ page = 1 }: GetDiscussionsParams = {}) => {
  return queryOptions({
    queryKey: ['discussions', { page }],
    queryFn: () => getDiscussions({ page }),
  });
};

type UseDiscussionsOptions = {
  params?: GetDiscussionsParams;
  queryConfig?: QueryConfig<typeof getDiscussionsQueryOptions>;
};

// 4. 自定义 Hook 导出
export const useDiscussions = ({
  params,
  queryConfig,
}: UseDiscussionsOptions = {}) => {
  return useQuery({
    ...getDiscussionsQueryOptions(params),
    ...queryConfig,
  });
};
```

### 2. 变更接口模板 (POST/PUT/DELETE / Mutation)

Mutation 接口必须负责**联动失效相关查询缓存**：

```typescript
// src/features/discussions/api/create-discussion.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { z } from 'zod';
import { api } from '@/lib/api-client';
import { MutationConfig } from '@/lib/react-query';
import { Discussion } from '@/types/api';
import { getDiscussionsQueryOptions } from './get-discussions';

// 1. Zod Schema 校验与类型推导
export const createDiscussionInputSchema = z.object({
  title: z.string().min(1, '标题不能为空').max(100, '标题不能超过 100 字'),
  body: z.string().min(1, '正文不能为空'),
});

export type CreateDiscussionInput = z.infer<typeof createDiscussionInputSchema>;

// 2. Fetcher 函数
export const createDiscussion = ({
  data,
}: {
  data: CreateDiscussionInput;
}): Promise<Discussion> => {
  return api.post('/discussions', data);
};

type UseCreateDiscussionOptions = {
  mutationConfig?: MutationConfig<typeof createDiscussion>;
};

// 3. Mutation Hook：内置精准缓存失效
export const useCreateDiscussion = ({
  mutationConfig,
}: UseCreateDiscussionOptions = {}) => {
  const queryClient = useQueryClient();
  const { onSuccess, ...restConfig } = mutationConfig || {};

  return useMutation({
    mutationFn: createDiscussion,
    onSuccess: (...args) => {
      // 核心：创建成功后，立刻失效 discussions 列表查询缓存
      queryClient.invalidateQueries({
        queryKey: getDiscussionsQueryOptions().queryKey,
      });
      onSuccess?.(...args);
    },
    ...restConfig,
  });
};
```

---

## 三、利用 `queryOptions` 实现路由数据预加载 (Prefetching)

通过分离 `queryOptions`，可以在路由 Loader 或组件鼠标悬浮（Hover）时提前触发数据加载，消除页面跳转后的白屏与 Loading 骨架屏：

```tsx
// src/app/routes/app/discussions/discussions.tsx
import { QueryClient, useQueryClient } from '@tanstack/react-query';
import { LoaderFunctionArgs } from 'react-router';
import { getDiscussionsQueryOptions } from '@/features/discussions/api/get-discussions';
import { getInfiniteCommentsQueryOptions } from '@/features/comments/api/get-comments';
import { DiscussionsList } from '@/features/discussions/components/discussions-list';

// 1. 路由 Loader 中预获取数据
export const clientLoader =
  (queryClient: QueryClient) =>
  async ({ request }: LoaderFunctionArgs) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page') || 1);
    const query = getDiscussionsQueryOptions({ page });

    return (
      queryClient.getQueryData(query.queryKey) ??
      (await queryClient.fetchQuery(query))
    );
  };

// 2. 交互时预加载关联数据（如 Hover 时预加载评论）
export const DiscussionsRoute = () => {
  const queryClient = useQueryClient();

  return (
    <DiscussionsList
      onDiscussionHover={(id) => {
        // 用户悬停在讨论项上时，预先拉取详情与评论
        queryClient.prefetchInfiniteQuery(getInfiniteCommentsQueryOptions(id));
      }}
    />
  );
};
```

---

## 四、基于 MSW (Mock Service Worker) 的接口模拟

在后端 API 未交付或测试环境中，利用 MSW 进行无侵入拦截模拟，无需修改业务代码中的 fetch URL：

```typescript
// src/testing/mocks/handlers/discussions.ts
import { http, HttpResponse } from 'msw';
import { env } from '@/config/env';

export const discussionsHandlers = [
  http.get(`${env.API_URL}/discussions`, ({ request }) => {
    return HttpResponse.json({
      data: [
        { id: '1', title: 'Bulletproof React 架构讨论', body: '如何划分模块？' },
      ],
      meta: { page: 1, totalPages: 1 },
    });
  }),

  http.post(`${env.API_URL}/discussions`, async ({ request }) => {
    const body = (await request.json()) as any;
    return HttpResponse.json({
      id: String(Date.now()),
      ...body,
    });
  }),
];
```
