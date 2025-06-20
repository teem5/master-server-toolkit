# Master Server Toolkit - Database

## Description
A system for accessing databases and APIs. Provides an abstraction for working with different types of data stores.

## IDatabaseAccessor

Base interface for all database accessors.

```csharp
public interface IDatabaseAccessor : IDisposable
{
    MstProperties CustomProperties { get; }
    Logger Logger { get; set; }
}
```

### Implementing your own accessor:
```csharp
public class MySQLAccessor : IDatabaseAccessor
{
    public MstProperties CustomProperties { get; } = new MstProperties();
    public Logger Logger { get; set; }
    
    // Implementation of MySQL methods
    public async Task<User> GetUserById(string userId)
    {
        // Logic to get a user
    }
    
    public void Dispose()
    {
        // Release resources
    }
}
```

## MstDbAccessor

Manager for working with different database accessors.

### Main methods:
```csharp
// Add an accessor
AddAccessor(IDatabaseAccessor access);

// Get accessor by type
T GetAccessor<T>() where T : class, IDatabaseAccessor;
```

### Usage example:
```csharp
// Create a central DB manager
var dbManager = new MstDbAccessor();

// Add various accessors
dbManager.AddAccessor(new MongoDbAccessor());
dbManager.AddAccessor(new RedisAccessor());
dbManager.AddAccessor(new MySQLAccessor());

// Get required accessor
var mongoDb = dbManager.GetAccessor<MongoDbAccessor>();
var redis = dbManager.GetAccessor<RedisAccessor>();

// Use accessor
var user = await mongoDb.GetUserById("user123");
await redis.SetCache("key", data, TimeSpan.FromMinutes(30));
```

## DatabaseAccessorFactory

Abstract factory for creating database accessors.

### Example implementation:
```csharp
public class GameDatabaseFactory : DatabaseAccessorFactory
{
    [Header("Database Settings")]
    [SerializeField] private string connectionString;
    [SerializeField] private DatabaseType dbType;
    
    public override void CreateAccessors()
    {
        switch (dbType)
        {
            case DatabaseType.MongoDB:
                var mongoDb = new MongoDbAccessor(connectionString);
                mongoDb.Logger = logger;
                Mst.Server.DbAccessors.AddAccessor(mongoDb);
                break;
                
            case DatabaseType.MySQL:
                var mysql = new MySQLAccessor(connectionString);
                mysql.Logger = logger;
                Mst.Server.DbAccessors.AddAccessor(mysql);
                break;
                
            case DatabaseType.Redis:
                var redis = new RedisAccessor(connectionString);
                redis.Logger = logger;
                Mst.Server.DbAccessors.AddAccessor(redis);
                break;
        }
        
        logger.Info("Database accessors created successfully");
    }
}
```

### Setup via Inspector:
1. Create a GameObject
2. Add a component that inherits from DatabaseAccessorFactory
3. Configure connection string and DB type
4. Accessors will be created automatically at start

## Accessor examples

### Redis accessor:
```csharp
public class RedisAccessor : IDatabaseAccessor
{
    private ConnectionMultiplexer _redis;
    private IDatabase _db;
    
    public Logger Logger { get; set; }
    public MstProperties CustomProperties { get; } = new MstProperties();
    
    public RedisAccessor(string connectionString)
    {
        _redis = ConnectionMultiplexer.Connect(connectionString);
        _db = _redis.GetDatabase();
    }
    
    public async Task SetCache(string key, string value, TimeSpan? expiry = null)
    {
        await _db.StringSetAsync(key, value, expiry);
    }
    
    public async Task<string> GetCache(string key)
    {
        return await _db.StringGetAsync(key);
    }
    
    public void Dispose()
    {
        _redis?.Dispose();
    }
}
```

### MongoDB accessor:
```csharp
public class MongoDbAccessor : IDatabaseAccessor
{
    private IMongoClient _client;
    private IMongoDatabase _database;
    
    public Logger Logger { get; set; }
    public MstProperties CustomProperties { get; } = new MstProperties();
    
    public MongoDbAccessor(string connectionString)
    {
        _client = new MongoClient(connectionString);
        _database = _client.GetDatabase("gamedb");
    }
    
    public async Task<User> GetUserById(string userId)
    {
        var collection = _database.GetCollection<User>("users");
        return await collection.Find(u => u.Id == userId).FirstOrDefaultAsync();
    }
    
    public void Dispose()
    {
        // MongoDB driver handles disposal automatically
    }
}
```

## Best practices

1. **One accessor - one responsibility**: each accessor should work with a single storage type
2. **Use a factory**: create accessors through DatabaseAccessorFactory
3. **Logging**: always set a Logger for your accessors
4. **Resource cleanup**: implement Dispose properly
5. **Asynchronous operations**: use async/await for database calls

## Structure for scaling

```
DatabaseAccessors/
├── SQL/
│   ├── MySQLAccessor.cs
│   └── PostgreSQLAccessor.cs
├── NoSQL/
│   ├── MongoDbAccessor.cs
│   └── CassandraAccessor.cs
├── Cache/
│   ├── RedisAccessor.cs
│   └── MemcachedAccessor.cs
└── Factories/
    ├── MainDatabaseFactory.cs
    └── CacheDatabaseFactory.cs
```
