# IPFly .NET SDK

A dependency-free C# client for the [IPFly](https://ipfly.world) IP geolocation API. One file — it handles caching, retries, rate limiting, request de-duplication, and batch lookups for you.

- **Zero third-party dependencies** — only `System.Net.Http` and `System.Text.Json` from the BCL
- **Fully async** — `Task`-based throughout, with `CancellationToken` support on every call
- **TTL + LRU caching** — in-memory by default, or a shared JSON file for persistence across restarts
- **Resilient** — automatic retry with exponential backoff + jitter on network/5xx failures
- **In-flight de-duplication** — two callers awaiting the same IP at the same moment share one HTTP request (via `ConcurrentDictionary<string, Lazy<Task<T>>>`)
- **Efficient batching** — `BatchLookupAsync` with bounded concurrency via `SemaphoreSlim`
- **Typed errors** — a single `IPFlyException` with `.Status` and `.Code`
- **DI-friendly** — accepts an externally managed `HttpClient` (use `IHttpClientFactory` in ASP.NET Core)
- Targets **.NET 6+** (works on .NET 6, 7, 8, 9)

---

## Where this SDK belongs

This client is meant to run **server-side** — an ASP.NET Core API, a worker service, a console app, an Azure Function. It is not meant to be shipped inside a client-side app (Blazor WebAssembly, MAUI, WPF) where your API token could be extracted from the compiled app. If you need geolocation results in a client-side app, put this SDK behind your own backend endpoint and call that endpoint from the client (see the [minimal API proxy example](#examples) below), or use the IPFly JavaScript SDK with a domain-restricted token for prototyping.

---

## Installation

No NuGet package is published — copy `IPFlyClient.cs` into your project.

```bash
# from your project root
curl -O https://ipfly.world/libs/dotnet-sdk/IPFlyClient.cs
```

```csharp
using IPFly;

var client = new IPFlyClient("YOUR_TOKEN");
```

### Requirements

- .NET 6.0 or later (uses nullable reference types, `Environment.ProcessId`, pattern-matching `or` combinators)
- No NuGet packages required

---

## Quick start

```csharp
using System;
using IPFly;

var client = new IPFlyClient("YOUR_TOKEN", new IPFlyClientOptions
{
    Include = "security", // optional: request the security/ASN field set on every call
});

// Look up a specific IP
var data = await client.LookupAsync("8.8.8.8");
Console.WriteLine($"{data.GetProperty("city").GetString()}, " +
                   $"{data.GetProperty("country_name").GetString()}, " +
                   $"VPN: {data.GetProperty("security").GetProperty("is_vpn").GetBoolean()}");

// Look up the caller's own IP (omit the ip argument)
var me = await client.LookupSelfAsync();
Console.WriteLine($"You appear to be in {me.GetProperty("city").GetString()}");

// Handle errors explicitly
try
{
    await client.LookupAsync("not-an-ip");
}
catch (IPFlyException err)
{
    Console.WriteLine($"{err.Code}: {err.Message}");
}

client.Dispose(); // only needed if you didn't supply your own HttpClient
```

Lookups return a [`System.Text.Json.JsonElement`](https://learn.microsoft.com/dotnet/api/system.text.json.jsonelement) — the standard BCL type for dynamic JSON, so there's no extra model type to learn: `data.GetProperty("city").GetString()`, `data.GetProperty("security").GetProperty("is_vpn").GetBoolean()`, and so on.

---

## Configuration reference

Passed as the second constructor argument: `new IPFlyClient(token, new IPFlyClientOptions { ... })`.

| Property | Type | Default | Description |
|---|---|---|---|
| `BaseUrl` | `string` | `https://ipfly.world/api` | Override to point at your own backend proxy. |
| `Include` | `string?` | `null` | Default `include` value sent with every request (e.g. `"security"`). Overridable per call. |
| `Timeout` | `TimeSpan` | `8s` | Per-request timeout. |
| `Retries` | `int` | `2` | Retry attempts for network errors and `5xx` responses. `4xx` is never retried. |
| `RetryDelay` | `TimeSpan` | `300ms` | Base delay for exponential backoff; jitter is added automatically. |
| `RateLimit` | `double` | `0` (unlimited) | Max requests/second this client will issue. |
| `Concurrency` | `int` | `6` | Default max parallel requests for `BatchLookupAsync`. |
| `Cache` | `CacheOptions` | see below | Response cache settings. |
| `HttpClient` | `HttpClient?` | `null` | Supply your own (e.g. from `IHttpClientFactory`) — **recommended** in ASP.NET Core. If omitted, the SDK creates and owns one, disposed when the client is disposed. |
| `Debug` | `bool` | `false` | Write internal lifecycle events to `Console.Error` (tokens are redacted). |
| `OnRequest` | `Action<string?, string>?` | `null` | Fired right before a request is sent: `(ip, url)`. |
| `OnResponse` | `Action<string?, JsonElement, bool>?` | `null` | Fired after a successful lookup: `(ip, data, fromCache)`. |
| `OnError` | `Action<string?, IPFlyException>?` | `null` | Fired when a lookup ultimately fails: `(ip, error)`. |
| `OnCacheHit` | `Action<string?, JsonElement>?` | `null` | Fired when a cached response is served instead of a network call: `(ip, data)`. |

### `CacheOptions`

| Property | Type | Default | Description |
|---|---|---|---|
| `Enabled` | `bool` | `true` | Set `false` to disable caching entirely. |
| `Ttl` | `TimeSpan` | `5 minutes` | How long a cached response stays valid. |
| `MaxSize` | `int` | `500` | Max cached entries before least-recently-used entries are evicted. |
| `Storage` | `CacheStorage` | `Memory` | `Memory` or `File` — see below. |
| `Path` | `string` | temp dir | File path used when `Storage` is `File`. |

**A .NET-specific note:** unlike PHP (where each web request is usually a fresh process, so an in-memory cache is useless across requests without something like APCu), a typical ASP.NET Core app runs as one long-lived process. That means the default `Memory` storage **already behaves like a shared cache across requests** — you generally don't need anything extra for a single-instance deployment. For multi-instance deployments behind a load balancer, implement `ICacheStore` yourself over a distributed cache (Redis, `IDistributedCache`, etc.) and pass it in — see [Extending the cache](#extending-the-cache).

```csharp
var client = new IPFlyClient("YOUR_TOKEN", new IPFlyClientOptions
{
    Cache = new CacheOptions { Storage = CacheStorage.File, Ttl = TimeSpan.FromHours(1) },
});
```

---

## API

### `LookupAsync(string? ip = null, LookupOptions? options = null, CancellationToken ct = default)`
Look up a single IP. Pass `null` (or omit) to geolocate the caller.

```csharp
await client.LookupAsync("1.1.1.1", new LookupOptions
{
    Include = "security", // overrides the client-level Include for this call only
    SkipCache = true,      // force a fresh network request, bypassing the cache
    Timeout = TimeSpan.FromSeconds(3), // override the client's default timeout for this call
});
```
Returns a `Task<JsonElement>`. Throws `IPFlyException` on failure.

### `LookupSelfAsync(LookupOptions? options = null, CancellationToken ct = default)`
Shorthand for `LookupAsync(null, options, ct)` — geolocates the caller's own IP.

### `BatchLookupAsync(IEnumerable<string> ips, int? concurrency = null, LookupOptions? options = null, CancellationToken ct = default)`
Looks up many IPs with bounded concurrency (`SemaphoreSlim`, sized by `concurrency` or the client default). This call **never throws** for an individual failure — it returns `IReadOnlyList<BatchResult>` in the same order as `ips`:

```csharp
var results = await client.BatchLookupAsync(new[] { "8.8.8.8", "1.1.1.1", "not-an-ip" });
foreach (var r in results)
{
    if (r.Ok)
        Console.WriteLine($"{r.Ip} -> {r.Data!.Value.GetProperty("city").GetString()}");
    else
        Console.WriteLine($"{r.Ip} failed: {r.Error!.Code}");
}
```

`BatchResult` is a small class: `Ip` (`string`), `Ok` (`bool`), `Data` (`JsonElement?`), `Error` (`IPFlyException?`).

### `ClearCache()`
Empties the response cache immediately.

### `CacheStats()`
Returns a `(bool Enabled, int Size, double TtlSeconds, int MaxSize)` tuple — useful for logging or a health-check endpoint.

### `IPFlyClient.IsValidIp(string ip)`
Static helper — validates an IPv4 or IPv6 string (via `IPAddress.TryParse`) without making a request.

```csharp
IPFlyClient.IsValidIp("8.8.8.8");               // true
IPFlyClient.IsValidIp("2001:4860:4860::8888");  // true
IPFlyClient.IsValidIp("not-an-ip");             // false
```

### `Dispose()`
Disposes the internally created `HttpClient` — a no-op if you supplied your own via `IPFlyClientOptions.HttpClient` (you own its lifetime in that case).

---

## Response shape

Successful lookups return a `JsonElement` with the same structure the IPFly API returns, unchanged — see the [IPFly API docs](https://ipfly.world) for the full field reference. Shape depends on your plan and the `Include` value requested (`security` and ASN/company fields require Pro or higher).

```csharp
var data = await client.LookupAsync("8.8.8.8");

string? city = data.GetProperty("city").GetString();
string? country = data.GetProperty("country_name").GetString();
bool isVpn = data.GetProperty("security").GetProperty("is_vpn").GetBoolean();
string? asnName = data.GetProperty("asn").GetProperty("name").GetString();
```

> `JsonElement.GetProperty` throws if the property is missing. Use `TryGetProperty` for optional fields (e.g. `state_prov`, `zipcode`), which are plan- and location-dependent.

---

## Error handling

All failures throw `IPFlyException`, an `Exception` subclass with:

| Member | Description |
|---|---|
| `Message` | Human-readable description. |
| `Status` | `int?` — HTTP status code, if the request reached the server (`null` for network/timeout errors). |
| `Code` | `string` — machine-readable code, see table below. |
| `Cause` | `object?` — the underlying exception or decoded response body, when available. |

| `Code` | Meaning |
|---|---|
| `MISSING_TOKEN` | No token was supplied when creating the client. |
| `INVALID_IP` | The IP string failed local validation before any request was sent. |
| `TIMEOUT` | The request exceeded the configured timeout. |
| `NETWORK_ERROR` | The request failed before a response was received (DNS, connection refused, etc). |
| `HTTP_ERROR` / server-provided code | The API responded with a non-2xx status. Check `Status` (401 = bad token, 403 = plan/permission issue, 404 = private/bogon IP, 429/5xx = retry-worthy). |
| `BAD_RESPONSE` | The server responded but the body wasn't valid JSON. |

```csharp
try
{
    await client.LookupAsync("8.8.8.8");
}
catch (IPFlyException err)
{
    if (err.Code == "TIMEOUT")
    {
        // retry later, log, fall back to a default
    }
    else if (err.Status == 401)
    {
        // token is invalid — surface a config error, don't retry
    }
    logger.LogWarning(err, "IPFly lookup failed: {Message}", err.Message);
}
```

---

## Performance notes

- **Use `IHttpClientFactory`.** Pass your own `HttpClient` via `IPFlyClientOptions.HttpClient` in ASP.NET Core (`services.AddHttpClient()` + inject a client) rather than letting the SDK create its own — this is the standard .NET guidance to avoid socket exhaustion from creating many short-lived `HttpClient` instances.
- **Caching** avoids re-querying the same IP within the TTL window. Because a .NET web app is normally one long-lived process, the default in-memory cache already behaves like a shared cache across requests — no extra backend needed for a single instance.
- **De-duplication**: if two requests call `LookupAsync("1.2.3.4")` concurrently, only one HTTP request goes out; both `await` the same underlying `Task`.
- **Rate limiting** is a courtesy limiter on the client side — it smooths bursts so you don't blow through your plan's requests/second in a tight loop.
- **Retries** only apply to transient failures (network errors, timeouts, `5xx`). A `401`/`403`/`404` fails fast since retrying won't fix a bad token or a private IP.

### Extending the cache

For a multi-instance deployment behind a load balancer, implement `ICacheStore` over whatever distributed cache you already use:

```csharp
public sealed class RedisCacheStore : IPFly.ICacheStore
{
    // Wrap StackExchange.Redis (or IDistributedCache) here, serializing
    // CacheEntry with System.Text.Json the same way FileCacheStore does.
    public IPFly.CacheEntry? Get(string key) => throw new NotImplementedException();
    public void Set(string key, IPFly.CacheEntry entry) => throw new NotImplementedException();
    public void Delete(string key) => throw new NotImplementedException();
    public IReadOnlyList<string> Keys() => throw new NotImplementedException();
    public int Size() => throw new NotImplementedException();
    public void Clear() => throw new NotImplementedException();
}
```
`CacheManager` is internal, so wire a custom store in by adapting the `ICacheStore` interface in your own composition — or simply fork the file if you want `CacheManager` to accept an `ICacheStore` directly (it's a small change to `IPFlyClientOptions`/`IPFlyClient`'s constructor if you need it).

---

## Examples

### ASP.NET Core minimal API proxy (keep the token server-side)

```csharp
// Program.cs
using IPFly;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHttpClient();
builder.Services.AddSingleton(sp => new IPFlyClient(
    builder.Configuration["IPFly:Token"]!,
    new IPFlyClientOptions
    {
        HttpClient = sp.GetRequiredService<IHttpClientFactory>().CreateClient("ipfly"),
        Cache = new CacheOptions { Ttl = TimeSpan.FromMinutes(10) },
    }));

var app = builder.Build();

app.MapGet("/api/geo", async (HttpContext ctx, IPFlyClient client) =>
{
    var ip = ctx.Request.Query["ip"].FirstOrDefault() ?? ctx.Connection.RemoteIpAddress?.ToString();
    try
    {
        var data = await client.LookupAsync(ip, new LookupOptions { Include = "security" });
        return Results.Json(data);
    }
    catch (IPFlyException err)
    {
        return Results.Json(new { error = err.Code, message = err.Message }, statusCode: err.Status ?? 502);
    }
});

app.Run();
```

Your frontend (a SPA, a mobile app, Blazor WebAssembly, etc.) calls `/api/geo` on your own server — the IPFly token stays in this process and is never sent to the client.

### Bulk-enriching a list of IPs

```csharp
using IPFly;

using var client = new IPFlyClient(Environment.GetEnvironmentVariable("IPFLY_TOKEN")!);

var ips = await File.ReadAllLinesAsync("visitors.txt");
var results = await client.BatchLookupAsync(ips, concurrency: 8);

foreach (var r in results)
{
    Console.WriteLine(r.Ok
        ? $"{r.Ip} -> {r.Data!.Value.GetProperty("country_name").GetString()}"
        : $"{r.Ip} -> error: {r.Error!.Code}");
}
```

> `IPFlyClient` implements `IDisposable` (not `IAsyncDisposable`), so a plain `using` is what disposes it — only needed when you let the SDK create its own internal `HttpClient` rather than supplying one yourself.

---

## License

MIT — use it, fork it, ship it.
