# アクセシビリティ設定機能 実装指示書
# Claude Code 専用ファイル

---

## ⚠️ Claude Code への指示

**このファイルを読んだら、以下の順番でコードを実装してください。**

```
STEP 1: 「1. 追加するパッケージ」のコマンドを実行する
STEP 2: 「2. フォルダ・ファイル構成」のファイルをすべて作成する
STEP 3: 「3〜10」のコードを各ファイルに書き込む
STEP 4: 「12. 動作確認チェックリスト」を実行して確認する
STEP 5: エラーがあれば自分で修正して、全項目にチェックが入るまで繰り返す
```

**実装中に判断が必要な場合は、このファイルの内容を優先してください。**
**勝手に仕様を変えず、書いてある通りに実装してください。**

---

## 概要

このファイルだけで「アクセシビリティ設定パネル」を完全実装できます。
既存のプロジェクトに追加する機能として実装してください。

---

## 実装する機能

1. ヘッダーの歯車アイコン（⚙️）をタップで設定パネルを開く
2. 設定パネルで以下を変更できる
   - **もじの大きさ**：ふつう / 大きい / とても大きい
   - **ふりがな**：ひょうじする / ひょうじしない
   - **はいけいの色**：しろ / くろ / きいろ / あお
3. 設定はブラウザに記憶し、次回ログイン後も維持する
4. 「もとにもどす」ボタンで初期設定に戻す

---

## 1. 追加するパッケージ

**👉 以下のコマンドを実行してください。**

```bash
# 追加インストールは不要
# lucide-react（歯車アイコン）はすでにプロジェクトに含まれている想定
# なければインストール
npm install lucide-react
```

---

## 2. フォルダ・ファイル構成

**👉 以下のファイルをすべて新規作成してください。**
**既存ファイル（Header.tsx・AppLayout.tsx）は上書きせず、追記・修正してください。**

```
src/
├── contexts/
│   └── AccessibilityContext.tsx   ← 設定の状態管理
├── components/
│   ├── layout/
│   │   ├── Header.tsx             ← 歯車ボタンを追加
│   │   └── AppLayout.tsx          ← Providerを追加
│   └── ui/
│       ├── AccessibilityPanel.tsx ← 設定パネル本体
│       └── Furigana.tsx           ← ふりがなコンポーネント
└── styles/
    └── accessibility.css          ← 文字サイズ・背景色のCSS
```

---

## 3. 型定義

**👉 `src/types/accessibility.ts` を新規作成して、以下のコードを書いてください。**

```typescript
// src/types/accessibility.ts

export type FontSize = 'normal' | 'large' | 'xlarge';
export type BgColor = 'white' | 'black' | 'yellow' | 'blue';

export type AccessibilitySettings = {
  fontSize: FontSize;
  furigana: boolean;
  bgColor: BgColor;
};

export const DEFAULT_SETTINGS: AccessibilitySettings = {
  fontSize: 'normal',
  furigana: false,
  bgColor: 'white',
};
```

---

## 4. Context（設定の状態管理）

**👉 `src/contexts/AccessibilityContext.tsx` を新規作成して、以下のコードを書いてください。**

```typescript
// src/contexts/AccessibilityContext.tsx

import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import type { AccessibilitySettings, FontSize, BgColor } from '@/types/accessibility';
import { DEFAULT_SETTINGS } from '@/types/accessibility';

const STORAGE_KEY = 'accessibility_settings';

type AccessibilityContextType = {
  settings: AccessibilitySettings;
  setFontSize: (size: FontSize) => void;
  setFurigana: (enabled: boolean) => void;
  setBgColor: (color: BgColor) => void;
  resetSettings: () => void;
};

const AccessibilityContext = createContext<AccessibilityContextType | null>(null);

export function AccessibilityProvider({ children }: { children: ReactNode }) {
  const [settings, setSettings] = useState<AccessibilitySettings>(() => {
    // ブラウザに保存された設定を読み込む
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      return saved ? JSON.parse(saved) : DEFAULT_SETTINGS;
    } catch {
      return DEFAULT_SETTINGS;
    }
  });

  // 設定が変わるたびにブラウザへ保存・bodyにクラスを適用
  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(settings));

    const body = document.body;
    // フォントサイズクラス
    body.classList.remove('font-normal', 'font-large', 'font-xlarge');
    body.classList.add(`font-${settings.fontSize}`);
    // 背景色クラス
    body.classList.remove('bg-white', 'bg-black', 'bg-yellow', 'bg-blue');
    body.classList.add(`bg-theme-${settings.bgColor}`);
  }, [settings]);

  const setFontSize = (fontSize: FontSize) =>
    setSettings(prev => ({ ...prev, fontSize }));

  const setFurigana = (furigana: boolean) =>
    setSettings(prev => ({ ...prev, furigana }));

  const setBgColor = (bgColor: BgColor) =>
    setSettings(prev => ({ ...prev, bgColor }));

  const resetSettings = () => setSettings(DEFAULT_SETTINGS);

  return (
    <AccessibilityContext.Provider
      value={{ settings, setFontSize, setFurigana, setBgColor, resetSettings }}
    >
      {children}
    </AccessibilityContext.Provider>
  );
}

// カスタムフック
export function useAccessibility() {
  const ctx = useContext(AccessibilityContext);
  if (!ctx) throw new Error('AccessibilityProvider が必要です');
  return ctx;
}
```

---

## 5. CSSファイル

**👉 `src/styles/accessibility.css` を新規作成して、以下のコードを書いてください。**

```css
/* src/styles/accessibility.css */
/* このファイルを src/index.css に @import する */

/* ========== 文字サイズ ========== */
.font-normal  { font-size: 16px !important; }
.font-large   { font-size: 20px !important; }
.font-xlarge  { font-size: 24px !important; }

/* ========== 背景色・文字色 ========== */

/* しろ（デフォルト）*/
.bg-theme-white {
  background-color: #ffffff;
  color: #1a202c;
}

/* くろ */
.bg-theme-black {
  background-color: #000000 !important;
  color: #ffffff !important;
}
.bg-theme-black a,
.bg-theme-black button,
.bg-theme-black th,
.bg-theme-black td {
  color: #ffffff !important;
  border-color: #ffffff !important;
}
.bg-theme-black input,
.bg-theme-black select,
.bg-theme-black textarea {
  background-color: #222222 !important;
  color: #ffffff !important;
  border-color: #ffffff !important;
}

/* きいろ */
.bg-theme-yellow {
  background-color: #ffff00 !important;
  color: #000000 !important;
}
.bg-theme-yellow a,
.bg-theme-yellow button { color: #000080 !important; }
.bg-theme-yellow input,
.bg-theme-yellow select { background-color: #ffffcc !important; }

/* あお */
.bg-theme-blue {
  background-color: #003087 !important;
  color: #ffffff !important;
}
.bg-theme-blue a { color: #ffff00 !important; }
.bg-theme-blue input,
.bg-theme-blue select {
  background-color: #001a52 !important;
  color: #ffffff !important;
}

/* ========== ふりがな（ruby要素）========== */
ruby rt {
  font-size: 0.6em;
  line-height: 1;
}
/* ふりがな非表示時 */
.furigana-hidden ruby rt {
  display: none;
}
```

---

## 6. ふりがなコンポーネント

**👉 `src/components/ui/Furigana.tsx` を新規作成して、以下のコードを書いてください。**

```typescript
// src/components/ui/Furigana.tsx
// 業務でよく出る漢字にふりがなを自動でつけるコンポーネント

import { useAccessibility } from '@/contexts/AccessibilityContext';

// ふりがな辞書（必要に応じて追加する）
const FURIGANA_MAP: Record<string, string> = {
  '利用者':   'りようしゃ',
  '事業所':   'じぎょうしょ',
  '送付':     'そうふ',
  '計画':     'けいかく',
  '作成':     'さくせい',
  '報告書':   'ほうこくしょ',
  '管理':     'かんり',
  '登録':     'とうろく',
  '一覧':     'いちらん',
  '履歴':     'りれき',
  '送付済み': 'そうふずみ',
  '送付待ち': 'そうふまち',
  '完了':     'かんりょう',
  '確認':     'かくにん',
  '承認':     'しょうにん',
  '氏名':     'しめい',
  '住所':     'じゅうしょ',
  '郵便番号': 'ゆうびんばんごう',
  '電話番号': 'でんわばんごう',
  '権限':     'けんげん',
  '設定':     'せってい',
};

type Props = {
  text: string;
  className?: string;
};

export function Furigana({ text, className }: Props) {
  const { settings } = useAccessibility();

  if (!settings.furigana) {
    // ふりがな非表示時はそのままテキストを返す
    return <span className={className}>{text}</span>;
  }

  // ふりがなを付与する処理
  const parts: React.ReactNode[] = [];
  let remaining = text;
  let key = 0;

  while (remaining.length > 0) {
    // 辞書の中で一致する最長の語を探す
    const match = Object.entries(FURIGANA_MAP).find(([kanji]) =>
      remaining.startsWith(kanji)
    );

    if (match) {
      const [kanji, reading] = match;
      parts.push(
        <ruby key={key++}>
          {kanji}
          <rt>{reading}</rt>
        </ruby>
      );
      remaining = remaining.slice(kanji.length);
    } else {
      // 一致しない文字はそのまま出力
      parts.push(<span key={key++}>{remaining[0]}</span>);
      remaining = remaining.slice(1);
    }
  }

  return <span className={className}>{parts}</span>;
}
```

---

## 7. 設定パネルコンポーネント

**👉 `src/components/ui/AccessibilityPanel.tsx` を新規作成して、以下のコードを書いてください。**
**ボタンサイズは最低60px以上を維持してください（障害者が使うため変更不可）。**

```typescript
// src/components/ui/AccessibilityPanel.tsx

import { useAccessibility } from '@/contexts/AccessibilityContext';
import type { FontSize, BgColor } from '@/types/accessibility';
import { X, RotateCcw } from 'lucide-react';

type Props = {
  isOpen: boolean;
  onClose: () => void;
};

export function AccessibilityPanel({ isOpen, onClose }: Props) {
  const { settings, setFontSize, setFurigana, setBgColor, resetSettings } =
    useAccessibility();

  if (!isOpen) return null;

  // フォントサイズの選択肢
  const fontSizes: { value: FontSize; label: string; sample: string }[] = [
    { value: 'normal', label: 'ふつう',       sample: 'あいう' },
    { value: 'large',  label: '大きい',       sample: 'あいう' },
    { value: 'xlarge', label: 'とても大きい', sample: 'あいう' },
  ];

  // 背景色の選択肢
  const bgColors: { value: BgColor; label: string; bg: string; text: string }[] = [
    { value: 'white',  label: 'しろ',   bg: '#ffffff', text: '#000000' },
    { value: 'black',  label: 'くろ',   bg: '#000000', text: '#ffffff' },
    { value: 'yellow', label: 'きいろ', bg: '#ffff00', text: '#000000' },
    { value: 'blue',   label: 'あお',   bg: '#003087', text: '#ffffff' },
  ];

  return (
    <>
      {/* 背景オーバーレイ */}
      <div
        className="fixed inset-0 z-40 bg-black bg-opacity-50"
        onClick={onClose}
        aria-hidden="true"
      />

      {/* 設定パネル */}
      <div
        role="dialog"
        aria-modal="true"
        aria-label="つかいやすさの設定"
        className="fixed top-0 right-0 z-50 h-full w-full max-w-sm overflow-y-auto bg-white shadow-2xl"
      >
        {/* ヘッダー */}
        <div className="flex items-center justify-between bg-gray-100 p-4">
          <h2 className="text-xl font-bold">⚙️ つかいやすさの設定</h2>
          <button
            onClick={onClose}
            aria-label="とじる"
            className="flex h-12 w-12 items-center justify-center rounded-full bg-gray-200 text-gray-700 hover:bg-gray-300"
          >
            <X size={24} />
          </button>
        </div>

        <div className="p-4 space-y-8">

          {/* ① もじの大きさ */}
          <section>
            <h3 className="mb-3 text-lg font-bold">もじの大きさ</h3>
            <div className="flex gap-3">
              {fontSizes.map(({ value, label, sample }) => (
                <button
                  key={value}
                  onClick={() => setFontSize(value)}
                  aria-pressed={settings.fontSize === value}
                  className={[
                    'flex flex-1 flex-col items-center justify-center rounded-xl border-4 py-4 transition-all',
                    settings.fontSize === value
                      ? 'border-blue-600 bg-blue-50 font-bold'
                      : 'border-gray-200 bg-white hover:border-gray-400',
                  ].join(' ')}
                >
                  <span
                    style={{
                      fontSize: value === 'normal' ? '16px'
                              : value === 'large'  ? '22px' : '28px',
                      lineHeight: 1.2,
                    }}
                  >
                    {sample}
                  </span>
                  <span className="mt-2 text-sm">{label}</span>
                  {settings.fontSize === value && (
                    <span className="mt-1 text-xs text-blue-600">✓ 選択中</span>
                  )}
                </button>
              ))}
            </div>
          </section>

          {/* ② ふりがな */}
          <section>
            <h3 className="mb-3 text-lg font-bold">よみがな（ふりがな）</h3>
            <div className="flex gap-3">
              {[
                { value: true,  label: 'ひょうじする',    sample: <>田中<span className="text-xs">（たなか）</span></> },
                { value: false, label: 'ひょうじしない',  sample: <>田中</> },
              ].map(({ value, label, sample }) => (
                <button
                  key={String(value)}
                  onClick={() => setFurigana(value)}
                  aria-pressed={settings.furigana === value}
                  className={[
                    'flex flex-1 flex-col items-center justify-center rounded-xl border-4 py-5 transition-all',
                    settings.furigana === value
                      ? 'border-blue-600 bg-blue-50 font-bold'
                      : 'border-gray-200 bg-white hover:border-gray-400',
                  ].join(' ')}
                >
                  <span className="text-base">{sample}</span>
                  <span className="mt-2 text-sm">{label}</span>
                  {settings.furigana === value && (
                    <span className="mt-1 text-xs text-blue-600">✓ 選択中</span>
                  )}
                </button>
              ))}
            </div>
          </section>

          {/* ③ はいけいの色 */}
          <section>
            <h3 className="mb-3 text-lg font-bold">はいけいの色</h3>
            <div className="grid grid-cols-2 gap-3">
              {bgColors.map(({ value, label, bg, text }) => (
                <button
                  key={value}
                  onClick={() => setBgColor(value)}
                  aria-pressed={settings.bgColor === value}
                  style={{ backgroundColor: bg, color: text }}
                  className={[
                    'flex h-20 flex-col items-center justify-center rounded-xl border-4 transition-all',
                    settings.bgColor === value
                      ? 'border-blue-500 ring-2 ring-blue-500'
                      : 'border-gray-300 hover:border-gray-500',
                  ].join(' ')}
                >
                  <span className="text-lg font-bold">{label}</span>
                  {settings.bgColor === value && (
                    <span className="text-xs">✓ 選択中</span>
                  )}
                </button>
              ))}
            </div>
          </section>

          {/* もとにもどすボタン */}
          <button
            onClick={() => {
              resetSettings();
            }}
            className="flex w-full items-center justify-center gap-2 rounded-xl border-2 border-gray-400 py-4 text-gray-700 hover:bg-gray-100"
          >
            <RotateCcw size={18} />
            <span className="font-bold">もとにもどす</span>
          </button>

        </div>
      </div>
    </>
  );
}
```

---

## 8. ヘッダーに歯車ボタンを追加

**👉 既存の `src/components/layout/Header.tsx` を開き、以下の内容を参考に歯車ボタンと設定パネルを追加してください。**
**既存のコードを削除せず、追記する形で修正してください。**

```typescript
// src/components/layout/Header.tsx
// 既存のHeaderに以下を追加する

import { useState } from 'react';
import { Settings } from 'lucide-react';
import { AccessibilityPanel } from '@/components/ui/AccessibilityPanel';

export function Header() {
  const [isPanelOpen, setIsPanelOpen] = useState(false);

  return (
    <>
      <header className="flex items-center justify-between bg-navy-700 px-4 py-3">
        {/* 既存のロゴ・タイトル */}
        <h1 className="text-white font-bold text-lg">郵便物送付管理システム</h1>

        <div className="flex items-center gap-3">
          {/* ⚙️ アクセシビリティ設定ボタン */}
          <button
            onClick={() => setIsPanelOpen(true)}
            aria-label="つかいやすさの設定をひらく"
            className="flex h-12 w-12 items-center justify-center rounded-full text-white hover:bg-white hover:bg-opacity-20 focus:outline-none focus:ring-2 focus:ring-white"
          >
            <Settings size={26} />
          </button>

          {/* 既存のユーザーメニューなど */}
        </div>
      </header>

      {/* 設定パネル */}
      <AccessibilityPanel
        isOpen={isPanelOpen}
        onClose={() => setIsPanelOpen(false)}
      />
    </>
  );
}
```

---

## 9. AppLayoutにProviderを追加

**👉 既存の `src/components/layout/AppLayout.tsx` を開き、`AccessibilityProvider` で全体を囲んでください。**
**既存の構造を崩さないよう注意してください。**

```typescript
// src/components/layout/AppLayout.tsx
// AccessibilityProviderで全体を囲む

import { Outlet } from 'react-router-dom';
import { AccessibilityProvider } from '@/contexts/AccessibilityContext';
import { Header } from './Header';
import { SideNav } from './SideNav';

export function AppLayout() {
  return (
    <AccessibilityProvider>
      <div className="flex h-screen flex-col">
        <Header />
        <div className="flex flex-1 overflow-hidden">
          <SideNav />
          <main id="main" className="flex-1 overflow-y-auto p-6">
            <Outlet />
          </main>
        </div>
      </div>
    </AccessibilityProvider>
  );
}
```

---

## 10. CSSをindex.cssに追記

**👉 `src/index.css` の末尾に以下の1行を追加してください。**

```css
/* src/index.css の末尾に追記 */
@import './styles/accessibility.css';
```

---

## 11. ふりがなの使い方

**👉 既存の画面コンポーネントで漢字が含まれるラベル・見出し・テーブルヘッダーに `<Furigana>` コンポーネントを適用してください。**
**特に利用者名・事業所名・ステータス表示を優先的に対応してください。**

```typescript
// 利用者名など漢字が多い箇所で使う
import { Furigana } from '@/components/ui/Furigana';

// 使用例（ふりがなON時は自動でルビが付く）
<Furigana text="利用者一覧" />
// → ON時:  利用者（りようしゃ）一覧（いちらん）
// → OFF時: 利用者一覧

<Furigana text="送付履歴" />
// → ON時:  送付（そうふ）履歴（りれき）
// → OFF時: 送付履歴

// テーブルのヘッダーでも使う
<th><Furigana text="事業所" /></th>
<th><Furigana text="送付" /></th>
<th><Furigana text="確認" /></th>
```

---

## 12. 動作確認チェックリスト

**👉 実装が完了したら、以下をすべて実行して確認してください。**
**1つでもチェックが入らない項目があれば、自分でデバッグして修正してください。**
**すべてのチェックが完了してから「実装完了」と報告してください。**

実装後、以下をすべて確認してください。

```
□ ヘッダーの歯車アイコンをクリックで設定パネルが開く
□ パネル外をクリックするとパネルが閉じる
□ ✕ボタンでパネルが閉じる
□ 「大きい」を選ぶと全体の文字が大きくなる
□ 「とても大きい」を選ぶとさらに大きくなる
□ 「ふりがなをひょうじする」でルビが表示される
□ 「くろ」を選ぶと背景が黒・文字が白になる
□ 「きいろ」を選ぶと背景が黄色になる
□ 「もとにもどす」で設定がリセットされる
□ ページをリロードしても設定が維持されている
□ キーボード（Tabキー）でボタンにフォーカスできる
□ 選択中の設定に「✓ 選択中」が表示されている
```

---

## 13. ふりがな辞書の追加方法

**👉 実装後にふりがなが付いていない漢字を見つけた場合は、`Furigana.tsx` の `FURIGANA_MAP` に追記してください。**

新しい漢字を追加したい場合は `Furigana.tsx` の `FURIGANA_MAP` に追記するだけです。

```typescript
// 追加例
const FURIGANA_MAP: Record<string, string> = {
  // 既存のエントリー...
  '相談員':   'そうだんいん',    // ← 追加
  'モニタリング': 'もにたりんぐ', // ← カタカナも追加可能
};
```

---

## 14. 実装完了後にCLAUDE.mdを更新する

**👉 すべての実装・動作確認が完了したら、プロジェクトルートの `CLAUDE.md` に以下を追記してください。**

```markdown
## アクセシビリティ設定パネル（実装済み）

- 実装完了日：（実装した日付を記入）
- 実装ファイル：
  - src/types/accessibility.ts
  - src/contexts/AccessibilityContext.tsx
  - src/styles/accessibility.css
  - src/components/ui/Furigana.tsx
  - src/components/ui/AccessibilityPanel.tsx
  - src/components/layout/Header.tsx（修正）
  - src/components/layout/AppLayout.tsx（修正）
- 動作確認：全12項目チェック済み
```

**⚠️ CLAUDE.mdは上書きせず、末尾に追記してください。**
