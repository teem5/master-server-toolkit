# Master Server Toolkit - Analytics

## Description
A module for collecting, storing, and analyzing game events and metrics with support for custom queries and data filtering.

## Main Components

### AnalyticsModule (Server)
```csharp
// Settings
[SerializeField] protected float saveDebounceTime = 5f; // Delay before saving to DB
[SerializeField] protected bool useAnalytics = true;    // Enable/disable analytics
[SerializeField] protected DatabaseAccessorFactory databaseAccessorFactory; // Database factory
```

### AnalyticsModuleClient (Client)
```csharp
// Send a one-time event
void SendEvent(string key, string category, Dictionary<string, string> data);

// Send a session event (once per session)
void SendSessionEvent(string key, string category, Dictionary<string, string> data);
```

## Event Structure

### AnalyticsDataInfoPacket
```csharp
public class AnalyticsDataInfoPacket : IAnalyticsInfoData
{
    public string Id { get; set; }                       // Unique event ID
    public string UserId { get; set; }                   // User ID
    public string Key { get; set; }                      // Event key
    public string Category { get; set; }                 // Event category
    public DateTime Timestamp { get; set; }              // Event time
    public Dictionary<string, string> Data { get; set; } // Event data
    public bool IsSessionEvent { get; set; }             // Session event flag
}
```

## Usage Examples

### Sending events from the client
```csharp
// Get the client
var analytics = Mst.Client.Analytics;

// Simple gameplay event
var data = new Dictionary<string, string>
{
    { "level", "2" },
    { "difficulty", "normal" },
    { "time", "125.5" }
};

analytics.SendEvent("level_completed", "gameplay", data);

// Session event (sent only once per session)
var sessionData = new Dictionary<string, string>
{
    { "device", SystemInfo.deviceModel },
    { "os", SystemInfo.operatingSystem },
    { "screen_resolution", $"{Screen.width}x{Screen.height}" }
};

analytics.SendSessionEvent("session_start", "system", sessionData);
```

### Gameplay tracking system
```csharp
public class AnalyticsTracker : MonoBehaviour
{
    // Track items
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

    // Track combat system
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

    // Track purchases
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

### Database accessor implementation
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

    // Implement other interface methods...
}
```

### Registering the database
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

## Data requests on the server
```csharp
// Get the module
var analyticsModule = Mst.Server.Modules.GetModule<AnalyticsModule>();

// Get all events (with pagination)
var allEvents = await analyticsModule.GetAll(100, 0); // 100 events, page 0

// Get events by key
var levelEvents = await analyticsModule.GetByKey("level_completed", 100, 0);

// Get events by user
var userEvents = await analyticsModule.GetByUserId(userId, 100, 0);

// Get events in a date range
var dateStart = DateTime.UtcNow.AddDays(-7);
var dateEnd = DateTime.UtcNow;
var weekEvents = await analyticsModule.GetByTimestampRange(dateStart, dateEnd, 1000, 0);

// Get events with a custom query
var customQuery = "{ $and: [ { 'Key': 'purchase' }, { 'Data.price': { $gt: '10' } } ] }";
var expensivePurchases = await analyticsModule.GetWithQuery(customQuery, 100, 0);
```

## Web panel integration
```csharp
// Web panel controller for analytics
public class AnalyticsWebController : WebController
{
    private AnalyticsModule analyticsModule;

    public override void Initialize(WebServerModule server)
    {
        base.Initialize(server);

        analyticsModule = server.GetModule<AnalyticsModule>();

        // API routes
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

    // Other handlers...
}
```

## Best practices

1. **Use debouncing** to batch event processing
2. **Group events by categories** for easier analysis
3. **Limit the amount of collected data** – gather only what you need
4. **Document event keys** to ensure consistency
5. **Analyze data regularly** to identify trends
6. **Use unique identifiers** to avoid duplicates
