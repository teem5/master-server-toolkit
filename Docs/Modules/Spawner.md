# Master Server Toolkit - Spawner

## Description
Module for managing game server launch processes in different regions with load balancing and queue support.

## Main Components

### SpawnersModule
```csharp
// Settings
[SerializeField] protected int createSpawnerPermissionLevel = 0; // Minimum permission level to register a spawner
[SerializeField] protected float queueUpdateFrequency = 0.1f;    // Queue update frequency
[SerializeField] protected bool enableClientSpawnRequests = true; // Allow spawn requests from clients

// Events
public event Action<RegisteredSpawner> OnSpawnerRegisteredEvent;  // When a spawner registers
public event Action<RegisteredSpawner> OnSpawnerDestroyedEvent;   // When a spawner is removed
public event SpawnedProcessRegistrationHandler OnSpawnedProcessRegisteredEvent; // When a process registers
```

### RegisteredSpawner
```csharp
// Main properties
public int SpawnerId { get; }                // Spawner unique ID
public IPeer Peer { get; }                   // Connection to the spawner
public SpawnerOptions Options { get; }       // Spawner settings
public int ProcessesRunning { get; }         // Number of running processes
public int QueuedTasks { get; }              // Number of tasks in queue

// Methods
public bool CanSpawnAnotherProcess();        // Can it spawn one more process
public int CalculateFreeSlotsCount();        // Calculate free slot count
```

### SpawnerOptions
```csharp
// Spawner options
public string MachineIp { get; set; }           // Spawner machine IP
public int MaxProcesses { get; set; } = 5;      // Maximum number of processes
public string Region { get; set; } = "Global";  // Spawner region
public Dictionary<string, string> CustomOptions { get; set; } // Additional options
```

### SpawnTask
```csharp
// Properties
public int Id { get; }                      // Task ID
public RegisteredSpawner Spawner { get; }   // Spawner executing the task
public MstProperties Options { get; }       // Spawn options
public SpawnStatus Status { get; }          // Current status
public string UniqueCode { get; }           // Unique security code
public IPeer Requester { get; set; }        // Requesting client
public IPeer RegisteredPeer { get; private set; } // Registered process

// Events
public event Action<SpawnStatus> OnStatusChangedEvent; // When status changes
```

## SpawnStatus values
```csharp
public enum SpawnStatus
{
    None,               // Initial state
    Queued,             // In queue
    ProcessStarted,     // Process started
    ProcessRegistered,  // Process registered
    Finalized,          // Finalized
    Aborted,            // Aborted
    Killed              // Killed
}
```

## Workflow

### Spawner registration
```csharp
// Client
var options = new SpawnerOptions
{
    MachineIp = "192.168.1.10",
    MaxProcesses = 10,
    Region = "eu-west",
    CustomOptions = new Dictionary<string, string> {
        { "mapsList", "map1,map2,map3" }
    }
};

Mst.Client.Spawners.RegisterSpawner(options, (successful, spawnerId) =>
{
    if (successful)
        Debug.Log($"Spawner registered with ID: {spawnerId}");
});

// Server
RegisteredSpawner spawner = CreateSpawner(peer, options);
```

### Spawn game server request
```csharp
// Client
var spawnOptions = new MstProperties();
spawnOptions.Set(Mst.Args.Names.RoomName, "MyGame");
spawnOptions.Set(Mst.Args.Names.RoomMaxPlayers, 10);
spawnOptions.Set(Mst.Args.Names.RoomRegion, "eu-west");

Mst.Client.Spawners.RequestSpawn(spawnOptions, (successful, spawnId) =>
{
    if (successful)
    {
        Debug.Log($"Spawn request created with ID: {spawnId}");
        
        // Subscribe to status changes
        Mst.Client.Spawners.OnStatusChangedEvent += OnSpawnStatusChanged;
    }
});

// Server
SpawnTask task = Spawn(options, region);
```

### Selecting a spawner to launch
```csharp
// Filter spawners by region
var spawners = GetSpawnersInRegion(region);

// Sort by available slots
var availableSpawners = spawners
    .OrderByDescending(s => s.CalculateFreeSlotsCount())
    .Where(s => s.CanSpawnAnotherProcess())
    .ToList();

// Choose spawner
if (availableSpawners.Count > 0)
{
    return availableSpawners[0];
}
```

### Finalizing the spawn process
```csharp
// On the process side
var finalizationData = new MstProperties();
finalizationData.Set("roomId", 12345);
finalizationData.Set("connectionAddress", "192.168.1.10:12345");

// Send finalization data
Mst.Client.Spawners.FinalizeSpawn(spawnTaskId, finalizationData);

// On the client side
Mst.Client.Spawners.GetFinalizationData(spawnId, (successful, data) =>
{
    if (successful)
    {
        string connectionAddress = data.AsString("connectionAddress");
        int roomId = data.AsInt("roomId");
        
        // Connect to the room
        ConnectToRoom(connectionAddress, roomId);
    }
});
```

## Управление регионами

```csharp
// Get all regions
List<RegionInfo> regions = spawnersModule.GetRegions();

// Get spawners in a specific region
List<RegisteredSpawner> regionalSpawners = spawnersModule.GetSpawnersInRegion("eu-west");

// Create a spawner in a region
var options = new SpawnerOptions { Region = "eu-west" };
var spawner = spawnersModule.CreateSpawner(peer, options);
```

## Балансировка нагрузки

```csharp
// Example of balancing by least loaded servers
public RegisteredSpawner GetLeastBusySpawner(string region)
{
    var spawners = GetSpawnersInRegion(region);
    
    // If there are no spawners in the region, use all available
    if (spawners.Count == 0)
        spawners = GetSpawners();
    
    // Sort by load
    return spawners
        .OrderByDescending(s => s.CalculateFreeSlotsCount())
        .FirstOrDefault(s => s.CanSpawnAnotherProcess());
}

// Расширенная логика выбора спаунера
public RegisteredSpawner GetOptimalSpawner(MstProperties options)
{
    string region = options.AsString(Mst.Args.Names.RoomRegion, "");
    string gameMode = options.AsString("gameMode", "");
    
    // Filter by region and game mode
    var filtered = spawnersList.Values
        .Where(s => (string.IsNullOrEmpty(region) || s.Options.Region == region) &&
                  (string.IsNullOrEmpty(gameMode) || 
                   s.Options.CustomOptions.ContainsKey("gameModes") && 
                   s.Options.CustomOptions["gameModes"].Contains(gameMode)))
        .ToList();
    
    return filtered
        .OrderByDescending(s => s.CalculateFreeSlotsCount())
        .FirstOrDefault(s => s.CanSpawnAnotherProcess());
}
```

## Практический пример

### System setup:
```csharp
// 1. Register spawners
RegisterSpawner("EU", "10.0.0.1", 10);
RegisterSpawner("US", "10.0.1.1", 15);
RegisterSpawner("ASIA", "10.0.2.1", 8);

// 2. Client request to create a game server
var options = new MstProperties();
options.Set(Mst.Args.Names.RoomName, "CustomGame");
options.Set(Mst.Args.Names.RoomMaxPlayers, 16);
options.Set(Mst.Args.Names.RoomRegion, "EU");
options.Set("gameMode", "deathmatch");

// 3. Handle request, choose spawner and start process
SpawnTask task = spawnersModule.Spawn(options, "EU");

// 4. Client waits for process completion
// 5. Spawned process registers and sends connection data
```

## Best Practices

1. **Group spawners by region** for optimal latency
2. **Configure process limits** for each spawner based on server capacity
3. **Use custom options** for flexible spawner setup
4. **Implement fault tolerance** – redirect tasks to another region if a spawner is unavailable
5. **Monitor spawner load** to detect bottlenecks
6. **Add timeouts to tasks** in the queue to avoid stalling
