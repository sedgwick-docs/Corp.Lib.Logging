# Corp.Lib.Logging

A standardized, enterprise-grade logging library for .NET Standard 2.0 built on top of [Serilog](https://serilog.net/). This library provides rich structured logging with automatic sensitive data masking, correlation ID tracking, and multiple output sinks configured out of the box. Targeting .NET Standard 2.0 enables compatibility across .NET Core 2.0+, .NET 5+, and .NET Framework 4.6.1+.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [Logger Class](#logger-class)
  - [Extension Methods](#extension-methods)
  - [CorrelationIdMessageHandler](#correlationidmessagehandler)
  - [CorrelationId Class](#correlationid-class)
- [Configuration Reference](#configuration-reference)
- [Log Files](#log-files)
- [Log Entry Format](#log-entry-format)
- [Automatic Data Masking](#automatic-data-masking)
- [Correlation ID](#correlation-id)
- [Binary Data Handling](#binary-data-handling)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Dependencies](#dependencies)

## Features

- **Structured Logging** - JSON-formatted log entries with rich contextual information
- **Automatic Sensitive Data Masking** - Built-in masking for passwords, SSNs, credit cards, and IBANs
- **Correlation ID Tracking** - Track requests across services with automatic correlation ID propagation
- **Multiple Log Files** - Separate files for application logs, error logs, and detailed debug logs
- **Session Support** - Optional session-based correlation ID management for web applications
- **Swagger Integration** - Automatic correlation ID header in Swagger UI
- **Automatic Request Logging** - Optional HTTP request/response logging via Serilog middleware
- **Enriched Log Entries** - Automatic enrichment with machine name, environment username, Windows username, and exception details
- **Binary Data Protection** - Automatic transformation of binary data to prevent log bloat
- **Async File Writing** - Non-blocking log writes for better application performance
- **HTTP Client Integration** - Built-in `DelegatingHandler` for automatic correlation ID propagation in HTTP calls

## Installation

Install the NuGet package:

```bash
dotnet add package Corp.Lib.Logging
```

Or via the Package Manager Console:

```powershell
Install-Package Corp.Lib.Logging
```

## Quick Start

### 1. Add Configuration

Add the required configuration to your `appsettings.json`. The library requires `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment` properties, plus either a `LoggerSettings` section or an `ApplicationName` property:

#### Option A: Using LoggerSettings (Recommended)

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "LoggerSettings": {
    "Name": "MyApplication",
    "Level": "Information",
    "Path": "logs",
    "EnableAutomaticRequestLogging": true,
    "SessionIdleTime": 60
  }
}
```

#### Option B: Using ApplicationName (Minimal Configuration)

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "ApplicationName": "MyApplication"
}
```

> **Note:** The final application name used in log files and entries is constructed as: `{TargetedVoyagerInstance}.{TargetedVoyagerEnvironment}.{Name}` (e.g., `PROD.US.MyApplication`)

### 2. Configure Your Application

#### ASP.NET Core Web API

```csharp
using Corp.Lib.Logging.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();

// Add logging (false = no session support for APIs)
builder.Services.AddLogging(enableSession: false);

// Add Swagger with correlation ID header
builder.Services.AddSwaggerWithLogging();

var app = builder.Build();

// Enable logging middleware
app.UseLogging(useSession: false);

// Enable Swagger
app.UseSwaggerWithLogging();

app.MapControllers();
app.Run();
```

#### ASP.NET Core Web Application (Razor Pages / MVC)

```csharp
using Corp.Lib.Logging.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllersWithViews();

// Add logging with session support for correlation ID tracking
builder.Services.AddLogging(enableSession: true);

// Add Swagger with correlation ID header (optional)
builder.Services.AddSwaggerWithLogging();

var app = builder.Build();

app.UseStaticFiles();

// Enable logging middleware with session
app.UseLogging(useSession: true);

// Enable Swagger (optional)
app.UseSwaggerWithLogging();

app.UseRouting();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

#### .NET Worker Service / Console Application

```csharp
using Corp.Lib.Logging;

// Use the static Logger directly - no setup required
Logger.Log.Information("Application starting...");

// Don't forget to flush on shutdown
Logger.CloseAndFlush();
```

### 3. Write Log Messages

```csharp
using Corp.Lib.Logging;

public class MyService
{
    public void DoWork()
    {
        // Different log levels
        Logger.Log.Verbose("Detailed trace information");
        Logger.Log.Debug("Debug information");
        Logger.Log.Information("Processing started");
        Logger.Log.Warning("Something unusual happened");
        Logger.Log.Error("An error occurred");
        Logger.Log.Fatal("Critical failure");

        // Structured logging with properties
        Logger.Log.Information("Processing order {OrderId} for customer {CustomerId}", 
            orderId, customerId);

        // Logging objects (automatically serialized)
        var order = new Order { Id = 123, Total = 99.99m };
        Logger.Log.Information("Order details: {@Order}", order);

        // Logging exceptions
        try
        {
            // ... code that might throw
        }
        catch (Exception ex)
        {
            Logger.Log.Error(ex, "Failed to process order {OrderId}", orderId);
        }

        // Access the current correlation ID
        var correlationId = Logger.CorrelationId;
    }
}
```

---

## API Reference

### Logger Class

**Namespace:** `Corp.Lib.Logging`

The `Logger` class is the primary entry point for all logging operations. It provides a fully configured, static Serilog logger instance with enterprise-grade features including automatic data masking, correlation ID tracking, and multiple output sinks.

```csharp
using Corp.Lib.Logging;
```

#### Properties

##### `Logger.Log`

```csharp
public static Serilog.ILogger Log { get; }
```

Returns a fully configured Serilog `ILogger` instance with all features enabled. This is the primary property you'll use for all logging operations.

**Features automatically configured:**
- Minimum level overrides for Microsoft namespaces to reduce noise
- Binary data destructuring to prevent log bloat
- Sensitive property masking (passwords, SSNs, tokens)
- Enrichment with application name, correlation ID, machine name, usernames, and exception details
- Pattern-based masking for credit cards and IBANs
- Multiple file sinks (app, error, details) with async writing
- Console output in DEBUG builds

**Usage:**

```csharp
// Basic logging
Logger.Log.Information("Application started");

// Structured logging with named parameters
Logger.Log.Information("User {UserId} logged in from {IpAddress}", userId, ipAddress);

// Logging objects with destructuring
Logger.Log.Debug("Request payload: {@Request}", requestObject);

// Logging exceptions
try
{
    await ProcessAsync();
}
catch (Exception ex)
{
    Logger.Log.Error(ex, "Processing failed for item {ItemId}", itemId);
}
```

**Available Log Methods on `ILogger`:**

| Method | Description |
|--------|-------------|
| `Verbose(string message, params object[] args)` | Extremely detailed trace information |
| `Debug(string message, params object[] args)` | Internal system events for debugging |
| `Information(string message, params object[] args)` | Normal operational events |
| `Warning(string message, params object[] args)` | Unusual events that don't prevent operation |
| `Error(string message, params object[] args)` | Failures in the current operation |
| `Error(Exception ex, string message, params object[] args)` | Failures with exception details |
| `Fatal(string message, params object[] args)` | Critical failures requiring immediate attention |
| `Fatal(Exception ex, string message, params object[] args)` | Critical failures with exception details |

---

##### `Logger.CorrelationId`

```csharp
public static Guid CorrelationId { get; }
```

Gets the current correlation ID for the request context. This ID is automatically included in all log entries and can be used to trace requests across services.

**Correlation ID Resolution Order:**
1. HTTP header `X-Correlation-Id` (if present in the request)
2. Session-stored correlation ID (if session is enabled)
3. Configuration file `CorrelationId` setting (for worker services)
4. `Guid.Empty` if no correlation ID is available

**Usage:**

```csharp
// Get the current correlation ID
var correlationId = Logger.CorrelationId;

// Include in outbound HTTP requests
httpClient.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId.ToString());

// Log the correlation ID
Logger.Log.Information("Processing request with correlation {CorrelationId}", correlationId);
```

---

##### `Logger.MinimumLevel`

```csharp
public static Serilog.Events.LogEventLevel MinimumLevel { get; set; }
```

Gets or sets the minimum log level threshold. Events below this level are not written to log files.

**Values:** `Verbose`, `Debug`, `Information`, `Warning`, `Error`, `Fatal`

**Default:** `LogEventLevel.Information`

> **Note:** This property is automatically set from the `LoggerSettings.Level` configuration value when `Logger.Log` is first accessed.

---

##### `Logger.Name`

```csharp
public static string Name { get; set; }
```

Gets or sets the application name. This value is:
- Included in every log entry as the `Application` property
- Used as part of the log file names (e.g., `MyApp-app_20240115.log`)

> **Note:** This property is automatically set from the `LoggerSettings.Name` configuration value.

---

##### `Logger.FilePath`

```csharp
public static string FilePath { get; set; }
```

Gets or sets the directory path where log files are written.

**Default:** `"logs"` (relative to application directory)

> **Note:** This property is automatically set from the `LoggerSettings.Path` configuration value.

---

##### `Logger.EnableAutomaticRequestLogging`

```csharp
public static bool EnableAutomaticRequestLogging { get; set; }
```

Gets or sets whether automatic HTTP request/response logging is enabled via Serilog middleware.

**Default:** `false`

> **Note:** This property is automatically set from the `LoggerSettings.EnableAutomaticRequestLogging` configuration value.

---

#### Methods

##### `Logger.CloseAndFlush()`

```csharp
public static void CloseAndFlush()
```

Flushes all pending log entries to disk and closes the logger. **This method must be called on application shutdown** to ensure all log entries are written.

**Usage:**

```csharp
// For console applications
AppDomain.CurrentDomain.ProcessExit += (s, e) => Logger.CloseAndFlush();

// For worker services
public override async Task StopAsync(CancellationToken cancellationToken)
{
    Logger.Log.Information("Worker stopping...");
    Logger.CloseAndFlush();
    await base.StopAsync(cancellationToken);
}

// For web applications using UseLogging(), this is called automatically
```

> **Note:** When using `app.UseLogging()` in web applications, `CloseAndFlush()` is automatically registered to be called on application shutdown.

---

### Extension Methods

**Namespace:** `Corp.Lib.Logging.Extensions`

The `ServiceCollectionExtensions` class provides extension methods for `IServiceCollection` and `IApplicationBuilder` to simplify logging configuration in ASP.NET Core applications.

```csharp
using Corp.Lib.Logging.Extensions;
```

#### IServiceCollection Extensions

##### `AddLogging(this IServiceCollection services, bool enableSession)`

```csharp
public static IServiceCollection AddLogging(this IServiceCollection services, bool enableSession)
```

Configures logging services for the application. Call this method in your `Program.cs` during service configuration.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `services` | `IServiceCollection` | The service collection to add logging services to. |
| `enableSession` | `bool` | Set to `true` to enable session-based correlation ID tracking. Use `true` for web applications (MVC/Razor Pages) and `false` for APIs. |

**What it configures:**
- `IHttpContextAccessor` for HTTP context access
- Session services with secure cookie settings (when `enableSession` is `true`)
- Distributed memory cache for session storage (when `enableSession` is `true`)
- Serilog request logging integration (when `EnableAutomaticRequestLogging` is `true` in configuration)

**Usage:**

```csharp
var builder = WebApplication.CreateBuilder(args);

// For APIs - no session needed
builder.Services.AddLogging(enableSession: false);

// For web applications - session enables correlation ID persistence
builder.Services.AddLogging(enableSession: true);
```

**Session Configuration:**
When `enableSession` is `true`, the session is configured with:
- Cookie name: Application name from `LoggerSettings.Name`
- Secure policy: `Always` (HTTPS only)
- HTTP only: `true` (not accessible via JavaScript)
- Essential: `true` (required for functionality)
- Idle timeout: From `LoggerSettings.SessionIdleTime` (default 60 minutes)

---

##### `AddSwaggerWithLogging(this IServiceCollection services)`

```csharp
public static IServiceCollection AddSwaggerWithLogging(this IServiceCollection services)
```

Configures Swagger/OpenAPI with automatic correlation ID header support. Every API endpoint in Swagger UI will include a required `X-Correlation-Id` header field.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `services` | `IServiceCollection` | The service collection to add Swagger services to. |

**What it configures:**
- API explorer services
- Swagger generation with custom schema IDs
- Correlation ID header operation filter

**Usage:**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddLogging(enableSession: false);
builder.Services.AddSwaggerWithLogging();  // Add after AddLogging

var app = builder.Build();
```

---

#### IApplicationBuilder Extensions

##### `UseLogging(this IApplicationBuilder app, bool useSession)`

```csharp
public static IApplicationBuilder UseLogging(this IApplicationBuilder app, bool useSession)
```

Enables logging middleware for the application. Call this method in your `Program.cs` during middleware configuration.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `app` | `IApplicationBuilder` | The application builder to add logging middleware to. |
| `useSession` | `bool` | Set to `true` if session was enabled in `AddLogging()`. Must match the value used in `AddLogging()`. |

**What it enables:**
- Session middleware with correlation ID tracking (when `useSession` is `true`)
- Serilog request logging middleware (when `EnableAutomaticRequestLogging` is `true`)
- Automatic `Logger.CloseAndFlush()` registration on application shutdown

**Usage:**

```csharp
var app = builder.Build();

// For APIs
app.UseLogging(useSession: false);

// For web applications
app.UseLogging(useSession: true);

// Continue with other middleware
app.UseAuthorization();
app.MapControllers();
```

**Middleware Order:**
`UseLogging()` should be called early in the middleware pipeline, typically after `UseStaticFiles()` but before `UseRouting()` and `UseAuthorization()`.

---

##### `UseSwaggerWithLogging(this IApplicationBuilder app)`

```csharp
public static IApplicationBuilder UseSwaggerWithLogging(this IApplicationBuilder app)
```

Enables Swagger UI with correlation ID header support.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `app` | `IApplicationBuilder` | The application builder to enable Swagger middleware on. |

**Usage:**

```csharp
var app = builder.Build();

app.UseLogging(useSession: false);
app.UseSwaggerWithLogging();  // Enable Swagger UI

app.MapControllers();
app.Run();
```

---

### CorrelationIdMessageHandler

**Namespace:** `Corp.Lib.Logging.MessageHandlers`

```csharp
using Corp.Lib.Logging.MessageHandlers;
```

A `DelegatingHandler` that automatically adds the current correlation ID to outbound HTTP requests. Use this handler with `HttpClient` to propagate correlation IDs across service boundaries.

#### Class Definition

```csharp
public class CorrelationIdMessageHandler : DelegatingHandler
```

#### Description

When added to an `HttpClient`'s handler pipeline, this handler automatically adds the `X-Correlation-Id` header to every outbound HTTP request. The correlation ID value is retrieved from the current request context using the same resolution logic as `Logger.CorrelationId`.

#### Usage with HttpClientFactory (Recommended)

```csharp
// In Program.cs or Startup.cs
builder.Services.AddTransient<CorrelationIdMessageHandler>();

builder.Services.AddHttpClient("MyApiClient", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.AddHttpMessageHandler<CorrelationIdMessageHandler>();
```

```csharp
// In your service
public class MyService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public MyService(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    public async Task CallExternalApiAsync()
    {
        var client = _httpClientFactory.CreateClient("MyApiClient");
        
        // X-Correlation-Id header is automatically added
        var response = await client.GetAsync("/api/data");
    }
}
```

#### Usage with Named/Typed HttpClient

```csharp
// Register the handler
builder.Services.AddTransient<CorrelationIdMessageHandler>();

// Configure a typed client
builder.Services.AddHttpClient<IExternalApiClient, ExternalApiClient>()
    .AddHttpMessageHandler<CorrelationIdMessageHandler>();
```

#### Manual Usage (Not Recommended)

```csharp
var handler = new CorrelationIdMessageHandler
{
    InnerHandler = new HttpClientHandler()
};

using var client = new HttpClient(handler);
var response = await client.GetAsync("https://api.example.com/data");
```

#### HTTP Header Details

| Header Name | Value | Description |
|-------------|-------|-------------|
| `X-Correlation-Id` | `Guid` string | The current correlation ID from the request context |

---

### CorrelationId Class

**Namespace:** `Corp.Lib.Logging.Enrichers`

```csharp
using Corp.Lib.Logging.Enrichers;
```

A Serilog log event enricher that adds correlation ID information to log entries. This class can also be used directly to retrieve the current correlation ID.

#### Class Definition

```csharp
public class CorrelationId : Serilog.Core.ILogEventEnricher
```

#### Constructor

```csharp
public CorrelationId()
```

Creates a new instance of the `CorrelationId` enricher using the default `HttpContextAccessor`.

#### Methods

##### `GetCorrelationId()`

```csharp
public Guid GetCorrelationId()
```

Retrieves the current correlation ID from the available context.

**Resolution Order:**
1. **HTTP Header:** Checks for `X-Correlation-Id` header in the current HTTP request
2. **HTTP Context Items:** Checks for correlation ID stored in `HttpContext.Items` (set by session middleware)
3. **Configuration:** Falls back to `LoggerSettings.CorrelationId` from `appsettings.json` (useful for worker services)
4. **Default:** Returns `Guid.Empty` if no correlation ID is available

**Returns:** `Guid` - The current correlation ID or `Guid.Empty`

**Usage:**

```csharp
var correlationIdEnricher = new CorrelationId();
var correlationId = correlationIdEnricher.GetCorrelationId();

// Or use the static property (recommended)
var correlationId = Logger.CorrelationId;
```

##### `Enrich(LogEvent, ILogEventPropertyFactory)`

```csharp
public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
```

Enriches a log event with the correlation ID property. This method is called automatically by Serilog during log event processing.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `logEvent` | `LogEvent` | The log event to enrich |
| `propertyFactory` | `ILogEventPropertyFactory` | Factory for creating log event properties |

> **Note:** You typically don't need to call this method directly. It's automatically invoked by Serilog when the enricher is registered via `.Enrich.WithCorrelationId()`.

---

## Configuration Reference

### Required Root Properties

The following properties must be defined at the root level of your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US"
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `TargetedVoyagerInstance` | `string` | **Yes** | The Voyager instance identifier (e.g., `PROD`, `DEV`, `QA`). |
| `TargetedVoyagerEnvironment` | `string` | **Yes** | The Voyager environment identifier (e.g., `US`, `UK`, `CA`). |

### Application Name Configuration

The application name can be specified in one of two ways:

#### Option 1: LoggerSettings.Name (Recommended)

```json
{
  "LoggerSettings": {
    "Name": "MyApplication"
  }
}
```

#### Option 2: ApplicationName (Fallback)

If `LoggerSettings` is not present, the library falls back to the root-level `ApplicationName` property:

```json
{
  "ApplicationName": "MyApplication"
}
```

> **Note:** The final application name is constructed as: `{TargetedVoyagerInstance}.{TargetedVoyagerEnvironment}.{Name}` (e.g., `PROD.US.MyApplication`)

### LoggerSettings Properties

Add the `LoggerSettings` section to your `appsettings.json` for full control over logging behavior:

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "LoggerSettings": {
    "Name": "MyApplication",
    "Level": "Information",
    "Path": "logs",
    "CorrelationId": null,
    "EnableAutomaticRequestLogging": false,
    "SessionIdleTime": 60
  }
}
```

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `Name` | `string` | **Yes** | - | Application name used in log file names and log entries. Must not be empty. |
| `Level` | `string` | No | `"Information"` | Minimum log level: `Verbose`, `Debug`, `Information`, `Warning`, `Error`, `Fatal` |
| `Path` | `string` | No | `"logs"` | Directory path where log files are written. Can be relative or absolute. |
| `CorrelationId` | `Guid?` | No | `null` | Optional fixed correlation ID. Useful for batch jobs and worker services. |
| `EnableAutomaticRequestLogging` | `bool` | No | `false` | Enable automatic HTTP request/response logging via Serilog middleware. |
| `SessionIdleTime` | `int` | No | `60` | Session timeout in minutes when session is enabled. |

### Example Configurations

#### Development Environment

```json
{
  "TargetedVoyagerInstance": "DEV",
  "TargetedVoyagerEnvironment": "US",
  "LoggerSettings": {
    "Name": "MyApp",
    "Level": "Verbose",
    "Path": "C:\\Logs\\MyApp",
    "EnableAutomaticRequestLogging": true
  }
}
```

#### Production Environment

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "LoggerSettings": {
    "Name": "MyApp",
    "Level": "Information",
    "Path": "D:\\Logs\\MyApp",
    "EnableAutomaticRequestLogging": false
  }
}
```

#### Worker Service with Fixed Correlation ID

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "LoggerSettings": {
    "Name": "MyWorkerService",
    "Level": "Information",
    "Path": "logs",
    "CorrelationId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### Minimal Configuration (Using ApplicationName Fallback)

```json
{
  "TargetedVoyagerInstance": "PROD",
  "TargetedVoyagerEnvironment": "US",
  "ApplicationName": "MySimpleApp"
}
```

---

## Log Files

The library creates up to three log files based on the configured minimum level:

| File Pattern | Contents | Rolling | Retention |
|--------------|----------|---------|-----------|
| `{Name}-app_{date}.log` | Information and Warning level entries | Daily | Unlimited |
| `{Name}-error_{date}.log` | Error and Fatal level entries | Daily | Unlimited |
| `{Name}-details_{date}_{hour}.log` | All entries (Verbose+) | Hourly | 24 files |

### File Creation Rules

- **App Log**: Created when level is `Verbose`, `Debug`, `Information`, or `Warning`
- **Error Log**: Always created
- **Details Log**: Only created when level is `Verbose` or `Debug`

### File Characteristics

All log files:
- Roll over at 20MB file size
- Use shared file access (multiple processes can write)
- Are written asynchronously for performance
- Use JSON format with one entry per line

---

## Log Entry Format

Each log entry is a JSON object with the following structure:

```json
{
  "Timestamp": "2024-01-15T10:30:45.123Z",
  "Message": "Processing order 12345",
  "Level": "Information",
  "Application": "MyApp",
  "CorrelationId": "550e8400-e29b-41d4-a716-446655440000",
  "EnvironmentUserName": "DOMAIN\\username",
  "LogEntryId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "MachineName": "SERVER01",
  "WindowsUsername": "DOMAIN\\username",
  "Properties": {
    "OrderId": 12345
  }
}
```

### Log Entry Properties

| Property | Description |
|----------|-------------|
| `Timestamp` | UTC timestamp of the log entry |
| `Message` | The formatted log message |
| `Level` | Log level (Verbose, Debug, Information, Warning, Error, Fatal) |
| `Application` | Application name from configuration |
| `CorrelationId` | Request correlation ID for distributed tracing |
| `EnvironmentUserName` | The environment's current username |
| `Exception` | Exception details (only present when logging exceptions) |
| `LogEntryId` | Unique identifier for each log entry |
| `MachineName` | Server/machine name |
| `WindowsUsername` | Windows identity username |
| `Properties` | Additional structured properties from the log message |

---

## Automatic Data Masking

### Property Name Masking

The following property names are automatically masked when logging objects:

| Property Names | Masked As |
|----------------|-----------|
| `Password`, `Pwd` | `*********` |
| `SocialSecurityNumber`, `SocialSecurityNum`, `SocialSecurityNo` | `*********` |
| `SocSecNum`, `SSNumber`, `SSNum`, `SSNo`, `SSN` | `*********` |
| `Token` | `*********` |

**Example:**

```csharp
var user = new { Username = "john", Password = "secret123", Token = "abc123" };
Logger.Log.Information("User: {@User}", user);
// Output: User: { Username: "john", Password: "*********", Token: "*********" }
```

### Pattern-Based Masking

Credit card numbers and IBANs are automatically detected and masked in log messages:

| Pattern | Masking Behavior |
|---------|------------------|
| Credit Cards | First 6 digits preserved, remaining masked (e.g., `411111**********`) |
| IBANs | Fully masked with `**********` |

**Example:**

```csharp
Logger.Log.Information("Payment card: {CardNumber}", "4111-1111-1111-1111");
// Output: Payment card: 411111**********
```

---

## Correlation ID

The correlation ID is a GUID that tracks a request across multiple services and log entries.

### How It Works

1. **Web Applications with Session**: The correlation ID is generated and stored in the session, then automatically propagated to all log entries
2. **APIs**: The correlation ID is read from the `X-Correlation-Id` HTTP header
3. **Worker Services**: Use the optional `CorrelationId` configuration setting for a fixed correlation ID

### HTTP Header Name

```
X-Correlation-Id
```

### Propagating Correlation IDs

#### Using CorrelationIdMessageHandler (Recommended)

```csharp
// Register in DI
builder.Services.AddTransient<CorrelationIdMessageHandler>();
builder.Services.AddHttpClient("ExternalApi")
    .AddHttpMessageHandler<CorrelationIdMessageHandler>();

// Usage - correlation ID is automatically added
var client = httpClientFactory.CreateClient("ExternalApi");
await client.GetAsync("https://api.example.com/data");
```

#### Manual Propagation

```csharp
var correlationId = Logger.CorrelationId;
httpClient.DefaultRequestHeaders.Add("X-Correlation-Id", correlationId.ToString());
```

### Swagger Integration

When using `AddSwaggerWithLogging()`, Swagger UI automatically includes a required `X-Correlation-Id` input field for all API endpoints with:
- Type: String (UUID format)
- Location: Header
- Required: Yes
- Default: `00000000-0000-0000-0000-000000000000`

---

## Binary Data Handling

Large binary data is automatically transformed to prevent log bloat:

| Type | Transformation |
|------|----------------|
| `byte[]` | `"{length} bytes"` |
| `sbyte[]` | `"{length} bytes"` |
| `Stream` | `"{length} bytes"` |
| `BinaryReader` | `"{length} bytes"` |
| `BinaryWriter` | `"{length} bytes"` |
| `IFormFile` | `"{length} bytes"` |
| `IFormFileCollection` | `"{count} files"` |

**Example:**

```csharp
var fileBytes = new byte[1024000];
Logger.Log.Information("File data: {@Data}", fileBytes);
// Output: File data: "1024000 bytes"
```

---

## Best Practices

### 1. Always Call CloseAndFlush on Shutdown

```csharp
// For web applications using UseLogging() - handled automatically!

// For console applications
AppDomain.CurrentDomain.ProcessExit += (s, e) => Logger.CloseAndFlush();

// For worker services
public override async Task StopAsync(CancellationToken cancellationToken)
{
    Logger.Log.Information("Worker stopping...");
    Logger.CloseAndFlush();
    await base.StopAsync(cancellationToken);
}
```

### 2. Use Structured Logging

```csharp
// ? Good - structured logging preserves property types
Logger.Log.Information("Order {OrderId} processed for {CustomerId}", orderId, customerId);

// ? Avoid - string interpolation loses structure
Logger.Log.Information($"Order {orderId} processed for {customerId}");
```

### 3. Use Appropriate Log Levels

| Level | Use For |
|-------|---------|
| **Verbose** | Extremely detailed traces, only for debugging specific issues |
| **Debug** | Internal system events, useful during development |
| **Information** | Normal operational events (request processed, job completed) |
| **Warning** | Unusual events that don't prevent operation |
| **Error** | Failures in the current operation (not application-wide) |
| **Fatal** | Critical failures that require immediate attention |

### 4. Include Context in Error Logs

```csharp
try
{
    await ProcessOrderAsync(order);
}
catch (Exception ex)
{
    Logger.Log.Error(ex, "Failed to process order {OrderId} for customer {CustomerId}", 
        order.Id, order.CustomerId);
    throw;
}
```

### 5. Log Objects with @ Destructuring

```csharp
// ? Serializes the entire object with all properties
Logger.Log.Information("Processing {@Order}", order);

// ?? Only logs ToString() result
Logger.Log.Information("Processing {Order}", order);
```

### 6. Use CorrelationIdMessageHandler for HTTP Clients

```csharp
// ? Automatic correlation ID propagation
builder.Services.AddHttpClient("MyApi")
    .AddHttpMessageHandler<CorrelationIdMessageHandler>();

// ? Manual propagation is error-prone
client.DefaultRequestHeaders.Add("X-Correlation-Id", Logger.CorrelationId.ToString());
```

---

## Troubleshooting

### Logs Not Being Written

1. Verify `LoggerSettings` exists in `appsettings.json` (or `ApplicationName` as a fallback)
2. Check that the `Name` property is set (required)
3. Ensure the `Path` directory exists and is writable
4. Verify the application has file system permissions
5. Check that `Logger.CloseAndFlush()` is being called on shutdown

### Missing Correlation IDs

1. For web apps: Ensure `builder.Services.AddLogging(true)` and `app.UseLogging(true)` are called
2. For APIs: Ensure the `X-Correlation-Id` header is being sent by clients
3. For worker services: Set `CorrelationId` in `LoggerSettings`
4. Check that `UseLogging()` is called before other middleware that might need correlation IDs

### Console Output Not Showing

Console output is only enabled in DEBUG builds (`#if DEBUG`). For release builds, check the log files in the configured path.

### HttpClient Not Sending Correlation ID

1. Ensure `CorrelationIdMessageHandler` is registered: `builder.Services.AddTransient<CorrelationIdMessageHandler>()`
2. Ensure the handler is added to the client: `.AddHttpMessageHandler<CorrelationIdMessageHandler>()`
3. Use `IHttpClientFactory` to create clients, not `new HttpClient()`

### Configuration Not Loading

1. Ensure `appsettings.json` is in the application's base directory
2. Check that the file is valid JSON
3. Verify required root properties exist: `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment`
4. Verify either `LoggerSettings.Name` or `ApplicationName` is configured
5. Check for case sensitivity issues (properties are case-insensitive)

### "TargetedVoyagerInstance" or "TargetedVoyagerEnvironment" Error

If you see a `ConfigurationErrorsException` mentioning these properties:
1. Add both `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment` to the root of your `appsettings.json`
2. Ensure both properties have non-empty string values
3. Example:
   ```json
   {
     "TargetedVoyagerInstance": "PROD",
     "TargetedVoyagerEnvironment": "US"
   }
   ```

---

## Dependencies

This package depends on the following NuGet packages:

| Package | Version | Purpose |
|---------|---------|---------|
| Masking.Serilog | 1.0.13+ | Property name-based masking |
| Microsoft.AspNetCore.Http | 2.2.0+ | HTTP context and accessor types |
| Microsoft.AspNetCore.Session | 2.2.0+ | Session middleware support |
| Microsoft.Extensions.Caching.Memory | 8.0.1+ | Distributed memory cache |
| Microsoft.Extensions.Hosting.Abstractions | 8.0.1+ | Application lifetime management |
| Serilog | 4.3.0+ | Core structured logging |
| Serilog.AspNetCore | 7.0.0+ | ASP.NET Core integration |
| Serilog.Enrichers.Environment | 3.0.1+ | Machine name and username enrichment |
| Serilog.Enrichers.Sensitive | 2.1.0+ | Pattern-based sensitive data masking |
| Serilog.Exceptions | 8.4.0+ | Detailed exception logging |
| Serilog.Expressions | 5.0.0+ | Log output formatting templates |
| Serilog.Formatting.Compact | 3.0.0+ | Compact JSON formatting |
| Serilog.Sinks.Async | 2.1.0+ | Asynchronous log writing |
| Serilog.Sinks.Console | 6.1.1+ | Console output (DEBUG builds) |
| Serilog.Sinks.File | 7.0.0+ | File output with rolling |
| Swashbuckle.AspNetCore | 6.9.0+ | Swagger/OpenAPI integration |
| System.Configuration.ConfigurationManager | 8.0.1+ | Configuration file support |
| System.Text.Json | 8.0.5+ | JSON serialization |

---

## Version History

- **10.0.8** - Retargeted to .NET Standard 2.0 for broad compatibility across .NET Core 2.0+, .NET 5+, and .NET Framework 4.6.1+. Extension methods now use `IServiceCollection` and `IApplicationBuilder` instead of `WebApplicationBuilder` and `WebApplication`.
- **10.0.1** - Initial release targeting .NET 10

---

## License

Copyright © 2026 Sedgwick. All rights reserved.

---

## Support

For issues, questions, or feature requests, please contact the development team or submit an issue through Azure DevOps.
