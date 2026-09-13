---
name: prisma-patterns
description: TypeScriptバックエンドにおけるPrisma ORMのパターン — スキーマ設計、クエリ最適化、トランザクション、ページネーション、およびupdateManyがレコードではなくカウントのみを返す問題、$transactionのタイムアウト、migrate devによるDBリセット、一括更新で@updatedAtがスキップされる問題、サーバーレスでのコネクション枯渇などの重大な落とし穴の回避。Prismaスキーマやクエリの記述、トランザクション、マイグレーション、サーバーレス接続制限のデバッグ時に使用します。
metadata:
  origin: ECC
---

# Prisma Patterns（Prisma ORM パターン集）

TypeScriptバックエンドにおけるPrisma ORMの実践的なプロダクションパターンと、見落としがちな落とし穴の解説。

> **パターンを適用する前にバージョンを確認してください:**
> ```bash
> npx prisma --version
> ```
> バージョン間の主な注意点:
> - `relationJoins` は個別クエリではなく JOIN を用いてリレーションを取得できますが、大規模な1対多や深い include では行数の爆発を招く可能性があります（双方でベンチマークを推奨）。
> - `omit` 修飾子および `prisma.$extends` クライアント拡張APIが追加されています。
> - 新しいセットアップでは、`@prisma/client` ではなく `prisma` パッケージ名が使われたり、ドライバーアダプター（`@prisma/adapter-pg` 等）が必要な場合があります。

## 有効化のタイミング

- Prismaスキーマのモデルやリレーションを設計・変更するとき
- クエリ、トランザクション、またはページネーションロジックを実装するとき
- `updateMany`、`deleteMany`、その他の一括処理（bulk operations）を使用するとき
- データベースマイグレーションを実行・計画するとき
- サーバーレス環境（Vercel, AWS Lambda, Cloudflare Workers）にデプロイするとき
- 論理削除（ソフトデリート）やマルチテナントの行フィルタリングを実装するとき

## コアコンセプト

### ID生成戦略

| 戦略 | 使用すべき場合 | 避けるべき場合 |
|---|---|---|
| `@default(cuid())` | デフォルトの推奨 — URLセーフ、ソート可能、衝突なし | 外部システム向けに連番IDが必要な場合 |
| `@default(uuid())` | 非Prismaシステムとの相互運用性が必要な場合 | 書き込み頻度の高いテーブル（ランダムUUIDはB-treeインデックスを断片化させる） |
| `@default(autoincrement())` | 内部結合テーブル、監査ログ | 外部公開ID（総レコード数が推測されるため） |

### スキーマのデフォルト設計

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique  // @unique で自動的にインデックスが作成されるため @@index は不要
  name      String
  role      Role      @default(USER)
  posts     Post[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  deletedAt DateTime?

  @@index([createdAt])
  @@index([deletedAt, createdAt]) // 論理削除 ＋ ソートクエリ用の複合インデックス
}
```

- 外部キー、および `WHERE` や `ORDER BY` で使用されるカラムには `@@index` を追加する。
- 将来的に論理削除が必要になる可能性がある場合は、最初から `deletedAt DateTime?` を宣言しておく（後からの追加は本番テーブルへのマイグレーションが必要）。
- `@updatedAt` は単一の `update` および `upsert` でのみ自動更新されます（一括更新の落とし穴に注意）。

### `include` vs `select`

| | `include` | `select` |
|---|---|---|
| 取得内容 | すべてのスカラーフィールド ＋ 指定したリレーション | 指定したフィールドのみ |
| 推奨用途 | 大半のフィールドと関連データが必要な場合 | ホットパス、巨大テーブル、オーバーフェッチの防止 |
| パフォーマンス | カラム数が多いテーブルでは過剰取得の恐れあり | 最小限のペイロードで高速 |

APIレスポンスから生のPrismaエンティティを直接返してはなりません — 露出フィールドを制御するために必ずレスポンスDTOにマッピングしてください:

```ts
// BAD: passwordHash や deletedAt などの内部フィールドが漏洩する
return await prisma.user.findUniqueOrThrow({ where: { id } });

// GOOD: 明示的なDTOマッピング
const user = await prisma.user.findUniqueOrThrow({ where: { id } });
return { id: user.id, name: user.name, email: user.email };
```

### トランザクション形式の使い分け

| 状況 | 推奨形式 |
|---|---|
| 相互依存のない独立した操作 | 配列形式（Array form） |
| 後続のステップが直前の結果に依存する場合 | インタラクティブ形式（Interactive form） |
| 外部通信（メール送信、外部HTTP API）を含む場合 | トランザクションの完全に外部で実行 |

```ts
// 配列形式 — 1回の往復でバッチ実行
const [user, post] = await prisma.$transaction([
  prisma.user.update({ where: { id }, data: { name } }),
  prisma.post.create({ data: { title, authorId: id } }),
]);

// インタラクティブ形式 — 必ず tx クライアントを使用し、外側の prisma を呼んではならない
const post = await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUniqueOrThrow({ where: { id } });
  if (user.role !== 'ADMIN') throw new Error('Forbidden');
  return tx.post.create({ data: { title, authorId: user.id } });
});
```

### PrismaClient のシングルトン化

`PrismaClient` のインスタンスごとに個別のコネクションプールが開かれます。必ず1回だけインスタンス化してください。

```ts
// lib/prisma.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

`globalThis` パターンにより、Next.js などのホットリロード時に重複インスタンスが生成されるのを防ぎます。

### N+1 問題の防止

ループ内でリレーションをフェッチすると、行ごとにクエリが発行されます。

```ts
// BAD: N+1 — ユーザーごとにクエリが発行される
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({ where: { authorId: user.id } });
}

// GOOD: 単一クエリで一括取得
const users = await prisma.user.findMany({ include: { posts: true } });
```

## コード例

### カーソルベースのページネーション（大量データやフィード向け）

```ts
async function getPosts(cursor?: string, limit = 20) {
  const items = await prisma.post.findMany({
    where: { published: true },
    orderBy: [
      { createdAt: 'desc' },
      { id: 'desc' }, // 重複タイムスタンプ時のページネーション崩れを防ぐセカンダリソート
    ],
    take: limit + 1,
    ...(cursor && { cursor: { id: cursor }, skip: 1 }),
  });

  const hasNextPage = items.length > limit;
  if (hasNextPage) items.pop();

  return { items, nextCursor: hasNextPage ? items[items.length - 1].id : null };
}
```

`limit + 1` 件取得して末尾を pop することで、追加の count クエリなしで次ページの有無を判定できます。

### エラーハンドリング

```ts
import { Prisma } from '@prisma/client';

try {
  await prisma.user.create({ data: { email } });
} catch (e) {
  if (e instanceof Prisma.PrismaClientKnownRequestError) {
    if (e.code === 'P2002') throw new ConflictError('メールアドレスが既に存在します');
    if (e.code === 'P2025') throw new NotFoundError('レコードが見つかりません');
    if (e.code === 'P2003') throw new BadRequestError('参照先レコードが存在しません');
  }
  throw e;
}
```

## 重大な落とし穴とアンチパターン

### `updateMany` はレコードではなく件数（count）のみを返す

```ts
// BAD: 戻り値は { count: 2 } であり、更新後レコードは取得できない
const users = await prisma.user.updateMany({ where: { role: 'GUEST' }, data: { role: 'USER' } });

// GOOD: 先に対象IDを取得してから更新し、必要な行のみ再取得する
const targets = await prisma.user.findMany({ where: { role: 'GUEST' }, select: { id: true } });
const ids = targets.map((u) => u.id);
await prisma.user.updateMany({ where: { id: { in: ids } }, data: { role: 'USER' } });
const updated = await prisma.user.findMany({ where: { id: { in: ids } } });
```

`deleteMany` も同様で、削除された行自体は返されず `{ count: n }` のみが返されます。

### `$transaction` のインタラクティブ形式は5秒でタイムアウトする

```ts
// BAD: トランザクション内部で外部APIを呼ぶとデフォルトの5秒を超えて失敗する
await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUniqueOrThrow({ where: { id } });
  await sendWelcomeEmail(user.email); // 外部通信
  await tx.user.update({ where: { id }, data: { emailSent: true } });
});

// GOOD: 外部通信はトランザクションの外側で行う
const user = await prisma.user.findUniqueOrThrow({ where: { id } });
await sendWelcomeEmail(user.email);
await prisma.user.update({ where: { id }, data: { emailSent: true } });
```

### `migrate dev` はデータベースを全消去（リセット）することがある

`migrate dev` はスキーマの乖離を検出すると、DBリセットを促すプロンプトを表示し、全データを破棄する恐れがあります。

```bash
# 共有開発環境、ステージング、本番環境では絶対に実行してはならない
npx prisma migrate dev

# 本番・ステージング・CI環境では常に deploy を使用すること
npx prisma migrate deploy
```

### `@updatedAt` は `updateMany` では更新されない

`@updatedAt` が自動更新されるのは `update` と `upsert` のみです。一括更新では古い値のまま放置されます。

```ts
// BAD: updatedAt が更新されない
await prisma.post.updateMany({ where: { authorId }, data: { published: true } });

// GOOD: 手動で現在時刻を設定する
await prisma.post.updateMany({
  where: { authorId },
  data: { published: true, updatedAt: new Date() },
});
```

### 論理削除と `findUniqueOrThrow` の組み合わせの罠

`findUniqueOrThrow` はDB上にレコードが存在しない場合にのみエラーを投げます。論理削除された行もDB上には残っているため、そのまま返されてしまいます。また、複合ユニークインデックスでない限り `where` に `deletedAt: null` を追加すると型エラーになります。**`findFirstOrThrow`** を使用してください。

```ts
// BAD: 論理削除されたユーザーが返されてしまう
const user = await prisma.user.findUniqueOrThrow({ where: { id } });

// GOOD: findFirstOrThrow を使用して deletedAt: null を条件に含める
const user = await prisma.user.findFirstOrThrow({ where: { id, deletedAt: null } });
```

### `where` のない `deleteMany` はテーブル全体を消去する

```ts
// BAD: テーブルの全行が無言で削除される
await prisma.post.deleteMany();

// GOOD: 必ず where 条件を指定する
await prisma.post.deleteMany({ where: { authorId: userId } });
```

## ベストプラクティスまとめ

- CI/CD では常に `migrate deploy` を使い、`migrate dev` はローカル開発のみにとどめる。
- エンティティをレスポンスDTOにマッピングし、内部フィールドの漏洩を防ぐ。
- サービス境界で `PrismaClientKnownRequestError` をキャッチし、ドメインエラーに変換する。
- 手動の null チェックの代わりに `findFirstOrThrow` 等を活用する。
- サーバーレス環境では `connection_limit=1` と外部コネクションプーラー（PgBouncer等）を併用する。
- `deleteMany` には必ず `where` を渡す。
- `updateMany` では手動で `updatedAt: new Date()` を付与する。
