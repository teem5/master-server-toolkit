# Master Server Toolkit - Rooms

## 설명
게임 룸에 대한 등록, 관리 및 제공 모듈.공개 및 개인 실을 만들고 매개 변수를 관리하고 플레이어 액세스를 제어 할 수 있습니다.

## RoomsModule

마스터 서버의 객실 관리를위한 메인 클래스.

### 설정 :
```csharp
[Header("Permissions")]
[SerializeField] protected int registerRoomPermissionLevel = 0;
```

## 방의 등록

### 게임 서버 :
```csharp
// Создание опций комнаты
var options = new RoomOptions
{
    Name = "Epic Battle Arena",
    RoomIp = "192.168.1.100",
    RoomPort = 7777,
    MaxConnections = 16,
    IsPublic = true,
    Password = "", // пустой для публичной комнаты
    Region = "RU",
    CustomOptions = new MstProperties()
};

// Добавляем кастомные свойства
options.CustomOptions.Set("map", "forest_arena");
options.CustomOptions.Set("gameMode", "battle_royale");
options.CustomOptions.Set("difficulty", "hard");

// Регистрация на master server
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

### 자동 등록 :
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

## 객실 관리

### 매개 변수 업데이트 :
```csharp
// Изменение опций комнаты
var newOptions = room.Options;
newOptions.MaxConnections = 20;
newOptions.CustomOptions.Set("gameState", "playing");

Mst.Server.Rooms.SaveRoomOptions(room.RoomId, newOptions);

// Управление игроками
room.AddPlayer(peerId, peer);
room.RemovePlayer(peerId);

// Получение списка игроков
var players = room.Players.Values;
```

### 방의 파괴 :
```csharp
// При завершении игры
Mst.Server.Rooms.DestroyRoom(room.RoomId, (successful, error) =>
{
    if (successful)
    {
        Debug.Log("Room destroyed successfully");
    }
});

// Автоматическое уничтожение при отключении
void OnApplicationQuit()
{
    if (registeredRoom != null)
    {
        Mst.Server.Rooms.DestroyRoom(registeredRoom.RoomId);
    }
}
```

## 방에 연결

## 검색 및 연결 :
```csharp
// Поиск публичных комнат
Mst.Client.Matchmaker.FindGames((games) =>
{
    foreach (var game in games)
    {
        Debug.Log($"Room: {game.Name}, Players: {game.OnlinePlayers}/{game.MaxPlayers}");
    }
});

// Подключение к конкретной комнате
var roomId = 12345;
Mst.Client.Rooms.GetAccess(roomId, "", (access, error) =>
{
    if (access != null)
    {
        ConnectToGameServer(access.RoomIp, access.RoomPort, access.Token);
    }
});
```

### 비밀번호 연결 :
```csharp
// Подключение к приватной комнате
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

## 게임 서버와 통합

### RoomServerManager:
```csharp
public class RoomServerManager : MonoBehaviour
{
    [Header("Refs")]
    public NetworkManager networkManager;
    
    private RegisteredRoom currentRoom;
    
    void Start()
    {
        // Запуск сервера
        networkManager.StartServer();
        
        // Регистрация комнаты
        RegisterRoom();
    }
    
    public override void OnServerConnect(NetworkConnectionToClient conn)
    {
        // Проверка токена доступа
        ValidatePlayerAccess(conn);
    }
    
    private void ValidatePlayerAccess(NetworkConnectionToClient conn)
    {
        // Получение токена от клиента
        string accessToken = GetTokenFromClient(conn);
        
        // Проверка на master server
        Mst.Server.Rooms.ValidateAccess(currentRoom.RoomId, accessToken, (userData, error) =>
        {
            if (userData != null)
            {
                // Доступ разрешен
                acceptedPlayers[conn.connectionId] = userData;
                Debug.Log($"Player {userData.Username} validated");
            }
            else
            {
                // Отклонить подключение
                conn.Disconnect();
            }
        });
    }
}
```

## 필터링 룸

### 기준 별 검색 :
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

## 카스트 필터링 :
```csharp
// На стороне сервера - переопределение GetPublicRoomOptions
public override MstProperties GetPublicRoomOptions(IPeer player, RegisteredRoom room, MstProperties playerFilters)
{
    var roomData = base.GetPublicRoomOptions(player, room, playerFilters);
    
    // Добавляем дополнительную информацию
    roomData.Set("serverVersion", "1.2.3");
    roomData.Set("ping", CalculatePing(player, room));
    
    // Скрываем некоторые данные
    if (!IsAdminPlayer(player))
    {
        roomData.Remove("adminPort");
    }
    
    return roomData;
}
```

## 방의 이벤트

### 이벤트 구독 :
```csharp
// На мастер сервере
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

// В игровой комнате
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

## 객실 목록의 경우 UI

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

## 모범 사례

1. ** 게임 서버에서 항상 액세스 토큰을 확인하십시오 **
2. ** 유연한 객실 설정에 맞춤 속성 사용 **
3. ** 서버 시작 후 객실 등록 **
4. ** 플레이어 수를 실시간으로 업데이트하십시오.
5. ** 청소 자원 ** 방을 파괴 할 때
6. ** 개인 실에 비밀번호 **를 사용하십시오
7. ** 객실 상태 모니터 ** 리소스 최적화
8. ** 필터 사용 ** 객실에 대한 최고의 UX 검색을 위해
