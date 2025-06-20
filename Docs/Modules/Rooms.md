# Master Server Toolkit - Rooms

## Description
Module for registering, managing and providing access to game rooms. It allows creation of public and private rooms, adjustment of their parameters and control of player access.

## RoomsModule

Main class for managing rooms on the master server.

### Setup:
```csharp
[Header("Permissions")]
[SerializeField] protected int registerRoomPermissionLevel = 0;
```

## Room registration

### From the game server:
```csharp
// Create room options
var options = new RoomOptions
{
    Name = "Epic Battle Arena",
    RoomIp = "192.168.1.100",
    RoomPort = 7777,
    MaxConnections = 16,
    IsPublic = true,
    Password = "", // empty for a public room
    Region = "RU",
    CustomOptions = new MstProperties()
};

// Add custom properties
options.CustomOptions.Set("map", "forest_arena");
options.CustomOptions.Set("gameMode", "battle_royale");
options.CustomOptions.Set("difficulty", "hard");

// Register on the master server
Mst.Server.Rooms.RegisterRoom(options, (room, error) =>
{
    if (room != null)
    {
        Debug.Log($"Room registered with ID: {room.RoomId}");
    }
    else
    {
        Debug.LogError($"Failed to register room: {error}");
    }
});
```

### Автоматическая регистрация:
```csharp
public class GameServerManager : MonoBehaviour
{
    [Header("Room Settings")]
    public string roomName = "Game Room";
    public int maxPlayers = 10;
    public bool isPublic = true;
    
    void Start()
    {
        RegisterGameRoom();
    }
    
    private void RegisterGameRoom()
    {
        var options = new RoomOptions
        {
            Name = roomName,
            RoomIp = GetServerIp(),
            RoomPort = NetworkManager.singleton.networkPort,
            MaxConnections = maxPlayers,
            IsPublic = isPublic
        };
        
        Mst.Server.Rooms.RegisterRoom(options);
    }
}
```

## Управление комнатой

### Обновление параметров:
```csharp
// Modify room options
var newOptions = room.Options;
newOptions.MaxConnections = 20;
newOptions.CustomOptions.Set("gameState", "playing");

Mst.Server.Rooms.SaveRoomOptions(room.RoomId, newOptions);

// Manage players
room.AddPlayer(peerId, peer);
room.RemovePlayer(peerId);

// Get the player list
var players = room.Players.Values;
```

### Уничтожение комнаты:
```csharp
// When the game ends
Mst.Server.Rooms.DestroyRoom(room.RoomId, (successful, error) =>
{
    if (successful)
    {
        Debug.Log("Room destroyed successfully");
    }
});

// Automatic destruction on shutdown
void OnApplicationQuit()
{
    if (registeredRoom != null)
    {
        Mst.Server.Rooms.DestroyRoom(registeredRoom.RoomId);
    }
}
```

## Подключение к комнате

### Поиск и подключение:
```csharp
// Search for public rooms
Mst.Client.Matchmaker.FindGames((games) =>
{
    foreach (var game in games)
    {
        Debug.Log($"Room: {game.Name}, Players: {game.OnlinePlayers}/{game.MaxPlayers}");
    }
});

// Connect to a specific room
var roomId = 12345;
Mst.Client.Rooms.GetAccess(roomId, "", (access, error) =>
{
    if (access != null)
    {
        ConnectToGameServer(access.RoomIp, access.RoomPort, access.Token);
    }
});
```

### Подключение с паролем:
```csharp
// Connect to a private room
Mst.Client.Rooms.GetAccess(roomId, "secret_password", (access, error) =>
{
    if (access != null)
    {
        Debug.Log("Access granted!");
        JoinGameServer(access.RoomIp, access.RoomPort, access.Token);
    }
    else
    {
        Debug.LogError("Invalid password");
    }
});
```

## Integration with the game server

### RoomServerManager:
```csharp
public class RoomServerManager : MonoBehaviour
{
    [Header("Refs")]
    public NetworkManager networkManager;
    
    private RegisteredRoom currentRoom;
    
    void Start()
    {
        // Start the server
        networkManager.StartServer();
        
        // Register the room
        RegisterRoom();
    }
    
    public override void OnServerConnect(NetworkConnectionToClient conn)
    {
        // Validate access token
        ValidatePlayerAccess(conn);
    }
    
    private void ValidatePlayerAccess(NetworkConnectionToClient conn)
    {
        // Get token from client
        string accessToken = GetTokenFromClient(conn);
        
        // Check with the master server
        Mst.Server.Rooms.ValidateAccess(currentRoom.RoomId, accessToken, (userData, error) =>
        {
            if (userData != null)
            {
                // Access granted
                acceptedPlayers[conn.connectionId] = userData;
                Debug.Log($"Player {userData.Username} validated");
            }
            else
            {
                // Reject connection
                conn.Disconnect();
            }
        });
    }
}
```

## Фильтрация комнат

### Поиск по критериям:
```csharp
// Создание фильтров
var filters = new MstProperties();
filters.Set("map", "forest");
filters.Set("gameMode", "pvp");
filters.Set("maxPlayers", 10);

// Поиск комнат с фильтрами
Mst.Client.Matchmaker.FindGames(filters, (games) =>
{
    var filteredRooms = games.Where(g => 
        g.Type == GameInfoType.Room &&
        g.OnlinePlayers < g.MaxPlayers &&
        !g.IsPasswordProtected
    );
});
```

### Custom filtering:
```csharp
// На стороне сервера - переопределение GetPublicRoomOptions
public override MstProperties GetPublicRoomOptions(IPeer player, RegisteredRoom room, MstProperties playerFilters)
{
    var roomData = base.GetPublicRoomOptions(player, room, playerFilters);
    
    // Add additional information
    roomData.Set("serverVersion", "1.2.3");
    roomData.Set("ping", CalculatePing(player, room));
    
    // Hide some data
    if (!IsAdminPlayer(player))
    {
        roomData.Remove("adminPort");
    }
    
    return roomData;
}
```

## События комнат

### Subscribing to events:
```csharp
// On the master server
roomsModule.OnRoomRegisteredEvent += (room) =>
{
    Debug.Log($"New room registered: {room.Options.Name}");
    NotifyMatchmakingSystem(room);
};

roomsModule.OnRoomDestroyedEvent += (room) =>
{
    Debug.Log($"Room destroyed: {room.RoomId}");
    CleanupResources(room);
};

// In the game room
room.OnPlayerJoinedEvent += (player) =>
{
    SendWelcomeMessage(player);
    UpdatePlayerCount();
};

room.OnPlayerLeftEvent += (player) =>
{
    SavePlayerProgress(player);
    CheckIfRoomShouldClose();
};
```

## UI для списка комнат

```csharp
public class RoomListUI : MonoBehaviour
{
    [Header("UI")]
    public GameObject roomItemPrefab;
    public Transform roomList;
    public Button refreshButton;
    
    void Start()
    {
        refreshButton.onClick.AddListener(RefreshRoomList);
        RefreshRoomList();
    }
    
    void RefreshRoomList()
    {
        // Очистка списка
        foreach (Transform child in roomList)
        {
            Destroy(child.gameObject);
        }
        
        // Получение комнат
        Mst.Client.Matchmaker.FindGames((games) =>
        {
            foreach (var game in games)
            {
                if (game.Type == GameInfoType.Room)
                {
                    CreateRoomItem(game);
                }
            }
        });
    }
    
    void CreateRoomItem(GameInfoPacket room)
    {
        var item = Instantiate(roomItemPrefab, roomList);
        var roomUI = item.GetComponent<RoomItem>();
        
        roomUI.Setup(room);
        roomUI.OnJoinClicked = () => JoinRoom(room.Id);
    }
    
    void JoinRoom(int roomId)
    {
        Mst.Client.Rooms.GetAccess(roomId, (access, error) =>
        {
            if (access != null)
            {
                NetworkManager.singleton.networkAddress = access.RoomIp;
                NetworkManager.singleton.StartClient();
            }
        });
    }
}
```

## Best Practices

1. **Always validate access tokens** on the game server
2. **Use custom properties** for flexible room configuration
3. **Register rooms after the server starts**
4. **Update player counts** in real time
5. **Clean up resources** when destroying a room
6. **Use passwords** for private rooms
7. **Monitor room status** to optimize resources
8. **Apply filters** for a better room search UX
