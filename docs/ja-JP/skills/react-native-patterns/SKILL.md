---
name: react-native-patterns
description: React NativeおよびExpoアプリの開発パターン — Expo Routerによる画面遷移、関心事ごとの状態分離（サーバー/クライアント/ルート/フォーム）、Zodによる検証を伴うTanStack Queryのデータフェッチ、パフォーマンスの高いリストレンダリング、NativeWind/StyleSheetによるスタイリング、ネイティブAPIの扱い、セキュアストレージ。React Native / Expoの画面、コンポーネント、ナビゲーション、データ層を構築・編集する場合に使用します。
origin: ECC
---

# React Native / Expo Patterns（React Native / Expo パターン）

Expo を使用した本番品質の React Native アプリケーションを構築するための実践的パターン集。ナビゲーション、状態管理、データ取得、リスト、スタイリング、ネイティブAPIを網羅しています。本スキルは `rules/react-native/` ルールセットと連携します: ルールは「*何を*強制すべきか」を規定し、本スキルは「*どのように*実装するか」を示します。

以下で言及するライブラリ（NativeWind, Zustand/Jotai, TanStack Query 等）は一般的かつ実績のある選択肢としての例示です — 特定のパッケージよりもパターンの本質が重要であり、同等のライブラリであれば同様に機能します。検証にはECCの既存のTypeScriptルールと一貫性を保つため Zod を使用します。

これらのパターンは、New Architecture（最新のExpo SDKにおけるデフォルト、SDK 55以降は必須）上のマネージド Expo ワークフロー（Expo Router, EAS, `expo-*` モジュール）を前提としています。ブラウザの DOM を前提としたものではありません — React Native には `<div>` もURLバーも、Webブラウザ標準のデータフェッチ挙動も存在しません。

## 有効化のタイミング

以下の場合に使用します:

- React Native / Expo の画面、コンポーネント、またはナビゲーションを構築・編集するとき
- Expo Router によるファイルベースルーティング（`app/` ディレクトリ）をセットアップするとき
- 状態の適切な配置場所（サーバーキャッシュ vs クライアントストア vs ルートパラメータ vs フォーム）を決定するとき
- TanStack Query によるデータ取得を実装し、Zod でレスポンスを検証するとき
- 長大または重いリストをレンダリングするとき
- スタイリング手法（NativeWind または StyleSheet）を選択・適用するとき
- ネイティブデバイスAPI（カメラ、位置情報、通知）またはセキュアストレージにアクセスするとき
- モバイル特有の問題について React Native コードをレビューするとき

Web/React-DOM のパターン（URLを状態として直接扱う、`<div>`、ブラウザ向けSWR等）をここに適用してはなりません。

## コアコンセプト

### プロジェクト構造（Expo Router）

`app/` 配下のファイルベースルーティング。ルートファイルは薄く保ちます: パラメータを読み取って検証し、実際の画面コンポーネント（`components/` または `features/` に配置）へ委譲します。

```
app/
  _layout.tsx          # ルートスタック
  (tabs)/
    _layout.tsx        # タブナビゲーター
    index.tsx          # ホーム画面
  user/[id].tsx        # 動的ルート
components/
features/
  user/UserProfile.tsx
```

### ナビゲーション: ルートパラメータの検証

ディープリンクや動的ルートから渡されるパラメータは信頼できない文字列です。使用前に必ず Zod で検証してください。

```tsx
// app/user/[id].tsx
import { useLocalSearchParams, router } from 'expo-router'
import { z } from 'zod'
import { UserProfile } from '@/features/user/UserProfile'

const Params = z.object({ id: z.string().uuid() })

export default function UserRoute() {
  const parsed = Params.safeParse(useLocalSearchParams())
  if (!parsed.success) {
    router.replace('/not-found')
    return null
  }
  return <UserProfile userId={parsed.data.id} />
}
```

### 状態管理: 関心事の明確な分離

サーバーから取得したデータをクライアントのグローバルストアに二重保持してはなりません。それぞれの関心事に適した場所があります。

| 関心事 | 一般的な選択肢 |
|---------|------|
| サーバー状態（リモートデータ） | サーバーキャッシュライブラリ（TanStack Query, SWR） |
| クライアント/UI状態 | 軽量ストア（Zustand, Jotai）または Context |
| ルート/ナビゲーション状態 | Expo Router のパラメータ |
| フォーム状態 | フォームライブラリ（React Hook Form 等） ＋ スキーマ検証 |
| シークレット / トークン | `expo-secure-store` |
| 非機密情報の永続化 | `AsyncStorage` / MMKV |

状態の共有が真に必要になるまでは、コンポーネントローカルの `useState` を優先してください。

### データ取得: キャッシュライブラリ ＋ Zod

`useEffect` 内での手動 fetch ではなく、サーバーキャッシュライブラリ（TanStack Query 等）を使用します。システムの境界でバリデーションを行い、スキーマから型を推論します。ローディング、エラー、空（empty）の状態を明示的に処理してください。

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { z } from 'zod'

const User = z.object({ id: z.string(), email: z.string().email() })
type User = z.infer<typeof User>

export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: async (): Promise<User> => User.parse(await api.getUser(id)),
  })
}

export function useUpdateEmail(id: string) {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (email: string) => api.updateEmail(id, email),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['user', id] }),
  })
}
```

### リスト: 仮想化を徹底（ScrollView内で大きな配列をmap展開してはならない）

```tsx
import { FlatList } from 'react-native'

<FlatList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={renderItem}          // memo化すること
  initialNumToRender={10}
  windowSize={5}
/>
```

大規模または多様なレイアウトを持つリストには、Shopifyの `FlashList` の利用を検討してください。

### スタイリング: システムを1つに統一

`StyleSheet.create()` はフレームワーク標準の選択肢であり、ユーティリティクラスライブラリ（NativeWind 等）も一般的です。どちらか1つを選択し、一貫性を維持してください。頻繁に再描画されるホットパス上で、JSX内にインラインでスタイルオブジェクトを生成してはなりません。

```tsx
// NativeWind の場合
<View className="p-4 rounded-2xl bg-white">
  <Text className="text-base font-semibold">Hello</Text>
</View>

// StyleSheet の場合
const styles = StyleSheet.create({ card: { padding: 16, borderRadius: 16, backgroundColor: '#fff' } })
<View style={styles.card}>...</View>
```

### ネイティブAPI: カスタムフックへの隠蔽とクリーンアップ

Expo SDKの呼び出しやイベント購読は JSX の中に直接書かず、`use*` フックの中にカプセル化します。必ずクリーンアップ関数を実装してください。

```tsx
import { useEffect, useState } from 'react'
import * as Location from 'expo-location'

type LocationState =
  | { status: 'loading' }
  | { status: 'denied' }
  | { status: 'granted'; coords: Location.LocationObjectCoords }

export function useCurrentLocation() {
  const [state, setState] = useState<LocationState>({ status: 'loading' })

  useEffect(() => {
    let active = true
    ;(async () => {
      const { status } = await Location.requestForegroundPermissionsAsync()
      if (status !== 'granted') {
        if (active) setState({ status: 'denied' })
        return
      }
      const pos = await Location.getCurrentPositionAsync({})
      if (active) setState({ status: 'granted', coords: pos.coords })
    })()
    return () => { active = false }   // アンマウント後の古い結果を無視
  }, [])

  return state
}
```

### 認証トークンのセキュアストレージ保存

```tsx
import * as SecureStore from 'expo-secure-store'

await SecureStore.setItemAsync('auth_token', token)   // iOS Keychain / Android Keystore
const token = await SecureStore.getItemAsync('auth_token')
```

## コード例

### 画面全体の実装例: ルート → クエリ → リスト → 各種状態

```tsx
// app/(tabs)/orders.tsx
import { memo, useCallback } from 'react'
import { FlatList, Text, View } from 'react-native'
import { useQuery } from '@tanstack/react-query'
import { z } from 'zod'

const OrderSchema = z.object({ id: z.string(), total: z.number(), status: z.string() })
const OrdersSchema = z.array(OrderSchema)
type Order = z.infer<typeof OrderSchema>

function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: async () => OrdersSchema.parse(await api.listOrders()),
  })
}

// レンダリング間で参照を安定させるために memo 化
const OrderRow = memo(function OrderRow({ item }: { item: Order }) {
  return (
    <View className="px-4 py-3 border-b border-neutral-200">
      <Text className="font-medium">#{item.id}</Text>
      <Text className="text-neutral-500">{item.status} · ${item.total}</Text>
    </View>
  )
})

export default function OrdersScreen() {
  const { data, isLoading, isError, refetch, isRefetching } = useOrders()
  const renderItem = useCallback(({ item }: { item: Order }) => <OrderRow item={item} />, [])

  if (isLoading) return <Centered><Text>Loading…</Text></Centered>
  if (isError) return <Centered><Text accessibilityRole="alert">注文を読み込めませんでした。</Text></Centered>
  if (!data?.length) return <Centered><Text>注文履歴がありません。</Text></Centered>

  return (
    <FlatList
      data={data}
      keyExtractor={(o) => o.id}
      onRefresh={refetch}
      refreshing={isRefetching}
      renderItem={renderItem}
    />
  )
}
```

## アンチパターン

```tsx
// BAD: ScrollView 内で大きな配列を直接 map 展開（仮想化されず、メモリ肥大化とカクつきの原因）
<ScrollView>{items.map((i) => <Row key={i.id} item={i} />)}</ScrollView>
// GOOD: FlatList または FlashList を使用する

// BAD: サーバーデータをクライアントのストアにコピー（真実の二重化、古いデータの残存）
const useStore = create((set) => ({ users: [], setUsers: (u) => set({ users: u }) }))
useEffect(() => { getUsers().then(setUsers) }, [])
// GOOD: TanStack Query がサーバー状態を管理し、必要な派生データのみ算出する

// BAD: トークンを AsyncStorage に保存（暗号化されないため危険）
await AsyncStorage.setItem('auth_token', token)
// GOOD: expo-secure-store を使用する

// BAD: ディープリンクのパラメータを検証なしに信用する
const { id } = useLocalSearchParams(); fetchUser(id)
// GOOD: 使用前に必ず Zod で検証する

// BAD: 再描画のたびにインラインでスタイルオブジェクトを再生成する
<View style={{ padding: 16, backgroundColor: '#fff' }} />
// GOOD: モジュールスコープで StyleSheet.create を定義するか、NativeWind を使用する

// BAD: 本物のシークレットをクライアントバンドルに含める
const STRIPE_SECRET = 'sk_live_...'
// GOOD: 特権的なAPI呼び出しはバックエンド側で行い、クライアントには公開キーのみ配信する
```

## ベストプラクティス

- ルートファイルは薄く保ち、ビジネスロジックは画面コンポーネントやカスタムフックに配置する。
- 外部からのすべての入力（APIレスポンス、ルート引数、Push通知データ）を Zod で検証する。
- サーバー状態は TanStack Query 等に任せ、クライアントのグローバルストアは最小限に留める。
- ローディング、エラー、および空（データなし）の状態を必ずUIとして実装する。
- リストは必ず仮想化し、`renderItem` はメモ化、安定した `keyExtractor` を提供する。
- アニメーションには `react-native-reanimated` を使用し、UIスレッドで動かす（JSスレッドをブロックしない）。
- 認証トークンは `expo-secure-store` に保存する。認可判定をクライアント側に委ねてはならない。
- セーフエリア、Dynamic Type（OSのフォントサイズ設定）、およびアクセシビリティラベル/ロールを初期から考慮する。
- リリース前に、すべてのネイティブ依存パッケージが New Architecture と互換性があるか確認する。

## 関連スキル

- `frontend-patterns` — React/Next.js（Web）のパターン。共通概念の理解には役立ちますが、DOM依存のコードは流用できません。
- `coding-standards` — TypeScript/JavaScriptのコーディング規約。
- `tdd-workflow`, `e2e-testing` — テスト手法（RNでは Jest + React Native Testing Library、Maestro/Detox を使用）。
- `security-review` — モバイルのバンドル/シークレット管理を補完する一般的なセキュリティチェックリスト。
