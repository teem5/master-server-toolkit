# Master Server Toolkit - World Rooms

## Description
The WorldRooms module extends the basic Rooms module to create persistent game areas (locations) in an open world with automatic launching and management.

## Main Components

### WorldRoomsModule
```csharp
// Settings
[Header("Zones Settings"), SerializeField]
private string[] zoneScenes; // Zone scenes that will be started automatically

// Dependencies
protected SpawnersModule spawnersModule; // For launching zone servers
```

## Automatic World Zone Creation

```csharp
// Automatically launch all zones when a spawner registers
private async void Spawners_OnSpawnerRegisteredEvent(RegisteredSpawner spawner)
{
    await Task.Delay(100);

    foreach (string zoneScene in zoneScenes)
    {
        spawnersModule.Spawn(SpawnerProperties(zoneScene)).WhenDone(task =>
        {
            logger.Info($"{zoneScene} zone status is: {task.Status}");
        });
    }
}
```

## Configuring Zone Properties

```csharp
// Create settings for launching a zone on the spawner
protected virtual MstProperties SpawnerProperties(string zoneId)
{
    var properties = new MstProperties();
    properties.Set(Mst.Args.Names.RoomName, zoneId);        // Room name
    properties.Set(Mst.Args.Names.RoomOnlineScene, zoneId); // Scene name
    properties.Set(Mst.Args.Names.RoomIsPrivate, true);     // Private room
    properties.Set(MstDictKeys.WORLD_ZONE, zoneId);         // World zone marker
    
    return properties;
}
```

## Retrieving Zone Information

```csharp
// On the server - find zone room by ID
RegisteredRoom zoneRoom = roomsList.Values
    .Where(r => r.Options.CustomOptions.AsString(MstDictKeys.WORLD_ZONE) == zoneId)
    .FirstOrDefault();

// Prepare zone information to send to the client
var game = new GameInfoPacket
{
    Id = zoneRoom.RoomId,
    Address = zoneRoom.Options.RoomIp + ":" + zoneRoom.Options.RoomPort,
    MaxPlayers = zoneRoom.Options.MaxConnections,
    Name = zoneRoom.Options.Name,
    OnlinePlayers = zoneRoom.OnlineCount,
    Properties = GetPublicRoomOptions(message.Peer, zoneRoom, null),
    IsPasswordProtected = !string.IsNullOrEmpty(zoneRoom.Options.Password),
    Type = GameInfoType.Room,
    Region = zoneRoom.Options.Region
};
```

## Client Zone Info Request

```csharp
// Request zone info by its ID
Mst.Client.Connection.SendMessage(MstOpCodes.GetZoneRoomInfo, "Forest", (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        // Retrieve zone data
        var zoneInfo = response.AsPacket<GameInfoPacket>();
        
        // Connect to the zone server
        string address = zoneInfo.Address;
        int roomId = zoneInfo.Id;
        
        ConnectToZone(address, roomId);
    }
    else
    {
        Debug.LogError("Failed to get zone room info");
    }
});
```

## Switching Between Zones

```csharp
// Client code for switching between zones
public void RequestZoneTransition(string targetZoneId, Vector3 spawnPosition)
{
    // 1. Request target zone information
    Mst.Client.Connection.SendMessage(MstOpCodes.GetZoneRoomInfo, targetZoneId, (status, response) =>
    {
        if (status == ResponseStatus.Success)
        {
            // 2. Get zone information
            var zoneInfo = response.AsPacket<GameInfoPacket>();
            
            // 3. Save spawn position for the new zone
            PlayerPrefs.SetFloat("SpawnPosX", spawnPosition.x);
            PlayerPrefs.SetFloat("SpawnPosY", spawnPosition.y);
            PlayerPrefs.SetFloat("SpawnPosZ", spawnPosition.z);
            
            // 4. Disconnect from current zone
            Mst.Client.Connection.Disconnect();
            
            // 5. Connect to the new zone
            ConnectToZone(zoneInfo.Address, zoneInfo.Id);
        }
    });
}

// Connect to the zone server
private void ConnectToZone(string address, int roomId)
{
    // Parse address in "ip:port" format
    string[] addressParts = address.Split(':');
    string ip = addressParts[0];
    int port = int.Parse(addressParts[1]);
    
    // Connect to the server
    Mst.Client.Connection.Connect(ip, port, (successful, connectedPeer) =>
    {
        if (successful)
        {
            Debug.Log($"Connected to zone server: {address}");
            
            // Join the room after connecting to the server
            Mst.Client.Rooms.JoinRoom(roomId, "", (successfulJoin, roomAccess) =>
            {
                if (successfulJoin)
                {
                    Debug.Log($"Joined zone room: {roomId}");
                }
            });
        }
    });
}
```

## Persistent World Integration

```csharp
// Example zone server code for handling player connection
protected override void OnPlayerJoinedRoom(RoomPlayer player)
{
    base.OnPlayerJoinedRoom(player);
    
    // Get user extension
    var userExt = player.GetExtension<IUserPeerExtension>();
    if (userExt != null)
    {
        // Load player data for the current zone
        LoadPlayerZoneData(userExt.UserId);
        
        // Send data about other players in the zone
        SendZonePlayersData(player);
    }
}

// Load player data for a specific zone
private void LoadPlayerZoneData(string userId)
{
    // Retrieve data from profile or database
    var zoneId = gameObject.scene.name;
    var zoneDataKey = $"zonedata_{zoneId}_{userId}";
    
    // Request data from profile
    Mst.Server.Profiles.GetProfileValues(userId, new string[] { zoneDataKey }, (success, data) =>
    {
        if (success && data.Has(zoneDataKey))
        {
            // Parse zone data (position, inventory, etc.)
            var zoneData = data.AsString(zoneDataKey);
            // Apply data to player
            ApplyZoneDataToPlayer(userId, zoneData);
        }
        else
        {
            // Use default data for new players in this zone
            ApplyDefaultZoneData(userId);
        }
    });
}
```

## Saving Data When Leaving a Zone

```csharp
// Zone server: handle player leaving the zone
protected override void OnPlayerLeftRoom(RoomPlayer player)
{
    base.OnPlayerLeftRoom(player);
    
    // Get user extension
    var userExt = player.GetExtension<IUserPeerExtension>();
    if (userExt != null)
    {
        // Save player data for the current zone
        SavePlayerZoneData(userExt.UserId);
    }
}

// Save player data for a specific zone
private void SavePlayerZoneData(string userId)
{
    // Gather player data (position, inventory, etc.)
    string zoneData = GenerateZoneDataForPlayer(userId);
    
    // Form the zone key
    var zoneId = gameObject.scene.name;
    var zoneDataKey = $"zonedata_{zoneId}_{userId}";
    
    // Save to profile
    var data = new MstProperties();
    data.Set(zoneDataKey, zoneData);
    
    Mst.Server.Profiles.SetProfileValues(userId, data);
}
```

## Usage Examples

### Setting Up Zone Transitions
```csharp
// The "Forest" zone connects to the "Cave" zone
public class ZoneTransition : MonoBehaviour
{
    [SerializeField] private string targetZoneId = "Cave";
    [SerializeField] private Vector3 spawnPosition = new Vector3(10, 0, 10);
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            // Send a request to move to another zone
            var zoneManager = FindObjectOfType<WorldZoneManager>();
            zoneManager.RequestZoneTransition(targetZoneId, spawnPosition);
        }
    }
}
```

### Global Zone Events
```csharp
// Global event system for synchronizing between zones
public class GlobalEventsManager : MonoBehaviour
{
    // Send a global event to all zones
    public void BroadcastGlobalEvent(string eventType, MstProperties eventData)
    {
        // Send the event to the master server
        var data = new MstProperties();
        data.Set("type", eventType);
        data.Set("data", eventData.ToJson());
        
        Mst.Client.Connection.SendMessage(MstOpCodes.GlobalZoneEvent, data);
    }
}

// On the master server
private Task GlobalZoneEventHandler(IIncomingMessage message)
{
    try
    {
        var data = MstProperties.FromBytes(message.AsBytes());
        string eventType = data.AsString("type");
        string eventData = data.AsString("data");
        
        // Relay to all zones
        foreach (var room in roomsList.Values)
        {
            if (room.Options.CustomOptions.Has(MstDictKeys.WORLD_ZONE))
            {
                room.SendMessage(MstOpCodes.GlobalZoneEvent, data.ToBytes());
            }
        }
        
        return Task.CompletedTask;
    }
    catch (System.Exception ex)
    {
        return Task.FromException(ex);
    }
}
```

## Best Practices

1. **Use zone separation** to reduce server load
2. **Save data** when moving between zones
3. **Optimize data transfer** – send only what is necessary
4. **Implement smooth transitions** between zones with loading screens
5. **Use a shared event system** for synchronization between zones
6. **Split responsibilities** between the zone server and the master server
