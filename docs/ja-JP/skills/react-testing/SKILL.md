---
name: react-testing
description: React Testing Library、Vitest/Jest、MSWによるネットワークモック、axeによるアクセシビリティアサーション、コンポーネントテストとPlaywright/Cypress E2Eの境界判断。Reactコンポーネント、フック、ページのテスト作成・修正時に使用。
metadata:
  origin: ECC
---

# React テスト

振る舞いに焦点を当てたコンポーネントテスト、カスタムフックテスト、アクセシビリティアサーション、ネットワークレベルモックのための包括的なReactテストパターン。

## いつ使用するか

- Reactコンポーネント、カスタムフック、ページのテスト作成
- レガシーの未テストコンポーネントへのテストカバレッジ追加
- EnzymeまたはクラスコンポーネントパターンからReact Testing Libraryへの移行
- 新しいReactプロジェクトへのVitestまたはJestのセットアップ
- テストでのHTTPリクエストモック
- アクセシビリティ違反のアサーション
- RTL vs Playwright Component Testing vs フルE2Eのテスト配置判断

## コアプリンシプル

ユーザーが見ること、行うことをテストする。実装の詳細ではない。

テストでは：

- 本番環境と同じプロバイダーでコンポーネントをレンダー
- アクセシブルなクエリ（role、label）と `userEvent` で操作
- 可視出力と観察可能な副作用（コールバック発火、リクエスト送信）をアサート

テストでやってはいけないこと：

- コンポーネントの状態、子に渡されたprops、呼び出されたフックの検査
- React自体やフレームワークフックのモック
- レンダー回数やユーザーに影響しないDOM構造のアサーション

## ライブラリ選択

| ランナー | いつ使うか | 備考 |
|---|---|---|
| **Vitest** | Vite、Remix、モダンな構成 | 高速、ネイティブESM、Jest互換API |
| **Jest** | Next.js、CRA、既存リポジトリ | 多くのReactプロジェクトのデフォルト |
| **Playwright Component Testing** | 実ブラウザエンジンが必要 | JSDOMに不足する機能がある場合 |
| **Cypress Component Testing** | 実ブラウザ、Cypress既存利用 | Playwright CTの代替 |

1つを選択。明確なレーン分離がない限り、同じリポジトリでRTL + VitestとPlaywright CTを両方実行しない。

## クエリ優先順位

React Testing Libraryは3段階のクエリを提供 — 上から順に使用：

1. **すべてのユーザーにアクセシブル**: `getByRole`, `getByLabelText`, `getByPlaceholderText`, `getByText`, `getByDisplayValue`
2. **セマンティック**: `getByAltText`, `getByTitle`
3. **テストID（最終手段）**: `getByTestId`

```tsx
// 最善
screen.getByRole("button", { name: /save/i });

// 入力に適切
screen.getByLabelText("Email");

// 最終手段
screen.getByTestId("save-btn");
```

バリアント：

- `getBy*` — マッチしない場合スロー
- `queryBy*` — `null` を返す（「不在をアサート」に使用）
- `findBy*` — 非同期、Promiseを返す（非同期処理後に表示される要素に使用）

## `userEvent` によるユーザー操作

```tsx
import userEvent from "@testing-library/user-event";

test("フォームを送信する", async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(<UserForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText("Email"), "user@example.com");
  await user.click(screen.getByRole("button", { name: /save/i }));

  expect(onSubmit).toHaveBeenCalledWith({ email: "user@example.com" });
});
```

- 常に userEvent 呼び出しを `await`
- テストごとに一度 `userEvent.setup()` を呼び、返された `user` を再利用
- `userEvent` は実ブラウザシーケンスをシミュレート。`fireEvent` は単一のsyntheticイベントをディスパッチ — `userEvent` を推奨

## 非同期パターン

```tsx
// 非同期処理後に表示される要素
expect(await screen.findByText("Loaded")).toBeInTheDocument();

// 副作用アサーション
await waitFor(() => expect(saveSpy).toHaveBeenCalled());

// 消えるべき要素
await waitForElementToBeRemoved(() => screen.queryByText("Loading"));
```

`setTimeout` + アサーションは使わない — フレーキー。上記のマッチャーを使用。

## MSWによるネットワークモック

Mock Service Workerはネットワーク層でモックする。コンポーネント、フック、fetchライブラリすべてが本番と同じ動作をする。

### セットアップ

```ts
// test/setup.ts
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users/:id", ({ params }) =>
    HttpResponse.json({ id: params.id, name: "Alice" }),
  ),
  http.post("/api/users", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: "new-id", ...body }, { status: 201 });
  }),
];

export const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

`onUnhandledRequest: "error"` を設定して、モックされていないリクエストがテストを大きく失敗させる — サイレントパスはレッドより悪い。

### テスト単位のオーバーライド

```tsx
test("500でエラーをレンダーする", async () => {
  server.use(
    http.get("/api/users/:id", () => new HttpResponse(null, { status: 500 })),
  );
  render(<UserPage id="1" />);
  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument();
});
```

## プロバイダーラッピング

`test-utils.tsx` でプロバイダーを一度ラップ：

```tsx
// test-utils.tsx
import { render, RenderOptions } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

export function renderWithProviders(
  ui: React.ReactElement,
  options?: RenderOptions,
) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return render(
    <QueryClientProvider client={queryClient}>
      <ThemeProvider theme={lightTheme}>
        <MemoryRouter>{ui}</MemoryRouter>
      </ThemeProvider>
    </QueryClientProvider>,
    options,
  );
}

export * from "@testing-library/react";
```

すべてのテストファイルで `import { renderWithProviders, screen } from "test-utils"` を使用。

## カスタムフックテスト

```tsx
import { renderHook, act } from "@testing-library/react";

test("useCounterがインクリメントとデクリメントする", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);

  act(() => result.current.decrement());
  expect(result.current.count).toBe(0);
});

test("useCounterが初期値を受け入れる", () => {
  const { result } = renderHook(() => useCounter(10));
  expect(result.current.count).toBe(10);
});

test("useUserがユーザーデータをフェッチする", async () => {
  // QueryClientをラッパーの外でテストごとに1回インスタンス化する。
  // ラッパークロージャ内で作成するとレンダーごとにキャッシュ状態がリセットされ、フレーキーなテストになる。
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );

  const { result } = renderHook(() => useUser("1"), { wrapper });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toEqual({ id: "1", name: "Alice" });
});
```

- 状態変更の呼び出しを `act` でラップ
- フックのパブリックAPIのみをテスト
- コンテキストを使うフックには `wrapper` を渡す

## アクセシビリティアサーション

```tsx
import { axe, toHaveNoViolations } from "jest-axe"; // or vitest-axe
expect.extend(toHaveNoViolations);

test("UserCardにa11y違反がない", async () => {
  const { container } = render(<UserCard user={mockUser} />);
  expect(await axe(container)).toHaveNoViolations();
});
```

すべてのインタラクティブコンポーネントのコンポーネントテストでaxeを実行。検出するもの：

- フォーム入力のラベル不足
- 無効なARIA使用
- 色のコントラスト不良（限定的 — JSDOMには実CSSエンジンがないため、インラインスタイルのみ有効。視覚的コントラストはPlaywrightで）
- 画像のalt テキスト不足
- 見出し順序の違反

クロスリンク: [skills/accessibility/SKILL.md](../accessibility/SKILL.md) でより広範なa11yテストプレイブックを参照。

## スナップショットテストを使わない場合

レンダー出力のスナップショット：

- スタイル変更のたびに壊れる
- レビュー時にラバースタンプされる
- 振る舞いではなく実装の詳細（DOM構造）をテスト

許容されるスナップショットの用途：

- 純粋なデータシリアライズ関数（`formatInvoice(invoice)` → 安定した文字列）
- 生成された設定ファイル（例：webpack設定出力）

コンポーネントの視覚的回帰には、Playwright/CypressのスクリーンショットまたはPercy/Chromaticを使用 — DOM文字列ではなく実際の視覚的差分。

## Playwright / Cypressを使うべき場合

JSDOM（Vitest/Jestで使用）ではできないこと：

- 実レイアウトのレンダー（flexbox、grid、ビューポートクエリ）
- ネイティブブラウザアニメーション、CSSトランジションの実行
- スクロール動作、ドラッグ＆ドロップ、クリップボードからの貼り付けのテスト
- iframe、ポップアップ、ダウンロード、クロスオリジンフローの処理
- 完全なDevToolsサポート付きの制御された環境での実ネットワーク実行

これらのいずれかについては、Playwright Component Testing（実ブラウザでのコンポーネントテスト）またはフルE2Eを使用。[e2e-testing skill](../e2e-testing/SKILL.md) を参照。

判断境界：

- フック、プレゼンテーショナルコンポーネント、ロジック付きフォーム → RTL
- レイアウトが重要なコンポーネントやJSDOMにないブラウザAPIを使用 → Playwright CT
- 複数ページにわたる完全なユーザーフロー → Playwright/Cypress E2E

## カバレッジ目標

| レイヤー | 目標 |
|---|---|
| 純粋ユーティリティ | >=90% |
| カスタムフック | >=85% |
| プレゼンテーショナルコンポーネント | >=80% — 行数ではなく振る舞い |
| コンテナコンポーネント | >=70% — ゴールデンパス + エラー状態 |
| ページ | E2Eで別途カバー。スモークテスト最低限 |

`vitest.config.ts` / `jest.config.js` で設定：

```ts
// vitest.config.ts
test: {
  coverage: {
    provider: "v8",
    reporter: ["text", "html", "lcov"],
    thresholds: {
      lines: 80,
      functions: 80,
      branches: 70,
      statements: 80,
    },
  },
}
```

## アンチパターン

- `container.querySelector("...")` — アクセシビリティクエリをバイパス。実ユーザーが失敗するケースでテストがパスする
- レンダー回数のアサーション — 実装の詳細
- `jest.mock("react", ...)` — Reactをモックしない。代わりにコンポーネントをリファクタ
- デフォルトで子コンポーネントをモック — 分離ではなく統合をテスト。子に重い副作用がある場合のみモック
- `act()` 警告の無視 — 実バグのシグナル（アンマウント後の状態更新、非同期ラッピング不足）
- テスト間でのミュータブル状態の共有 — テスト順序変更でフレーキー
- `it.skip()` を外してもパスするテスト — テストは実際にはアサートしていない

## TDDワークフロー

```
RED     -> 次の要件の失敗テストを書く
GREEN   -> テストをパスさせる最小限のコンポーネントコードを書く
REFACTOR -> コンポーネントを改善、テストはグリーンのまま
REPEAT  -> 次の要件
```

新しいコンポーネントの場合：

1. コンポーネントのprop型とシグネチャを定義
2. 最もシンプルなケースの最初のテストを書く
3. 正しい理由で失敗することを確認
4. パスするのに十分なだけ実装
5. 次のテストケースを追加
6. 3つ目の類似テストがパターンを示したらリファクタ

## テストコマンド

```bash
# Vitest
vitest                            # ウォッチ
vitest run                        # ワンショット
vitest run --coverage             # カバレッジ付き
vitest run path/to/file.test.tsx  # 単一ファイル

# Jest
jest --watch
jest --coverage
jest path/to/file.test.tsx

# CIモード
CI=true vitest run --coverage
```

## 関連

- ルール: [rules/react/testing.md](../../rules/react/testing.md)
- スキル: [react-patterns](../react-patterns/SKILL.md), [accessibility](../accessibility/SKILL.md), [e2e-testing](../e2e-testing/SKILL.md), [tdd-workflow](../tdd-workflow/SKILL.md)
- エージェント: `react-reviewer`（コードレビュー時のテスト品質レビュー）, `tdd-guide`（TDDプロセスの強制）
- コマンド: `/react-test`, `/react-review`

## 例

### MSW と userEvent を使ったフォーム送信

```tsx
test("ユーザーフォームを送信して成功を表示する", async () => {
  server.use(
    http.post("/api/users", () =>
      HttpResponse.json({ id: "1", name: "Alice" }, { status: 201 }),
    ),
  );

  const user = userEvent.setup();
  renderWithProviders(<UserForm />);

  await user.type(screen.getByLabelText("Name"), "Alice");
  await user.type(screen.getByLabelText("Email"), "alice@example.com");
  await user.click(screen.getByRole("button", { name: /save/i }));

  expect(await screen.findByText(/saved successfully/i)).toBeInTheDocument();
});
```

### エラーバウンダリのテスト

```tsx
function Broken() {
  throw new Error("boom");
}

test("エラーバウンダリがフォールバックをレンダーする", () => {
  // 予期されるスローに対するReactのconsole.errorノイズを抑制し、
  // スパイが他のテストにリークして実エラーを隠さないよう復元する。
  const errorSpy = vi.spyOn(console, "error").mockImplementation(() => {});
  try {
    render(
      <ErrorBoundary fallback={<div>問題が発生しました</div>}>
        <Broken />
      </ErrorBoundary>,
    );

    expect(screen.getByText("問題が発生しました")).toBeInTheDocument();
  } finally {
    errorSpy.mockRestore();
  }
});
```

### Suspenseバウンダリのテスト

```tsx
test("ローディングの後にコンテンツを表示する", async () => {
  renderWithProviders(
    <Suspense fallback={<div>読み込み中...</div>}>
      <UserDetail id="1" />
    </Suspense>,
  );

  expect(screen.getByText("読み込み中...")).toBeInTheDocument();
  expect(await screen.findByText("Alice")).toBeInTheDocument();
});
```
