# 実装ガイド - TaskFlow

## はじめに

このドキュメントは、TaskFlowプロジェクトを実際に実装する際の具体的な手順とサンプルコードを提供します。
ご指定いただいた学習要件（Server Components、並列データフェッチ、キャッシュ戦略など）を全て満たすように構成されています。

## 1. プロジェクトセットアップ

### 1.1 初期セットアップコマンド

```bash
# Next.jsプロジェクトの作成
npx create-next-app@latest taskflow --typescript --tailwind --app --eslint

# ディレクトリ移動
cd taskflow

# 必要なパッケージのインストール
npm install @prisma/client prisma
npm install next-auth@beta @auth/prisma-adapter
npm install zod
npm install lucide-react
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
npm install date-fns
npm install bcryptjs
npm install @types/bcryptjs --save-dev

# 開発用パッケージ
npm install --save-dev @types/node
npm install --save-dev jest @testing-library/react @testing-library/jest-dom
npm install --save-dev @playwright/test
npm install --save-dev eslint-config-prettier prettier
```

### 1.2 TypeScript設定（tsconfig.json）

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 1.3 Tailwind CSS設定（tailwind.config.ts）

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
        },
      },
      animation: {
        'fade-in': 'fadeIn 0.5s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [],
}

export default config
```

## 2. データベースセットアップ

### 2.1 Prisma初期化

```bash
# Prisma初期化
npx prisma init

# .envファイルを作成し、データベース接続情報を設定
# DATABASE_URL="postgresql://user:password@localhost:5432/taskflow"
```

### 2.2 スキーマ定義（prisma/schema.prisma）

```prisma
// 前述の詳細設計書のスキーマを使用
// ここでは主要な部分のみ記載

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  password      String
  name          String
  avatarUrl     String?
  role          UserRole  @default(MEMBER)
  emailVerified DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  memberships   ProjectMember[]
  assignedTasks Task[]
  comments      Comment[]
  notifications Notification[]
  sessions      Session[]

  @@index([email])
}

// ... 他のモデルも同様に定義
```

### 2.3 マイグレーション実行

```bash
# マイグレーション作成
npx prisma migrate dev --name init

# Prisma Client生成
npx prisma generate
```

## 3. Server Components実装例

### 3.1 プロジェクト一覧（Server Component）

```typescript
// app/(dashboard)/projects/page.tsx
import { Suspense } from "react";
import { ProjectList } from "@/components/server/projects/ProjectList";
import { ProjectListSkeleton } from "@/components/server/projects/ProjectListSkeleton";
import { CreateProjectButton } from "@/components/client/projects/CreateProjectButton";
import { auth } from "@/lib/auth/config";
import { redirect } from "next/navigation";

export default async function ProjectsPage({
  searchParams,
}: {
  searchParams: { status?: string; q?: string };
}) {
  // 認証チェック
  const session = await auth();
  if (!session?.user?.id) {
    redirect("/login");
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl font-bold">プロジェクト一覧</h1>
        <CreateProjectButton />
      </div>
      
      {/* Suspenseで囲むことで、独立したローディング状態を実現 */}
      <Suspense fallback={<ProjectListSkeleton />}>
        <ProjectList 
          userId={session.user.id}
          filter={{
            status: searchParams.status,
            search: searchParams.q,
          }}
        />
      </Suspense>
    </div>
  );
}
```

### 3.2 データフェッチ層の実装

```typescript
// lib/data/projects.ts
import "server-only"; // Server Componentでのみ実行を保証
import { prisma } from "@/lib/db/prisma";
import { unstable_cache } from "next/cache";
import { cache } from "react";

// プロジェクト一覧取得（並列フェッチ対応）
export const getProjectsWithStats = unstable_cache(
  async (userId: string, filter?: any) => {
    // 並列でデータを取得
    const [projects, stats] = await Promise.all([
      // プロジェクト一覧
      prisma.project.findMany({
        where: {
          members: {
            some: { userId },
          },
          ...(filter?.status && { status: filter.status }),
          ...(filter?.search && {
            OR: [
              { name: { contains: filter.search, mode: "insensitive" } },
              { description: { contains: filter.search, mode: "insensitive" } },
            ],
          }),
        },
        include: {
          members: {
            include: { user: true },
          },
          _count: {
            select: { tasks: true },
          },
        },
        orderBy: { updatedAt: "desc" },
      }),
      
      // 統計情報
      prisma.project.groupBy({
        by: ["status"],
        where: {
          members: {
            some: { userId },
          },
        },
        _count: true,
      }),
    ]);
    
    return { projects, stats };
  },
  ["projects-with-stats"],
  {
    revalidate: 300, // 5分間キャッシュ
    tags: ["projects"],
  }
);

// Preloadパターンの実装
export const preloadProjects = (userId: string, filter?: any) => {
  // voidで囲むことで、Promiseの返り値を無視
  void getProjectsWithStats(userId, filter);
};
```

### 3.3 カンバンボード実装（Server + Client Hybrid）

```typescript
// components/server/tasks/KanbanBoard.tsx
import "server-only";
import { getTasksByProject } from "@/lib/data/tasks";
import { TaskColumn } from "./TaskColumn";
import { DragDropProvider } from "@/components/client/providers/DragDropProvider";

const COLUMNS = [
  { id: "todo", label: "未着手", color: "bg-gray-100" },
  { id: "in_progress", label: "進行中", color: "bg-blue-100" },
  { id: "review", label: "レビュー待ち", color: "bg-yellow-100" },
  { id: "done", label: "完了", color: "bg-green-100" },
] as const;

interface KanbanBoardProps {
  projectId: string;
}

export async function KanbanBoard({ projectId }: KanbanBoardProps) {
  // 各カラムのタスクを並列取得
  const tasksPromises = COLUMNS.map(column =>
    getTasksByProject(projectId, { status: column.id })
  );
  
  const tasksByColumn = await Promise.all(tasksPromises);
  
  return (
    <DragDropProvider>
      <div className="flex gap-4 overflow-x-auto pb-4">
        {COLUMNS.map((column, index) => (
          <TaskColumn
            key={column.id}
            status={column.id}
            label={column.label}
            color={column.color}
            tasks={tasksByColumn[index]}
            projectId={projectId}
          />
        ))}
      </div>
    </DragDropProvider>
  );
}
```

## 4. Server Actions実装例

### 4.1 タスク作成アクション

```typescript
// lib/actions/tasks.ts
"use server";

import { z } from "zod";
import { revalidatePath, revalidateTag } from "next/cache";
import { prisma } from "@/lib/db/prisma";
import { auth } from "@/lib/auth/config";
import { metrics } from "@/lib/monitoring/metrics";

// バリデーションスキーマ
const CreateTaskSchema = z.object({
  title: z.string().min(1, "タイトルは必須です").max(200),
  description: z.string().max(2000).optional(),
  projectId: z.string().uuid("有効なプロジェクトIDを指定してください"),
  assigneeId: z.string().uuid().optional(),
  priority: z.enum(["low", "medium", "high"]),
  deadline: z.string().datetime().optional(),
});

export async function createTask(prevState: any, formData: FormData) {
  // パフォーマンス計測
  return metrics.trackServerAction("createTask", async () => {
    // 認証チェック
    const session = await auth();
    if (!session?.user?.id) {
      return { success: false, error: "認証が必要です" };
    }
    
    // バリデーション
    const validatedFields = CreateTaskSchema.safeParse({
      title: formData.get("title"),
      description: formData.get("description"),
      projectId: formData.get("projectId"),
      assigneeId: formData.get("assigneeId"),
      priority: formData.get("priority"),
      deadline: formData.get("deadline"),
    });
    
    if (!validatedFields.success) {
      return {
        success: false,
        errors: validatedFields.error.flatten().fieldErrors,
      };
    }
    
    try {
      // トランザクション処理
      const result = await prisma.$transaction(async (tx) => {
        // タスク作成
        const task = await tx.task.create({
          data: {
            ...validatedFields.data,
            status: "todo",
            progress: 0,
            position: await getNextPosition(tx, validatedFields.data.projectId, "todo"),
          },
          include: {
            assignee: true,
            project: true,
          },
        });
        
        // 通知作成（担当者がいる場合）
        if (task.assigneeId && task.assigneeId !== session.user.id) {
          await tx.notification.create({
            data: {
              userId: task.assigneeId,
              type: "TASK_ASSIGNED",
              title: "新しいタスクが割り当てられました",
              message: `「${task.title}」が割り当てられました`,
              relatedUrl: `/tasks/${task.id}`,
            },
          });
        }
        
        return task;
      });
      
      // キャッシュ無効化
      revalidateTag("tasks");
      revalidateTag(`project-${validatedFields.data.projectId}`);
      revalidatePath(`/projects/${validatedFields.data.projectId}`);
      
      return { success: true, data: result };
    } catch (error) {
      console.error("Task creation error:", error);
      return { success: false, error: "タスクの作成に失敗しました" };
    }
  });
}

// ポジション計算ヘルパー
async function getNextPosition(
  tx: any,
  projectId: string,
  status: string
): Promise<number> {
  const lastTask = await tx.task.findFirst({
    where: { projectId, status },
    orderBy: { position: "desc" },
    select: { position: true },
  });
  
  return (lastTask?.position ?? 0) + 1000;
}
```

### 4.2 useActionStateを使用したフォーム

```typescript
// components/client/forms/TaskForm.tsx
"use client";

import { useActionState, useEffect } from "react";
import { createTask } from "@/lib/actions/tasks";
import { Button } from "@/components/client/ui/Button";

interface TaskFormProps {
  projectId: string;
  assignees: Array<{ id: string; name: string }>;
}

export function TaskForm({ projectId, assignees }: TaskFormProps) {
  // Server Actionとの連携
  const [state, formAction, isPending] = useActionState(
    createTask,
    { success: false }
  );
  
  useEffect(() => {
    if (state.success) {
      // 成功時の処理（Toast表示など）
      console.log("Task created successfully");
    }
  }, [state.success]);
  
  return (
    <form action={formAction} className="space-y-4">
      <input type="hidden" name="projectId" value={projectId} />
      
      <div>
        <label htmlFor="title" className="block text-sm font-medium mb-1">
          タスク名 <span className="text-red-500">*</span>
        </label>
        <input
          id="title"
          name="title"
          type="text"
          required
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          disabled={isPending}
        />
        {state.errors?.title && (
          <p className="text-red-500 text-sm mt-1">
            {state.errors.title[0]}
          </p>
        )}
      </div>
      
      <div>
        <label htmlFor="description" className="block text-sm font-medium mb-1">
          説明
        </label>
        <textarea
          id="description"
          name="description"
          rows={4}
          className="w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          disabled={isPending}
        />
      </div>
      
      <div className="grid grid-cols-2 gap-4">
        <div>
          <label htmlFor="assigneeId" className="block text-sm font-medium mb-1">
            担当者
          </label>
          <select
            id="assigneeId"
            name="assigneeId"
            className="w-full px-3 py-2 border rounded-md"
            disabled={isPending}
          >
            <option value="">未割り当て</option>
            {assignees.map((user) => (
              <option key={user.id} value={user.id}>
                {user.name}
              </option>
            ))}
          </select>
        </div>
        
        <div>
          <label htmlFor="priority" className="block text-sm font-medium mb-1">
            優先度
          </label>
          <select
            id="priority"
            name="priority"
            defaultValue="medium"
            className="w-full px-3 py-2 border rounded-md"
            disabled={isPending}
          >
            <option value="low">低</option>
            <option value="medium">中</option>
            <option value="high">高</option>
          </select>
        </div>
      </div>
      
      <Button
        type="submit"
        disabled={isPending}
        className="w-full"
      >
        {isPending ? "作成中..." : "タスクを作成"}
      </Button>
    </form>
  );
}
```

## 5. パフォーマンス最適化実装

### 5.1 並列データフェッチの実装

```typescript
// app/(dashboard)/projects/[projectId]/page.tsx
import { Suspense } from "react";
import { preloadProjectTasks, preloadProjectMembers } from "@/lib/data/preload";
import { ProjectHeader } from "@/components/server/projects/ProjectHeader";
import { TaskBoard } from "@/components/server/tasks/TaskBoard";
import { ProjectStats } from "@/components/server/projects/ProjectStats";
import { ProjectActivity } from "@/components/server/projects/ProjectActivity";

export default async function ProjectDetailPage({
  params: { projectId },
}: {
  params: { projectId: string };
}) {
  // Preloadパターン - データフェッチを先行開始
  preloadProjectTasks(projectId);
  preloadProjectMembers(projectId);
  
  return (
    <div className="container mx-auto px-4 py-8">
      {/* ヘッダーは独立してレンダリング */}
      <ProjectHeader projectId={projectId} />
      
      <div className="grid grid-cols-1 lg:grid-cols-4 gap-6 mt-6">
        {/* メインコンテンツ */}
        <div className="lg:col-span-3">
          <Suspense fallback={<div className="animate-pulse bg-gray-200 h-96 rounded" />}>
            <TaskBoard projectId={projectId} />
          </Suspense>
        </div>
        
        {/* サイドバー - 独立した Suspense 境界 */}
        <div className="space-y-6">
          <Suspense fallback={<div className="animate-pulse bg-gray-200 h-48 rounded" />}>
            <ProjectStats projectId={projectId} />
          </Suspense>
          
          <Suspense fallback={<div className="animate-pulse bg-gray-200 h-64 rounded" />}>
            <ProjectActivity projectId={projectId} />
          </Suspense>
        </div>
      </div>
    </div>
  );
}
```

### 5.2 キャッシュ戦略の実装

```typescript
// lib/cache/strategies.ts
import { unstable_cache } from "next/cache";

// 静的コンテンツ（変更頻度：低）
export const createStaticCache = <T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyParts: string[]
): T => {
  return unstable_cache(fn, keyParts, {
    revalidate: false, // 静的キャッシュ
    tags: [...keyParts, "static"],
  }) as T;
};

// 頻繁に更新されるデータ（変更頻度：高）
export const createDynamicCache = <T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyParts: string[]
): T => {
  return unstable_cache(fn, keyParts, {
    revalidate: 60, // 1分キャッシュ
    tags: [...keyParts, "dynamic"],
  }) as T;
};

// ユーザー固有データ（変更頻度：中）
export const createUserCache = <T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyParts: string[],
  userId: string
): T => {
  return unstable_cache(fn, [...keyParts, userId], {
    revalidate: 300, // 5分キャッシュ
    tags: [...keyParts, `user-${userId}`],
  }) as T;
};

// ISR（Incremental Static Regeneration）対応
export const createISRCache = <T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyParts: string[],
  revalidateSeconds: number = 3600
): T => {
  return unstable_cache(fn, keyParts, {
    revalidate: revalidateSeconds,
    tags: [...keyParts, "isr"],
  }) as T;
};
```

## 6. エラーハンドリング実装

### 6.1 グローバルエラーハンドリング

```typescript
// app/error.tsx
"use client";

import { useEffect } from "react";
import { Button } from "@/components/client/ui/Button";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // エラーロギング
    console.error("Application error:", error);
    
    // 本番環境ではSentryに送信
    if (process.env.NODE_ENV === "production") {
      // Sentry.captureException(error);
    }
  }, [error]);
  
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full bg-white shadow-lg rounded-lg p-6">
        <div className="flex items-center justify-center w-12 h-12 mx-auto bg-red-100 rounded-full">
          <svg
            className="w-6 h-6 text-red-600"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              strokeLinecap="round"
              strokeLinejoin="round"
              strokeWidth={2}
              d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
            />
          </svg>
        </div>
        
        <h2 className="mt-4 text-xl font-semibold text-center text-gray-900">
          エラーが発生しました
        </h2>
        
        <p className="mt-2 text-sm text-center text-gray-600">
          申し訳ございません。予期しないエラーが発生しました。
        </p>
        
        {process.env.NODE_ENV === "development" && (
          <details className="mt-4 p-4 bg-gray-100 rounded text-xs">
            <summary className="cursor-pointer font-medium">
              エラー詳細（開発環境のみ）
            </summary>
            <pre className="mt-2 whitespace-pre-wrap break-all">
              {error.message}
              {error.stack}
            </pre>
          </details>
        )}
        
        <div className="mt-6 flex gap-3">
          <Button
            onClick={reset}
            className="flex-1"
          >
            再試行
          </Button>
          
          <Button
            onClick={() => window.location.href = "/"}
            variant="outline"
            className="flex-1"
          >
            ホームに戻る
          </Button>
        </div>
      </div>
    </div>
  );
}
```

## 7. 学習のポイントと実装上の注意

### 7.1 Server Components使用時の注意点

```typescript
// ❌ 悪い例：Server ComponentでuseStateを使用
export async function BadServerComponent() {
  const [count, setCount] = useState(0); // エラー！
  return <div>{count}</div>;
}

// ✅ 良い例：必要な部分だけClient Component
export async function GoodServerComponent() {
  const data = await fetchData(); // Server側でデータ取得
  
  return (
    <div>
      <h1>{data.title}</h1>
      <InteractiveCounter /> {/* Client Component */}
    </div>
  );
}
```

### 7.2 並列フェッチのパターン

```typescript
// ❌ 悪い例：直列フェッチ
const user = await getUser(userId);
const projects = await getProjects(user.id);
const tasks = await getTasks(projects[0].id);

// ✅ 良い例：並列フェッチ
const [user, projects, tasks] = await Promise.all([
  getUser(userId),
  getProjects(userId),
  getTasks(projectId),
]);

// ✅ より良い例：Suspenseで分割
<Suspense fallback={<UserSkeleton />}>
  <UserInfo userId={userId} />
</Suspense>
<Suspense fallback={<ProjectsSkeleton />}>
  <ProjectsList userId={userId} />
</Suspense>
```

### 7.3 キャッシュ戦略の選択

```typescript
// 静的コンテンツ → revalidate: false
export const getStaticContent = unstable_cache(
  async () => { /* ... */ },
  ["static-content"],
  { revalidate: false }
);

// 動的コンテンツ → 短いrevalidate
export const getDynamicContent = unstable_cache(
  async () => { /* ... */ },
  ["dynamic-content"],
  { revalidate: 60 } // 1分
);

// ユーザー固有 → タグベースの無効化
export const getUserContent = unstable_cache(
  async (userId: string) => { /* ... */ },
  ["user-content"],
  { 
    revalidate: 300,
    tags: [`user-${userId}`]
  }
);
```

## 8. 開発ワークフロー

### 8.1 推奨開発手順

1. **要件確認**: 要件定義書を読み、機能の理解
2. **設計確認**: 基本設計書・詳細設計書で実装方針を確認
3. **実装**:
   - データモデル作成（Prisma）
   - Server Component作成
   - データフェッチ層実装
   - Client Component作成（必要な部分のみ）
   - Server Actions実装
4. **テスト**: ユニットテスト → 統合テスト → E2Eテスト
5. **最適化**: パフォーマンス測定と改善

### 8.2 デバッグTips

```typescript
// Server Component のデバッグ
console.log("[Server]", data); // サーバーログに出力

// Client Component のデバッグ
useEffect(() => {
  console.log("[Client]", data); // ブラウザコンソールに出力
}, [data]);

// Server Action のデバッグ
export async function myAction() {
  console.log("[Action Start]");
  try {
    // 処理
    console.log("[Action Success]");
  } catch (error) {
    console.error("[Action Error]", error);
  }
}
```

## まとめ

このプロジェクトを通じて、以下の技術を実践的に習得できます：

1. **Server Components**: サーバーサイドでのレンダリングとデータフェッチ
2. **並列データフェッチ**: Promise.all、Suspense、Preloadパターン
3. **キャッシュ戦略**: Static/Dynamic Rendering、revalidate設定
4. **Server Actions**: フォーム処理の簡素化
5. **TypeScript**: 型安全な開発
6. **最適化**: パフォーマンスとUXの向上

段階的に実装を進めることで、Modern React開発の実践的なスキルを身につけることができます。
