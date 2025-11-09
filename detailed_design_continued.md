# 詳細設計書 - TaskFlow（続き）

## 9. テスト仕様（続き）

### 9.1 ユニットテスト例（続き）

```typescript
// /tests/unit/actions/tasks.test.ts
import { createTask, updateTaskStatus } from "@/lib/actions/tasks";
import { prisma } from "@/lib/db/prisma";
import { auth } from "@/lib/auth/config";

jest.mock("@/lib/db/prisma");
jest.mock("@/lib/auth/config");

describe("Task Actions", () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  describe("createTask", () => {
    it("should create a task successfully", async () => {
      const mockSession = { user: { id: "user1" } };
      (auth as jest.Mock).mockResolvedValue(mockSession);
      
      const mockTask = {
        id: "task1",
        title: "Test Task",
        projectId: "project1",
      };
      
      (prisma.task.create as jest.Mock).mockResolvedValue(mockTask);
      
      const formData = new FormData();
      formData.append("title", "Test Task");
      formData.append("projectId", "project1");
      formData.append("priority", "medium");
      
      const result = await createTask(formData);
      
      expect(result.success).toBe(true);
      expect(result.data).toEqual(mockTask);
      expect(prisma.task.create).toHaveBeenCalledWith({
        data: expect.objectContaining({
          title: "Test Task",
          projectId: "project1",
          priority: "medium",
          status: "todo",
          progress: 0,
        }),
      });
    });
    
    it("should return error when validation fails", async () => {
      const mockSession = { user: { id: "user1" } };
      (auth as jest.Mock).mockResolvedValue(mockSession);
      
      const formData = new FormData();
      // タイトルなしで送信
      formData.append("projectId", "project1");
      
      const result = await createTask(formData);
      
      expect(result.success).toBe(false);
      expect(result.errors).toBeDefined();
      expect(prisma.task.create).not.toHaveBeenCalled();
    });
  });
});
```

### 9.2 統合テスト例

```typescript
// /tests/integration/projects.test.ts
import { GET } from "@/app/api/projects/route";
import { auth } from "@/lib/auth/config";
import { prisma } from "@/lib/db/prisma";

jest.mock("@/lib/auth/config");

describe("Projects API Integration", () => {
  beforeAll(async () => {
    // テストデータのセットアップ
    await prisma.user.create({
      data: {
        id: "test-user",
        email: "test@example.com",
        password: "hashed",
        name: "Test User",
      },
    });
    
    await prisma.project.createMany({
      data: [
        {
          id: "project1",
          name: "Test Project 1",
          status: "ACTIVE",
        },
        {
          id: "project2",
          name: "Test Project 2",
          status: "COMPLETED",
        },
      ],
    });
  });
  
  afterAll(async () => {
    // クリーンアップ
    await prisma.project.deleteMany();
    await prisma.user.deleteMany();
  });
  
  it("should fetch user projects", async () => {
    (auth as jest.Mock).mockResolvedValue({
      user: { id: "test-user" },
    });
    
    const request = new Request("http://localhost/api/projects");
    const response = await GET(request);
    const data = await response.json();
    
    expect(response.status).toBe(200);
    expect(data.projects).toHaveLength(2);
    expect(data.projects[0].name).toBe("Test Project 1");
  });
});
```

### 9.3 E2Eテスト例

```typescript
// /tests/e2e/task-management.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Task Management", () => {
  test.beforeEach(async ({ page }) => {
    // ログイン
    await page.goto("/login");
    await page.fill('input[name="email"]', "test@example.com");
    await page.fill('input[name="password"]', "testpass123");
    await page.click('button[type="submit"]');
    await page.waitForURL("/dashboard");
  });
  
  test("should create a new task", async ({ page }) => {
    await page.goto("/projects/test-project");
    
    // 新規タスクボタンをクリック
    await page.click('button:has-text("新規タスク")');
    
    // フォーム入力
    await page.fill('input[name="title"]', "E2E Test Task");
    await page.fill('textarea[name="description"]', "This is a test task");
    await page.selectOption('select[name="priority"]', "high");
    
    // 送信
    await page.click('button:has-text("タスクを作成")');
    
    // タスクが作成されたことを確認
    await expect(page.locator('text="E2E Test Task"')).toBeVisible();
  });
  
  test("should drag and drop task to change status", async ({ page }) => {
    await page.goto("/projects/test-project/tasks");
    
    // タスクカードを取得
    const taskCard = page.locator('[data-task-id="task1"]');
    const inProgressColumn = page.locator('[data-column="in_progress"]');
    
    // ドラッグ&ドロップ
    await taskCard.dragTo(inProgressColumn);
    
    // ステータスが更新されたことを確認
    await expect(inProgressColumn.locator('[data-task-id="task1"]')).toBeVisible();
  });
});
```

## 10. 監視・ロギング詳細

### 10.1 カスタムメトリクス収集

```typescript
// /lib/monitoring/metrics.ts
import { headers } from "next/headers";

interface PerformanceMetric {
  name: string;
  value: number;
  tags?: Record<string, string>;
}

class MetricsCollector {
  private metrics: PerformanceMetric[] = [];
  
  track(name: string, value: number, tags?: Record<string, string>) {
    this.metrics.push({
      name,
      value,
      tags: {
        ...tags,
        timestamp: new Date().toISOString(),
      },
    });
    
    // 本番環境では外部サービスに送信
    if (process.env.NODE_ENV === "production") {
      this.sendToAnalytics({ name, value, tags });
    }
  }
  
  async trackServerAction<T>(
    name: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const start = performance.now();
    
    try {
      const result = await fn();
      const duration = performance.now() - start;
      
      this.track(`server_action.${name}`, duration, {
        status: "success",
      });
      
      return result;
    } catch (error) {
      const duration = performance.now() - start;
      
      this.track(`server_action.${name}`, duration, {
        status: "error",
        error: error instanceof Error ? error.message : "unknown",
      });
      
      throw error;
    }
  }
  
  private sendToAnalytics(metric: PerformanceMetric) {
    // Vercel Analytics APIに送信
    fetch("https://vitals.vercel-insights.com/v1/vitals", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        dsn: process.env.VERCEL_ANALYTICS_ID,
        metrics: [metric],
      }),
    }).catch(console.error);
  }
}

export const metrics = new MetricsCollector();
```

### 10.2 エラートラッキング

```typescript
// /lib/monitoring/error-tracking.ts
import * as Sentry from "@sentry/nextjs";

export function initErrorTracking() {
  if (process.env.NODE_ENV === "production") {
    Sentry.init({
      dsn: process.env.SENTRY_DSN,
      environment: process.env.VERCEL_ENV || "production",
      tracesSampleRate: 0.1,
      beforeSend(event, hint) {
        // 個人情報をマスク
        if (event.user) {
          event.user = {
            id: event.user.id,
            // email等は送信しない
          };
        }
        return event;
      },
      integrations: [
        new Sentry.BrowserTracing(),
        new Sentry.Replay({
          maskAllText: true,
          blockAllMedia: true,
        }),
      ],
    });
  }
}

export function captureError(
  error: unknown,
  context?: Record<string, any>
) {
  console.error("Error captured:", error);
  
  if (process.env.NODE_ENV === "production") {
    Sentry.captureException(error, {
      extra: context,
    });
  }
}
```

## 11. デプロイメント設計

### 11.1 環境変数設定

```env
# .env.example
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/taskflow"

# Authentication
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-secret-here"

# Vercel Blob Storage
BLOB_READ_WRITE_TOKEN="vercel_blob_token"

# External Services
SENTRY_DSN="https://xxx@sentry.io/xxx"
VERCEL_ANALYTICS_ID="xxx"
SENDGRID_API_KEY="xxx"
REDIS_URL="redis://localhost:6379"

# Feature Flags
ENABLE_NOTIFICATIONS="true"
ENABLE_FILE_UPLOAD="true"
MAX_FILE_SIZE_MB="5"
```

### 11.2 CI/CDパイプライン

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "20"
          cache: "npm"
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run type check
        run: npm run type-check
      
      - name: Run linting
        run: npm run lint
      
      - name: Run tests
        run: npm run test:ci
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: "--prod"
```

## 12. 実装チェックリスト

### 12.1 Phase 1（MVP）チェックリスト

- [ ] **環境構築**
  - [ ] Next.js 14.2+のセットアップ
  - [ ] TypeScript設定
  - [ ] Tailwind CSS設定
  - [ ] ESLint/Prettier設定
  - [ ] PostgreSQLセットアップ
  - [ ] Prisma設定

- [ ] **認証システム**
  - [ ] NextAuth.js設定
  - [ ] ログイン画面
  - [ ] 新規登録画面
  - [ ] セッション管理
  - [ ] 認証ミドルウェア

- [ ] **基本UI構築**
  - [ ] レイアウトコンポーネント
  - [ ] ナビゲーション
  - [ ] ダッシュボード画面
  - [ ] エラーバウンダリ
  - [ ] Loading UI

- [ ] **プロジェクト機能**
  - [ ] プロジェクト一覧（Server Component）
  - [ ] プロジェクト作成（Server Action）
  - [ ] プロジェクト詳細
  - [ ] メンバー管理

- [ ] **タスク機能**
  - [ ] タスク一覧（Server Component）
  - [ ] タスク作成（Server Action）
  - [ ] タスク編集
  - [ ] カンバンボード（ドラッグ&ドロップ）
  - [ ] ステータス更新

- [ ] **データフェッチ最適化**
  - [ ] データフェッチ層の実装
  - [ ] 並列フェッチの実装
  - [ ] Suspense境界の設定
  - [ ] Preloadパターンの実装

### 12.2 Phase 2（機能拡張）チェックリスト

- [ ] **コメント機能**
  - [ ] コメント投稿（Server Action）
  - [ ] コメント一覧表示
  - [ ] メンション機能
  - [ ] Markdown対応

- [ ] **通知システム**
  - [ ] 通知データモデル
  - [ ] 通知一覧
  - [ ] リアルタイム通知（WebSocket）
  - [ ] メール通知

- [ ] **検索・フィルタ**
  - [ ] グローバル検索
  - [ ] 詳細フィルタ
  - [ ] 検索結果画面
  - [ ] フィルタ保存機能

- [ ] **ファイルアップロード**
  - [ ] Vercel Blob Storage設定
  - [ ] ファイルアップロードUI
  - [ ] ファイル一覧表示
  - [ ] ファイルダウンロード

### 12.3 Phase 3（最適化）チェックリスト

- [ ] **パフォーマンス最適化**
  - [ ] Static Rendering設定
  - [ ] キャッシュ戦略実装
  - [ ] 画像最適化
  - [ ] Bundle最適化
  - [ ] Core Web Vitals改善

- [ ] **レポート機能**
  - [ ] ダッシュボード統計
  - [ ] バーンダウンチャート
  - [ ] CSV/PDFエクスポート
  - [ ] カスタムレポート

- [ ] **外部連携**
  - [ ] GitHub連携
  - [ ] Slack連携
  - [ ] カレンダー連携

- [ ] **テスト・品質**
  - [ ] ユニットテスト（80%カバレッジ）
  - [ ] 統合テスト
  - [ ] E2Eテスト
  - [ ] パフォーマンステスト

- [ ] **運用準備**
  - [ ] エラー監視（Sentry）
  - [ ] アナリティクス設定
  - [ ] ログ収集
  - [ ] バックアップ設定
  - [ ] ドキュメント整備

## 13. トラブルシューティングガイド

### 13.1 よくある問題と解決策

| 問題 | 原因 | 解決策 |
|------|------|---------|
| Server Componentでエラー | Client専用APIの使用 | `"use client"`の追加、またはServer Component対応APIに変更 |
| Hydration Error | サーバーとクライアントの不一致 | 動的コンテンツをuseEffectで処理 |
| キャッシュが更新されない | revalidateの未設定 | revalidateTag/revalidatePathの追加 |
| Server Action失敗 | バリデーションエラー | Zodスキーマの確認、エラーハンドリング追加 |
| ドラッグ&ドロップ不具合 | Server Component内でのイベント | Client Componentでラップ |
| データフェッチが遅い | 直列フェッチ | Promise.allで並列化 |
| TypeScriptエラー | 型定義の不一致 | Prisma generateの実行、型定義の更新 |

## 14. まとめ

この詳細設計書では、React + Next.js + TypeScriptを使用したタスク管理システム「TaskFlow」の実装に必要な全ての技術仕様を定義しました。

### 重要な実装ポイント

1. **Server Components優先**: データフェッチとレンダリングをサーバーサイドで実行
2. **並列データフェッチ**: Promise.all、Suspense、Preloadパターンの活用
3. **キャッシュ戦略**: Static Rendering優先、適切なrevalidate設定
4. **コンポーネント分離**: Container/Presentationalパターンでテスタビリティ向上
5. **Server Actions**: フォーム処理とデータ更新の効率化
6. **最適化**: パフォーマンスとユーザー体験の継続的改善

これらのドキュメントを参照しながら、段階的に実装を進めることで、モダンなReact開発のベストプラクティスを実践的に学習できます。
