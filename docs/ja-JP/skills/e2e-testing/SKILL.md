---
name: e2e-testing
description: Playwright E2Eテストパターン、Page Object Model、設定、CI/CD統合、アーティファクト管理、および不安定なテスト戦略。
origin: ECC
---

# E2E テストパターン

安定的で高速、保守可能なE2Eテストスイートを構築するための包括的なPlaywrightパターン。

## テストファイルの構成

```
tests/
├── e2e/
│   ├── auth/
│   │   ├── login.spec.ts
│   │   ├── logout.spec.ts
│   │   └── register.spec.ts
│   ├── features/
│   │   ├── browse.spec.ts
│   │   ├── search.spec.ts
│   │   └── create.spec.ts
│   └── api/
│       └── endpoints.spec.ts
├── fixtures/
│   ├── auth.ts
│   └── data.ts
└── playwright.config.ts
```

## Page Object Model (POM)

```typescript
import { Page, Locator } from '@playwright/test'

export class ItemsPage {
  readonly page: Page
  readonly searchInput: Locator
  readonly itemCards: Locator
  readonly createButton: Locator

  constructor(page: Page) {
    this.page = page
    this.searchInput = page.locator('[data-testid="search-input"]')
    this.itemCards = page.locator('[data-testid="item-card"]')
    this.createButton = page.locator('[data-testid="create-btn"]')
  }

  async goto() {
    await this.page.goto('/items')
    await this.page.waitForLoadState('networkidle')
  }

  async search(query: string) {
    await this.searchInput.fill(query)
    await this.page.waitForResponse(resp => resp.url().includes('/api/search'))
    await this.page.waitForLoadState('networkidle')
  }

  async getItemCount() {
    return await this.itemCards.count()
  }
}
```

## テスト構造

```typescript
import { test, expect } from '@playwright/test'
import { ItemsPage } from '../../pages/ItemsPage'

test.describe('アイテム検索', () => {
  let itemsPage: ItemsPage

  test.beforeEach(async ({ page }) => {
    itemsPage = new ItemsPage(page)
    await itemsPage.goto()
  })

  test('キーワードで検索できること', async ({ page }) => {
    await itemsPage.search('test')

    const count = await itemsPage.getItemCount()
    expect(count).toBeGreaterThan(0)

    await expect(itemsPage.itemCards.first()).toContainText(/test/i)
    await page.screenshot({ path: 'artifacts/search-results.png' })
  })

  test('結果なしを適切に処理すること', async ({ page }) => {
    await itemsPage.search('xyznonexistent123')

    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
    expect(await itemsPage.getItemCount()).toBe(0)
  })
})
```

## Playwright 設定

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'playwright-results.xml' }],
    ['json', { outputFile: 'playwright-results.json' }]
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10000,
    navigationTimeout: 30000,
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
})
```

## 不安定なテストのパターン

### 隔離

```typescript
test('不安定: 複雑な検索', async ({ page }) => {
  test.fixme(true, '不安定 - Issue #123')
  // テストコード...
})

test('条件付きスキップ', async ({ page }) => {
  test.skip(process.env.CI, 'CIで不安定 - Issue #123')
  // テストコード...
})
```

### 不安定性の特定

```bash
npx playwright test tests/search.spec.ts --repeat-each=10
npx playwright test tests/search.spec.ts --retries=3
```

### よくある原因と修正方法

**競合状態:**
```typescript
// 悪い例: 要素の準備完了を仮定
await page.click('[data-testid="button"]')

// 良い例: 自動待機ロケーター
await page.locator('[data-testid="button"]').click()
```

**ネットワークタイミング:**
```typescript
// 悪い例: 任意のタイムアウト
await page.waitForTimeout(5000)

// 良い例: 特定の条件を待機
await page.waitForResponse(resp => resp.url().includes('/api/data'))
```

**アニメーションタイミング:**
```typescript
// 悪い例: アニメーション中にクリック
await page.click('[data-testid="menu-item"]')

// 良い例: 安定するまで待機
await page.locator('[data-testid="menu-item"]').waitFor({ state: 'visible' })
await page.waitForLoadState('networkidle')
await page.locator('[data-testid="menu-item"]').click()
```

## アーティファクト管理

### スクリーンショット

```typescript
await page.screenshot({ path: 'artifacts/after-login.png' })
await page.screenshot({ path: 'artifacts/full-page.png', fullPage: true })
await page.locator('[data-testid="chart"]').screenshot({ path: 'artifacts/chart.png' })
```

### トレース

```typescript
await browser.startTracing(page, {
  path: 'artifacts/trace.json',
  screenshots: true,
  snapshots: true,
})
// ... テストアクション ...
await browser.stopTracing()
```

### 動画

```typescript
// playwright.config.ts内
use: {
  video: 'retain-on-failure',
  videosPath: 'artifacts/videos/'
}
```

## CI/CD 統合

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
        env:
          BASE_URL: ${{ vars.STAGING_URL }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## テストレポートテンプレート

```markdown
# E2E テストレポート

**日付:** YYYY-MM-DD HH:MM
**所要時間:** Xm Ys
**ステータス:** 合格 / 不合格

## サマリー
- 合計: X | 合格: Y (Z%) | 不合格: A | 不安定: B | スキップ: C

## 失敗したテスト

### テスト名
**ファイル:** `tests/e2e/feature.spec.ts:45`
**エラー:** 要素が表示されることが期待されました
**スクリーンショット:** artifacts/failed.png
**推奨修正:** [説明]

## アーティファクト
- HTMLレポート: playwright-report/index.html
- スクリーンショット: artifacts/*.png
- 動画: artifacts/videos/*.webm
- トレース: artifacts/*.zip
```

## ウォレット / Web3 テスト

```typescript
test('ウォレット接続', async ({ page, context }) => {
  // ウォレットプロバイダーのモック
  await context.addInitScript(() => {
    window.ethereum = {
      isMetaMask: true,
      request: async ({ method }) => {
        if (method === 'eth_requestAccounts')
          return ['0x1234567890123456789012345678901234567890']
        if (method === 'eth_chainId') return '0x1'
      }
    }
  })

  await page.goto('/')
  await page.locator('[data-testid="connect-wallet"]').click()
  await expect(page.locator('[data-testid="wallet-address"]')).toContainText('0x1234')
})
```

## 金融 / クリティカルフローテスト

```typescript
test('取引実行', async ({ page }) => {
  // 本番環境ではスキップ — 実際のお金が関わる
  test.skip(process.env.NODE_ENV === 'production', '本番環境ではスキップ')

  await page.goto('/markets/test-market')
  await page.locator('[data-testid="position-yes"]').click()
  await page.locator('[data-testid="trade-amount"]').fill('1.0')

  // プレビューの確認
  const preview = page.locator('[data-testid="trade-preview"]')
  await expect(preview).toContainText('1.0')

  // 確認してブロックチェーンを待機
  await page.locator('[data-testid="confirm-trade"]').click()
  await page.waitForResponse(
    resp => resp.url().includes('/api/trade') && resp.status() === 200,
    { timeout: 30000 }
  )

  await expect(page.locator('[data-testid="trade-success"]')).toBeVisible()
})
```
