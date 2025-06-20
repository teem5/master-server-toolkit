# Master Server Toolkit - Analytics

## 설명
사용자 쿼리 및 데이터 여과를 지원하는 게임 이벤트 및 메트릭의 수집, 저장 및 분석 모듈.

## 주요 구성 요소

### AnalyticsModule (서버)
```csharp
// Настройки
[SerializeField] protected float saveDebounceTime = 5f; // Задержка перед сохранением в БД
[SerializeField] protected bool useAnalytics = true;    // Включить/выключить аналитику
[SerializeField] protected DatabaseAccessorFactory databaseAccessorFactory; // Фабрика базы данных
```

### AnalyticsModuleClient (클라이언트)
```csharp
// Отправка события (единоразовое)
void SendEvent(string key, string category, Dictionary<string, string> data);

// Отправка события сессии (только один раз за сессию)
void SendSessionEvent(string key, string category, Dictionary<string, string> data);
```

## 사건의 구조

### AnalyticsDataInfoPacket
```csharp
public class AnalyticsDataInfoPacket : IAnalyticsInfoData
{
    public string Id { get; set; }                       // Уникальный ID события
    public string UserId { get; set; }                   // ID пользователя
    public string Key { get; set; }                      // Ключ события
    public string Category { get; set; }                 // Категория события
    public DateTime Timestamp { get; set; }              // Время события
    public Dictionary<string, string> Data { get; set; } // Данные события
    public bool IsSessionEvent { get; set; }             // Событие сессии
}
```

## 사용의 예

### 클라이언트로부터 이벤트 보내기
```csharp
// Получение клиента
var analytics = Mst.Client.Analytics;

// Простое событие игры
var data = new Dictionary<string, string>
{
    { "level", "2" },
    { "difficulty", "normal" },
    { "time", "125.5" }
};

analytics.SendEvent("level_completed", "gameplay", data);

// Событие сессии (отправляется только один раз за сессию)
var sessionData = new Dictionary<string, string>
{
    { "device", SystemInfo.deviceModel },
    { "os", SystemInfo.operatingSystem },
    { "screen_resolution", $"{Screen.width}x{Screen.height}" }
};

analytics.SendSessionEvent("session_start", "system", sessionData);
```

### 게임 동작 추적을위한 시스템
```csharp
public class AnalyticsTracker : MonoBehaviour
{
    // Отслеживание предметов
    public void TrackItemAcquired(string itemId, string source)
    {
        var data = new Dictionary<string, string>
        {
            { "item_id", itemId },
            { "source", source },
            { "player_level", PlayerStats.Level.ToString() }
        };
        
        Mst.Client.Analytics.SendEvent("item_acquired", "economy", data);
    }
    
    // Отслеживание боевой системы
    public void TrackEnemyDefeated(string enemyType, string weaponUsed, int timeToKill)
    {
        var data = new Dictionary<string, string>
        {
            { "enemy_type", enemyType },
            { "weapon", weaponUsed },
            { "time_to_kill", timeToKill.ToString() },
            { "player_health", PlayerStats.Health.ToString() }
        };
        
        Mst.Client.Analytics.SendEvent("enemy_defeated", "combat", data);
    }
    
    // Отслеживание покупок
    public void TrackPurchase(string productId, float price, string currency)
    {
        var data = new Dictionary<string, string>
        {
            { "product_id", productId },
            { "price", price.ToString() },
            { "currency", currency },
            { "player_balance", PlayerStats.Balance.ToString() }
        };
        
        Mst.Client.Analytics.SendEvent("purchase", "economy", data);
    }
}
```

### 데이터베이스 액세스 인터페이스 구현
```csharp
public class MongoAnalyticsAccessor : IAnalyticsDatabaseAccessor
{
    private MongoClient client;
    private IMongoDatabase database;
    private IMongoCollection<AnalyticsDataInfoPacket> eventsCollection;
    
    public void Initialize(string connectionString)
    {
        client = new MongoClient(connectionString);
        database = client.GetDatabase("game_analytics");
        eventsCollection = database.GetCollection<AnalyticsDataInfoPacket>("events");
    }
    
    public async Task Insert(IEnumerable<IAnalyticsInfoData> eventsData)
    {
        var documents = eventsData.Cast<AnalyticsDataInfoPacket>().ToList();
        await eventsCollection.InsertManyAsync(documents);
    }
    
    public async Task<IEnumerable<IAnalyticsInfoData>> GetByKey(string eventKey, int size, int page)
    {
        var filter = Builders<AnalyticsDataInfoPacket>.Filter.Eq(e => e.Key, eventKey);
        var options = new FindOptions<AnalyticsDataInfoPacket>
        {
            Limit = size,
            Skip = page * size
        };
        
        var cursor = await eventsCollection.FindAsync(filter, options);
        return await cursor.ToListAsync();
    }
    
    // Реализация других методов интерфейса...
}
```

### 데이터베이스 등록
```csharp
public class AnalyticsDatabaseFactory : DatabaseAccessorFactory
{
    [SerializeField] private string connectionString = "mongodb://localhost:27017";
    
    public override void CreateAccessors()
    {
        var accessor = new MongoAnalyticsAccessor();
        accessor.Initialize(connectionString);
        
        Mst.Server.DbAccessors.AddAccessor(accessor);
    }
}
```

## 서버의 데이터 쿼리

```csharp
// Получение модуля
var analyticsModule = Mst.Server.Modules.GetModule<AnalyticsModule>();

// Получение всех событий (с пагинацией)
var allEvents = await analyticsModule.GetAll(100, 0); // 100 событий, страница 0

// Получение событий по ключу
var levelEvents = await analyticsModule.GetByKey("level_completed", 100, 0);

// Получение событий пользователя
var userEvents = await analyticsModule.GetByUserId(userId, 100, 0);

// Получение событий в диапазоне дат
var dateStart = DateTime.UtcNow.AddDays(-7);
var dateEnd = DateTime.UtcNow;
var weekEvents = await analyticsModule.GetByTimestampRange(dateStart, dateEnd, 1000, 0);

// Получение событий с пользовательским запросом
var customQuery = "{ $and: [ { 'Key': 'purchase' }, { 'Data.price': { $gt: '10' } } ] }";
var expensivePurchases = await analyticsModule.GetWithQuery(customQuery, 100, 0);
```

## 웹 패널과 통합

```csharp
// Контроллер для веб-панели аналитики
public class AnalyticsWebController : WebController
{
    private AnalyticsModule analyticsModule;
    
    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);
        
        analyticsModule = server.GetModule<AnalyticsModule>();
        
        // API маршруты
        WebServer.RegisterGetHandler("api/analytics/events", GetEventsHandler, true);
        WebServer.RegisterGetHandler("api/analytics/users", GetUsersHandler, true);
        WebServer.RegisterGetHandler("api/analytics/dashboard", GetDashboardHandler, true);
    }
    
    private async Task<IHttpResult> GetEventsHandler(HttpListenerRequest request)
    {
        string eventType = request.QueryString["type"];
        int size = int.Parse(request.QueryString["size"] ?? "100");
        int page = int.Parse(request.QueryString["page"] ?? "0");
        
        var events = await analyticsModule.GetByKey(eventType, size, page);
        
        return new JsonResult(events);
    }
    
    // Другие обработчики...
}
```

## 모범 사례

1. ** 이벤트 패키지 처리에 Debows ** 사용 **
2. ** 카테고리 별 그룹 이벤트 **보다 편리한 분석을 위해
3. ** 수집 된 데이터 금액 제한 ** - 필요한 정보 만 수집하십시오.
4. ** 이벤트 이벤트 문서 ** 일관성을 보장합니다.
5. ** 데이터를 정기적으로 분석하여 추세를 식별합니다.
6. ** 이벤트에 고유 식별자 ** 사용하여 복제를 제거하십시오.
