---
name: react-performance
description: Vercel Engineeringの React Best Practicesから適応したReactとNext.jsのパフォーマンス最適化パターン。ウォーターフォール、バンドルサイズ、サーバーサイド、クライアントフェッチ、再レンダー、レンダリング、JSマイクロパフォーマンス、アドバンスドの8つの優先カテゴリにわたる70以上のルールを整理。React/Next.jsコードの作成、レビュー、リファクタリング時に使用。
metadata:
  origin: ECC
---

# React パフォーマンス

React 18/19とNext.jsのパフォーマンス最適化パターン。[Vercel Labs `react-best-practices`](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)（MIT, v1.0.0）から適応。優先度別にルールを整理し、アクティブなコードレビューとリファクタリングのためのデシジョンツリーガイダンスを提供。

## いつ使用するか

- React/Next.jsコードのパフォーマンスに関する作成・レビュー
- 遅いページロード、遅いインタラクション、クライアントの高CPU診断
- バンドルサイズまたはLighthouse Core Web Vitalsの回帰監査
- Server Components / APIルートでのウォーターフォール除去
- クライアントサイドの再レンダー削減
- 長いリスト、アニメーション、ハイドレーションの最適化
- `app/`, `pages/`, `components/`、データレイヤーに触れるPRの最適化選択監査

## 優先度インデックス

| 優先度 | カテゴリ | プレフィックス | いつ重要か |
|---|---|---|---|
| 1 — クリティカル | ウォーターフォールの除去 | `async-` | `await` の後に独立した `await` が続くとき |
| 2 — クリティカル | バンドルサイズ最適化 | `bundle-` | ファーストロードJS、ルートレベルインポート、サードパーティライブラリ |
| 3 — 高 | サーバーサイドパフォーマンス | `server-` | RSC、Server Actions、APIルート、SSR |
| 4 — 中高 | クライアントサイドデータフェッチ | `client-` | SWR / TanStack Query / フック内の生 `fetch` |
| 5 — 中 | 再レンダー最適化 | `rerender-` | 高頻度の状態更新、親子ファンアウト |
| 6 — 中 | レンダリングパフォーマンス | `rendering-` | 長いリスト、アニメーション、ハイドレーション |
| 7 — 低中 | JavaScriptパフォーマンス | `js-` | ホットループ、頻繁なアロケーション |
| 8 — 低 | アドバンスドパターン | `advanced-` | エフェクト-イベント統合、安定ref |

## 1. ウォーターフォールの除去（クリティカル）

> 「ウォーターフォールは#1のパフォーマンスキラー」— すべての逐次 `await` がフルネットワークレイテンシーを追加。

### awaitの前に安価な条件チェック

リモートデータをawaitする前に同期条件（props、env、ハードコードフラグ）を確認。

```ts
// 不正
async function Page({ id }: { id: string }) {
  const flag = await getFlag("show-page");
  if (!flag || !id) return null;
  const data = await getData(id);
}

// 正しい — 安価な同期条件を先にショートサーキット
async function Page({ id }: { id: string }) {
  if (!id) return null;
  const flag = await getFlag("show-page");
  if (!flag) return null;
  const data = await getData(id);
}
```

### awaitを使用箇所まで遅延

`await` をデータを使用するブランチに移動。

```ts
// 不正 — データが必要かどうか判断する前にawait
const user = await getUser(id);
if (mode === "guest") return renderGuest();
return renderUser(user);

// 正しい
if (mode === "guest") return renderGuest();
const user = await getUser(id);
return renderUser(user);
```

### 独立した処理にPromise.all

```ts
// 不正 — 逐次
const user = await getUser(id);
const posts = await getPosts(id);
const followers = await getFollowers(id);

// 正しい — 並列
const [user, posts, followers] = await Promise.all([
  getUser(id),
  getPosts(id),
  getFollowers(id),
]);
```

### 部分的な依存関係 — 早期開始、遅延await

```ts
// 正しい — すべてのPromiseを開始、各結果が必要な時のみawait
const userP = getUser(id);
const postsP = getPosts(id);
const profile = await getProfile(id);
if (profile.private) return null;
const [user, posts] = await Promise.all([userP, postsP]);
```

### ストリーミングのためのSuspense

`<Suspense>` バウンダリをデータの近くに配置して、遅いサブツリーがストリームする間にページが描画できるものを描画。トレードオフ：コンテンツ到着時のレイアウトシフト — スペースを確保（スケルトンまたは `min-height`）。

### Server Components：コンポジションによる並列化

```tsx
// 不正 — 1つのコンポーネント内の兄弟awaitが逐次実行
export default async function Page() {
  const user = await getUser();
  const cart = await getCart();
  return <View user={user} cart={cart} />;
}

// 正しい — 子に分割、Reactが並列実行
export default async function Page() {
  return (
    <View>
      <UserSection />
      <CartSection />
    </View>
  );
}
```

## 2. バンドルサイズ最適化（クリティカル）

### バレルでなく直接インポート

バレル `index.ts` ファイルはツリーシェイキングで大部分が除去されても、バンドラーにモジュールグラフ全体をたどらせる。直接インポートで多くの実アプリで200-800msのファーストロードJSを節約。

```ts
// 不正
import { Button, Card, Modal } from "@/components";

// 正しい
import { Button } from "@/components/Button";
import { Card } from "@/components/Card";
import { Modal } from "@/components/Modal";
```

### 重いコンポーネントのダイナミックインポート

```tsx
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("./HeavyChart"), {
  loading: () => <Skeleton />,
  ssr: false, // クライアントのみの場合
});
```

### サードパーティスクリプトの遅延

アナリティクス、ロギング、サポートウィジェットはハイドレーション後にロード。`next/script` の `strategy="afterInteractive"`（デフォルト）または `"lazyOnload"` を使用。

## 3. サーバーサイドパフォーマンス（高）

### Server ActionsをAPIルートと同様に認証

すべての `"use server"` 関数はパブリックエンドポイント。アクション内で認証AND認可 — 呼び出し側Client Componentのゲーティングに依存しない。

### リクエスト単位の重複排除に `React.cache()`

```ts
import { cache } from "react";

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
```

`React.cache` は単一リクエスト内で重複排除。同じレンダーで3つのServer Componentsから `getUser("1")` を呼び出す = 1回のDBクエリ。

### Client Componentsに渡すデータを最小化

Clientが必要なものだけをシリアライズ。DB層でフィールドを除去、ページネーション、カラム射影。

## 4. クライアントサイドデータフェッチ（中高）

### SWR / TanStack Queryで重複排除

複数コンポーネントが `useUser(id)` を呼ぶ場合、1つのネットワークリクエストと1つのキャッシュエントリを共有すべき。共有データに自前の `useEffect` + `fetch` を使わない。

## 5. 再レンダー最適化（中）

### コールバックのみで使用される状態をサブスクライブしない

```tsx
// 不正 — countが変わるたびに再レンダー
const count = useStore((s) => s.count);
const handler = () => doSomething(count);

// 正しい — 呼び出し時に一度読む
const handler = () => {
  const count = useStore.getState().count;
  doSomething(count);
};
```

### レンダー中に導出、useEffectを使わない

```tsx
// 不正
const [full, setFull] = useState("");
useEffect(() => setFull(`${first} ${last}`), [first, last]);

// 正しい
const full = `${first} ${last}`;
```

### 安定コールバックのための関数型 `setState`

```tsx
// 正しい
const increment = useCallback(() => setCount((c) => c + 1), []);
```

### 非緊急更新に `startTransition`

```tsx
const [pending, startTransition] = useTransition();
startTransition(() => setFilters(newFilters));
```

### コンポーネント内にコンポーネントを定義しない

```tsx
// 不正 — OuterのレンダーごとにInnerが新しいコンポーネントになる
function Outer() {
  const Inner = () => <span />;
  return <Inner />;
}
```

各レンダーが新しい `Inner` 型を作成し、リコンシリエーションを無効化してchildrenをアンマウント。

## 6. レンダリングパフォーマンス（中）

### 長いリストに `content-visibility: auto`

```css
.row { content-visibility: auto; contain-intrinsic-size: auto 80px; }
```

ブラウザがオフスクリーンレンダリングをスキップ — 数百行のリストで大きな効果。

### ハイドレーションのフリッカー防止にインラインスクリプト

ハイドレーション前に必要な値（テーマ、ロケール）は、Reactマウント前に `document.documentElement.dataset.*` を設定する `<script>` をインライン化。

## 7. JavaScriptパフォーマンス（低中）

- **DOM/CSS変更のバッチ化** — プロパティごとではなくクラス切替や `cssText` で適用
- **繰り返しルックアップに `Map`** — リニアスキャン `O(n)` vs `O(1)`
- **`filter().map()` を1パスに統合** — `flatMap` または単一 `for`
- **ループの外に `RegExp` をホイスト** — コンパイルは無料ではない
- **メンバーシップに `Set`/`Map`** — `Array.includes` `O(n)` vs `O(1)`

## 8. アドバンスドパターン（低）

### アプリロード時に一度だけ初期化

モジュールレベルシングルトン（テレメトリ、ロガー）にはモジュールスコープフラグでガード — `useEffect` ではない。

## Lighthouse / Web Vitalsマッピング

| メトリクス | 最も関連するカテゴリ |
|---|---|
| **LCP** (Largest Contentful Paint) | ウォーターフォール、バンドルサイズ、リソースヒント |
| **INP** (Interaction to Next Paint) | 再レンダー、レンダリング、JavaScript |
| **CLS** (Cumulative Layout Shift) | レンダリング（Suspense配置、画像寸法） |
| **TBT** (Total Blocking Time) | バンドルサイズ、JavaScript、サードパーティ遅延 |

## 関連

- スキル: [react-patterns](../react-patterns/SKILL.md), [react-testing](../react-testing/SKILL.md), [frontend-patterns](../frontend-patterns/SKILL.md), [accessibility](../accessibility/SKILL.md), [nextjs-turbopack](../nextjs-turbopack/SKILL.md)
- ルール: [rules/react/](../../rules/react/)
- エージェント: `react-reviewer` コードレビューでこれらのルールを強制。`react-build-resolver` 関連ビルド障害を処理
- コマンド: `/react-review`, `/react-build`, `/react-test`

## 帰属

Vercel Labs `react-best-practices` skill（MIT License, copyright Vercel Engineering, v1.0.0 January 2026）から適応。出典: [https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices)。

このスキルはオリジナルの70ルールカタログを単一のナビゲーション可能なリファレンスに再構成・適応したもの。拡張例付きの完全なオリジナルルールセットは上流リポジトリを参照。
