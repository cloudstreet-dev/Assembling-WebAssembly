# Chapter 24: .NET and Blazor

## .NET WebAssembly

Microsoft's .NET runtime can compile C# applications to WebAssembly via **Blazor WebAssembly**. This enables running full .NET applications in the browser.

**Key features**:
- Full C# and .NET support
- Rich ecosystem (.NET libraries)
- Razor component model (like React/Vue)
- Strong typing and tooling
- SignalR for real-time communication

**Tradeoffs**:
- Large download size (~2-5MB initial)
- Startup time
- Currently web-focused (server/WASI support limited)

## Blazor WebAssembly

Blazor is .NET's web framework with two hosting models:
- **Blazor Server**: Runs on server, UI updates via SignalR
- **Blazor WebAssembly**: Runs entirely in browser via WASM

We'll focus on Blazor WebAssembly.

## Setup

### Prerequisites

```bash
# Install .NET SDK (8.0 or later)
# Download from https://dotnet.microsoft.com/download

dotnet --version
```

### Create Project

```bash
dotnet new blazorwasm -o MyBlazorApp
cd MyBlazorApp

# Run
dotnet run
```

Visit https://localhost:5001

## Project Structure

```
MyBlazorApp/
├── wwwroot/
│   ├── index.html        # Entry point
│   └── css/
├── Pages/
│   ├── Index.razor       # Home page component
│   ├── Counter.razor     # Counter example
│   └── FetchData.razor   # Data fetching example
├── Shared/
│   ├── MainLayout.razor  # Layout component
│   └── NavMenu.razor     # Navigation
├── Program.cs            # Entry point
└── App.razor             # Root component
```

## Razor Components

### Basic Component

**Pages/Greet.razor**:
```razor
@page "/greet"

<h3>Greeting</h3>

<input @bind="name" placeholder="Enter your name" />
<button @onclick="SayHello">Greet</button>

<p>@message</p>

@code {
    private string name = "";
    private string message = "";

    private void SayHello()
    {
        message = $"Hello, {name}!";
    }
}
```

### Component with Parameters

**Components/DisplayName.razor**:
```razor
<div class="name-display">
    <h4>@Title</h4>
    <p>@Name</p>
</div>

@code {
    [Parameter]
    public string Title { get; set; } = "Name";

    [Parameter]
    public string Name { get; set; } = "";
}
```

**Usage**:
```razor
<DisplayName Title="User" Name="Alice" />
<DisplayName Title="Admin" Name="Bob" />
```

## C# Code in Components

### Counter Example

**Pages/Counter.razor**:
```razor
@page "/counter"

<PageTitle>Counter</PageTitle>

<h1>Counter</h1>

<p role="status">Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### Lifecycle Methods

```razor
@implements IDisposable

<h3>Component Lifecycle</h3>

@code {
    protected override void OnInitialized()
    {
        Console.WriteLine("Component initialized");
    }

    protected override async Task OnInitializedAsync()
    {
        // Async initialization
        await Task.Delay(1000);
    }

    protected override void OnParametersSet()
    {
        Console.WriteLine("Parameters set");
    }

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            Console.WriteLine("First render complete");
        }
    }

    public void Dispose()
    {
        Console.WriteLine("Component disposed");
    }
}
```

## JavaScript Interop

### Calling JavaScript from C#

**wwwroot/js/utils.js**:
```javascript
window.utils = {
    showAlert: function(message) {
        alert(message);
    },

    getWindowWidth: function() {
        return window.innerWidth;
    }
};
```

**In Component**:
```razor
@inject IJSRuntime JS

<button @onclick="CallJS">Call JavaScript</button>
<p>Window width: @windowWidth</p>

@code {
    private int windowWidth = 0;

    private async Task CallJS()
    {
        await JS.InvokeVoidAsync("utils.showAlert", "Hello from C#!");
        windowWidth = await JS.InvokeAsync<int>("utils.getWindowWidth");
    }
}
```

### Calling C# from JavaScript

```csharp
// DotNetObjectReference allows JS to call C# methods
public class Interop
{
    [JSInvokable]
    public static string GetMessage()
    {
        return "Hello from C#!";
    }

    [JSInvokable]
    public static int Add(int a, int b)
    {
        return a + b;
    }
}
```

**Register in Program.cs**:
```csharp
builder.Services.AddSingleton<Interop>();
```

**JavaScript**:
```javascript
// Call static method
const message = await DotNet.invokeMethodAsync('MyBlazorApp', 'GetMessage');
console.log(message);

// Call with parameters
const sum = await DotNet.invokeMethodAsync('MyBlazorApp', 'Add', 5, 3);
console.log(sum);  // 8
```

## HTTP and APIs

### HttpClient

**Service**:
```csharp
public class WeatherService
{
    private readonly HttpClient _http;

    public WeatherService(HttpClient http)
    {
        _http = http;
    }

    public async Task<WeatherForecast[]> GetForecastAsync()
    {
        return await _http.GetFromJsonAsync<WeatherForecast[]>("sample-data/weather.json");
    }
}

public class WeatherForecast
{
    public DateTime Date { get; set; }
    public int TemperatureC { get; set; }
    public string Summary { get; set; }
}
```

**Register in Program.cs**:
```csharp
builder.Services.AddScoped<WeatherService>();
```

**Component**:
```razor
@page "/weather"
@inject WeatherService WeatherService

<h3>Weather Forecast</h3>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        forecasts = await WeatherService.GetForecastAsync();
    }
}
```

## State Management

### Cascade Parameters

```razor
<CascadingValue Value="@theme">
    <ChildComponent />
</CascadingValue>

@code {
    private string theme = "dark";
}
```

**Child component**:
```razor
<div class="@Theme">
    Content styled with theme
</div>

@code {
    [CascadingParameter]
    public string Theme { get; set; }
}
```

### App State Service

```csharp
public class AppState
{
    public string UserName { get; set; } = "";

    public event Action OnChange;

    public void SetUserName(string name)
    {
        UserName = name;
        NotifyStateChanged();
    }

    private void NotifyStateChanged() => OnChange?.Invoke();
}
```

**Register**:
```csharp
builder.Services.AddSingleton<AppState>();
```

**Use**:
```razor
@inject AppState AppState
@implements IDisposable

<p>Current user: @AppState.UserName</p>

<input @bind="userName" />
<button @onclick="UpdateUser">Update</button>

@code {
    private string userName = "";

    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }

    private void UpdateUser()
    {
        AppState.SetUserName(userName);
    }

    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

## Forms and Validation

```razor
@page "/contact"

<EditForm Model="@contactModel" OnValidSubmit="@HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div class="form-group">
        <label>Name:</label>
        <InputText @bind-Value="contactModel.Name" class="form-control" />
        <ValidationMessage For="@(() => contactModel.Name)" />
    </div>

    <div class="form-group">
        <label>Email:</label>
        <InputText @bind-Value="contactModel.Email" class="form-control" />
        <ValidationMessage For="@(() => contactModel.Email)" />
    </div>

    <button type="submit" class="btn btn-primary">Submit</button>
</EditForm>

@code {
    private ContactModel contactModel = new();

    private void HandleValidSubmit()
    {
        Console.WriteLine($"Form submitted: {contactModel.Name}, {contactModel.Email}");
    }

    public class ContactModel
    {
        [Required]
        [StringLength(50, ErrorMessage = "Name is too long.")]
        public string Name { get; set; } = "";

        [Required]
        [EmailAddress]
        public string Email { get; set; } = "";
    }
}
```

## Dependency Injection

**Service**:
```csharp
public interface IDataService
{
    Task<List<Item>> GetItemsAsync();
}

public class DataService : IDataService
{
    private readonly HttpClient _http;

    public DataService(HttpClient http)
    {
        _http = http;
    }

    public async Task<List<Item>> GetItemsAsync()
    {
        return await _http.GetFromJsonAsync<List<Item>>("api/items");
    }
}
```

**Register**:
```csharp
builder.Services.AddScoped<IDataService, DataService>();
```

**Inject**:
```razor
@inject IDataService DataService

@code {
    protected override async Task OnInitializedAsync()
    {
        var items = await DataService.GetItemsAsync();
    }
}
```

## Performance Optimization

### Lazy Loading

**Program.cs**:
```csharp
builder.Services.AddScoped(sp => new HttpClient
{
    BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
});

// Lazy load assemblies
builder.Services.AddSingleton<LazyAssemblyLoader>();
```

### Virtualization

For large lists:
```razor
@using Microsoft.AspNetCore.Components.Web.Virtualization

<Virtualize Items="@items" Context="item">
    <div class="item">
        @item.Name
    </div>
</Virtualize>

@code {
    private List<Item> items = Enumerable.Range(1, 10000)
        .Select(i => new Item { Id = i, Name = $"Item {i}" })
        .ToList();
}
```

## Progressive Web App (PWA)

Add PWA support:

```bash
dotnet new blazorwasm -o MyPWA --pwa
```

**Generated files**:
- `service-worker.js` - Service worker for offline support
- `manifest.json` - PWA manifest

## Debugging

### Browser DevTools

1. Run with `dotnet run`
2. Open browser DevTools (F12)
3. Sources tab shows C# code (with source maps)
4. Set breakpoints in C# code
5. Inspect variables, call stack

### Visual Studio / Rider

Full debugging support:
- Breakpoints in C# code
- Step through code
- Watch variables
- Call stack inspection

## Real-World Example: Todo App

**Models/TodoItem.cs**:
```csharp
public class TodoItem
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public bool Completed { get; set; }
}
```

**Services/TodoService.cs**:
```csharp
public class TodoService
{
    private List<TodoItem> todos = new();
    private int nextId = 1;

    public List<TodoItem> GetAll() => todos;

    public void Add(string title)
    {
        todos.Add(new TodoItem { Id = nextId++, Title = title });
    }

    public void Toggle(int id)
    {
        var todo = todos.FirstOrDefault(t => t.Id == id);
        if (todo != null) todo.Completed = !todo.Completed;
    }

    public void Delete(int id)
    {
        todos.RemoveAll(t => t.Id == id);
    }
}
```

**Pages/Todo.razor**:
```razor
@page "/todo"
@inject TodoService TodoService

<h3>Todo List</h3>

<div class="input-group">
    <input @bind="newTodoTitle" @bind:event="oninput"
           @onkeyup="HandleKeyUp" class="form-control" placeholder="New todo" />
    <button @onclick="AddTodo" class="btn btn-primary">Add</button>
</div>

<ul class="list-group mt-3">
    @foreach (var todo in TodoService.GetAll())
    {
        <li class="list-group-item d-flex justify-content-between align-items-center">
            <div>
                <input type="checkbox" @bind="todo.Completed"
                       @onchange="@(() => TodoService.Toggle(todo.Id))" />
                <span class="@(todo.Completed ? "text-decoration-line-through" : "")">
                    @todo.Title
                </span>
            </div>
            <button @onclick="@(() => TodoService.Delete(todo.Id))"
                    class="btn btn-sm btn-danger">Delete</button>
        </li>
    }
</ul>

@code {
    private string newTodoTitle = "";

    private void AddTodo()
    {
        if (!string.IsNullOrWhiteSpace(newTodoTitle))
        {
            TodoService.Add(newTodoTitle);
            newTodoTitle = "";
        }
    }

    private void HandleKeyUp(KeyboardEventArgs e)
    {
        if (e.Key == "Enter")
        {
            AddTodo();
        }
    }
}
```

## Deployment

### Build

```bash
dotnet publish -c Release -o output
```

**Output**:
- `wwwroot/` - Static files
- `_framework/` - .NET runtime and assemblies

### Hosting

**Static site hosting** (GitHub Pages, Netlify, etc.):
```bash
# wwwroot/ contains everything needed
# Just serve as static files
```

**Azure Static Web Apps**:
```yaml
# .github/workflows/azure-static-web-apps.yml
name: Deploy to Azure
on: [push]
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build And Deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_TOKEN }}
          app_location: "/"
          output_location: "wwwroot"
```

## Limitations

- **Size**: Initial download 2-5MB (improving with AOT)
- **Startup**: Cold start can be slow
- **Browser-only**: WASI support is experimental
- **No true threading**: Limited to web workers

## When to Use Blazor WebAssembly

**Good fit**:
- .NET teams building web apps
- Enterprise applications with complex business logic
- Rich, interactive UIs
- Strong typing requirements

**Not ideal**:
- Small, lightweight apps (size matters)
- Need instant startup
- Server-side WASM (use WASI languages instead)

## Next Steps

Blazor WebAssembly brings the full .NET ecosystem to the browser, enabling C# developers to build rich web applications entirely in C#. While the payload is large, the developer experience and access to .NET libraries make it compelling for enterprise applications.

We've completed Part 4's deep dives into major WASM languages. Next, in Part 5, we'll survey additional languages with WASM support—from mature options like Zig and Swift to experimental implementations like Python and beyond.
