---
name: react-patterns
description: React 18/19パターン。フックの規律、サーバー/クライアントコンポーネント境界、Suspense + エラーバウンダリ、フォームアクション、データフェッチ、状態管理デシジョンツリー、アクセシビリティファーストのコンポジション。Reactコンポーネントの作成・レビュー時に使用。
metadata:
  origin: ECC
---

# React パターン

堅牢でアクセシブル、パフォーマンスに優れたコンポーネントツリーを構築するためのReact 18/19のイディオマティックパターン。

## いつ使用するか

- React関数コンポーネント、カスタムフック、コンポーネントツリーの作成・修正
- JSX/TSXファイルのレビュー
- 状態の設計やコンポーネントコンポジション
- クラスコンポーネントや古い `forwardRef`/`useEffect` 多用コードの移行
- ローカル状態、リフトアップ状態、コンテキスト、外部ストアの選択
- Server Components / Client Components（Next.js App Router, RSC）の利用
- React 19アクションまたはcontrolled inputsでのフォーム実装
- TanStack Query / SWR / RSC fetchでのデータフェッチ接続

## コアプリンシプル

### 1. レンダーはPropsとStateの純粋関数

```tsx
// 良い例: レンダー中に導出
function Cart({ items }: { items: CartItem[] }) {
  const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
  return <span>{formatMoney(total)}</span>;
}

// 悪い例: 導出状態を別途保持
function Cart({ items }: { items: CartItem[] }) {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((sum, i) => sum + i.price * i.qty, 0));
  }, [items]);
  return <span>{formatMoney(total)}</span>;
}
```

`useEffect` 内の導出状態はレンダーサイクルを追加し、同期ずれを起こし、データフローを不明瞭にする。

### 2. 副作用はレンダー外で

エフェクト、ミューテーション、ネットワークコール、サブスクリプションはイベントハンドラまたは `useEffect` に配置する — レンダー本体には決して置かない。

### 3. 継承よりコンポジション

Reactにはコンポーネントの継承モデルがない。`children`、render props、またはコンポーネントpropsでコンポジションする。

## フックの規律

完全なルールセットは [rules/react/hooks.md](../../rules/react/hooks.md) を参照。ハイライト：

- トップレベルのみ、条件付き呼び出し禁止
- すべてのサブスクリプション、インターバル、リスナーをクリーンアップ
- 新しい状態が古い状態に依存する場合は関数型アップデータ (`setX(prev => prev + 1)`)
- デフォルトはメモ化しない — プロファイラーまたは依存チェーンが必要性を証明した場合のみ `useMemo`/`useCallback` を追加
- 同じフックシーケンスが2つ以上のコンポーネントに現れる場合のみカスタムフックを抽出

## 状態の配置デシジョンツリー

```
1つのコンポーネントのみで使用？
  -> そのコンポーネント内で useState

親 + 数個の子孫で使用？
  -> 最も近い共通祖先にリフトアップ

離れたブランチ間で使用 AND 低頻度の読み取り（テーマ、認証、ロケール）？
  -> React Context

ツリー全体で共有される高頻度の更新？
  -> 外部ストア（Zustand, Jotai, Redux Toolkit）

サーバーから導出される？
  -> サーバー状態ライブラリ（TanStack Query, SWR, RSC fetch）
```

ほとんどのページにはコンテキストやグローバルストアは不要。リフトアップの重複が辛くなるまで抽象化に抵抗すること。

## サーバー / クライアントコンポーネント (RSC)

```tsx
// Server Component - デフォルト、async、自身のJSを配信しない
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await db.product.findUnique({ where: { id: params.id } });
  if (!product) notFound();
  return <ProductView product={product} />;
}

// Client Component - "use client" でオプトイン
"use client";
export function AddToCartButton({ productId }: { productId: string }) {
  const [pending, startTransition] = useTransition();
  return (
    <button
      disabled={pending}
      onClick={() => startTransition(() => addToCart(productId))}
    >
      {pending ? "追加中..." : "カートに追加"}
    </button>
  );
}
```

境界：

- Server -> Client: シリアライズ可能なpropsまたは `children` を渡す
- Client -> Server: `<form action={...}>` またはイベントハンドラから命令的にServer Actionsを呼び出す
- Client ComponentファイルからServer Componentを `import` しない — 代わりに `children` を介してコンポジションする

## Suspense + エラーバウンダリ

```tsx
<ErrorBoundary fallback={<ErrorView />}>
  <Suspense fallback={<UserSkeleton />}>
    <UserDetail id={id} />
  </Suspense>
</ErrorBoundary>
```

- Suspenseバウンダリはルートルートではなくデータの近くに配置 — コンテンツを段階的に表示
- Error Boundaryはクラスベースのまま。フックフレンドリーなラッパーには `react-error-boundary` を使用
- バウンダリは子のレンダー、ライフサイクル、コンストラクタで投げられたエラーをキャッチ — イベントハンドラや非同期コードではキャッチしない

## フォーム

### React 19 フォームアクション（新コード推奨）

```tsx
"use client";
import { useActionState } from "react";

const initial = { error: null as string | null };

async function updateUserAction(_prev: typeof initial, formData: FormData) {
  "use server";
  const parsed = UserSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: "無効な入力" };
  await db.user.update({ where: { id: parsed.data.id }, data: parsed.data });
  return { error: null };
}

export function UserForm() {
  const [state, formAction, pending] = useActionState(updateUserAction, initial);
  return (
    <form action={formAction}>
      <input name="name" required />
      <button type="submit" disabled={pending}>保存</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

### Controlled inputs

値が他のUIを駆動する場合、キーストロークごとにフォーマットする場合、リアルタイムバリデーションを実装する場合にcontrolledを使用。

### 複雑なフォーム

マルチステップフォーム、動的フィールド配列、クロスフィールドバリデーションの場合：ライブラリ（React Hook Form、TanStack Form）を使用。些細な複雑さを超えるフォームの自前状態管理はメンテナンスの罠。

## データフェッチ デシジョンマトリクス

| ニーズ | ツール |
|---|---|
| Next.js App Routerでのリクエスト単位データ | RSC `await fetch()` |
| クライアントサイドキャッシュ + ミューテーション + 無効化 | TanStack Query |
| 軽量クライアントキャッシュ + 再検証 | SWR |
| リアルタイムサブスクリプション | Server-Sent Events、WebSockets、またはライブラリのサブスクリプションAPI |
| ワンオフのfire-and-forget | イベントハンドラ内の `fetch()` |

アプリケーションデータに `useEffect` + `fetch` を使用しない — レースコンディション、キャッシュなし、リトライなし、Suspense統合なし。

## コンポジションレシピ

### `children` によるスロット

```tsx
<Layout>
  <Header />
  <Main>{content}</Main>
</Layout>
```

### 名前付きスロット

```tsx
<Page header={<Nav />} sidebar={<Filters />}>
  <Results />
</Page>
```

### 複合コンポーネント（Contextによる共有状態）

```tsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Trigger value="profile">プロフィール</Tabs.Trigger>
    <Tabs.Trigger value="settings">設定</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="profile"><Profile /></Tabs.Panel>
  <Tabs.Panel value="settings"><Settings /></Tabs.Panel>
</Tabs>
```

### Render prop / function-as-child

親がレンダリング出力にパラメータを渡す必要がある場合に有用：

```tsx
<DataLoader id={id}>
  {({ data, isLoading }) => isLoading ? <Spinner /> : <UserCard user={data} />}
</DataLoader>
```

モダンな代替案：同じ形状を返すフック (`useData(id)`) — 通常こちらの方がクリーン。

## パフォーマンス

### `React.memo` が実際に有効な場合

以下の場合のみコンポーネントを `React.memo` でラップ：

1. 頻繁に再レンダーされる
2. レンダー間でpropsが通常同じ
3. レンダーが計測可能なほど高コスト

`React.memo` はすべてのレンダーで等価性チェックを追加する。ほとんどのレンダーでpropsが異なる場合、チェックは純粋なオーバーヘッド。

### レンダーカスケードの回避

- 可能な場合は状態を上ではなく下にリフト
- コンテキストを分割：関心ごとに1コンテキスト。`themeContext` の変更で認証コンシューマーが再レンダーされないように
- 外部状態ライブラリには `useSyncExternalStore` を使用 — 安全な並行レンダリングに必須

### リスト

- 安定した `key` propsを提供（データベースID、配列インデックスではない）
- 非自明な行で表示アイテム数が〜50を超えたら `@tanstack/react-virtual` または `react-window` で仮想化

## アクセシビリティファーストのコンポジション

- `role` 属性に手を出す前に常にセマンティックHTML（`<button>`, `<a>`, `<nav>`, `<main>`）をレンダー
- すべてのインタラクティブ要素はキーボードで到達可能でなければならない
- フォーム入力にはラベルが必要 — `<label htmlFor>` またはアイコンで視覚的にラベル付けされている場合は `aria-label`
- ルート変更とモーダル開閉時にフォーカスを管理
- コンポーネントテストで `axe` を実行（[skills/react-testing](../react-testing/SKILL.md) 参照）
- クロスリンク: [skills/accessibility/SKILL.md](../accessibility/SKILL.md) でWCAG基準とパターンライブラリをカバー

## ルーティング

このスキルはルーター非依存。上記のパターンはReact Router、TanStack Router、Next.js App Router、Remix Routerで動作する。ルーター固有のパターン（ローダー、アクション、ネストレイアウト）はルーターのドキュメントに従う — それらはReactコアの上にレイヤーされるフレームワークの関心事。

## スコープ外（ポインタセクション）

- **Next.js固有**: App Routerデータロード、Route Handlers、Middleware、Parallel Routes — 別の関心事、Next.jsドキュメントを使用
- **React Native**: プラットフォーム固有のパターンが十分に異なるため、別の `react-native-patterns` スキルを参照
- **Remix**: Loader/actionの規約はRSCと重複するがRemixドキュメントに従う

## 関連

- ルール: [rules/react/](../../rules/react/) — コーディングスタイル、フック、パターン、セキュリティ、テスト
- スキル: [react-performance](../react-performance/SKILL.md) Vercel由来のパフォーマンスルールセット、[frontend-patterns](../frontend-patterns/SKILL.md) クロスフレームワークUI関心事、[accessibility](../accessibility/SKILL.md)、[angular-developer](../angular-developer/SKILL.md) フレームワーク比較
- エージェント: `react-reviewer` コードレビュー用、`react-build-resolver` ビルド/バンドラーエラー用
- コマンド: `/react-review`, `/react-build`, `/react-test`

## 例

### デバウンス検索のカスタムフック

```tsx
function useDebounce<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

function SearchBox() {
  const [query, setQuery] = useState("");
  const debounced = useDebounce(query, 300);
  const { data } = useQuery({
    queryKey: ["search", debounced],
    queryFn: () => searchApi(debounced),
    enabled: debounced.length > 0,
  });
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results items={data ?? []} />
    </>
  );
}
```

### React 19 `useOptimistic` を使った楽観的UI

```tsx
"use client";
import { useOptimistic } from "react";

export function MessageList({ messages }: { messages: Message[] }) {
  const [optimistic, addOptimistic] = useOptimistic(
    messages,
    (state, newMessage: Message) => [...state, newMessage],
  );

  async function send(formData: FormData) {
    const text = String(formData.get("text"));
    addOptimistic({ id: "pending", text, sender: "me" });
    await saveMessage(text);
  }

  return (
    <>
      <ul>{optimistic.map((m) => <li key={m.id}>{m.text}</li>)}</ul>
      <form action={send}>
        <input name="text" />
        <button type="submit">送信</button>
      </form>
    </>
  );
}
```

### コンテキスト分割によるレンダーカスケードの回避

```tsx
// 2つのコンテキスト: 1つはほとんど変わらず、もう1つは頻繁に変わる
const ThemeContext = createContext<Theme>("light");
const NotificationsContext = createContext<Notification[]>([]);

// ThemeContextのみを消費するコンポーネントは通知が変更されても再レンダーされない
```
