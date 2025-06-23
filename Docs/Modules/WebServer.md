# Master Server Toolkit - Web Server

## Description
WebServer module provides a built-in HTTP server for managing the game server via web interface, API and system monitoring. It allows creating RESTful APIs for external services and an admin panel.

## WebServerModule

Main class of the web server module.

### Setup:
```csharp
[Header("Http Server Settings")]
[SerializeField] protected bool autostart = true;
[SerializeField] protected string httpAddress = "127.0.0.1";
[SerializeField] protected int httpPort = 5056;
[SerializeField] protected string[] defaultIndexPage = new string[] { "index", "home" };

[Header("User Credentials Settings")]
[SerializeField] protected bool useCredentials = true;
[SerializeField] protected string username = "admin";
[SerializeField] protected string password = "admin";

[Header("Assets")]
[SerializeField] protected TextAsset startPage;
```

### Start and stop:
```csharp
// Initialization with auto start
public override void Initialize(IServer server)
{
    if (autostart)
    {
        Listen();
    }
}

// Start manually
var webServer = Mst.Server.Modules.GetModule<WebServerModule>();
webServer.Listen(); // Using default settings

// Start with a specific URL
webServer.Listen("http://localhost:8080");

// Stop the server
webServer.Stop();
```

### Command line arguments:
```bash
--web-username "admin"    # Username for web interface access
--web-password "secret"   # Password for web interface access
--web-port 8080           # Port for the HTTP server
--web-address "0.0.0.0"   # Listen address (0.0.0.0 for all interfaces)
```

## Registering handlers

### HTTP methods (GET, POST, PUT, DELETE):
```csharp
// GET request
webServer.RegisterGetHandler("stats", GetStatisticsHandler);

// POST request
webServer.RegisterPostHandler("users/create", CreateUserHandler, true); // true - requires authorization

// PUT request
webServer.RegisterPutHandler("users/update", UpdateUserHandler, true);

// DELETE request
webServer.RegisterDeleteHandler("users/delete", DeleteUserHandler, true);

// Example handler
private async Task<IHttpResult> GetStatisticsHandler(HttpListenerRequest request)
{
    var stats = new
    {
        playersOnline = 120,
        roomsCount = 25,
        serverUptime = "10 hours"
    };
    
    return new JsonResult(stats);
}
```

## HTTP results

### Result types:
```csharp
// Text response
var stringResult = new StringResult("Hello, World!");
stringResult.ContentType = "text/plain"; // default

// JSON response
var jsonResult = new JsonResult(new { status = "ok", users = 5 });

// Status code
var badRequest = new BadRequest(); // 400
var unauthorized = new Unauthorized(); // 401
var notFound = new NotFound(); // 404
var serverError = new InternalServerError("Database error"); // 500
```

### Custom response:
```csharp
public class HtmlResult : HttpResult
{
    public string Html { get; set; }
    
    public HtmlResult(string html)
    {
        Html = html;
        ContentType = "text/html";
        StatusCode = 200;
    }
    
    public override async Task ExecuteResultAsync(HttpListenerResponse response)
    {
        using (var writer = new StreamWriter(response.OutputStream))
        {
            await writer.WriteAsync(Html);
        }
    }
}

// Usage
private async Task<IHttpResult> GetDashboardHandler(HttpListenerRequest request)
{
    string html = "<html><body><h1>Dashboard</h1></body></html>";
    return new HtmlResult(html);
}
```

## Web Controllers

Web controllers allow you to create structured groups of handlers.

### Creating a controller:
```csharp
public class UsersController : WebController
{
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Security settings
        UseCredentials = true;
        
        // Register routes
        WebServer.RegisterGetHandler("users/list", GetUsersList);
        WebServer.RegisterGetHandler("users/view", GetUserDetails);
        WebServer.RegisterPostHandler("users/create", CreateUser, true);
        WebServer.RegisterPutHandler("users/update", UpdateUser, true);
        WebServer.RegisterDeleteHandler("users/delete", DeleteUser, true);
    }
    
    private async Task<IHttpResult> GetUsersList(HttpListenerRequest request)
    {
        // Logic for getting the list of users
        var users = new[] 
        { 
            new { id = 1, name = "John" },
            new { id = 2, name = "Alice" }
        };
        
        return new JsonResult(users);
    }
    
    private async Task<IHttpResult> GetUserDetails(HttpListenerRequest request)
    {
        // Get request parameters
        string id = request.QueryString["id"];
        
        if (string.IsNullOrEmpty(id))
            return new BadRequest();
        
        // Logic for obtaining user data
        var user = new { id = int.Parse(id), name = "John", email = "john@example.com" };
        
        return new JsonResult(user);
    }
    
    private async Task<IHttpResult> CreateUser(HttpListenerRequest request)
    {
        // Read request body
        using (var reader = new StreamReader(request.InputStream))
        {
            string body = await reader.ReadToEndAsync();
            // Parse JSON and create user
            
            return new JsonResult(new { success = true, message = "User created" });
        }
    }
    
    // Other methods...
}
```

### Adding a controller:
```csharp
// 1. By adding to a GameObject
gameObject.AddComponent<UsersController>();

// 2. Or programmatically
var controller = new UsersController();
Controllers.TryAdd(controller.GetType(), controller);
controller.Initialize(this);
```

## Built-in controllers

### InfoWebController:
```csharp
// Provides information about the server
// GET /info - general information
// GET /info/modules - list of modules
// GET /info/connections - connection information
```

### NotificationWebController:
```csharp
// Sending notifications via the web interface
// POST /notifications/all - send to all
// POST /notifications/room/{roomId} - send to a room
// POST /notifications/user/{userId} - send to a user

// Example usage:
// curl -X POST http://localhost:5056/notifications/all
//   -u admin:admin
//   -H "Content-Type: application/json"
//   -d '{"message":"Server will restart in 5 minutes"}'
```

### AnalyticsWebController:
```csharp
// Retrieve analytics data
// GET /analytics/users - user statistics
// GET /analytics/rooms - room statistics
```

## Handling requests

### Reading request parameters:
```csharp
private async Task<IHttpResult> SearchHandler(HttpListenerRequest request)
{
    // Query parameters (?name=value)
    string query = request.QueryString["q"];
    string limit = request.QueryString["limit"] ?? "10";
    
    // Headers
    string authorization = request.Headers["Authorization"];
    string contentType = request.ContentType;
    
    // Cookies
    Cookie sessionCookie = request.Cookies["session"];
    
    // URL parameters
    string[] segments = request.Url.Segments;
    
    // Result
    return new JsonResult(new { 
        query = query,
        limit = int.Parse(limit),
        results = new[] { "result1", "result2" }
    });
}
```

### Reading request body:
```csharp
private async Task<IHttpResult> CreateItemHandler(HttpListenerRequest request)
{
    // Check Content-Type
    if (!request.ContentType.Contains("application/json"))
        return new BadRequest();
    
    // Read request body
    string body;
    using (var reader = new StreamReader(request.InputStream, request.ContentEncoding))
    {
        body = await reader.ReadToEndAsync();
    }
    
    try
    {
        // JSON deserialization
        var data = JsonUtility.FromJson<ItemData>(body);
        
        // Create an object
        string itemId = Guid.NewGuid().ToString();
        
        return new JsonResult(new { 
            success = true, 
            id = itemId 
        });
    }
    catch (Exception ex)
    {
        return new BadRequest();
    }
}

[Serializable]
public class ItemData
{
    public string name;
    public int quantity;
}
```

## Authentication and authorization

### Configuring basic authentication:
```csharp
// 1. In the Inspector
[SerializeField] protected bool useCredentials = true;
[SerializeField] protected string username = "admin";
[SerializeField] protected string password = "admin";

// 2. In code during handler registration
// true - requires authorization, false - public access
webServer.RegisterGetHandler("admin/stats", AdminStatsHandler, true);
```

### Custom authorization:
```csharp
public class CustomAuthController : WebController
{
    private Dictionary<string, string> apiKeys = new Dictionary<string, string>
    {
        { "client1", "key1" },
        { "client2", "key2" }
    };
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Disable built-in Basic Authentication
        UseCredentials = false;
        
        WebServer.RegisterGetHandler("api/data", GetApiData);
    }
    
    private async Task<IHttpResult> GetApiData(HttpListenerRequest request)
    {
        // Check the API key
        string apiKey = request.Headers["X-API-Key"];
        
        if (string.IsNullOrEmpty(apiKey) || !apiKeys.ContainsValue(apiKey))
            return new Unauthorized();
        
        // Authorized access
        return new JsonResult(new { data = "Secure API data" });
    }
}
```

## Creating a web interface

### Built-in start page:
```csharp
[Header("Assets")]
[SerializeField] protected TextAsset startPage;

// HTML page with placeholders
// #MST-TITLE# will be replaced with "MST v.X.X.X"
// #MST-GREETINGS# will be replaced with "MST v.X.X.X"
```

### Creating an HTML controller:
```csharp
public class DashboardController : WebController
{
    [SerializeField] private TextAsset dashboardHtmlTemplate;
    [SerializeField] private TextAsset loginHtmlTemplate;
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Public pages
        WebServer.RegisterGetHandler("login", LoginPage, false);
        
        // Protected pages
        WebServer.RegisterGetHandler("dashboard", DashboardPage, true);
        WebServer.RegisterGetHandler("dashboard/users", UsersPage, true);
    }
    
    private async Task<IHttpResult> LoginPage(HttpListenerRequest request)
    {
        var result = new StringResult(loginHtmlTemplate.text);
        result.ContentType = "text/html";
        return result;
    }
    
    private async Task<IHttpResult> DashboardPage(HttpListenerRequest request)
    {
        // Get statistics
        var playersCount = Mst.Server.ConnectionsCount;
        var roomsCount = Mst.Server.Modules.GetModule<RoomsModule>()?.GetRooms().Count ?? 0;
        
        // Substitute data into the template
        string html = dashboardHtmlTemplate.text
            .Replace("{{PLAYERS_COUNT}}", playersCount.ToString())
            .Replace("{{ROOMS_COUNT}}", roomsCount.ToString())
            .Replace("{{SERVER_UPTIME}}", GetServerUptime());
        
        var result = new StringResult(html);
        result.ContentType = "text/html";
        return result;
    }
}
```

## RESTful API for external services

### Game API example:
```csharp
public class GameApiController : WebController
{
    private AuthModule authModule;
    private ProfilesModule profilesModule;
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Get dependencies
        authModule = server.GetModule<AuthModule>();
        profilesModule = server.GetModule<ProfilesModule>();
        
        // API routes
        WebServer.RegisterGetHandler("api/players", GetPlayers, true);
        WebServer.RegisterGetHandler("api/players/online", GetOnlinePlayers, true);
        WebServer.RegisterGetHandler("api/player", GetPlayer, true);
        WebServer.RegisterPostHandler("api/player/reward", AddReward, true);
    }
    
    private async Task<IHttpResult> GetOnlinePlayers(HttpListenerRequest request)
    {
        var onlinePlayers = authModule.LoggedInUsers.Select(u => new {
            id = u.UserId,
            username = u.Username,
            isGuest = u.IsGuest
        }).ToList();
        
        return new JsonResult(onlinePlayers);
    }
    
    private async Task<IHttpResult> AddReward(HttpListenerRequest request)
    {
        // Read request body
        using (var reader = new StreamReader(request.InputStream))
        {
            string body = await reader.ReadToEndAsync();
            var data = JsonUtility.FromJson<RewardData>(body);
            
            // Find user
            var user = authModule.GetLoggedInUserById(data.userId);
            
            if (user == null)
                return new NotFound();
            
            // Find profile
            if (!user.Peer.TryGetExtension(out ProfilePeerExtension profileExt))
                return new NotFound();
            
            // Add currency
            if (profileExt.Profile.TryGet(ProfilePropertyOpCodes.currency, out ObservableDictionary currencies))
            {
                currencies.Set(data.currencyType, currencies.Get(data.currencyType, 0) + data.amount);
                return new JsonResult(new { success = true });
            }
            else
            {
                return new InternalServerError("Currency property not found");
            }
        }
    }
    
    [Serializable]
    private class RewardData
    {
        public string userId;
        public string currencyType;
        public int amount;
    }
}
```

## Integration with external systems

### Webhooks:
```csharp
// Send a webhook when a game event occurs
public class WebhookManager : MonoBehaviour
{
    [SerializeField] private string webhookUrl = "https://example.com/webhook";
    
    private AuthModule authModule;
    
    private void Start()
    {
        authModule = Mst.Server.Modules.GetModule<AuthModule>();
        
        // Subscribe to events
        authModule.OnUserLoggedInEvent += OnUserLoggedIn;
        authModule.OnUserLoggedOutEvent += OnUserLoggedOut;
    }
    
    private async void OnUserLoggedIn(IUserPeerExtension user)
    {
        var data = new WebhookData
        {
            eventType = "user.login",
            userId = user.UserId,
            username = user.Username,
            timestamp = DateTime.UtcNow.ToString("o")
        };
        
        await SendWebhookAsync(data);
    }
    
    private async void OnUserLoggedOut(IUserPeerExtension user)
    {
        var data = new WebhookData
        {
            eventType = "user.logout",
            userId = user.UserId,
            username = user.Username,
            timestamp = DateTime.UtcNow.ToString("o")
        };
        
        await SendWebhookAsync(data);
    }
    
    private async Task SendWebhookAsync(WebhookData data)
    {
        try
        {
            using (var client = new WebClient())
            {
                client.Headers[HttpRequestHeader.ContentType] = "application/json";
                string json = JsonUtility.ToJson(data);
                await client.UploadStringTaskAsync(webhookUrl, json);
            }
        }
        catch (Exception ex)
        {
            Debug.LogError($"Failed to send webhook: {ex.Message}");
        }
    }
    
    [Serializable]
    private class WebhookData
    {
        public string eventType;
        public string userId;
        public string username;
        public string timestamp;
    }
}
```

## Best Practices

1. **Use controllers to organize logic** – group related handlers into controllers
2. **Enable authorization for admin functions** – set `UseCredentials = true` on protected routes
3. **Handle errors and return correct HTTP codes** – use result classes like BadRequest, NotFound, etc.
4. **Validate input data** – always check request parameters and body before use
5. **Create API documentation** – describe available routes, parameters and results
6. **Separate read and write operations** – use GET for reads, POST/PUT/DELETE for changes
7. **Configure CORS if necessary** – for access from external web apps
8. **Restrict access by IP** – for critical APIs
9. **Use JSON for data exchange** – preferred over other formats
10. **Implement API statistics tracking** – monitor usage and performance
