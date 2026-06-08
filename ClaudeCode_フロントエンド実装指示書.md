# 郵便物送付管理システム フロントエンド実装指示書
# Claude Code 専用ファイル

---

## このファイルの使い方

このファイルをClaude Codeに渡し、以下のように指示してください。

```
この指示書に従ってReact + TypeScriptのフロントエンドを実装してください。
まず「1. セットアップ」を実行し、次に「実装する画面」を指定してください。
```

---

## 1. セットアップ

### 1.1 プロジェクト作成

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
```

### 1.2 依存パッケージインストール

```bash
# スタイリング（DADS）
npm install tailwindcss @digital-go-jp/tailwind-theme-plugin postcss autoprefixer
npx tailwindcss init -p

# ルーティング
npm install react-router-dom

# APIデータ取得
npm install @tanstack/react-query axios

# フォーム・バリデーション
npm install react-hook-form @hookform/resolvers zod
```

### 1.3 DADSコンポーネントのコピー

```bash
# DADSのコードスニペットをクローン（npmパッケージではないため）
git clone https://github.com/digital-go-jp/design-system-example-components-react.git dads-temp
cp -r dads-temp/src/components ./src/components/dads
cp -r dads-temp/src/tokens ./src/tokens
rm -rf dads-temp
```

### 1.4 設定ファイル

**tailwind.config.js**
```js
export default {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      fontFamily: { mono: ['Noto Sans Mono'] },
    },
  },
  plugins: [require('@digital-go-jp/tailwind-theme-plugin')],
};
```

**src/index.css**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**vite.config.ts**
```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
  server: {
    proxy: {
      '/api': 'http://localhost:8080',  // Spring Boot
    },
  },
});
```

---

## 2. フォルダ構成

```
src/
├── components/
│   ├── dads/               ← DADSのコードスニペット（コピー済み）
│   ├── layout/
│   │   ├── AppLayout.tsx   ← 認証後の共通レイアウト
│   │   ├── Header.tsx
│   │   └── SideNav.tsx
│   └── ui/
│       ├── StatusBadge.tsx
│       └── ConfirmModal.tsx
├── pages/
│   ├── login/
│   │   └── LoginPage.tsx
│   ├── dashboard/
│   │   └── DashboardPage.tsx
│   ├── mail-sends/
│   │   ├── ByOfficePage.tsx
│   │   ├── CreatePage.tsx
│   │   └── HistoryPage.tsx
│   ├── users/
│   │   ├── UserListPage.tsx
│   │   ├── UserDetailPage.tsx
│   │   └── UserForm.tsx
│   ├── offices/
│   │   ├── OfficeListPage.tsx
│   │   └── OfficeForm.tsx
│   ├── staffs/
│   │   ├── StaffListPage.tsx
│   │   └── StaffForm.tsx
│   └── password/
│       └── ChangePasswordPage.tsx
├── api/
│   ├── client.ts           ← axiosインスタンス
│   ├── auth.ts
│   ├── dashboard.ts
│   ├── mailSends.ts
│   ├── users.ts
│   ├── offices.ts
│   └── staffs.ts
├── types/
│   ├── mailSend.ts
│   ├── user.ts
│   ├── office.ts
│   └── staff.ts
├── schemas/                ← Zodバリデーションスキーマ
│   ├── loginSchema.ts
│   ├── mailSendSchema.ts
│   ├── userSchema.ts
│   ├── officeSchema.ts
│   ├── staffSchema.ts
│   └── passwordSchema.ts
└── App.tsx
```

---

## 3. TypeScript 型定義

**src/types/mailSend.ts**
```ts
export type SendType = 'PLAN' | 'MONITORING';
export type SendStatus = 'PENDING' | 'SENT' | 'DONE';

export type MailSend = {
  id: number;
  userId: number;
  userName: string;
  officeId: number;
  officeName: string;
  sendType: SendType;
  sendMonth: string;       // "2025-05"形式
  status: SendStatus;
  isOverdue: boolean;      // send_month < 当月 AND status=PENDING
  batchId: number | null;
  createdAt: string;
  updatedAt: string;
};

export type MailSendByOffice = {
  office: {
    id: number;
    name: string;
    address: string;
  };
  mailSends: MailSend[];
};

export type MailSendBatch = {
  batchId: number;
  sentAt: string;
  updatedCount: number;
};
```

**src/types/user.ts**
```ts
export type User = {
  id: number;
  name: string;
  nameKana: string | null;
  birthDate: string | null;
  notes: string | null;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
};

export type UserWithOffices = User & {
  offices: Office[];
};
```

**src/types/office.ts**
```ts
export type Office = {
  id: number;
  name: string;
  postalCode: string | null;
  address: string | null;
  phone: string | null;
  isActive: boolean;
};
```

**src/types/staff.ts**
```ts
export type Role = 'ADMIN' | 'STAFF';

export type Staff = {
  id: number;
  name: string;
  email: string;
  role: Role;
  isActive: boolean;
  forcePasswordChange: boolean;
};
```

---

## 4. APIクライアント

**src/api/client.ts**
```ts
import axios from 'axios';

export const client = axios.create({
  baseURL: '/api',
  withCredentials: true,    // セッションCookieを送信
  headers: { 'Content-Type': 'application/json' },
});

// 401（未認証）の場合はログインページへリダイレクト
client.interceptors.response.use(
  res => res,
  err => {
    if (err.response?.status === 401) {
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);
```

**src/api/mailSends.ts**
```ts
import { client } from './client';
import type { MailSend, MailSendByOffice, MailSendBatch } from '@/types/mailSend';

// 事業所別一覧（送付先別一覧ビュー用）
export const getMailSendsByOffice = (status?: string) =>
  client.get<MailSendByOffice[]>('/mail-sends/by-office', { params: { status } })
    .then(r => r.data);

// 一覧取得
export const getMailSends = (params?: {
  status?: string;
  sendMonth?: string;
  userId?: number;
  officeId?: number;
}) => client.get<MailSend[]>('/mail-sends', { params }).then(r => r.data);

// 新規登録
export const createMailSend = (data: {
  userId: number;
  officeId: number;
  sendType: string;
  sendMonth: string;
}) => client.post<MailSend>('/mail-sends', data).then(r => r.data);

// まとめて送付済みにする
export const createBatch = (data: {
  mailSendIds: number[];
  notes?: string;
}) => client.post<MailSendBatch>('/mail-send-batches', data).then(r => r.data);
```

**src/api/dashboard.ts**
```ts
import { client } from './client';

export type DashboardData = {
  currentMonth: string;
  summary: { pending: number; sent: number; done: number };
  overdueMonths: { month: string; count: number }[];
  recentHistory: {
    id: number;
    officeName: string;
    userName: string;
    sendType: string;
    sentAt: string;
  }[];
};

export const getDashboard = () =>
  client.get<DashboardData>('/dashboard').then(r => r.data);
```

---

## 5. ルーティング設定

**src/App.tsx**
```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { AppLayout } from '@/components/layout/AppLayout';
import { LoginPage } from '@/pages/login/LoginPage';
import { DashboardPage } from '@/pages/dashboard/DashboardPage';
import { ByOfficePage } from '@/pages/mail-sends/ByOfficePage';
import { CreatePage } from '@/pages/mail-sends/CreatePage';
import { HistoryPage } from '@/pages/mail-sends/HistoryPage';
import { UserListPage } from '@/pages/users/UserListPage';
import { UserDetailPage } from '@/pages/users/UserDetailPage';
import { OfficeListPage } from '@/pages/offices/OfficeListPage';
import { StaffListPage } from '@/pages/staffs/StaffListPage';
import { ChangePasswordPage } from '@/pages/password/ChangePasswordPage';

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/password/change" element={<ChangePasswordPage />} />
          <Route element={<AppLayout />}>
            <Route path="/" element={<DashboardPage />} />
            <Route path="/mail-sends/by-office" element={<ByOfficePage />} />
            <Route path="/mail-sends/new" element={<CreatePage />} />
            <Route path="/mail-sends/history" element={<HistoryPage />} />
            <Route path="/users" element={<UserListPage />} />
            <Route path="/users/:id" element={<UserDetailPage />} />
            <Route path="/offices" element={<OfficeListPage />} />
            <Route path="/staffs" element={<StaffListPage />} />
          </Route>
          <Route path="*" element={<Navigate to="/" />} />
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  );
}
```

---

## 6. Zodバリデーションスキーマ

**src/schemas/loginSchema.ts**
```ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string()
    .min(1, 'メールアドレスを入力してください')
    .email('メールアドレスは ◯◯@◯◯.◯◯ の形で入力してください'),
  password: z.string()
    .min(1, 'パスワードを入力してください')
    .min(8, 'パスワードは8文字以上で入力してください'),
});
export type LoginForm = z.infer<typeof loginSchema>;
```

**src/schemas/mailSendSchema.ts**
```ts
import { z } from 'zod';

export const createMailSendSchema = z.object({
  userId:    z.number({ required_error: '利用者を選んでください' }),
  officeId:  z.number({ required_error: '送付先の事業所を選んでください' }),
  sendType:  z.enum(['PLAN', 'MONITORING'], {
    required_error: '送付の種類を選んでください',
  }),
  sendMonth: z.string().min(1, '送付する月を選んでください'),
});
export type CreateMailSendForm = z.infer<typeof createMailSendSchema>;

export const batchSchema = z.object({
  mailSendIds: z.array(z.number()).min(1, '送付済みにしたい項目にチェックを入れてください'),
  notes: z.string().max(500, 'メモは500文字以内で入力してください').optional(),
});
export type BatchForm = z.infer<typeof batchSchema>;
```

**src/schemas/userSchema.ts**
```ts
import { z } from 'zod';

export const userSchema = z.object({
  name: z.string()
    .min(1, '氏名を入力してください')
    .max(100, '氏名は100文字以内で入力してください'),
  nameKana: z.string()
    .max(100, 'ふりがなは100文字以内で入力してください')
    .regex(/^[ぁ-ん\s]*$/, 'ふりがなはひらがなで入力してください（例：やまだ はなこ）')
    .optional().or(z.literal('')),
  birthDate: z.string()
    .refine(v => !v || new Date(v) < new Date(), '生年月日は過去の日付を入力してください')
    .optional().or(z.literal('')),
  notes: z.string().max(1000, '備考は1000文字以内で入力してください').optional(),
});
export type UserForm = z.infer<typeof userSchema>;
```

**src/schemas/officeSchema.ts**
```ts
import { z } from 'zod';

export const officeSchema = z.object({
  name: z.string()
    .min(1, '事業所名を入力してください')
    .max(200, '事業所名は200文字以内で入力してください'),
  postalCode: z.string()
    .regex(/^\d{3}-?\d{4}$/, '郵便番号は数字7桁で入力してください（例：123-4567）')
    .optional().or(z.literal('')),
  address: z.string().max(500, '住所は500文字以内で入力してください').optional(),
  phone: z.string()
    .regex(/^[\d\-\+\(\)\s]*$/, '電話番号は数字とハイフンで入力してください（例：03-0000-0000）')
    .optional().or(z.literal('')),
});
export type OfficeForm = z.infer<typeof officeSchema>;
```

**src/schemas/staffSchema.ts**
```ts
import { z } from 'zod';

export const staffSchema = z.object({
  name: z.string().min(1, '氏名を入力してください').max(100),
  email: z.string()
    .min(1, 'メールアドレスを入力してください')
    .email('メールアドレスは ◯◯@◯◯.◯◯ の形で入力してください'),
  password: z.string()
    .min(8, 'パスワードは8文字以上で、英字（a〜z）と数字（0〜9）をまぜて設定してください')
    .regex(/^(?=.*[a-zA-Z])(?=.*\d).+$/,
      'パスワードは8文字以上で、英字（a〜z）と数字（0〜9）をまぜて設定してください'),
  role: z.enum(['ADMIN', 'STAFF'], { required_error: '権限を選んでください' }),
});
export type StaffForm = z.infer<typeof staffSchema>;
```

**src/schemas/passwordSchema.ts**
```ts
import { z } from 'zod';

export const changePasswordSchema = z.object({
  currentPassword: z.string().min(1, '今のパスワードを入力してください'),
  newPassword: z.string()
    .min(8, '新しいパスワードは8文字以上で、英字（a〜z）と数字（0〜9）をまぜて設定してください')
    .regex(/^(?=.*[a-zA-Z])(?=.*\d).+$/,
      '新しいパスワードは8文字以上で、英字（a〜z）と数字（0〜9）をまぜて設定してください'),
  confirmPassword: z.string(),
}).refine(d => d.newPassword === d.confirmPassword, {
  message: '上で入力した新しいパスワードと同じ内容を入力してください',
  path: ['confirmPassword'],
});
export type ChangePasswordForm = z.infer<typeof changePasswordSchema>;
```

---

## 7. 共通コンポーネント実装指示

### StatusBadge

```tsx
// src/components/ui/StatusBadge.tsx
// PENDING（当月）: オレンジ / PENDING（月遅れ）: 赤 / SENT: ブルー / DONE: グレー
type Props = {
  status: 'PENDING' | 'SENT' | 'DONE';
  isOverdue?: boolean;
};
```

### ConfirmModal

```tsx
// src/components/ui/ConfirmModal.tsx
// 重要操作（送付済みにする・削除・無効化）前に必ず表示する確認モーダル
// DADSのDialogコンポーネントを使用
// Storybook: https://design.digital.go.jp/dads/react/?path=/docs/component-dialog--docs
type Props = {
  isOpen: boolean;
  title: string;
  description: string;
  items?: string[];      // 対象一覧（任意）
  noteLabel?: string;    // メモ入力欄（任意）
  onConfirm: (note?: string) => void;
  onCancel: () => void;
};
```

---

## 8. 各画面の実装ポイント

### ダッシュボード（/）
- TanStack Queryで `/api/dashboard` を取得
- `summary.pending` が0より大きい場合、送付待ちカードをオレンジ強調
- `overdueMonths` が1件以上の場合のみ月遅れアラートバナーを表示
- 送付待ちカードクリック → `/mail-sends/by-office?status=PENDING` へ遷移

### 送付先別一覧ビュー（/mail-sends/by-office）
- TanStack Queryで `/api/mail-sends/by-office` を取得
- `isOverdue=true` のレコードは赤バッジ表示
- チェックボックスで複数選択 → ConfirmModalを表示 → `/api/mail-send-batches` へPOST

### 送付物作成（/mail-sends/new）
- STEP1で利用者を選択後、STEP2の事業所セレクトを `/api/users/{userId}/offices` で動的に絞り込む
- 確認画面を必ず挟む（ConfirmModal）
- 登録完了後 → `/mail-sends/by-office` へ遷移

### 利用者詳細（/users/:id）
- 基本情報 + 紐付き事業所一覧 + 最近の送付履歴（5件）を表示
- 事業所の紐付け解除はConfirmModalで確認後 `/api/users/{userId}/offices/{officeId}` をDELETE

---

## 9. DADSコンポーネント 参照先

各コンポーネントの詳細な使い方はStorybookで確認してください。

```
Storybook URL: https://design.digital.go.jp/dads/react/
```

| 本システムで使うコンポーネント | Storybookのパス |
|---|---|
| ボタン | component/button |
| テキスト入力 | component/input |
| セレクトボックス | component/select |
| チェックボックス | component/checkbox |
| ラジオボタン | component/radiobutton |
| テキストエリア | component/textarea |
| ラベル | component/label |
| エラーメッセージ | component/errormessage |
| ダイアログ（モーダル） | component/dialog |
| テーブル | component/table |

---

## 10. 開発サーバー起動確認

```bash
# フロントエンド起動（ポート5173）
npm run dev

# バックエンド（Spring Boot）が localhost:8080 で起動していること
# vite.config.ts のproxyで /api → 8080 に転送される
```
