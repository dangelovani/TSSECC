---
name: dotnet-patterns
description: C#と.NET言語固有のパターン、規約、依存性注入、async/await、およびロバストで保守可能な.NETアプリケーション構築のためのベストプラクティス。
origin: ECC
---

# .NET 開発パターン

堅牢で高性能、保守可能なアプリケーションを構築するための慣用的なC#と.NETパターン。

## いつ使用するか

- 新しいC#コードを書くとき
- C#コードをレビューするとき
- 既存の.NETアプリケーションをリファクタリングするとき
- ASP.NET Coreでサービスアーキテクチャを設計するとき

## 基本原則

### 1. 不変性を優先する

データモデルにはレコードとinit専用プロパティを使用する。可変性は明示的で正当な理由がある場合のみ選択すべき。

```csharp
// 良い例: 不変な値オブジェクト
public sealed record Money(decimal Amount, string Currency);

// 良い例: initセッターによる不変なDTO
public sealed class CreateOrderRequest
{
    public required string CustomerId { get; init; }
    public required IReadOnlyList<OrderItem> Items { get; init; }
}

// 悪い例: パブリックセッターを持つ可変モデル
public class Order
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
}
```

### 2. 暗黙より明示

null許容性、アクセス修飾子、意図を明確にする。

```csharp
// 良い例: 明示的なアクセス修飾子とnull許容性
public sealed class UserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository repository, ILogger<UserService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task<User?> FindByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _repository.FindByIdAsync(id, cancellationToken);
    }
}
```

### 3. 抽象に依存する

サービス境界にはインターフェースを使用する。DIコンテナ経由で登録する。

```csharp
// 良い例: インターフェースベースの依存関係
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(string customerId, CancellationToken cancellationToken);
    Task AddAsync(Order order, CancellationToken cancellationToken);
}

// 登録
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
```

## Async/Await パターン

### 適切なAsync使用法

```csharp
// 良い例: 末端までasync、CancellationToken付き
public async Task<OrderSummary> GetOrderSummaryAsync(
    Guid orderId,
    CancellationToken cancellationToken)
{
    var order = await _repository.FindByIdAsync(orderId, cancellationToken)
        ?? throw new NotFoundException($"Order {orderId} not found");

    var customer = await _customerService.GetAsync(order.CustomerId, cancellationToken);

    return new OrderSummary(order, customer);
}

// 悪い例: asyncのブロッキング
public OrderSummary GetOrderSummary(Guid orderId)
{
    var order = _repository.FindByIdAsync(orderId, CancellationToken.None).Result; // デッドロックの危険
    return new OrderSummary(order);
}
```

### 並列非同期操作

```csharp
// 良い例: 独立した操作の同時実行
public async Task<DashboardData> LoadDashboardAsync(CancellationToken cancellationToken)
{
    var ordersTask = _orderService.GetRecentAsync(cancellationToken);
    var metricsTask = _metricsService.GetCurrentAsync(cancellationToken);
    var alertsTask = _alertService.GetActiveAsync(cancellationToken);

    await Task.WhenAll(ordersTask, metricsTask, alertsTask);

    return new DashboardData(
        Orders: await ordersTask,
        Metrics: await metricsTask,
        Alerts: await alertsTask);
}
```

## オプションパターン

設定セクションを厳密に型付けされたオブジェクトにバインドする。

```csharp
public sealed class SmtpOptions
{
    public const string SectionName = "Smtp";

    public required string Host { get; init; }
    public required int Port { get; init; }
    public required string Username { get; init; }
    public bool UseSsl { get; init; } = true;
}

// 登録
builder.Services.Configure<SmtpOptions>(
    builder.Configuration.GetSection(SmtpOptions.SectionName));

// インジェクション経由での使用
public class EmailService(IOptions<SmtpOptions> options)
{
    private readonly SmtpOptions _smtp = options.Value;
}
```

## リザルトパターン

期待される失敗に対して例外をスローする代わりに、明示的な成功/失敗を返す。

```csharp
public sealed record Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(T value) { IsSuccess = true; Value = value; }
    private Result(string error) { IsSuccess = false; Error = error; }

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
}

// 使用例
public async Task<Result<Order>> PlaceOrderAsync(CreateOrderRequest request)
{
    if (request.Items.Count == 0)
        return Result<Order>.Failure("注文には少なくとも1つのアイテムが必要です");

    var order = Order.Create(request);
    await _repository.AddAsync(order, CancellationToken.None);
    return Result<Order>.Success(order);
}
```

## EF Coreを使用したリポジトリパターン

```csharp
public sealed class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public SqlOrderRepository(AppDbContext db) => _db = db;

    public async Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _db.Orders
            .Include(o => o.Items)
            .AsNoTracking()
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<IReadOnlyList<Order>> FindByCustomerAsync(
        string customerId,
        CancellationToken cancellationToken)
    {
        return await _db.Orders
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .AsNoTracking()
            .ToListAsync(cancellationToken);
    }

    public async Task AddAsync(Order order, CancellationToken cancellationToken)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(cancellationToken);
    }
}
```

## ミドルウェアとパイプライン

```csharp
// カスタムミドルウェア
public sealed class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation(
                "リクエスト {Method} {Path} が {ElapsedMs}ms で完了、ステータス {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}
```

## Minimal API パターン

```csharp
// ルートグループによる整理
var orders = app.MapGroup("/api/orders")
    .RequireAuthorization()
    .WithTags("Orders");

orders.MapGet("/{id:guid}", async (
    Guid id,
    IOrderRepository repository,
    CancellationToken cancellationToken) =>
{
    var order = await repository.FindByIdAsync(id, cancellationToken);
    return order is not null
        ? TypedResults.Ok(order)
        : TypedResults.NotFound();
});

orders.MapPost("/", async (
    CreateOrderRequest request,
    IOrderService service,
    CancellationToken cancellationToken) =>
{
    var result = await service.PlaceOrderAsync(request, cancellationToken);
    return result.IsSuccess
        ? TypedResults.Created($"/api/orders/{result.Value!.Id}", result.Value)
        : TypedResults.BadRequest(result.Error);
});
```

## ガード句

```csharp
// 良い例: 明確なバリデーションによる早期リターン
public async Task<ProcessResult> ProcessPaymentAsync(
    PaymentRequest request,
    CancellationToken cancellationToken)
{
    ArgumentNullException.ThrowIfNull(request);

    if (request.Amount <= 0)
        throw new ArgumentOutOfRangeException(nameof(request.Amount), "金額は正の値でなければなりません");

    if (string.IsNullOrWhiteSpace(request.Currency))
        throw new ArgumentException("通貨は必須です", nameof(request.Currency));

    // ハッピーパスはネストなしで続行
    var gateway = _gatewayFactory.Create(request.Currency);
    return await gateway.ChargeAsync(request, cancellationToken);
}
```

## 避けるべきアンチパターン

| アンチパターン | 修正方法 |
|---|---|
| `async void` メソッド | `Task` を返す（イベントハンドラを除く） |
| `.Result` や `.Wait()` | `await` を使用する |
| `catch (Exception) { }` | コンテキスト付きで処理または再スローする |
| コンストラクタ内の `new Service()` | コンストラクタインジェクションを使用する |
| `public` フィールド | 適切なアクセサを持つプロパティを使用する |
| ビジネスロジック内の `dynamic` | ジェネリクスまたは明示的な型を使用する |
| 可変な `static` ステート | DIスコーピングまたは `ConcurrentDictionary` を使用する |
| ループ内の `string.Format` | `StringBuilder` または補間文字列ハンドラを使用する |
