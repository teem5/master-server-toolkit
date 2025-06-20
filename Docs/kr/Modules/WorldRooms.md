# Master Server Toolkit - World Rooms

## 설명
Worldroom 모듈은 기본 룸 모듈의 기능을 확장하여 오픈 월드에서 지속적인 게임 영역 (위치)을 생성하며 자동으로 시작하고 관리 할 수 ​​있습니다.

## 주요 구성 요소

### WorldRoomsModule
```csharp
// Настройки
[Header("Zones Settings"), SerializeField]
private string[] zoneScenes; // Сцены зон, которые будут автоматически запущены

// Зависимости
protected SpawnersModule spawnersModule; // Для запуска серверов зон
```

## 세계 구역의 자동 생성

```csharp
// При регистрации спаунера автоматически запускаются все зоны
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

## 영역의 속성 설정

```csharp
// Создание настроек для запуска зоны на спаунере
protected virtual MstProperties SpawnerProperties(string zoneId)
{
    var properties = new MstProperties();
    properties.Set(Mst.Args.Names.RoomName, zoneId);        // Имя комнаты
    properties.Set(Mst.Args.Names.RoomOnlineScene, zoneId); // Имя сцены
    properties.Set(Mst.Args.Names.RoomIsPrivate, true);     // Приватная комната
    properties.Set(MstDictKeys.WORLD_ZONE, zoneId);         // Маркер зоны мира
    
    return properties;
}
```

## 영역에 대한 정보를 얻습니다

```csharp
// На сервере - поиск комнаты зоны по ID
RegisteredRoom zoneRoom = roomsList.Values
    .Where(r => r.Options.CustomOptions.AsString(MstDictKeys.WORLD_ZONE) == zoneId)
    .FirstOrDefault();

// Формирование информации о зоне для отправки клиенту
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

## 클라이언트 영역에 대한 정보 요청

```csharp
// Запрос информации о зоне по ее ID
Mst.Client.Connection.SendMessage(MstOpCodes.GetZoneRoomInfo, "Forest", (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        // Получение данных о зоне
        var zoneInfo = response.AsPacket<GameInfoPacket>();
        
        // Подключение к серверу зоны
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

## 영역 간 전환

```csharp
// Клиентский код для перехода между зонами
public void RequestZoneTransition(string targetZoneId, Vector3 spawnPosition)
{
    // 1. Запрос информации о целевой зоне
    Mst.Client.Connection.SendMessage(MstOpCodes.GetZoneRoomInfo, targetZoneId, (status, response) =>
    {
        if (status == ResponseStatus.Success)
        {
            // 2. Получение информации о зоне
            var zoneInfo = response.AsPacket<GameInfoPacket>();
            
            // 3. Сохранение позиции появления для новой зоны
            PlayerPrefs.SetFloat("SpawnPosX", spawnPosition.x);
            PlayerPrefs.SetFloat("SpawnPosY", spawnPosition.y);
            PlayerPrefs.SetFloat("SpawnPosZ", spawnPosition.z);
            
            // 4. Отключение от текущей зоны
            Mst.Client.Connection.Disconnect();
            
            // 5. Подключение к новой зоне
            ConnectToZone(zoneInfo.Address, zoneInfo.Id);
        }
    });
}

// Подключение к серверу зоны
private void ConnectToZone(string address, int roomId)
{
    // Разбор адреса в формате "ip:port"
    string[] addressParts = address.Split(':');
    string ip = addressParts[0];
    int port = int.Parse(addressParts[1]);
    
    // Подключение к серверу
    Mst.Client.Connection.Connect(ip, port, (successful, connectedPeer) =>
    {
        if (successful)
        {
            Debug.Log($"Connected to zone server: {address}");
            
            // Присоединение к комнате после подключения к серверу
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

## 페르시아 세계와의 통합

```csharp
// Пример кода сервера зоны для обработки подключения игрока
protected override void OnPlayerJoinedRoom(RoomPlayer player)
{
    base.OnPlayerJoinedRoom(player);
    
    // Получение расширения пользователя
    var userExt = player.GetExtension<IUserPeerExtension>();
    if (userExt != null)
    {
        // Загрузка данных игрока для текущей зоны
        LoadPlayerZoneData(userExt.UserId);
        
        // Отправка данных о других игроках в зоне
        SendZonePlayersData(player);
    }
}

// Загрузка данных игрока для конкретной зоны
private void LoadPlayerZoneData(string userId)
{
    // Получение данных из профиля или базы данных
    var zoneId = gameObject.scene.name;
    var zoneDataKey = $"zonedata_{zoneId}_{userId}";
    
    // Запрос данных из профиля
    Mst.Server.Profiles.GetProfileValues(userId, new string[] { zoneDataKey }, (success, data) =>
    {
        if (success && data.Has(zoneDataKey))
        {
            // Разбор данных зоны (позиция, инвентарь и т.д.)
            var zoneData = data.AsString(zoneDataKey);
            // Применение данных к игроку
            ApplyZoneDataToPlayer(userId, zoneData);
        }
        else
        {
            // Использование данных по умолчанию для новых игроков в этой зоне
            ApplyDefaultZoneData(userId);
        }
    });
}
```

## 영역을 떠날 때 데이터 조건

```csharp
// Сервер зоны: обработка выхода игрока из зоны
protected override void OnPlayerLeftRoom(RoomPlayer player)
{
    base.OnPlayerLeftRoom(player);
    
    // Получение расширения пользователя
    var userExt = player.GetExtension<IUserPeerExtension>();
    if (userExt != null)
    {
        // Сохранение данных игрока для текущей зоны
        SavePlayerZoneData(userExt.UserId);
    }
}

// Сохранение данных игрока для конкретной зоны
private void SavePlayerZoneData(string userId)
{
    // Сбор данных игрока (позиция, инвентарь и т.д.)
    string zoneData = GenerateZoneDataForPlayer(userId);
    
    // Формирование ключа зоны
    var zoneId = gameObject.scene.name;
    var zoneDataKey = $"zonedata_{zoneId}_{userId}";
    
    // Сохранение в профиль
    var data = new MstProperties();
    data.Set(zoneDataKey, zoneData);
    
    Mst.Server.Profiles.SetProfileValues(userId, data);
}
```

## 사용의 예

### 영역 간의 전환 설정
```csharp
// Зона "Лес" соединяется с зоной "Пещера"
public class ZoneTransition : MonoBehaviour
{
    [SerializeField] private string targetZoneId = "Cave";
    [SerializeField] private Vector3 spawnPosition = new Vector3(10, 0, 10);
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            // Отправка запроса на переход в другую зону
            var zoneManager = FindObjectOfType<WorldZoneManager>();
            zoneManager.RequestZoneTransition(targetZoneId, spawnPosition);
        }
    }
}
```

### 구역의 글로벌 이벤트
```csharp
// Система глобальных событий для синхронизации между зонами
public class GlobalEventsManager : MonoBehaviour
{
    // Отправка глобального события всем зонам
    public void BroadcastGlobalEvent(string eventType, MstProperties eventData)
    {
        // Отправка события на мастер сервер
        var data = new MstProperties();
        data.Set("type", eventType);
        data.Set("data", eventData.ToJson());
        
        Mst.Client.Connection.SendMessage(MstOpCodes.GlobalZoneEvent, data);
    }
}

// На мастер сервере
private Task GlobalZoneEventHandler(IIncomingMessage message)
{
    try
    {
        var data = MstProperties.FromBytes(message.AsBytes());
        string eventType = data.AsString("type");
        string eventData = data.AsString("data");
        
        // Рассылка всем зонам
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

## 모범 사례

1. ** 구역 디비전 **를 사용하여 서버의 부하를 줄입니다.
2. ** 데이터 저장 ** 영역 사이를 건너면
3. ** 데이터 전송 최적화 ** - 필요한 정보 만 보내기
4. ** 매끄러운 전환 개선 ** 화면을 로딩하는 영역간에
5. ** 영역 간의 동기화를 위해 전체 이벤트 시스템을 사용하십시오 **
6. ** 공유 책임 ** 영역 서버와 마스터 서버간에
