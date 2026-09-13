---
name: frontend-a11y
description: >
  ReactおよびNext.js向けのアクセシビリティパターン — セマンティックHTML、ARIA属性、
  フォームのラベル付け、キーボードナビゲーション、フォーカス管理、スクリーンリーダー対応。
  インタラクティブなUIコンポーネントやフォームを構築する場合に使用します。
metadata:
  origin: community
---

# Frontend Accessibility Patterns（フロントエンド・アクセシビリティ）

React および Next.js のための実践的なアクセシビリティ（a11y）パターン。コードレビューで最も頻繁に指摘される問題（ラベル付けの欠落、不適切なARIAの使用、非セマンティックな対話要素、キーボード操作の不備など）を網羅しています。

## 有効化のタイミング

- フォームコンポーネント（`<input>`, `<select>`, `<textarea>`）の構築またはレビュー時
- 対話的要素（モーダル、ドロップダウン、ツールチップ、タブ）の作成時
- `<div>` や `<span>` に `onClick` を設定している箇所の修正時
- 要素に `aria-*` 属性を追加するとき
- キーボード操作やフォーカス管理を実装するとき
- コードレビューツール（CodeRabbit, ESLint a11y 等）からアクセシビリティの指摘を受けたとき
- スクリーンリーダー対応が求められるコンポーネントの構築時

## フォームのアクセシビリティ

`htmlFor` と `id` の接続漏れや、入力フィールドとエラーメッセージの非連携は、最も一般的な不備です。

### ラベルの紐付け

```tsx
// BAD: label と input に関連付けがなく、スクリーンリーダーが関連を認識できない
<label>メールアドレス</label>
<input type="email" />

// GOOD: htmlFor と input の id を一致させる
<label htmlFor="email">メールアドレス</label>
<input id="email" type="email" />
```

### 必須フィールド

```tsx
// BAD: 視覚的なアスタリスク（*）だけではスクリーンリーダーに何も伝わらない
<label htmlFor="email">メールアドレス *</label>
<input id="email" type="email" />

// GOOD: required でブラウザネイティブの検証を有効化し、aria-required で読み上げに対応させる
<label htmlFor="email">
  メールアドレス <span aria-hidden="true">*</span>
</label>
<input id="email" type="email" required aria-required="true" />
```

### エラーメッセージの紐付け

```tsx
// BAD: エラーテキストが視覚的に存在するだけで、input と関連付けられていない
<input id="email" type="email" />
<span className="error">無効なメールアドレスです</span>

// GOOD: aria-describedby でエラーメッセージを接続し、aria-invalid で無効状態を通知
<input
  id="email"
  type="email"
  aria-describedby="email-error"
  aria-invalid={!!error}
/>
{error && (
  <span id="email-error" role="alert">
    {error}
  </span>
)}
```

## セマンティック HTML

目的に合ったネイティブのHTML要素を使用します。スクリーンリーダーやキーボード操作はネイティブのセマンティクスに依存しています。

```tsx
// BAD: div には role もキーボードサポートもなく、アクセシブルな名前もない
<div onClick={handleClick}>送信</div>

// GOOD: button はフォーカス可能で、Enter/Space で発火し、「ボタン」として読み上げられる
<button type="button" onClick={handleClick}>送信</button>
```

```tsx
// BAD: 非セマンティックなナビゲーション
<div onClick={() => navigate('/home')}>ホーム</div>

// GOOD: aタグは右クリック、ホイールクリック（新しいタブ）、キーボード操作を自然にサポート
<a href="/home">ホーム</a>
```

```tsx
// BAD: 見出しレベルの飛び越し（h1 から h4）
<h1>ダッシュボード</h1>
<h4>最近のアクティビティ</h4>

// GOOD: 連続した見出しレベル
<h1>ダッシュボード</h1>
<h2>最近のアクティビティ</h2>
```

## ARIA 属性

ネイティブの HTML セマンティクスでは不十分な場合にのみ ARIA を使用します。**誤った ARIA は ARIA が全くない状態よりも有害です。**

### `aria-label` vs `aria-labelledby`

```tsx
// aria-label: インライン文字列ラベル — 画面上に表示テキストが存在しない場合に使用
<button aria-label="モーダルを閉じる">
  <XIcon />
</button>

// aria-labelledby: 他の要素のテキストを参照 — 表示ラベルが存在する場合に使用
<section aria-labelledby="section-title">
  <h2 id="section-title">最近の注文</h2>
  {/* コンテンツ */}
</section>
```

### 動的コンテンツのための `aria-live`

```tsx
// ページ再読み込みなしで更新されるコンテンツを通知するために aria-live を使用
// polite: ユーザーの現在の操作が終わるのを待ってから読み上げる
// assertive: 直ちに割り込んで読み上げる — 緊急のエラーにのみ使用

export function StatusMessage({ message, isError }: { message: string; isError?: boolean }) {
  return (
    <div role="status" aria-live={isError ? 'assertive' : 'polite'} aria-atomic="true">
      {message}
    </div>
  );
}
```

### アコーディオン展開状態の通知

```tsx
export function Accordion({ title, children }: { title: string; children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const contentId = useId();

  return (
    <div>
      <button aria-expanded={isOpen} aria-controls={contentId} onClick={() => setIsOpen(prev => !prev)}>
        {title}
      </button>
      <div id={contentId} hidden={!isOpen}>
        {children}
      </div>
    </div>
  );
}
```

## フォーカス管理

UIの状態が変化した際（特にモーダルを開閉したときなど）、フォーカスが論理的に移動する必要があります。

```tsx
export function Modal({ isOpen, onClose, title, children }: { isOpen: boolean; onClose: () => void; title: string; children: React.ReactNode }) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // 開く前のフォーカス要素を記憶し、モーダル内にフォーカスを移動
      previousFocusRef.current = document.activeElement as HTMLElement;
      modalRef.current?.focus();
    } else {
      // モーダルを閉じた際、トリガー元の要素へフォーカスを復帰
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div ref={modalRef} role="dialog" aria-modal="true" aria-labelledby="modal-title" tabIndex={-1} onKeyDown={e => e.key === 'Escape' && onClose()}>
      <h2 id="modal-title">{title}</h2>
      {children}
      <button onClick={onClose}>閉じる</button>
    </div>
  );
}
```

## 画像とアイコン

```tsx
// BAD: 装飾用アイコンがラベルなし画像として読み上げられてしまう
<img src="/icon.svg" />

// GOOD: 装飾用画像はスクリーンリーダーから隠蔽する
<img src="/decoration.png" alt="" aria-hidden="true" />

// GOOD: 意味を持つ画像には説明的な代替テキストを付与する
<img src="/chart.png" alt="1月から3月にかけて月間売上が23%増加した推移グラフ" />

// GOOD: アイコンのみのボタンにはアクセシブルなラベルを付与する
<button aria-label="項目を削除">
  <TrashIcon aria-hidden="true" />
</button>
```

## アニメーションの抑制（prefers-reduced-motion）

OS設定で「視覚効果を減らす」を選択しているユーザーに配慮します。

```tsx
export function useReducedMotion(): boolean {
  const [prefersReduced, setPrefersReduced] = useState(false);

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    setPrefersReduced(mq.matches);
    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}
```

## アンチパターン

```tsx
// BAD: キーボードサポートのない要素への onClick
<div onClick={handleClick}>クリック</div>

// BAD: role のない div に aria-label を付与（無視される）
<div aria-label="ナビゲーション">...</div>

// BAD: placeholder をラベルの代用にする
<input placeholder="メールアドレスを入力" />

// BAD: 正の tabIndex（タブ移動の順序を破壊する）
<button tabIndex={3}>送信</button>

// BAD: フォーカス可能な要素に対する aria-hidden="true"（キーボード操作でトラップされる）
<button aria-hidden="true">開く</button>
```

## チェックリスト

- [ ] すべての `<input>`, `<select>`, `<textarea>` が `htmlFor` / `id` で `<label>` と接続されている
- [ ] エラーメッセージが `aria-describedby` で接続され、`role="alert"` がマークされている
- [ ] `role`, `tabIndex`, `onKeyDown` なしで `<div>` や `<span>` に `onClick` が付与されていない
- [ ] アイコンのみのボタンに `aria-label` がある
- [ ] 装飾画像に `alt=""` および `aria-hidden="true"` が設定されている
- [ ] モーダルが閉じたときにフォーカスが復帰する
- [ ] 動的なコンテンツ更新に `aria-live` が使用されている
- [ ] アニメーションにおいて `prefers-reduced-motion` が尊重されている
