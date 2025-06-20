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

### Запуск и остановка:
```csharp
// Инициализация с автозапуском
public override void Initialize(IServer server)
{
    if (autostart)
    {
        Listen();
    }
}

// Запуск вручную
var webServer = Mst.Server.Modules.GetModule<WebServerModule>();
webServer.Listen(); // Используя настройки по умолчанию

// Запуск с указанием URL
webServer.Listen("http://localhost:8080");

// Остановка сервера
webServer.Stop();
```

### Аргументы командной строки:
```bash
--web-username "admin"    # Имя пользователя для доступа к веб-интерфейсу
--web-password "secret"   # Пароль для доступа к веб-интерфейсу
--web-port 8080           # Порт для HTTP сервера
--web-address "0.0.0.0"   # Адрес для прослушивания (0.0.0.0 для всех интерфейсов)
```

## Регистрация обработчиков

### HTTP методы (GET, POST, PUT, DELETE):
```csharp
// GET запрос
webServer.RegisterGetHandler("stats", GetStatisticsHandler);

// POST запрос
webServer.RegisterPostHandler("users/create", CreateUserHandler, true); // true - требует авторизацию

// PUT запрос
webServer.RegisterPutHandler("users/update", UpdateUserHandler, true);

// DELETE запрос
webServer.RegisterDeleteHandler("users/delete", DeleteUserHandler, true);

// Пример обработчика
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

## Результаты HTTP

### Типы результатов:
```csharp
// Текстовый ответ
var stringResult = new StringResult("Hello, World!");
stringResult.ContentType = "text/plain"; // по умолчанию

// JSON ответ
var jsonResult = new JsonResult(new { status = "ok", users = 5 });

// Код состояния
var badRequest = new BadRequest(); // 400
var unauthorized = new Unauthorized(); // 401
var notFound = new NotFound(); // 404
var serverError = new InternalServerError("Database error"); // 500
```

### Кастомный ответ:
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

// Использование
private async Task<IHttpResult> GetDashboardHandler(HttpListenerRequest request)
{
    string html = "<html><body><h1>Dashboard</h1></body></html>";
    return new HtmlResult(html);
}
```

## Web Controllers

Web контроллеры позволяют создавать структурированные группы обработчиков.

### Создание контроллера:
```csharp
public class UsersController : WebController
{
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Настройка безопасности
        UseCredentials = true;
        
        // Регистрация маршрутов
        WebServer.RegisterGetHandler("users/list", GetUsersList);
        WebServer.RegisterGetHandler("users/view", GetUserDetails);
        WebServer.RegisterPostHandler("users/create", CreateUser, true);
        WebServer.RegisterPutHandler("users/update", UpdateUser, true);
        WebServer.RegisterDeleteHandler("users/delete", DeleteUser, true);
    }
    
    private async Task<IHttpResult> GetUsersList(HttpListenerRequest request)
    {
        // Логика получения списка пользователей
        var users = new[] 
        { 
            new { id = 1, name = "John" },
            new { id = 2, name = "Alice" }
        };
        
        return new JsonResult(users);
    }
    
    private async Task<IHttpResult> GetUserDetails(HttpListenerRequest request)
    {
        // Получение параметров запроса
        string id = request.QueryString["id"];
        
        if (string.IsNullOrEmpty(id))
            return new BadRequest();
        
        // Логика получения данных пользователя
        var user = new { id = int.Parse(id), name = "John", email = "john@example.com" };
        
        return new JsonResult(user);
    }
    
    private async Task<IHttpResult> CreateUser(HttpListenerRequest request)
    {
        // Чтение тела запроса
        using (var reader = new StreamReader(request.InputStream))
        {
            string body = await reader.ReadToEndAsync();
            // Парсинг JSON и создание пользователя
            
            return new JsonResult(new { success = true, message = "User created" });
        }
    }
    
    // Другие методы...
}
```

### Добавление контроллера:
```csharp
// 1. Через добавление на GameObject
gameObject.AddComponent<UsersController>();

// 2. Или программно
var controller = new UsersController();
Controllers.TryAdd(controller.GetType(), controller);
controller.Initialize(this);
```

## Встроенные контроллеры

### InfoWebController:
```csharp
// Предоставляет информацию о сервере
// GET /info - общая информация
// GET /info/modules - список модулей
// GET /info/connections - информация о подключениях
```

### NotificationWebController:
```csharp
// Отправка уведомлений через веб-интерфейс
// POST /notifications/all - отправить всем
// POST /notifications/room/{roomId} - отправить в комнату
// POST /notifications/user/{userId} - отправить пользователю

// Пример использования:
// curl -X POST http://localhost:5056/notifications/all
//   -u admin:admin
//   -H "Content-Type: application/json"
//   -d '{"message":"Server will restart in 5 minutes"}'
```

### AnalyticsWebController:
```csharp
// Получение аналитических данных
// GET /analytics/users - статистика пользователей
// GET /analytics/rooms - статистика комнат
```

## Обработка запросов

### Чтение параметров запроса:
```csharp
private async Task<IHttpResult> SearchHandler(HttpListenerRequest request)
{
    // Query параметры (?name=value)
    string query = request.QueryString["q"];
    string limit = request.QueryString["limit"] ?? "10";
    
    // Заголовки
    string authorization = request.Headers["Authorization"];
    string contentType = request.ContentType;
    
    // Cookies
    Cookie sessionCookie = request.Cookies["session"];
    
    // URL параметры
    string[] segments = request.Url.Segments;
    
    // Результат
    return new JsonResult(new { 
        query = query,
        limit = int.Parse(limit),
        results = new[] { "result1", "result2" }
    });
}
```

### Чтение тела запроса:
```csharp
private async Task<IHttpResult> CreateItemHandler(HttpListenerRequest request)
{
    // Проверка Content-Type
    if (!request.ContentType.Contains("application/json"))
        return new BadRequest();
    
    // Чтение тела запроса
    string body;
    using (var reader = new StreamReader(request.InputStream, request.ContentEncoding))
    {
        body = await reader.ReadToEndAsync();
    }
    
    try
    {
        // Десериализация JSON
        var data = JsonUtility.FromJson<ItemData>(body);
        
        // Создание объекта
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

## Аутентификация и авторизация

### Настройка базовой аутентификации:
```csharp
// 1. В Inspector
[SerializeField] protected bool useCredentials = true;
[SerializeField] protected string username = "admin";
[SerializeField] protected string password = "admin";

// 2. В коде при регистрации обработчика
// true - требуется авторизация, false - публичный доступ
webServer.RegisterGetHandler("admin/stats", AdminStatsHandler, true);
```

### Кастомная авторизация:
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
        
        // Отключаем встроенную Basic Authentication
        UseCredentials = false;
        
        WebServer.RegisterGetHandler("api/data", GetApiData);
    }
    
    private async Task<IHttpResult> GetApiData(HttpListenerRequest request)
    {
        // Проверка API ключа
        string apiKey = request.Headers["X-API-Key"];
        
        if (string.IsNullOrEmpty(apiKey) || !apiKeys.ContainsValue(apiKey))
            return new Unauthorized();
        
        // Авторизованный доступ
        return new JsonResult(new { data = "Secure API data" });
    }
}
```

## Создание веб-интерфейса

### Встроенная стартовая страница:
```csharp
[Header("Assets")]
[SerializeField] protected TextAsset startPage;

// HTML страница с плейсхолдерами
// #MST-TITLE# - будет заменено на "MST v.X.X.X"
// #MST-GREETINGS# - будет заменено на "MST v.X.X.X"
```

### Создание HTML контроллера:
```csharp
public class DashboardController : WebController
{
    [SerializeField] private TextAsset dashboardHtmlTemplate;
    [SerializeField] private TextAsset loginHtmlTemplate;
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Публичные страницы
        WebServer.RegisterGetHandler("login", LoginPage, false);
        
        // Защищенные страницы
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
        // Получение статистики
        var playersCount = Mst.Server.ConnectionsCount;
        var roomsCount = Mst.Server.Modules.GetModule<RoomsModule>()?.GetRooms().Count ?? 0;
        
        // Подстановка данных в шаблон
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

## RESTful API для внешних сервисов

### Пример игрового API:
```csharp
public class GameApiController : WebController
{
    private AuthModule authModule;
    private ProfilesModule profilesModule;
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        // Получение зависимостей
        authModule = server.GetModule<AuthModule>();
        profilesModule = server.GetModule<ProfilesModule>();
        
        // API маршруты
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
        // Чтение тела запроса
        using (var reader = new StreamReader(request.InputStream))
        {
            string body = await reader.ReadToEndAsync();
            var data = JsonUtility.FromJson<RewardData>(body);
            
            // Поиск пользователя
            var user = authModule.GetLoggedInUserById(data.userId);
            
            if (user == null)
                return new NotFound();
            
            // Поиск профиля
            if (!user.Peer.TryGetExtension(out ProfilePeerExtension profileExt))
                return new NotFound();
            
            // Добавление валюты
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

## Интеграция с внешними системами

### Вебхуки (webhooks):
```csharp
// Отправка вебхука при событии в игре
public class WebhookManager : MonoBehaviour
{
    [SerializeField] private string webhookUrl = "https://example.com/webhook";
    
    private AuthModule authModule;
    
    private void Start()
    {
        authModule = Mst.Server.Modules.GetModule<AuthModule>();
        
        // Подписка на события
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
