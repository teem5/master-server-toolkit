# Master Server Toolkit - Logger

## Description
Flexible logging system with support for levels, named loggers and configurable outputs.

## Logger

Main class for logging messages.

### Creating a logger:
```csharp
// Create a named logger
Logger logger = Mst.Create.Logger("MyModule");
logger.LogLevel = LogLevel.Info;

// Get a logger via LogManager
Logger networkLogger = LogManager.GetLogger("Network");
```

### Log levels:
```csharp
LogLevel.All      // All messages
LogLevel.Trace    // Detailed trace
LogLevel.Debug    // Debug information
LogLevel.Info     // Informational messages
LogLevel.Warn     // Warnings
LogLevel.Error    // Errors
LogLevel.Fatal    // Critical errors
LogLevel.Off      // Disable logging
LogLevel.Global   // Use global level
```

### Logging methods:
```csharp
// Basic methods
logger.Trace("Entering method GetPlayer()");
logger.Debug("Player position: {0}", playerPos);
logger.Info("Player connected successfully");
logger.Warn("Connection latency is high");
logger.Error("Failed to load game data");
logger.Fatal("Critical server error");

// Conditional logging
logger.Debug(player != null, "Player found: " + player.Name);
logger.Log(LogLevel.Info, "Custom message");
```

## LogManager

Central manager for all loggers.

### Initialization:
```csharp
// Basic initialization
LogManager.Initialize(
    new[] { LogAppenders.UnityConsoleAppender },
    LogLevel.Info
);

// Initialization with custom appenders
LogManager.Initialize(new LogHandler[] {
    LogAppenders.UnityConsoleAppender,
    CustomFileAppender,
    NetworkLogAppender
}, LogLevel.Debug);
```

### Global settings:
```csharp
// Global level (for all loggers)
LogManager.GlobalLogLevel = LogLevel.Warn;

// Forced level (overrides everything)
LogManager.LogLevel = LogLevel.Off;
```

## Logs

Static class for quick logging without creating a logger.

```csharp
// Using static methods
Logs.Info("Server started");
Logs.Error("Connection failed");
Logs.Debug("Processing player data");

// Conditional logging
Logs.Warn(healthPoints < 10, "Player health is critical");
```

## Custom appenders

### Creating a file appender:
```csharp
public static void FileAppender(Logger logger, LogLevel logLevel, object message)
{
    string logPath = Path.Combine(Application.persistentDataPath, "game.log");
    string logEntry = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] [{logLevel}] [{logger.Name}] {message}\n";
    
    File.AppendAllText(logPath, logEntry);
}

// Register the appender
LogManager.AddAppender(FileAppender);
```

## Usage examples

### Module logging:
```csharp
public class NetworkManager : MonoBehaviour
{
    private Logger logger;
    
    void Awake()
    {
        logger = Mst.Create.Logger("Network");
        logger.LogLevel = LogLevel.Debug;
    }
    
    public async Task ConnectToServer(string ip, int port)
    {
        logger.Info($"Attempting connection to {ip}:{port}");
        
        try
        {
            logger.Debug("Creating socket...");
            // Connection code
            
            logger.Info("Successfully connected to server");
        }
        catch (Exception ex)
        {
            logger.Error($"Connection failed: {ex.Message}");
            throw;
        }
    }
}
```

## Configuration via arguments

```csharp
// On application start
void ConfigureLogging()
{
    // Get level from arguments
    string logLevelArg = Mst.Args.AsString(Mst.Args.Names.LogLevel, "Info");
    LogLevel level = (LogLevel)Enum.Parse(typeof(LogLevel), logLevelArg);
    
    LogManager.GlobalLogLevel = level;
    
    // Configure file output (if specified)
    if (Mst.Args.AsBool(Mst.Args.Names.EnableFileLog, false))
    {
        LogManager.AddAppender(FileAppender);
    }
}

// Launch example
// ./Game.exe -logLevel Debug -enableFileLog true
```

## Recommendations

1. **Naming Loggers**: Use Hierarchical Names
```csharp
Logger("Network.Client")
Logger("Network.Server")
Logger("Game.Player")
Logger("Game.UI")
```

2. **Levels in production**:
- Server: Info and above
- Client: Warn and above
- Development: Debug

3. **Производительность**:
```csharp
// Bad - always creates a string
logger.Debug($"Processing {listItems.Count} items");

// Okay - checks the level first
if (logger.IsLogging(LogLevel.Debug))
{
    logger.Debug($"Processing {listItems.Count} items");
}
```

4. **Message structure**:
```csharp
// Последовательный формат
logger.Info("Player [P12345] joined room [R67890] at position (10, 20, 30)");
```

5. **Confidentiality**:
```csharp
// Avoid logging sensitive data
logger.Info($"User logged in: {user.Id}"); // ✓
logger.Debug($"Password hash: {password.Substring(0, 8)}***"); // ✓
logger.Error($"Login failed for: {user.Password}"); // ✗
```

## Integration with MST

```csharp
// Use with modules
public class MyModule : BaseServerModule
{
    protected override void Initialize()
    {
        Logger.Info("Module initializing...");
        
        // Подписка на события
        Mst.Events.AddListener("playerConnected", msg => {
            Logger.Debug($"Player connected event: {msg}");
        });
    }
    
    protected override void OnDestroy()
    {
        Logger.Info("Module shutting down...");
        base.OnDestroy();
    }
}
```
