# Register Services (`RegisterServices`)

`RegisterServices` is the method plugins use to **register services with the dependency injection (DI) container**.

Its primary purpose is to configure **dependencies**. In modern software development, a DI container manages object creation and lifetime. Through this method, you can:

1.  **Request Services (Consumer)**: Configure external tools your plugin needs, such as HTTP clients or database connections.
2.  **Provide Services (Provider)**: Register your business logic classes so they can be consumed by other parts of your plugin—or even by other plugins.

The DI container automatically injects required services into consumers and manages their lifecycle for you.

## Hosting Model

SharwAPI uses the [Dependency Injection hosting model](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) to manage objects. This means you don't need to manually write `new ServiceClass()` in your code—the host application automatically manages object creation and injects dependencies based on your registration.

This approach offers two key benefits:
1. **Lifecycle Management**: The system automatically handles object disposal, helping prevent memory leaks.
2. **Instance Sharing**: You can easily share data across plugins or requests by controlling service lifetime.

## Service Lifetimes

When registering services, choose the appropriate [service lifetime](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/service-lifetimes) based on your business needs. This determines how long an instance lives in memory.

### 1. Transient
* **Behavior**: A new instance is created every time the service is requested.
* **Use Case**: Lightweight, stateless utility classes (e.g., simple calculators, data formatters).
* **Example**: `services.AddTransient<MyService>();`

### 2. Scoped
* **Behavior**: Within a single HTTP request, multiple resolutions of the service return the same instance. The instance is disposed when the request completes.
* **Use Case**: Database contexts (`DbContext`) or business services that need to maintain state within a request. This is the most common pattern in web development.
* **Example**: `services.AddScoped<MyService>();`

### 3. Singleton
* **Behavior**: A single instance is created for the entire application lifetime. All requests and plugins share this instance.
* **Use Case**: Caching services, global configuration readers, or background scheduled tasks.
* **Example**: `services.AddSingleton<MyService>();`

## Common Operations

### Requesting External Resources (Consumer Pattern)

When your plugin needs to access external networks, register the corresponding client (e.g., `HttpClient`).

```csharp
// Register a client named "google" and preset the base address
services.AddHttpClient("google", client =>
{
    client.BaseAddress = new Uri("https://google.com  ");
    client.DefaultRequestHeaders.Add("User-Agent", "SharwAPI-Plugin");
});

```

### Providing Public Services (I am a Provider)

If you write a powerful class (e.g., a database operation service) and want it to be used by the **current plugin** or even **other plugins**, you need to register it in the container.

```csharp
// Register as a singleton service: The entire system shares the same MyDatabaseService instance
services.AddSingleton<IDatabaseService, MyDatabaseService>();

```

Once registered, any plugin (including your own) simply needs to declare an `IDatabaseService` parameter in its constructor or route handler method, and the main program will automatically inject the instance you registered.

### Reading Configuration

The second parameter of the method, `IConfiguration`, is used to access configuration data. It contains the configuration file with the same name under the plugin directory (`config/your-plugin-name.json`).

```csharp
// Method 1: Read a specific field directly
var apiKey = configuration["ApiKey"];

// Method 2: Bind a configuration section to an object (Recommended)
services.Configure<MyOptions>(configuration);
```

For more details, see [Plugin Configuration](/en/plugin/plugin-config)


::: danger Do Not Build Manually
Please **NEVER** call `services.BuildServiceProvider()` inside this method.

```csharp
// Incorrect Example
var provider = services.BuildServiceProvider(); 

```

This operation forces the creation of a **container copy** independent of the main program, leading to serious consequences:

1. **Singleton Failure**: The main program and your plugin will hold separate singleton objects, making data communication impossible.
2. **Dependency Loss**: Your plugin will not be able to obtain other services registered afterward.
3. **Memory Risk**: This container copy might not be disposed of correctly, leading to memory leaks.
:::