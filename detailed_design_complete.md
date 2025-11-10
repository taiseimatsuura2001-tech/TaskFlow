# 詳細設計書 - TaskFlow

## 1. コンポーネント詳細設計

### 1.1 Server Components詳細

#### 1.1.1 ProjectList Component

```typescript
// /components/server/projects/ProjectList.tsx
import "server-only";
import { Suspense } from "react";
import { getProjects } from "@/lib/data/projects";
import { ProjectCard } from "./ProjectCard";
import { ProjectListSkeleton } from "./ProjectListSkeleton";

interface ProjectListProps {
  userId: string;
  filter?: {
    status?: string;
    search?: string;
  };
  sort?: {
    field: "createdAt" | "updatedAt" | "deadline";
    order: "asc" | "desc";
  };
}

export async function ProjectList({ 
  userId, 
  filter, 
  sort = { field: "updatedAt", order: "desc" } 
}: ProjectListProps) {
  // データフェッチ層を使用
  const projects = await getProjects(userId, filter, sort);
  
  if (projects.length === 0) {
    return (
      <div className="text-center py-8 text-gray-500">
        プロジェクトが見つかりません
      </div>
    );
  }
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {projects.map((project) => (
        <Suspense key={project.id} fallback={<ProjectCardSkeleton />}>
          <ProjectCard project={project} />
        </Suspense>
      ))}
    </div>
  );
}
```

#### 1.1.2 TaskBoard Component (Kanban)

```typescript
// /components/server/tasks/TaskBoard.tsx
import "server-only";
import { getTasksByProject } from "@/lib/data/tasks";
import { TaskColumn } from "./TaskColumn";
import { DragDropProvider } from "@/components/client/providers/DragDropProvider";

const TASK_STATUSES = [
  { id: "todo", label: "未着手", color: "gray" },
  { id: "in_progress", label: "進行中", color: "blue" },
  { id: "review", label: "レビュー待ち", color: "yellow" },
  { id: "done", label: "完了", color: "green" },
] as const;

interface TaskBoardProps {
  projectId: string;
}

export async function TaskBoard({ projectId }: TaskBoardProps) {
  // 並列データフェッチ
  const tasksByStatus = await Promise.all(
    TASK_STATUSES.map(async (status) => ({
      status: status.id,
      label: status.label,
      color: status.color,
      tasks: await getTasksByProject(projectId, { status: status.id }),
    }))
  );
  
  return (
    <DragDropProvider>
      <div className="flex gap-4 overflow-x-auto pb-4">
        {tasksByStatus.map((column) => (
          <TaskColumn
            key={column.status}
            status={column.status}
            label={column.label}
            color={column.color}
            tasks={column.tasks}
            projectId={projectId}
          />
        ))}
      </div>
    </DragDropProvider>
  );
}
```

#### 1.1.3 TaskDetail Component

```typescript
// /components/server/tasks/TaskDetail.tsx
import "server-only";
import { Suspense } from "react";
import { getTask, getTaskComments } from "@/lib/data/tasks";
import { TaskEditForm } from "@/components/client/forms/TaskEditForm";
import { CommentSection } from "./CommentSection";
import { AttachmentList } from "./AttachmentList";
import { TaskActivity } from "./TaskActivity";

interface TaskDetailProps {
  taskId: string;
}

export async function TaskDetail({ taskId }: TaskDetailProps) {
  // Preloadパターンでコメントを先読み
  const commentsPromise = getTaskComments(taskId);
  
  // メインのタスクデータを取得
  const task = await getTask(taskId);
  
  if (!task) {
    return <div>タスクが見つかりません</div>;
  }
  
  return (
    <div className="space-y-6">
      {/* タスク編集フォーム（Client Component） */}
      <TaskEditForm task={task} />
      
      {/* 添付ファイル一覧 */}
      <Suspense fallback={<div>ファイル読み込み中...</div>}>
        <AttachmentList taskId={taskId} />
      </Suspense>
      
      {/* コメントセクション（並列フェッチ） */}
      <Suspense fallback={<div>コメント読み込み中...</div>}>
        <CommentSection commentsPromise={commentsPromise} taskId={taskId} />
      </Suspense>
      
      {/* アクティビティログ */}
      <Suspense fallback={<div>履歴読み込み中...</div>}>
        <TaskActivity taskId={taskId} />
      </Suspense>
    </div>
  );
}
```

### 1.2 Client Components詳細

#### 1.2.1 TaskForm Component (with Server Action)

```typescript
// /components/client/forms/TaskForm.tsx
"use client";

import { useActionState } from "react";
import { createTask } from "@/lib/actions/tasks";
import { Button } from "@/components/client/ui/Button";
import { Input } from "@/components/client/ui/Input";
import { Select } from "@/components/client/ui/Select";
import { useToast } from "@/hooks/useToast";

interface TaskFormProps {
  projectId: string;
  assignees: Array<{ id: string; name: string }>;
}

export function TaskForm({ projectId, assignees }: TaskFormProps) {
  const { toast } = useToast();
  const [state, formAction, isPending] = useActionState(
    createTask,
    { success: false }
  );
  
  // エラー処理
  useEffect(() => {
    if (state.success) {
      toast({ title: "タスクを作成しました", type: "success" });
    } else if (state.error) {
      toast({ title: state.error, type: "error" });
    }
  }, [state]);
  
  return (
    <form action={formAction} className="space-y-4">
      <input type="hidden" name="projectId" value={projectId} />
      
      <div>
        <label htmlFor="title" className="block text-sm font-medium mb-1">
          タスク名 *
        </label>
        <Input
          id="title"
          name="title"
          required
          placeholder="タスクのタイトルを入力"
          disabled={isPending}
        />
        {state.errors?.title && (
          <p className="text-red-500 text-sm mt-1">{state.errors.title[0]}</p>
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
          className="w-full border rounded-md p-2"
          placeholder="タスクの詳細を入力"
          disabled={isPending}
        />
      </div>
      
      <div className="grid grid-cols-2 gap-4">
        <div>
          <label htmlFor="assigneeId" className="block text-sm font-medium mb-1">
            担当者
          </label>
          <Select
            id="assigneeId"
            name="assigneeId"
            disabled={isPending}
          >
            <option value="">未割り当て</option>
            {assignees.map((user) => (
              <option key={user.id} value={user.id}>
                {user.name}
              </option>
            ))}
          </Select>
        </div>
        
        <div>
          <label htmlFor="priority" className="block text-sm font-medium mb-1">
            優先度
          </label>
          <Select
            id="priority"
            name="priority"
            defaultValue="medium"
            disabled={isPending}
          >
            <option value="low">低</option>
            <option value="medium">中</option>
            <option value="high">高</option>
          </Select>
        </div>
      </div>
      
      <div>
        <label htmlFor="deadline" className="block text-sm font-medium mb-1">
          期限
        </label>
        <Input
          type="datetime-local"
          id="deadline"
          name="deadline"
          disabled={isPending}
        />
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

#### 1.2.2 DragDropArea Component

```typescript
// /components/client/interactive/DragDropArea.tsx
"use client";

import { useState, useCallback } from "react";
import { useDragDropContext } from "@/hooks/useDragDropContext";
import { updateTaskStatus } from "@/lib/actions/tasks";
import { useOptimistic } from "react";

interface DragDropAreaProps {
  status: string;
  tasks: Array<{ id: string; title: string; position: number }>;
  children: React.ReactNode;
}

export function DragDropArea({ status, tasks, children }: DragDropAreaProps) {
  const [isDragOver, setIsDragOver] = useState(false);
  const { draggedTask, setDraggedTask } = useDragDropContext();
  
  // Optimistic UI更新
  const [optimisticTasks, setOptimisticTasks] = useOptimistic(
    tasks,
    (state, newTask) => {
      return [...state.filter(t => t.id !== newTask.id), newTask]
        .sort((a, b) => a.position - b.position);
    }
  );
  
  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragOver(true);
  }, []);
  
  const handleDragLeave = useCallback(() => {
    setIsDragOver(false);
  }, []);
  
  const handleDrop = useCallback(async (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragOver(false);
    
    if (!draggedTask) return;
    
    // Optimistic update
    setOptimisticTasks({
      ...draggedTask,
      status,
      position: optimisticTasks.length,
    });
    
    // Server Action呼び出し
    const result = await updateTaskStatus(
      draggedTask.id,
      status,
      optimisticTasks.length
    );
    
    if (!result.success) {
      // エラー時はロールバック
      console.error("Failed to update task status");
    }
    
    setDraggedTask(null);
  }, [draggedTask, status, optimisticTasks]);
  
  return (
    <div
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      onDrop={handleDrop}
      className={`
        min-h-[400px] p-2 rounded-lg transition-colors
        ${isDragOver ? "bg-blue-50 border-blue-300" : "bg-gray-50"}
      `}
    >
      {children}
    </div>
  );
}
```

#### 1.2.3 SearchBar Component

```typescript
// /components/client/interactive/SearchBar.tsx
"use client";

import { useState, useCallback, useTransition } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { useDebouncedCallback } from "@/hooks/useDebounce";
import { SearchIcon } from "lucide-react";

export function SearchBar() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState(searchParams.get("q") || "");
  
  // 検索実行（デバウンス付き）
  const handleSearch = useDebouncedCallback(
    useCallback((value: string) => {
      const params = new URLSearchParams(searchParams);
      
      if (value) {
        params.set("q", value);
      } else {
        params.delete("q");
      }
      
      startTransition(() => {
        router.push(`/search?${params.toString()}`);
      });
    }, [router, searchParams]),
    300 // 300ms debounce
  );
  
  return (
    <div className="relative">
      <SearchIcon className="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400 w-4 h-4" />
      <input
        type="search"
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
          handleSearch(e.target.value);
        }}
        placeholder="検索..."
        className={`
          pl-10 pr-4 py-2 rounded-lg border
          ${isPending ? "bg-gray-100" : "bg-white"}
          focus:outline-none focus:ring-2 focus:ring-blue-500
        `}
        disabled={isPending}
      />
      {isPending && (
        <div className="absolute right-3 top-1/2 -translate-y-1/2">
          <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-gray-900" />
        </div>
      )}
    </div>
  );
}
```

## 2. データフェッチ層詳細設計

### 2.1 データフェッチ層の実装

```typescript
// /lib/data/projects.ts
import "server-only";
import { prisma } from "@/lib/db/prisma";
import { unstable_cache } from "next/cache";
import { cache } from "react";

// プロジェクト一覧取得（キャッシュあり）
export const getProjects = unstable_cache(
  async (
    userId: string,
    filter?: { status?: string; search?: string },
    sort?: { field: string; order: "asc" | "desc" }
  ) => {
    const where: any = {
      members: {
        some: { userId },
      },
    };
    
    if (filter?.status) {
      where.status = filter.status;
    }
    
    if (filter?.search) {
      where.OR = [
        { name: { contains: filter.search, mode: "insensitive" } },
        { description: { contains: filter.search, mode: "insensitive" } },
      ];
    }
    
    return prisma.project.findMany({
      where,
      include: {
        members: {
          include: { user: true },
        },
        _count: {
          select: { tasks: true },
        },
      },
      orderBy: sort ? { [sort.field]: sort.order } : { updatedAt: "desc" },
    });
  },
  ["projects"],
  {
    revalidate: 300, // 5分キャッシュ
    tags: ["project"],
  }
);

// 個別プロジェクト取得（Request Memoization）
export const getProject = cache(async (projectId: string) => {
  return prisma.project.findUnique({
    where: { id: projectId },
    include: {
      members: {
        include: { user: true },
      },
    },
  });
});

// プロジェクト統計取得
export const getProjectStats = cache(async (projectId: string) => {
  const [taskStats, memberStats, activityStats] = await Promise.all([
    // タスク統計
    prisma.task.groupBy({
      by: ["status"],
      where: { projectId },
      _count: true,
    }),
    
    // メンバー別タスク数
    prisma.task.groupBy({
      by: ["assigneeId"],
      where: { projectId, assigneeId: { not: null } },
      _count: true,
    }),
    
    // 直近のアクティビティ
    prisma.task.findMany({
      where: { projectId },
      orderBy: { updatedAt: "desc" },
      take: 10,
      select: {
        id: true,
        title: true,
        updatedAt: true,
        assignee: {
          select: { name: true, avatarUrl: true },
        },
      },
    }),
  ]);
  
  return {
    tasks: taskStats,
    members: memberStats,
    recentActivity: activityStats,
  };
});

// Preloadパターンの実装
export const preloadProject = (projectId: string) => {
  void getProject(projectId);
  void getProjectStats(projectId);
};
```

### 2.2 タスクデータフェッチ層

```typescript
// /lib/data/tasks.ts
import "server-only";
import { prisma } from "@/lib/db/prisma";
import { unstable_cache } from "next/cache";
import { cache } from "react";

// タスク一覧取得（ステータス別）
export const getTasksByProject = cache(
  async (projectId: string, filter?: { status?: string }) => {
    const where: any = { projectId };
    
    if (filter?.status) {
      where.status = filter.status;
    }
    
    return prisma.task.findMany({
      where,
      include: {
        assignee: {
          select: { id: true, name: true, avatarUrl: true },
        },
        labels: {
          include: {
            label: true,
          },
        },
        _count: {
          select: { comments: true, attachments: true },
        },
      },
      orderBy: { position: "asc" },
    });
  }
);

// タスク詳細取得
export const getTask = unstable_cache(
  async (taskId: string) => {
    return prisma.task.findUnique({
      where: { id: taskId },
      include: {
        project: true,
        assignee: true,
        labels: {
          include: { label: true },
        },
        attachments: true,
        parentTask: true,
        childTasks: true,
        relatedTasks: {
          include: {
            relatedTask: true,
          },
        },
      },
    });
  },
  ["task-detail"],
  {
    revalidate: 60,
    tags: ["task"],
  }
);

// コメント取得（並列フェッチ対応）
export const getTaskComments = cache(async (taskId: string) => {
  return prisma.comment.findMany({
    where: { taskId },
    include: {
      user: {
        select: { id: true, name: true, avatarUrl: true },
      },
    },
    orderBy: { createdAt: "desc" },
  });
});

// 複数タスクの並列取得
export const getMultipleTasks = cache(async (taskIds: string[]) => {
  const tasks = await Promise.all(
    taskIds.map(id => getTask(id))
  );
  return tasks.filter(Boolean);
});

// タスク検索
export const searchTasks = unstable_cache(
  async (query: string, filters?: any) => {
    return prisma.task.findMany({
      where: {
        OR: [
          { title: { contains: query, mode: "insensitive" } },
          { description: { contains: query, mode: "insensitive" } },
        ],
        ...filters,
      },
      include: {
        project: true,
        assignee: true,
      },
      take: 50,
    });
  },
  ["task-search"],
  {
    revalidate: 180, // 3分キャッシュ
  }
);
```

## 3. Server Actions詳細設計

### 3.1 プロジェクト関連アクション

```typescript
// /lib/actions/projects.ts
"use server";

import { z } from "zod";
import { revalidatePath, revalidateTag } from "next/cache";
import { prisma } from "@/lib/db/prisma";
import { auth } from "@/lib/auth/config";
import { redirect } from "next/navigation";

// バリデーションスキーマ
const ProjectSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(500).optional(),
  deadline: z.string().datetime().optional(),
});

// プロジェクト作成
export async function createProject(prevState: any, formData: FormData) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  const validatedFields = ProjectSchema.safeParse({
    name: formData.get("name"),
    description: formData.get("description"),
    deadline: formData.get("deadline"),
  });
  
  if (!validatedFields.success) {
    return {
      success: false,
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }
  
  try {
    const project = await prisma.project.create({
      data: {
        ...validatedFields.data,
        status: "active",
        members: {
          create: {
            userId: session.user.id,
            role: "owner",
          },
        },
      },
    });
    
    revalidateTag("project");
    revalidatePath("/projects");
    
    redirect(`/projects/${project.id}`);
  } catch (error) {
    return { success: false, error: "プロジェクトの作成に失敗しました" };
  }
}

// プロジェクトメンバー招待
export async function inviteProjectMember(
  projectId: string,
  email: string,
  role: "admin" | "member" | "viewer"
) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  // 権限チェック
  const membership = await prisma.projectMember.findUnique({
    where: {
      projectId_userId: {
        projectId,
        userId: session.user.id,
      },
    },
  });
  
  if (!membership || membership.role === "viewer") {
    return { success: false, error: "権限がありません" };
  }
  
  try {
    // ユーザー検索
    const user = await prisma.user.findUnique({
      where: { email },
    });
    
    if (!user) {
      return { success: false, error: "ユーザーが見つかりません" };
    }
    
    // メンバー追加
    await prisma.projectMember.create({
      data: {
        projectId,
        userId: user.id,
        role,
      },
    });
    
    // 通知作成
    await prisma.notification.create({
      data: {
        userId: user.id,
        type: "project_invite",
        title: "プロジェクトへの招待",
        message: `プロジェクトに招待されました`,
        relatedUrl: `/projects/${projectId}`,
      },
    });
    
    revalidateTag(`project-${projectId}`);
    
    return { success: true, data: { userId: user.id } };
  } catch (error) {
    return { success: false, error: "メンバーの追加に失敗しました" };
  }
}
```

### 3.2 タスク関連アクション

```typescript
// /lib/actions/tasks.ts
"use server";

import { z } from "zod";
import { auth } from "@/lib/auth/config";
import { prisma } from "@/lib/db/prisma";
import { revalidatePath, revalidateTag } from "next/cache";
import { put } from "@vercel/blob";

// タスク作成スキーマ
const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().max(2000).optional(),
  projectId: z.string().uuid(),
  assigneeId: z.string().uuid().optional(),
  priority: z.enum(["low", "medium", "high"]),
  deadline: z.string().datetime().optional(),
});

// タスク作成アクション
export async function createTask(formData: FormData) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
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
    const task = await prisma.task.create({
      data: {
        ...validatedFields.data,
        status: "todo",
        progress: 0,
      },
    });

    // キャッシュ無効化
    revalidateTag("task");
    revalidatePath(`/projects/${validatedFields.data.projectId}`);

    return { success: true, data: task };
  } catch (error) {
    return { success: false, error: "Failed to create task" };
  }
}

// タスクステータス更新アクション
export async function updateTaskStatus(
  taskId: string,
  status: string,
  position?: number
) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  try {
    const task = await prisma.task.update({
      where: { id: taskId },
      data: { 
        status,
        position,
        updatedAt: new Date(),
      },
    });

    // 関連キャッシュの無効化
    revalidateTag("task");
    revalidateTag(`task-${taskId}`);
    
    return { success: true, data: task };
  } catch (error) {
    return { success: false, error: "Failed to update task status" };
  }
}

// タスク更新スキーマ
const UpdateTaskSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  description: z.string().max(2000).optional(),
  status: z.enum(["todo", "in_progress", "review", "done"]).optional(),
  priority: z.enum(["low", "medium", "high"]).optional(),
  progress: z.number().min(0).max(100).optional(),
  assigneeId: z.string().uuid().nullable().optional(),
  deadline: z.string().datetime().nullable().optional(),
});

// タスク更新
export async function updateTask(taskId: string, data: any) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  const validatedFields = UpdateTaskSchema.safeParse(data);
  
  if (!validatedFields.success) {
    return {
      success: false,
      errors: validatedFields.error.flatten().fieldErrors,
    };
  }
  
  try {
    // 権限チェック
    const task = await prisma.task.findUnique({
      where: { id: taskId },
      include: {
        project: {
          include: {
            members: {
              where: { userId: session.user.id },
            },
          },
        },
      },
    });
    
    if (!task || task.project.members.length === 0) {
      return { success: false, error: "タスクが見つかりません" };
    }
    
    // 更新実行
    const updatedTask = await prisma.task.update({
      where: { id: taskId },
      data: validatedFields.data,
    });
    
    // 担当者変更時の通知
    if (validatedFields.data.assigneeId && 
        validatedFields.data.assigneeId !== task.assigneeId) {
      await prisma.notification.create({
        data: {
          userId: validatedFields.data.assigneeId,
          type: "TASK_ASSIGNED",
          title: "タスクが割り当てられました",
          message: `「${task.title}」が割り当てられました`,
          relatedUrl: `/tasks/${taskId}`,
        },
      });
    }
    
    // キャッシュ無効化
    revalidateTag("task");
    revalidateTag(`task-${taskId}`);
    revalidatePath(`/projects/${task.projectId}`);
    
    return { success: true, data: updatedTask };
  } catch (error) {
    return { success: false, error: "タスクの更新に失敗しました" };
  }
}

// コメント追加
export async function addComment(taskId: string, content: string) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  if (!content || content.trim().length === 0) {
    return { success: false, error: "コメントを入力してください" };
  }
  
  if (content.length > 2000) {
    return { success: false, error: "コメントは2000文字以内で入力してください" };
  }
  
  try {
    const comment = await prisma.comment.create({
      data: {
        taskId,
        userId: session.user.id,
        content: content.trim(),
      },
      include: {
        user: true,
      },
    });
    
    // メンション通知処理
    const mentions = content.match(/@(\w+)/g);
    if (mentions) {
      const usernames = mentions.map(m => m.substring(1));
      const users = await prisma.user.findMany({
        where: {
          name: { in: usernames },
        },
      });
      
      await Promise.all(
        users.map(user =>
          prisma.notification.create({
            data: {
              userId: user.id,
              type: "MENTION",
              title: "メンションされました",
              message: `${session.user.name}がコメントでメンションしました`,
              relatedUrl: `/tasks/${taskId}`,
            },
          })
        )
      );
    }
    
    revalidateTag(`task-${taskId}`);
    
    return { success: true, data: comment };
  } catch (error) {
    return { success: false, error: "コメントの投稿に失敗しました" };
  }
}

// ファイルアップロード
export async function uploadTaskAttachment(
  taskId: string,
  file: File
) {
  const session = await auth();
  
  if (!session?.user?.id) {
    return { success: false, error: "認証が必要です" };
  }
  
  // ファイルサイズチェック（5MB）
  if (file.size > 5 * 1024 * 1024) {
    return { success: false, error: "ファイルサイズは5MB以下にしてください" };
  }
  
  // MIMEタイプチェック
  const allowedTypes = [
    "image/jpeg", "image/png", "image/gif", "image/webp",
    "application/pdf", "text/plain", "text/csv",
    "application/msword", "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "application/vnd.ms-excel", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  ];
  
  if (!allowedTypes.includes(file.type)) {
    return { success: false, error: "対応していないファイル形式です" };
  }
  
  try {
    // Vercel Blob Storageにアップロード
    const blob = await put(file.name, file, {
      access: "public",
    });
    
    // データベースに記録
    const attachment = await prisma.taskAttachment.create({
      data: {
        taskId,
        fileName: file.name,
        fileUrl: blob.url,
        fileSize: file.size,
        mimeType: file.type,
      },
    });
    
    revalidateTag(`task-${taskId}`);
    
    return { success: true, data: attachment };
  } catch (error) {
    return { success: false, error: "ファイルのアップロードに失敗しました" };
  }
}
```

## 4. データベース詳細設計

### 4.1 Prisma Schema完全版

```prisma
// /lib/db/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ユーザーテーブル
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

  // Relations
  memberships    ProjectMember[]
  assignedTasks  Task[]
  comments       Comment[]
  notifications  Notification[]
  sessions       Session[]

  @@index([email])
}

// セッションテーブル（NextAuth用）
model Session {
  id           String   @id @default(cuid())
  userId       String
  expires      DateTime
  sessionToken String   @unique
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
}

// プロジェクトテーブル
model Project {
  id          String        @id @default(cuid())
  name        String
  description String?       @db.Text
  status      ProjectStatus @default(ACTIVE)
  deadline    DateTime?
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  // Relations
  members ProjectMember[]
  tasks   Task[]
  labels  Label[]

  @@index([status, updatedAt])
}

// プロジェクトメンバーテーブル
model ProjectMember {
  id        String      @id @default(cuid())
  projectId String
  userId    String
  role      MemberRole  @default(MEMBER)
  joinedAt  DateTime    @default(now())

  project Project @relation(fields: [projectId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([projectId, userId])
  @@index([userId])
  @@index([projectId])
}

// タスクテーブル
model Task {
  id           String       @id @default(cuid())
  projectId    String
  title        String
  description  String?      @db.Text
  status       TaskStatus   @default(TODO)
  priority     TaskPriority @default(MEDIUM)
  progress     Int          @default(0)
  position     Int          @default(0)
  assigneeId   String?
  parentTaskId String?
  deadline     DateTime?
  createdAt    DateTime     @default(now())
  updatedAt    DateTime     @updatedAt

  // Relations
  project      Project          @relation(fields: [projectId], references: [id], onDelete: Cascade)
  assignee     User?           @relation(fields: [assigneeId], references: [id], onDelete: SetNull)
  parentTask   Task?           @relation("TaskHierarchy", fields: [parentTaskId], references: [id])
  childTasks   Task[]          @relation("TaskHierarchy")
  comments     Comment[]
  attachments  TaskAttachment[]
  labels       TaskLabel[]
  relatedTasks TaskRelation[]  @relation("RelatedTask")
  relatedFrom  TaskRelation[]  @relation("RelatedFrom")

  @@index([projectId, status])
  @@index([assigneeId])
  @@index([parentTaskId])
  @@index([deadline])
}

// タスク関連テーブル
model TaskRelation {
  id             String       @id @default(cuid())
  taskId         String
  relatedTaskId  String
  relationType   RelationType

  task        Task @relation("RelatedTask", fields: [taskId], references: [id], onDelete: Cascade)
  relatedTask Task @relation("RelatedFrom", fields: [relatedTaskId], references: [id], onDelete: Cascade)

  @@unique([taskId, relatedTaskId])
  @@index([taskId])
  @@index([relatedTaskId])
}

// コメントテーブル
model Comment {
  id        String   @id @default(cuid())
  taskId    String
  userId    String
  content   String   @db.Text
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  task Task @relation(fields: [taskId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([taskId, createdAt])
  @@index([userId])
}

// 添付ファイルテーブル
model TaskAttachment {
  id         String   @id @default(cuid())
  taskId     String
  fileName   String
  fileUrl    String
  fileSize   Int
  mimeType   String
  uploadedAt DateTime @default(now())

  task Task @relation(fields: [taskId], references: [id], onDelete: Cascade)

  @@index([taskId])
}

// ラベルテーブル
model Label {
  id        String      @id @default(cuid())
  name      String
  color     String
  projectId String

  project Project     @relation(fields: [projectId], references: [id], onDelete: Cascade)
  tasks   TaskLabel[]

  @@unique([projectId, name])
  @@index([projectId])
}

// タスクラベル中間テーブル
model TaskLabel {
  taskId  String
  labelId String

  task  Task  @relation(fields: [taskId], references: [id], onDelete: Cascade)
  label Label @relation(fields: [labelId], references: [id], onDelete: Cascade)

  @@id([taskId, labelId])
  @@index([taskId])
  @@index([labelId])
}

// 通知テーブル
model Notification {
  id         String           @id @default(cuid())
  userId     String
  type       NotificationType
  title      String
  message    String
  relatedUrl String?
  isRead     Boolean          @default(false)
  createdAt  DateTime         @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, isRead, createdAt])
}

// Enum定義
enum UserRole {
  ADMIN
  MEMBER
  VIEWER
}

enum ProjectStatus {
  ACTIVE
  COMPLETED
  ARCHIVED
  ON_HOLD
}

enum MemberRole {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  REVIEW
  DONE
}

enum TaskPriority {
  LOW
  MEDIUM
  HIGH
}

enum RelationType {
  BLOCKS
  IS_BLOCKED_BY
  RELATES_TO
  DUPLICATES
}

enum NotificationType {
  TASK_ASSIGNED
  TASK_COMMENTED
  TASK_UPDATED
  PROJECT_INVITE
  MENTION
  DEADLINE_REMINDER
}
```

### 4.2 インデックス戦略

```sql
-- パフォーマンス最適化のための追加インデックス

-- 複合インデックス
CREATE INDEX idx_task_project_status_deadline 
ON "Task"("projectId", "status", "deadline");

CREATE INDEX idx_task_assignee_status 
ON "Task"("assigneeId", "status");

CREATE INDEX idx_notification_user_unread 
ON "Notification"("userId", "isRead", "createdAt" DESC);

-- 全文検索用インデックス
CREATE INDEX idx_task_search 
ON "Task" USING GIN(to_tsvector('english', "title" || ' ' || COALESCE("description", '')));

CREATE INDEX idx_project_search 
ON "Project" USING GIN(to_tsvector('english', "name" || ' ' || COALESCE("description", '')));

-- 統計クエリ用
CREATE INDEX idx_task_created_at 
ON "Task"("createdAt");

CREATE INDEX idx_task_updated_at 
ON "Task"("updatedAt" DESC);
```

## 5. 認証・認可詳細設計

### 5.1 NextAuth設定

```typescript
// /lib/auth/config.ts
import NextAuth from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { prisma } from "@/lib/db/prisma";
import bcrypt from "bcryptjs";
import { z } from "zod";

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: {
    strategy: "jwt",
    maxAge: 30 * 60, // 30分
  },
  pages: {
    signIn: "/login",
    error: "/auth/error",
  },
  providers: [
    CredentialsProvider({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        const validatedFields = LoginSchema.safeParse(credentials);
        
        if (!validatedFields.success) {
          return null;
        }
        
        const { email, password } = validatedFields.data;
        
        const user = await prisma.user.findUnique({
          where: { email },
        });
        
        if (!user || !user.password) {
          return null;
        }
        
        const passwordsMatch = await bcrypt.compare(password, user.password);
        
        if (!passwordsMatch) {
          return null;
        }
        
        return {
          id: user.id,
          email: user.email,
          name: user.name,
          image: user.avatarUrl,
        };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user, account }) {
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      if (token && session.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }
      return session;
    },
  },
});

// 認証ミドルウェア
export const authMiddleware = auth((req) => {
  const { pathname } = req.nextUrl;
  const isAuthenticated = !!req.auth;
  
  // 公開ルート
  const publicRoutes = ["/", "/login", "/register"];
  const isPublicRoute = publicRoutes.includes(pathname);
  
  if (!isAuthenticated && !isPublicRoute) {
    const url = new URL("/login", req.url);
    url.searchParams.set("callbackUrl", pathname);
    return Response.redirect(url);
  }
  
  return null;
});
```

## 6. エラー処理とロギング

### 6.1 エラー処理ユーティリティ

```typescript
// /lib/utils/error-handling.ts
import { ZodError } from "zod";

export class ApplicationError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number = 500
  ) {
    super(message);
    this.name = "ApplicationError";
  }
}

export function handleActionError(error: unknown) {
  console.error("Action error:", error);
  
  if (error instanceof ZodError) {
    return {
      success: false,
      error: "入力データが不正です",
      errors: error.flatten().fieldErrors,
    };
  }
  
  if (error instanceof ApplicationError) {
    return {
      success: false,
      error: error.message,
      code: error.code,
    };
  }
  
  if (error instanceof Error) {
    return {
      success: false,
      error: "予期しないエラーが発生しました",
      details: process.env.NODE_ENV === "development" ? error.message : undefined,
    };
  }
  
  return {
    success: false,
    error: "システムエラーが発生しました",
  };
}

// ロギングユーティリティ
export function logError(
  context: string,
  error: unknown,
  metadata?: Record<string, any>
) {
  const errorInfo = {
    timestamp: new Date().toISOString(),
    context,
    metadata,
    error: error instanceof Error ? {
      name: error.name,
      message: error.message,
      stack: error.stack,
    } : error,
  };
  
  // 本番環境ではSentryに送信
  if (process.env.NODE_ENV === "production") {
    // Sentry.captureException(error, { extra: metadata });
  }
  
  console.error(JSON.stringify(errorInfo, null, 2));
}
```

## 7. キャッシュ戦略詳細

### 7.1 キャッシュ設定

```typescript
// /lib/cache/config.ts
import { unstable_cache } from "next/cache";

export const CACHE_TAGS = {
  ALL_PROJECTS: "all-projects",
  PROJECT: (id: string) => `project-${id}`,
  ALL_TASKS: "all-tasks",
  TASK: (id: string) => `task-${id}`,
  USER: (id: string) => `user-${id}`,
  NOTIFICATIONS: (userId: string) => `notifications-${userId}`,
} as const;

export const CACHE_REVALIDATION = {
  STATIC: false as const,         // 静的コンテンツ
  FREQUENT: 60,                   // 1分
  NORMAL: 300,                     // 5分
  SLOW: 3600,                      // 1時間
  DAILY: 86400,                    // 1日
} as const;

// キャッシュヘルパー関数
export function createCachedFunction<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  keyParts: string[],
  options?: {
    revalidate?: number | false;
    tags?: string[];
  }
): T {
  return unstable_cache(fn, keyParts, {
    revalidate: options?.revalidate ?? CACHE_REVALIDATION.NORMAL,
    tags: options?.tags,
  }) as T;
}

// キャッシュ無効化ヘルパー
export async function invalidateCache(tags: string | string[]) {
  const { revalidateTag } = await import("next/cache");
  
  const tagArray = Array.isArray(tags) ? tags : [tags];
  tagArray.forEach(tag => revalidateTag(tag));
}
```

## 8. パフォーマンス最適化

### 8.1 画像最適化

```typescript
// /components/shared/OptimizedImage.tsx
import Image from "next/image";

interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  priority?: boolean;
  className?: string;
}

export function OptimizedImage({
  src,
  alt,
  width = 400,
  height = 300,
  priority = false,
  className = "",
}: OptimizedImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      priority={priority}
      className={className}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..."
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      quality={85}
    />
  );
}
```

### 8.2 Bundle分割戦略

```typescript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    optimizePackageImports: [
      "lucide-react",
      "@radix-ui/react-icons",
      "date-fns",
    ],
  },
  
  // チャンク分割の最適化
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.optimization.splitChunks = {
        chunks: "all",
        cacheGroups: {
          default: false,
          vendors: false,
          framework: {
            name: "framework",
            chunks: "all",
            test: /(?<!node_modules.*)[\\/]node_modules[\\/](react|react-dom|scheduler|prop-types|use-sync-external-store)[\\/]/,
            priority: 40,
            enforce: true,
          },
          lib: {
            test(module) {
              return module.size() > 160000 &&
                /node_modules[/\\]/.test(module.identifier());
            },
            name(module) {
              const hash = crypto.createHash("sha1");
              hash.update(module.identifier());
              return hash.digest("hex").substring(0, 8);
            },
            priority: 30,
            minChunks: 1,
            reuseExistingChunk: true,
          },
          commons: {
            name: "commons",
            chunks: "all",
            minChunks: 2,
            priority: 20,
          },
          shared: {
            name(module, chunks) {
              const hash = crypto
                .createHash("sha1")
                .update(chunks.reduce((acc, chunk) => acc + chunk.name, ""))
                .digest("hex");
              return hash;
            },
            priority: 10,
            minChunks: 2,
            reuseExistingChunk: true,
          },
        },
      };
    }
    
    return config;
  },
};

module.exports = nextConfig;
```

## 9〜14は詳細設計書（続き）ファイルに記載済みの内容と同じです

これで、詳細設計書の完全版が1つのファイルにまとまりました。全ての内容が含まれています。
