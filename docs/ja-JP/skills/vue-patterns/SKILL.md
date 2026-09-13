---
name: vue-patterns
description: Vue.js 3 Composition APIのパターン、コンポーネントアーキテクチャ、リアクティビティのベストプラクティス、Piniaによる状態管理、Vue Routerのナビゲーション、NuxtのSSRパターン。Vue、Nuxt、Vite、Piniaプロジェクトで有効化されます。Vue 3、Nuxt、Piniaコード（Composition API、リアクティビティ、ルーター遷移）を構築・レビューする場合に使用します。
origin: ECC
---

# Vue.js Patterns and Best Practices（Vue.js パターンとベストプラクティス）

Composition API（`<script setup>`）を用いた Vue.js 3 開発のための包括的ガイド。コンポーネント設計、リアクティビティ、状態管理、ルーティング、テスト、SSRパターンを網羅しています。バニラVueと異なる部分についてはNuxt特有のガイダンスも記載しています。

## 有効化のタイミング

以下の場合に使用します:
- Vue.js、Nuxt、Vite + Vue、または Pinia を使用しているプロジェクト
- Vueのコンポーネント設計、コンポーザブル（Composables）、リアクティビティ、または状態管理についての質問時
- 単一ファイルコンポーネント（`.vue` ファイル）のレビュー時
- Vue Router、Pinia ストア、または Vite/Vitest の設定時
- Vue特有のパフォーマンス、セキュリティ、またはSSRパターンの検討時

---

## 1. プロジェクト構造

### 推奨レイアウト（機能別構成）

```
src/
├── api/              # APIクライアントおよびエンドポイント定義
├── assets/           # 静的アセット（画像、フォント、アイコン）
├── components/       # 共通/再利用可能コンポーネント
│   ├── base/         # 基本UIプリミティブ（Button, Input, Modal）
│   └── features/     # 機能固有の共有コンポーネント
├── composables/      # 再利用可能な Composition API ロジック
├── layouts/          # ページレイアウト（任意）
├── pages/            # ルートレベルのページコンポーネント
├── router/           # Vue Router の設定
├── stores/           # Pinia ストア
├── types/            # TypeScript の型定義
├── utils/            # 純粋なユーティリティ関数
└── App.vue           # ルートコンポーネント
```

---

## 2. コンポーネント設計

### 単一ファイルコンポーネント（SFC）の記述順序

```vue
<script setup lang="ts">
// 1. インポート（vue → エコシステム → 絶対パス → 相対パス）
// 2. Props & Emits & Slots の定義
// 3. Composables の呼び出し
// 4. ローカル状態（ref/reactive）
// 5. 算出プロパティ（computed）
// 6. メソッド・関数
// 7. ウォッチャー（watch/watchEffect）
// 8. ライフサイクルフック（onMounted 等）
</script>

<template>
  <!-- テンプレート内容 -->
</template>

<style scoped>
  /* スコープ付きスタイル */
</style>
```

### Props のベストプラクティス

```ts
// デフォルト値付きの型ベースProps定義
interface Props {
  label: string;
  variant?: "primary" | "secondary";
  disabled?: boolean;
  items: Item[];
}

const props = withDefaults(defineProps<Props>(), {
  variant: "primary",
  disabled: false,
});
```

- 必ず型を指定し、必要に応じて `default` を設定する。
- Props を直接変更してはならない（イミュータブルの原則） — 変更が必要な場合はイベントを emit する。
- v-model バインディングには、Vue 3.4+ の `defineModel()` または `modelValue` ＋ `update:modelValue` を使用する。

### イベント定義（Emits）

```ts
const emit = defineEmits<{
  submit: [];
  "update:modelValue": [value: string];
  select: [id: string, index: number];
}>();
```

---

## 3. Composables（再利用可能ロジック）

### 構造

```ts
// composables/useDebounce.ts
export function useDebounce<T>(value: MaybeRef<T>, delay: number): Ref<T> {
  const debounced = ref(toValue(value)) as Ref<T>;

  let timer: ReturnType<typeof setTimeout>;
  watch(
    () => toValue(value),
    (newVal) => {
      clearTimeout(timer);
      timer = setTimeout(() => { debounced.value = newVal; }, delay);
    }
  );

  onUnmounted(() => clearTimeout(timer));
  return readonly(debounced);
}
```

### ルール
- 関数名は必ず `use` プレフィックスで始める。
- プリミティブ値そのものではなく、リアクティブな値（`ref`, `computed`）を返す。
- 引数は `MaybeRef` や `toValue()` を通じてリアクティブな入力を許容する。
- 副作用は `onUnmounted` やウォッチャーのクリーンアップ関数で確実に解除する。

---

## 4. 状態管理（Pinia Setup Store 推奨）

```ts
// stores/useCartStore.ts
export const useCartStore = defineStore("cart", () => {
  const items = ref<CartItem[]>([]);
  const isLoading = ref(false);

  const totalPrice = computed(() =>
    items.value.reduce((sum, i) => sum + i.price * i.quantity, 0)
  );
  const itemCount = computed(() =>
    items.value.reduce((sum, i) => sum + i.quantity, 0)
  );

  async function addItem(productId: string) {
    isLoading.value = true;
    try {
      const item = await fetchProduct(productId);
      const existing = items.value.find(i => i.id === item.id);
      if (existing) existing.quantity++;
      else items.value.push({ ...item, quantity: 1 });
    } finally {
      isLoading.value = false;
    }
  }

  return { items, isLoading, totalPrice, itemCount, addItem };
});
```

---

## 5. Vue 3.5+ の新機能

### Props のリアクティブな分割代入（Reactive Props Destructure）

Vue 3.5 では、`defineProps()` から分割代入した変数が自動的にリアクティビティを維持するようになりました:

```ts
// Vue 3.5+: 分割代入してもリアクティブ（toRefs は不要）
const { count = 0, msg = "hello" } = defineProps<{
  count?: number;
  msg?: string;
}>();

// 注意: 分割代入した変数を直接 watch することはできず、ゲッター関数が必要
watch(() => count, (newVal) => { ... });
```

### `useTemplateRef()`

テンプレート参照において、変数名の一致に依存する従来の ref の代わりに `useTemplateRef()` を使用します:

```ts
import { useTemplateRef } from "vue";
const inputEl = useTemplateRef<HTMLInputElement>("input");
// template 側の ref="input" 属性と紐づく
```

### `onWatcherCleanup()`

グローバルにインポート可能なウォッチャークリーンアップ関数（Vue 3.5+）:

```ts
import { watch, onWatcherCleanup } from "vue";

watch(userId, async (newId) => {
  const controller = new AbortController();
  onWatcherCleanup(() => controller.abort());
  // ... fetch with signal
});
```

### `useId()`

アクセシビリティやフォーム要素のための、SSRセーフな一意ID生成:

```ts
import { useId } from "vue";
const id = useId();
```

---

## アンチパターンと修正方法

| アンチパターン | なぜ問題か | 修正方法 |
|-------------|---------------|---------|
| `v-if` と `v-for` を同一要素で使用 | 実行順序が曖昧になりバグの原因となる | `computed` でフィルタリングした配列を `v-for` で回す |
| `v-for` の key に配列インデックスを使用 | 並び替えや削除時に状態の不整合が発生 | 一意なデータベースID（`item.id` 等）を使用する |
| Props を直接変更する | 単方向データフローに違反する | イベントを emit するか、`v-model` を使用する |
| ユーザー入力に `v-html` を使用 | XSS（クロスサイトスクリプティング）の脆弱性 | DOMPurify でサニタイズするか、通常のテキスト補間を使用 |
| Vue 3 での Mixins の使用 | 出所が不明瞭で名前衝突の原因となる | Composables に置き換える |
| Composable 内のモジュールスコープでの副作用 | 全インスタンスで副作用が共有されてしまう | `onMounted` / `onUnmounted` 内でスコープを絞る |
| 再代入される状態に `reactive()` を使用 | オブジェクトの再代入でリアクティビティが失われる | `ref()` を使用する |
| クリーンアップのないウォッチャー | メモリリークや競合状態の原因となる | `onWatcherCleanup()` や `onCleanup` を使用する |
| 新規 Vue 3 コードでの Options API の使用 | エコシステム全体の Composition API への移行に逆行 | `<script setup>` を使用する |
