# Master Server Toolkit - Lobbies

## 설명
플레이어 그룹 만들기, 준비 상태 관리 및 게임 시작을 포함하여 게임 로비를 만들고 관리하기위한 모듈.

## LobbiesModule

게임 로비 관리를위한 메인 클래스.

### 설정 :
```csharp
[Header("Configuration")]
public int createLobbiesPermissionLevel = 0;
public bool dontAllowCreatingIfJoined = true;
public int joinedLobbiesLimit = 1;
```

## 로비 만들기

로비 공장의 ### 등록 :
```csharp
// Создание фабрики
var factory = new LobbyFactoryAnonymous("Game_Lobby", lobbyModule);
factory.CreateNewLobbyHandler = (properties, user) =>
{
    var config = new LobbyConfig
    {
        Name = properties.AsString("name", "Game Lobby"),
        IsPublic = properties.AsBool("isPublic", true),
        MaxPlayers = properties.AsInt("maxPlayers", 10),
        MaxWaitTime = properties.AsInt("maxWaitTime", 60000),
        Teams = new List<LobbyTeam>
        {
            new LobbyTeam("Team A") { MaxPlayers = 5 },
            new LobbyTeam("Team B") { MaxPlayers = 5 }
        }
    };
    
    return new GameLobby(lobbyModule.NextLobbyId(), config, lobbyModule);
};

// Регистрация фабрики
lobbyModule.AddFactory(factory);
```

### 로비 생성 (클라이언트) :
```csharp
// Создание лобби
var properties = new MstProperties();
properties.Set(MstDictKeys.LOBBY_FACTORY_ID, "Game_Lobby");
properties.Set("name", "Epic Battle Room");
properties.Set("isPublic", true);
properties.Set("maxPlayers", 8);

Mst.Client.Connection.SendMessage(MstOpCodes.CreateLobby, properties, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        int lobbyId = response.AsInt();
        Debug.Log($"Lobby created: {lobbyId}");
    }
});
```

## 로비에 공동

### 이익 (클라이언트) :
```csharp
// Присоединение к лобби
Mst.Client.Connection.SendMessage(MstOpCodes.JoinLobby, lobbyId, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        var lobbyData = response.AsPacket<LobbyDataPacket>();
        Debug.Log($"Joined lobby: {lobbyData.Name}");
        
        // Подписка на события лобби
        SubscribeToLobbyEvents();
    }
});

// Покинуть лобби
Mst.Client.Connection.SendMessage(MstOpCodes.LeaveLobby, lobbyId);
```

로비의 ## 이벤트

### 이벤트 구독 :
```csharp
// Подписка на изменения
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberJoined, OnMemberJoined);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberLeft, OnMemberLeft);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyStateChange, OnLobbyStateChange);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberReadyStatusChange, OnMemberReadyChange);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyChatMessage, OnChatMessage);

// Обработчики
private void OnMemberJoined(IIncomingMessage message)
{
    var memberData = message.AsPacket<LobbyMemberData>();
    Debug.Log($"Player {memberData.Username} joined the lobby");
}

private void OnLobbyStateChange(IIncomingMessage message)
{
    var state = (LobbyState)message.AsInt();
    Debug.Log($"Lobby state changed to: {state}");
}
```

## 속성 관리

로비의 ### 속성 :
```csharp
// Установка свойств лобби (только владелец)
var properties = new MstProperties();
properties.Set("map", "forest");
properties.Set("gameMode", "battle");
properties.Set("maxPlayers", 12);

var packet = new LobbyPropertiesSetPacket
{
    LobbyId = currentLobbyId,
    Properties = properties
};

Mst.Client.Connection.SendMessage(MstOpCodes.SetLobbyProperties, packet);
```

플레이어의 ### 속성 :
```csharp
// Установка своих свойств
var myProperties = new Dictionary<string, string>
{
    { "character", "warrior" },
    { "level", "25" },
    { "color", "blue" }
};

Mst.Client.Connection.SendMessage(MstOpCodes.SetMyProperties, myProperties.ToBytes());
```

## 팀

### 플레이어의 준비 :
```csharp
// Установить себя готовым
Mst.Client.Connection.SendMessage(MstOpCodes.SetLobbyAsReady, 1);

// Отменить готовность
Mst.Client.Connection.SendMessage(MstOpCodes.SetLobbyAsReady, 0);
```

### 팀 연결 :
```csharp
// Присоединиться к команде
var joinTeamPacket = new LobbyJoinTeamPacket
{
    LobbyId = currentLobbyId,
    TeamName = "Team A"
};

Mst.Client.Connection.SendMessage(MstOpCodes.JoinLobbyTeam, joinTeamPacket);
```

## 로비에서 채팅

### 게시물 보내기 :
```csharp
// Отправка сообщения в чат лобби
string chatMessage = "Hello everyone!";
Mst.Client.Connection.SendMessage(MstOpCodes.SendMessageToLobbyChat, chatMessage);

// Получение сообщений
private void OnChatMessage(IIncomingMessage message)
{
    var chatData = message.AsPacket<LobbyChatPacket>();
    Debug.Log($"{chatData.Sender}: {chatData.Message}");
}
```

## 게임 출시

## 수동 출시 :
```csharp
// Начать игру (только владелец лобби)
Mst.Client.Connection.SendMessage(MstOpCodes.StartLobbyGame, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        Debug.Log("Game started!");
    }
});
```

### 자동 출시 :
```csharp
// Настройка автозапуска
var config = new LobbyConfig
{
    EnableReadySystem = true,
    MinPlayersToStart = 2,
    MaxWaitTime = 30000 // 30 секунд
};

// Игра начнется автоматически когда:
// 1. Минимум игроков готовы
// 2. Истекло время ожидания
```

## 사용자 로비 구현

```csharp
public class RankedGameLobby : BaseLobby
{
    public RankedGameLobby(int lobbyId, ILobbyFactory factory, 
        LobbiesModule module) : base(lobbyId, factory, module)
    {
        // Кастомные настройки
        EnableTeamSwitching = false;
        EnableReadySystem = true;
        MaxPlayers = 10;
    }
    
    public override bool AddPlayer(LobbyUserPeerExtension playerExt, out string error)
    {
        // Проверка рейтинга игрока
        var playerProfile = GetPlayerProfile(playerExt);
        if (playerProfile.Rating < RequiredRating)
        {
            error = "Rating too low for this lobby";
            return false;
        }
        
        // Автоматическое распределение по командам
        AutoBalanceTeams(playerExt);
        
        return base.AddPlayer(playerExt, out error);
    }
    
    protected override void OnGameStart()
    {
        // Кастомная логика старта
        CalculateTeamBalance();
        ApplyRatingModifiers();
        
        base.OnGameStart();
    }
}
```

## 다른 모듈과의 통합

## Spawner :
```csharp
// Автоматическое создание сервера при старте игры
protected override void OnGameStart()
{
    var spawnOptions = new MstProperties();
    spawnOptions.Set("map", Properties["map"]);
    spawnOptions.Set("gameMode", Properties["gameMode"]);
    
    SpawnersModule.Spawn(spawnOptions, (spawner, data) =>
    {
        GamePort = data.RoomPort;
        GameIp = data.RoomIp;
        // Игроки получат доступ к серверу
    });
}
```

매치 메이커와 ### :
```csharp
// Интеграция с матчмейкингом
var factory = new LobbyFactoryAnonymous("Matchmaking_Lobby", lobbyModule);
factory.CreateNewLobbyHandler = (properties, user) =>
{
    var matchmakingData = properties.AsPacket<MatchmakingRequestPacket>();
    var lobby = new MatchmakingLobby();
    
    // Настройка на основе данных матчмейкинга
    lobby.SetupMatchmaking(matchmakingData);
    
    return lobby;
};
```

## 모범 사례

1. ** 공장 사용 ** 다른 유형의 로비를 만듭니다.
2. ** 유효한 속성 ** 플레이어를 추가하기 전에
3. ** 공정한 게임을 위해 자동 잔액 설정 **
4. ** 프로세스 이벤트 ** 반응 형 UI를 생성합니다
5. ** 빈 로비를 청소하십시오 ** 무료 자원을 위해
6. ** 디버깅 및 분석을위한 로그 액션 **
7. ** 제한을 사용하여 남용으로부터 보호 **

## 예제 UI

### 로비의 간단한 목록 :
```csharp
public class LobbyListUI : MonoBehaviour
{
    [Header("UI References")]
    public GameObject lobbyItemPrefab;
    public Transform lobbyList;
    
    void Start()
    {
        RefreshLobbyList();
    }
    
    void RefreshLobbyList()
    {
        // Получение списка публичных лобби
        Mst.Client.Lobbies.GetPublicLobbies((lobbies) =>
        {
            foreach (var lobby in lobbies)
            {
                var item = Instantiate(lobbyItemPrefab, lobbyList);
                var itemComponent = item.GetComponent<LobbyListItem>();
                itemComponent.Setup(lobby);
            }
        });
    }
}
```
