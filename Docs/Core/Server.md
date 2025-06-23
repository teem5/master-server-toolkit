# Master Server Toolkit - Server

## Description
Basic infrastructure for creating server applications. Includes ServerBehaviour for the network server and BaseServerModule for modules.

## ServerBehaviour

Base class for creating server applications with automatic connection and module management.

### Main properties:
```csharp
[Header("Server Settings")]
public string serverIp = "localhost";
public int serverPort = 5000;
public ushort maxConnections = 0;
public string service = "mst";

[Header("Security Settings")]
public bool useSecure = false;
public string password = "mst";
```

### Server management:
```csharp
// Start server
server.StartServer();
server.StartServer(8080);
server.StartServer("192.168.1.100", 8080);

// Stop server
server.StopServer();

// Check status
bool isRunning = server.IsRunning;
int connectedClients = server.PeersCount;
```

### Server events:
```csharp
// Client connection
server.OnPeerConnectedEvent += (peer) => {
    Debug.Log($"Client {peer.Id} connected");
};

// Client disconnection
server.OnPeerDisconnectedEvent += (peer) => {
    Debug.Log($"Client {peer.Id} disconnected");
};

// Server start/stop
server.OnServerStartedEvent += () => Debug.Log("Server started");
server.OnServerStoppedEvent += () => Debug.Log("Server stopped");
```

### Registering handlers:
```csharp
// Register a message handler
server.RegisterMessageHandler(MstOpCodes.SignIn, HandleSignIn);

// Asynchronous handler
server.RegisterMessageHandler(CustomOpCodes.GetData, async (message) => {
    var data = await LoadDataAsync();
    message.Respond(data, ResponseStatus.Success);
});
```

## BaseServerModule

Base class for creating modular server components.

### Creating a module:
```csharp
public class AccountsModule : BaseServerModule
{
    protected override void Awake()
    {
        base.Awake();
        // Add dependencies
        AddDependency<DatabaseModule>();
        AddOptionalDependency<EmailModule>();
    }
    
    public override void Initialize(IServer server)
    {
        // Register handlers
        server.RegisterMessageHandler(MstOpCodes.SignIn, HandleSignIn);
        server.RegisterMessageHandler(MstOpCodes.SignUp, HandleSignUp);
    }
    
    private void HandleSignIn(IIncomingMessage message)
    {
        // Sign-in logic
    }
}
```

### Module dependencies:
```csharp
// Required dependencies
AddDependency<DatabaseModule>();
AddDependency<PermissionsModule>();

// Optional dependencies
AddOptionalDependency<EmailModule>();
AddOptionalDependency<AnalyticsModule>();

// Get other modules
var dbModule = Server.GetModule<DatabaseModule>();
var emailModule = Server.GetModule<EmailModule>();
```

## Advanced features

### Custom ServerBehaviour:
```csharp
public class GameServerBehaviour : ServerBehaviour
{
    protected override void OnPeerConnected(IPeer peer)
    {
        base.OnPeerConnected(peer);
        // Custom logic for a new player
        NotifyOtherPlayers(peer);
    }
    
    protected override void ValidateConnection(ProvideServerAccessCheckPacket packet, SuccessCallback callback)
    {
        // Custom connection validation
        if (CheckServerPassword(packet.Password) && CheckGameVersion(packet.Version))
        {
            callback.Invoke(true, string.Empty);
        }
        else
        {
            callback.Invoke(false, "Access denied");
        }
    }
}
```

### Statistics and monitoring:
```csharp
// Get server information
MstProperties info = server.Info();
MstJson jsonInfo = server.JsonInfo();

// Connection statistics
Debug.Log($"Active clients: {server.PeersCount}");
Debug.Log($"Total clients: {server.Info().Get("Total clients")}");
Debug.Log($"Highest clients: {server.Info().Get("Highest clients")}");
```

## Command line arguments

```bash
# Basic parameters
./Server.exe -masterip 192.168.1.100 -masterport 5000

# Security
./Server.exe -mstUseSecure true -certificatePath cert.pfx -certificatePassword pass

# Performance
./Server.exe -targetFrameRate 60 -clientInactivityTimeout 30
```

### Automatic setup from arguments:
```csharp
// Address and port
serverIp = Mst.Args.AsString(Mst.Args.Names.MasterIp, serverIp);
serverPort = Mst.Args.AsInt(Mst.Args.Names.MasterPort, serverPort);

// Security
useSecure = Mst.Args.AsBool(Mst.Args.Names.UseSecure, useSecure);
certificatePath = Mst.Args.AsString(Mst.Args.Names.CertificatePath, certificatePath);

// Timeouts
inactivityTimeout = Mst.Args.AsFloat(Mst.Args.Names.ClientInactivityTimeout, inactivityTimeout);
```

## Working with connections

### Peer management:
```csharp
// Get peer by ID
IPeer peer = server.GetPeer(peerId);

// Disconnect all clients
foreach (var peer in connectedPeers.Values)
{
    peer.Disconnect("Server maintenance");
}

// Authentication check
var securityInfo = peer.GetExtension<SecurityInfoPeerExtension>();
int permissionLevel = securityInfo.PermissionLevel;
```

### Permissions:
```csharp
[SerializeField]
private List<PermissionEntry> permissions = new List<PermissionEntry>
{
    new PermissionEntry { key = "admin", permissionLevel = 100 },
    new PermissionEntry { key = "moderator", permissionLevel = 50 }
};

// Permission check
if (peer.HasPermission(50))
{
    // Allowed for moderators and above
}
```

## Best practices

1. **Use modules for logical grouping**:
   - AuthModule - authentication
   - GameModule - game logic
   - ChatModule - chat
   - DatabaseModule - database operations

2. **Manage dependencies**:
   - Declare dependencies in Awake()
   - Use optional dependencies for additional features

3. **Handle errors**:
```csharp
protected override void OnMessageReceived(IIncomingMessage message)
{
    try
    {
        base.OnMessageReceived(message);
    }
    catch (Exception ex)
    {
        logger.Error($"Message error: {ex.Message}");
        message.Respond(ResponseStatus.Error);
    }
}
```

4. **Optimize performance**:
   - Limit server FPS
   - Adjust timeouts
   - Use the maximum number of connections

5. **Log important events**:
```csharp
logger.Info($"Server started on {Address}:{Port}");
logger.Debug($"Module {GetType().Name} initialized");
logger.Error($"Failed to handle message: {ex.Message}");
```
