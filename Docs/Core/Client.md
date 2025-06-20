# Master Server Toolkit - Client

## Description
A client system for connecting to the Master Server. It consists of basic classes and connection helpers.

## BaseClientBehaviour

Base class for creating client components.

### Main properties:
```csharp
// Current connection
public IClientSocket Connection { get; protected set; }

// Check connection
public bool IsConnected => Connection != null && Connection.IsConnected;

// Module logger
public Logger Logger { get; set; }
```

### Usage example:
```csharp
public class MyClientModule : BaseClientBehaviour
{
    protected override void OnInitialize()
    {
        // Initialization on start
        Logger.Info("Module started");
    }
    
    protected override void OnConnectionStatusChanged(ConnectionStatus status)
    {
        Logger.Info($"Connection status: {status}");
    }
}
```

### Основные методы:
```csharp
// Register a message handler
RegisterMessageHandler(IPacketHandler handler);
RegisterMessageHandler(ushort opCode, IncommingMessageHandler handler);

// Change connection
ChangeConnection(IClientSocket connection, bool clearHandlers = false);

// Clear connection
ClearConnection(bool clearHandlers = true);
```

## BaseClientModule

Base class for client modules.

### Пример:
```csharp
public class GameStatisticsModule : BaseClientModule
{
    public override void OnInitialize(BaseClientBehaviour parentBehaviour)
    {
        base.OnInitialize(parentBehaviour);
        
        // Register handlers
        parentBehaviour.RegisterMessageHandler(OpCodes.Statistics, HandleStatistics);
    }
    
    private void HandleStatistics(IIncommingMessage message)
    {
        // Handle statistics
    }
}
```

## ClientToMasterConnector

Component for automatic connection to the Master Server.

### Setup:
```csharp
// Add to a GameObject
var connector = GetComponent<ClientToMasterConnector>();

// Configure via Inspector or code
connector.serverIp = "192.168.1.100";
connector.serverPort = 5000;
connector.connectOnStart = true;
```

### Аргументы командной строки:
```bash
# Automatic IP and port configuration
./Client.exe -masterip 192.168.1.100 -masterport 5000
```

## ConnectionHelper

Base helper for creating connections with automatic attempts.

### Main settings:
```csharp
// Number of connection attempts
[SerializeField] protected int maxAttemptsToConnect = 5;

// Connection timeout
[SerializeField] protected float timeout = 5f;

// Auto connect
[SerializeField] protected bool connectOnStart = true;

// Secure connection
[SerializeField] protected bool useSecure = false;
```

### Events:
```csharp
// Successful connection
OnConnectedEvent

// Failed connection
OnFailedConnectEvent

// Disconnection
OnDisconnectedEvent
```

### Example of a custom connector:
```csharp
public class MyCustomConnector : ConnectionHelper<MyCustomConnector>
{
    protected override void Start()
    {
        // Custom logic before connecting
        base.Start();
    }
    
    protected override void OnConnectedEventHandler(IClientSocket client)
    {
        base.OnConnectedEventHandler(client);
        // Additional logic after connection
    }
}
```

## Module architecture

### Module hierarchy:
1. BaseClientBehaviour - main component
2. BaseClientModule - child modules
3. Modules are initialized automatically on start

### Example structure:
```
GameObject
├── MyClientBehaviour (BaseClientBehaviour)
└── ChildModules
    ├── AuthModule (BaseClientModule)
    ├── ChatModule (BaseClientModule)
    └── ProfileModule (BaseClientModule)
```

## Best practices
1. Use ConnectionHelper to manage the connection
2. Inherit from BaseClientBehaviour for main components
3. Create specialized modules via BaseClientModule
4. Register handlers in OnInitialize
5. Clean up resources in OnDestroy
