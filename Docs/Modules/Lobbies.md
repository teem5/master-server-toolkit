# Master Server Toolkit - Lobbies

## Description
Module for creating and managing game lobbies including party creation, ready checks, and starting games.

## LobbiesModule

Main class for lobby management.

### Setup:
```csharp
[Header("Configuration")]
public int createLobbiesPermissionLevel = 0;
public bool dontAllowCreatingIfJoined = true;
public int joinedLobbiesLimit = 1;
```

## Creating a lobby

### Registering a lobby factory:
```csharp
// Create the factory
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

// Register the factory
lobbyModule.AddFactory(factory);
```

### Creating a lobby (client):
```csharp
// Create a lobby
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

## Joining a lobby

### Joining (client):
```csharp
// Join a lobby
Mst.Client.Connection.SendMessage(MstOpCodes.JoinLobby, lobbyId, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        var lobbyData = response.AsPacket<LobbyDataPacket>();
        Debug.Log($"Joined lobby: {lobbyData.Name}");

        // Subscribe to lobby events
        SubscribeToLobbyEvents();
    }
});

// Leave the lobby
Mst.Client.Connection.SendMessage(MstOpCodes.LeaveLobby, lobbyId);
```

## Lobby events

### Subscribing to events:
```csharp
// Subscribe to changes
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberJoined, OnMemberJoined);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberLeft, OnMemberLeft);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyStateChange, OnLobbyStateChange);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyMemberReadyStatusChange, OnMemberReadyChange);
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.LobbyChatMessage, OnChatMessage);

// Handlers
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

## Property management

### Lobby properties:
```csharp
// Set lobby properties (owner only)
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

### Player properties:
```csharp
// Set your properties
var myProperties = new Dictionary<string, string>
{
    { "character", "warrior" },
    { "level", "25" },
    { "color", "blue" }
};

Mst.Client.Connection.SendMessage(MstOpCodes.SetMyProperties, myProperties.ToBytes());
```

## Commands

### Player ready state:
```csharp
// Set ready
Mst.Client.Connection.SendMessage(MstOpCodes.SetLobbyAsReady, 1);

// Cancel ready
Mst.Client.Connection.SendMessage(MstOpCodes.SetLobbyAsReady, 0);
```

### Joining a team:
```csharp
// Join a team
var joinTeamPacket = new LobbyJoinTeamPacket
{
    LobbyId = currentLobbyId,
    TeamName = "Team A"
};

Mst.Client.Connection.SendMessage(MstOpCodes.JoinLobbyTeam, joinTeamPacket);
```

## Lobby chat

### Sending messages:
```csharp
// Send a lobby chat message
string chatMessage = "Hello everyone!";
Mst.Client.Connection.SendMessage(MstOpCodes.SendMessageToLobbyChat, chatMessage);

// Receive messages
private void OnChatMessage(IIncomingMessage message)
{
    var chatData = message.AsPacket<LobbyChatPacket>();
    Debug.Log($"{chatData.Sender}: {chatData.Message}");
}
```

## Starting the game

### Manual start:
```csharp
// Start the game (lobby owner only)
Mst.Client.Connection.SendMessage(MstOpCodes.StartLobbyGame, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        Debug.Log("Game started!");
    }
});
```

### Automatic start:
```csharp
// Autostart settings
var config = new LobbyConfig
{
    EnableReadySystem = true,
    MinPlayersToStart = 2,
    MaxWaitTime = 30000 // 30 seconds
};

// The game will start automatically when:
// 1. Minimum players are ready
// 2. The waiting time expires
```

## Implementing a custom lobby
```csharp
public class RankedGameLobby : BaseLobby
{
    public RankedGameLobby(int lobbyId, ILobbyFactory factory,
        LobbiesModule module) : base(lobbyId, factory, module)
    {
        // Custom settings
        EnableTeamSwitching = false;
        EnableReadySystem = true;
        MaxPlayers = 10;
    }

    public override bool AddPlayer(LobbyUserPeerExtension playerExt, out string error)
    {
        // Check player rating
        var playerProfile = GetPlayerProfile(playerExt);
        if (playerProfile.Rating < RequiredRating)
        {
            error = "Rating too low for this lobby";
            return false;
        }

        // Auto assign to teams
        AutoBalanceTeams(playerExt);

        return base.AddPlayer(playerExt, out error);
    }

    protected override void OnGameStart()
    {
        // Custom start logic
        CalculateTeamBalance();
        ApplyRatingModifiers();

        base.OnGameStart();
    }
}
```

## Integration with other modules

### With Spawner:
```csharp
// Automatically create a server when the game starts
protected override void OnGameStart()
{
    var spawnOptions = new MstProperties();
    spawnOptions.Set("map", Properties["map"]);
    spawnOptions.Set("gameMode", Properties["gameMode"]);

    SpawnersModule.Spawn(spawnOptions, (spawner, data) =>
    {
        GamePort = data.RoomPort;
        GameIp = data.RoomIp;
        // Players will get access to the server
    });
}
```

### With Matchmaker:
```csharp
// Integration with matchmaking
var factory = new LobbyFactoryAnonymous("Matchmaking_Lobby", lobbyModule);
factory.CreateNewLobbyHandler = (properties, user) =>
{
    var matchmakingData = properties.AsPacket<MatchmakingRequestPacket>();
    var lobby = new MatchmakingLobby();

    // Configure based on matchmaking data
    lobby.SetupMatchmaking(matchmakingData);

    return lobby;
};
```

## Best practices

1. **Use factories** to create different lobby types
2. **Validate properties** before adding players
3. **Enable auto-balance** for fair matches
4. **Handle events** to build responsive UI
5. **Clean up empty lobbies** to free resources
6. **Log actions** for debugging and analytics
7. **Protect against abuse** with limits

## UI examples

### Simple lobby list:
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
        // Get a list of public lobbies
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
