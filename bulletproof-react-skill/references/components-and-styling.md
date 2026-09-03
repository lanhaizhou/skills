# 🧱 组件设计与样式方案 (Components & Styling)

在 Bulletproof React 体系中，组件设计的核心原则是：**就近组织（Colocation）**、**共享组件与业务组件严格分层**、**组合优于配置（Composition over Props）** 以及 **第三方库薄封装隔离**。

---

## 一、组件分层架构

应用中的组件被明确划分为三个层级：

```text
src/
├── components/
│   ├── ui/               # 层级 1：基础通用 UI 原子组件（零业务逻辑、高复用）
│   │   ├── button/
│   │   ├── dialog/
│   │   ├── input/
│   │   ├── table/
│   │   └── notifications/
│   └── layouts/          # 层级 2：结构性页面布局骨架（跨页面通用）
│       ├── content-layout.tsx
│       ├── dashboard-layout.tsx
│       └── auth-layout.tsx
└── features/
    └── [feature-name]/
        └── components/   # 层级 3：业务功能独享组件（含领域交互、API 调用）
```

| 层级 | 存放位置 | 业务耦合度 | 是否包含 API 调用 | 复用范围 |
| :--- | :--- | :--- | :--- | :--- |
| **基础 UI 组件** | `src/components/ui/` | **零耦合**（纯展示/基础交互） | 严禁包含 | 全局所有 Feature |
| **布局骨架组件** | `src/components/layouts/` | **弱耦合**（仅负责网格/骨架） | 仅限全局 Header/Sidebar 鉴权状态 | 全局路由页面 |
| **Feature 业务组件** | `src/features/*/components/` | **强耦合**（专注该业务） | 允许消费该 Feature API Hooks | 仅限本 Feature 内部或 App 路由 |

---

## 二、组合模式 (Composition) 消除 Props 膨胀

### ❌ 反模式：巨型配置化 Props
当一个组件接收过多控制性 Props（如 `hasHeader`, `isWarning`, `confirmBtnColor`, `renderExtraFooter` 等），组件内部充斥大量条件分支，维护成本极高：

```tsx
// ❌ 差的实践：Props 泛滥导致组件难以扩展
<ConfirmationDialog
  isOpen={isOpen}
  title="确认删除"
  description="此操作无法撤销"
  confirmText="删除"
  cancelText="取消"
  confirmButtonVariant="danger"
  showIcon={true}
  iconType="trash"
  onConfirm={handleDelete}
  onCancel={handleClose}
/>
```

### ✅ 推荐模式：利用 Slots / Children 自由组合
通过插槽组合与触发器包装，让调用方决定触发时机与内容排版：

```tsx
// src/components/ui/dialog/confirmation-dialog/confirmation-dialog.tsx
import * as React from 'react';
import { Button } from '@/components/ui/button';
import { Dialog, DialogContent, DialogFooter, DialogHeader, DialogTitle, DialogTrigger } from '@/components/ui/dialog';

export type ConfirmationDialogProps = {
  triggerButton: React.ReactElement;
  confirmButton: React.ReactElement;
  title: string;
  body?: string;
  children?: React.ReactNode;
};

export const ConfirmationDialog = ({
  triggerButton,
  confirmButton,
  title,
  body,
  children,
}: ConfirmationDialogProps) => {
  return (
    <Dialog>
      <DialogTrigger asChild>{triggerButton}</DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
        </DialogHeader>
        {body && <p className="text-sm text-gray-500">{body}</p>}
        {children}
        <DialogFooter className="flex justify-end space-x-2 mt-4">
          {confirmButton}
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
};
```

**使用方体验：**
```tsx
<ConfirmationDialog
  title="删除讨论"
  body="确定要永久删除该讨论帖吗？"
  triggerButton={<Button variant="danger">删除</Button>}
  confirmButton={
    <Button variant="danger" onClick={handleDeleteDiscussion}>
      确认删除
    </Button>
  }
/>
```

---

## 三、警惕内部渲染函数 (Nested Render Functions)

### ❌ 反模式：在单一组件中塞入大量辅助渲染函数
```tsx
// ❌ 极难维护：一个文件超 400 行，充满 renderXxx 函数
function UserDashboard() {
  function renderHeader() { return <header>...</header>; }
  function renderStats() { return <div>...</div>; }
  function renderRecentLogs() { return <ul>...</ul>; }

  return (
    <div>
      {renderHeader()}
      {renderStats()}
      {renderRecentLogs()}
    </div>
  );
}
```

### ✅ 推荐模式：拆分为聚焦的子组件（就近存放在同一目录）
```tsx
// ✅ 拆分成独立组件，逻辑隔离、状态互不干扰
function Header() { return <header>...</header>; }
function Stats() { return <div>...</div>; }
function RecentLogs() { return <ul>...</ul>; }

export function UserDashboard() {
  return (
    <div>
      <Header />
      <Stats />
      <RecentLogs />
    </div>
  );
}
```

---

## 四、第三方基础组件的绝缘层封装

无论是路由导航链接还是 UI 组件库（如 Radix、Headless UI、AntD 等），**切勿在业务组件中到处直接 import 第三方原生组件**。
在 `src/components/ui/` 中建立薄封装层：

```tsx
// src/components/ui/link/link.tsx
import { Link as RouterLink, LinkProps } from 'react-router';
import { cn } from '@/utils/cn';

export const Link = ({ className, children, ...props }: LinkProps) => {
  return (
    <RouterLink
      className={cn('text-blue-600 hover:text-blue-500 hover:underline', className)}
      {...props}
    >
      {children}
    </RouterLink>
  );
};
```

**收益**：未来若从 React Router 迁移至 TanStack Router 或 Next.js，只需修改 `src/components/ui/link/link.tsx` 一处文件，成百上千个业务 Feature 无需任何改动。

---

## 五、现代样式方案与类名合并工具 (`cn`)

推荐采用 **Tailwind CSS + Headless/Radix UI**（类似 Shadcn UI 思想），配合 `clsx` 与 `tailwind-merge` 构建 `cn` 工具函数：

```typescript
// src/utils/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

使用示例：
```tsx
// 组件内部保证默认样式可被外部传入的 className 精确覆盖而不发生类名冲突
export const Button = ({ variant = 'primary', className, ...props }) => {
  return (
    <button
      className={cn(
        'inline-flex items-center justify-center rounded-md font-medium transition-colors',
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'danger' && 'bg-red-600 text-white hover:bg-red-700',
        className
      )}
      {...props}
    />
  );
};
```
