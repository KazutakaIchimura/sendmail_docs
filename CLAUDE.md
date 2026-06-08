# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

郵便物送付管理システムのフロントエンド。React + TypeScript + Vite で構築し、デジタル庁デザインシステム（DADS）のコンポーネントを使用する。バックエンドは Spring Boot（localhost:8080）で、Vite の dev proxy 経由で `/api` に接続する。

## コマンド

すべてのコマンドは `frontend/` ディレクトリで実行する。

```bash
npm run dev       # 開発サーバー起動（localhost:5173）
npm run build     # 本番ビルド（tsc → vite build）
npm run preview   # ビルド済みファイルのプレビュー
```

## アーキテクチャ

### ルーティング（src/App.tsx）
- `/login` と `/password/change` は認証不要の独立ルート
- その他は `<AppLayout>` でラップ（Header + SideNav + Outlet）
- 未マッチは `/` にリダイレクト
- 401 レスポンス時は axios インターセプターが `/login` へ自動リダイレクト

### 認証（src/contexts/AuthContext.tsx）
- `AuthProvider` は `AppLayout` 内に配置（認証不要ルートには不要）
- `GET /api/auth/me` で現在のスタッフ情報を取得
- `useAuth()` で `currentStaff`・`isAdmin`・`logout` を取得
- スタッフ管理（`/staffs`）は SideNav で ADMIN のみ表示

### データフェッチ
TanStack Query (`useQuery` / `useMutation`) を使用。API 関数は `src/api/` に集約し、コンポーネントから直接 axios を呼ばない。

### フォームバリデーション
`react-hook-form` + `zod` の組み合わせ。スキーマは `src/schemas/` に定義。**Zod v4** がインストールされているため `required_error` は使えず、代わりに `error` を使う。

```ts
// NG（Zod v3 の書き方）
z.number({ required_error: '...' })
// OK（Zod v4）
z.number({ error: '...' })
```

### DADS コンポーネント
デジタル庁デザインシステムのコードスニペットを `src/components/dads/` にコピー済み。Storybook は [https://design.digital.go.jp/dads/react/](https://design.digital.go.jp/dads/react/) で参照。

**実際の props（Storybook のサンプルと異なる点）:**

| コンポーネント | インポートパス | 特記事項 |
|---|---|---|
| Button | `Button/Button` | `variant`: `'solid-fill'`\|`'outline'`\|`'text'`、`size` 必須 |
| Input | `Input/Input` | エラー時は `isError`（`isInvalid` ではない） |
| Select | `Select/Select` | エラー時は `isError` |
| Checkbox | `Checkbox/Checkbox` | エラー時は `isError` |
| Radio | `Radio/Radio` | RadioButton ではなく Radio |
| Textarea | `Textarea/Textarea` | エラー時は `isError` |
| Label | `Label/Label` | 必須マークは `RequirementBadge` を別途使う |
| ErrorText | `ErrorText/ErrorText` | `<p>` 相当。`ErrorMessage` ではない |
| RequirementBadge | `RequirementBadge/RequirementBadge` | `isOptional` で「任意」表示 |
| Dialog/DialogBody | `v1/Dialog/Dialog` | `dialogRef.showModal()` で開く |

- **注意**: `DatePicker` と `SeparatedDatePicker` は `react-aria-components` 依存のため `src/components/dads/index.ts` から除外済み。使う場合は先にパッケージをインストールする。
- コンポーネントは `@/components/dads/Button/Button` のようにファイル直接インポートする。

### 共通 UI コンポーネント（src/components/ui/）

| 用途 | コンポーネント |
|------|----------------|
| 確認モーダル | `ConfirmModal` — DADS Dialog を使用。`isDanger` で危険操作の赤ボタン |
| ステータスバッジ | `StatusBadge` — `isOverdue=true` で赤表示 |
| ページタイトル | `PageTitle` |
| 検索入力 | `SearchInput` |

### タイポグラフィ（DADS クラス）

```
text-std-32B-7  → 見出し大（32px Bold）
text-std-24B-7  → 見出し中（24px Bold）
text-std-17B-7  → 見出し小（17px Bold）
text-std-16N-7  → 本文（16px Normal）
text-std-14N-7  → 補足テキスト（14px Normal）
text-std-14B-7  → 補足テキスト Bold
```

### パスエイリアス
`@/` が `src/` に解決される（vite.config.ts + tsconfig.app.json に設定済み）。

## 主要な画面と API 対応

| 画面 | パス | 主な API |
|------|------|----------|
| ダッシュボード | `/` | `GET /api/dashboard` |
| 送付先別一覧 | `/mail-sends/by-office` | `GET /api/mail-sends/by-office` |
| 送付物作成 | `/mail-sends/new` | `POST /api/mail-sends`、`GET /api/users/{id}/offices` |
| 送付履歴 | `/mail-sends/history` | `GET /api/mail-sends` |
| 利用者一覧 | `/users` | `GET /api/users` |
| 利用者詳細 | `/users/:id` | `GET /api/users/{id}`、`DELETE /api/users/{id}/offices/{officeId}` |
| 利用者登録/編集 | `/users/new`、`/users/:id/edit` | `POST/PUT /api/users` |
| 事業所一覧 | `/offices` | `GET /api/offices` |
| 事業所登録/編集 | `/offices/new`、`/offices/:id/edit` | `POST/PUT /api/offices` |
| スタッフ管理 | `/staffs` | `GET /api/staffs`（ADMIN のみ） |
| スタッフ登録/編集 | `/staffs/new`、`/staffs/:id/edit` | `POST/PUT /api/staffs` |

## 重要な実装ルール

- **送付済み一括処理**: `POST /api/mail-send-batches` に `mailSendIds[]` を送る
- **ConfirmModal**: 送付済みにする・削除・無効化の前に必ず表示する
- **isOverdue フラグ**: `sendMonth < 当月 AND status=PENDING` の場合 true。赤バッジで表示
- **STEP 式フォーム（送付物作成）**: STEP1（利用者）→ STEP2（事業所、userId で動的絞り込み）→ STEP3（種別）→ STEP4（送付月）→ 確認
- **スタッフ安全設計**: 自己無効化禁止・ADMIN 最低 1 名維持（`StaffListPage` で `canDisable()` チェック）
- **Tailwind CSS v3**（v4 ではない）がインストールされているため、v4 の CSS-first 設定は使わない

## アクセシビリティ設定パネル（実装済み）

- 実装完了日：2026-05-20
- 実装ファイル：
  - `src/types/accessibility.ts`
  - `src/contexts/AccessibilityContext.tsx`
  - `src/styles/accessibility.css`
  - `src/components/ui/Furigana.tsx`
  - `src/components/ui/AccessibilityPanel.tsx`
  - `src/components/layout/Header.tsx`（修正）
  - `src/components/layout/AppLayout.tsx`（修正）
  - `src/index.css`（`@import './styles/accessibility.css'` 追記）
- 動作確認：全12項目チェック済み

### 概要
ヘッダーの歯車アイコン（⚙️）から設定パネルを開き、以下を変更できる：
- **もじの大きさ**：ふつう(16px) / 大きい(20px) / とても大きい(24px)
- **ふりがな**：ひょうじする / ひょうじしない（`<Furigana>` コンポーネントで適用）
- **はいけいの色**：しろ / くろ / きいろ / あお
- 設定は `localStorage` に保存され、リロード後も維持される
- `<body>` に `font-{size}` と `bg-theme-{color}` クラスを付与して CSS で切り替え

### 共通 UI コンポーネント追加分

| 用途 | コンポーネント |
|------|----------------|
| アクセシビリティ設定パネル | `AccessibilityPanel` — `isOpen`/`onClose` props |
| ふりがな表示 | `Furigana` — `text` props。設定OFFの場合は `<span>` をそのまま返す |

### AccessibilityProvider の配置
`AppLayout.tsx` で `AccessibilityProvider` が最外層、その内側に `AuthProvider` を配置している。
`Header.tsx` で `useAccessibility()` は使わず、`useState` で `isPanelOpen` を管理して `AccessibilityPanel` に渡す。
