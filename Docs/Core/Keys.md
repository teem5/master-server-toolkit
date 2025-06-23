# Master Server Toolkit - Keys

## Description
Central registry of keys and codes for network messages, events and dictionary data. Provides typed constants for all operations in the framework.

## MstOpCodes

Operation codes for network messages between client and server.

### Basic operations:
```csharp
// Errors and ping
MstOpCodes.Error        // Error messages
MstOpCodes.Ping         // Connection check

// Authentication
MstOpCodes.SignIn       // Sign in
MstOpCodes.SignUp       // Sign up
MstOpCodes.SignOut      // Sign out
```

### Usage example:
```csharp
// Send a message with a specific OpCode
client.SendMessage(MstOpCodes.SignIn, loginData);

// Register a handler for an OpCode
client.RegisterMessageHandler(MstOpCodes.LobbyInfo, HandleLobbyInfo);

// Create a custom OpCode
public static ushort MyCustomCode = "myCustomAction".ToUint16Hash();
```

### OpCode categories:

#### 1. Authentication and accounts:
```csharp
MstOpCodes.SignIn
MstOpCodes.SignUp
MstOpCodes.SignOut
MstOpCodes.GetPasswordResetCode
MstOpCodes.ConfirmEmail
MstOpCodes.ChangePassword
```

#### 2. Rooms and spawns:
```csharp
MstOpCodes.RegisterRoomRequest
MstOpCodes.DestroyRoomRequest
MstOpCodes.GetRoomAccessRequest
MstOpCodes.SpawnProcessRequest
MstOpCodes.CompleteSpawnProcess
```

#### 3. Lobbies:
```csharp
MstOpCodes.CreateLobby
MstOpCodes.JoinLobby
MstOpCodes.LeaveLobby
MstOpCodes.SetLobbyProperties
MstOpCodes.StartLobbyGame
```

#### 4. Chat:
```csharp
MstOpCodes.ChatMessage
MstOpCodes.JoinChannel
MstOpCodes.LeaveChannel
MstOpCodes.PickUsername
```

## MstEventKeys

Keys for the event system used by UI and game logic.

### UI events:
```csharp
// Dialogs
MstEventKeys.showOkDialogBox
MstEventKeys.hideOkDialogBox
MstEventKeys.showYesNoDialogBox

// Screens
MstEventKeys.showSignInView
MstEventKeys.hideSignInView
MstEventKeys.showLobbyListView
```

### Usage example:
```csharp
// Show a dialog
Mst.Events.Invoke(MstEventKeys.showOkDialogBox, "Welcome!");

// Subscribe to an event
Mst.Events.AddListener(MstEventKeys.gameStarted, OnGameStarted);

// Create custom events
public static string MyCustomEvent = "game.levelCompleted";
```

### Event categories:

#### 1. Navigation:
```csharp
MstEventKeys.goToZone
MstEventKeys.leaveRoom
MstEventKeys.showLoadingInfo
```

#### 2. Game events:
```csharp
MstEventKeys.gameStarted
MstEventKeys.gameOver
MstEventKeys.playerStartedGame
MstEventKeys.playerFinishedGame
```

#### 3. Visual elements:
```csharp
MstEventKeys.showLoadingInfo
MstEventKeys.hideLoadingInfo
MstEventKeys.showPickUsernameView
```

## MstDictKeys

Keys for dictionary data passed in messages.

### User data:
```csharp
MstDictKeys.USER_ID          // "-userId"
MstDictKeys.USER_NAME        // "-userName"
MstDictKeys.USER_EMAIL       // "-userEmail"
MstDictKeys.USER_AUTH_TOKEN  // "-userAuthToken"
```

### Usage example:
```csharp
// Create a message with data
var userData = new MstProperties();
userData.Set(MstDictKeys.USER_NAME, "Player1");
userData.Set(MstDictKeys.USER_EMAIL, "player@game.com");

// Send data
client.SendMessage(MstOpCodes.SignUp, userData);

// Receive data
string userName = message.AsString(MstDictKeys.USER_NAME);
```

### Key categories:

#### 1. Rooms:
```csharp
MstDictKeys.ROOM_ID
MstDictKeys.ROOM_CONNECTION_TYPE
MstDictKeys.WORLD_ZONE
```

#### 2. Lobby:
```csharp
MstDictKeys.LOBBY_FACTORY_ID
MstDictKeys.LOBBY_NAME
MstDictKeys.LOBBY_PASSWORD
MstDictKeys.LOBBY_TEAM
```

#### 3. Authentication:
```csharp
MstDictKeys.USER_ID
MstDictKeys.USER_PASSWORD
MstDictKeys.USER_AUTH_TOKEN
MstDictKeys.RESET_PASSWORD_CODE
```

## MstPeerPropertyCodes

Codes for peer properties (connected clients/servers).

```csharp
// Basic properties
MstPeerPropertyCodes.Start

// Registered entities
MstPeerPropertyCodes.RegisteredRooms
MstPeerPropertyCodes.RegisteredSpawners

// Client requests
MstPeerPropertyCodes.ClientSpawnRequest
```

### Usage example:
```csharp
// Set a peer property
peer.SetProperty(MstPeerPropertyCodes.RegisteredRooms, roomsList);

// Get a property
var rooms = peer.GetProperty(MstPeerPropertyCodes.RegisteredRooms);
```

## Creating custom keys

### Extending OpCodes:
```csharp
public static class CustomOpCodes
{
    public static ushort GetPlayerStats = "getPlayerStats".ToUint16Hash();
    public static ushort UpdateInventory = "updateInventory".ToUint16Hash();
    public static ushort CraftItem = "craftItem".ToUint16Hash();
}
```

### Extending EventKeys:
```csharp
public static class GameEventKeys
{
    public static string itemCrafted = "game.itemCrafted";
    public static string achievementUnlocked = "game.achievementUnlocked";
    public static string questCompleted = "game.questCompleted";
}
```

### Extending DictKeys:
```csharp
public static class CustomDictKeys
{
    public const string PLAYER_LEVEL = "-playerLevel";
    public const string INVENTORY_DATA = "-inventoryData";
    public const string ACHIEVEMENT_ID = "-achievementId";
}
```

## Best practices

1. **Use hashing for OpCodes**:
```csharp
// Good
public static ushort MyAction = "myAction".ToUint16Hash();

// Bad - direct numbers
public static ushort MyAction = 1234;
```

2. **Make keys descriptive**:
```csharp
// Good
MstDictKeys.USER_AUTH_TOKEN

// Bad
"-token"
```

3. **Group by functionality**:
```csharp
// Grouping by modules
public struct ChatOpCodes { }
public struct LobbyOpCodes { }
public struct AuthOpCodes { }
```

4. **Document custom keys**:
```csharp
/// <summary>
/// Gets player statistics for the current season
/// </summary>
public static ushort GetSeasonStats = "getSeasonStats".ToUint16Hash();
```

## Integration with other systems

```csharp
// Create a message with multiple keys
var message = new MstProperties();
message.Set(MstDictKeys.USER_ID, userId);
message.Set(MstDictKeys.ROOM_ID, roomId);
message.Set(CustomDictKeys.PLAYER_LEVEL, level);

// Send over the network
Mst.Server.SendMessage(peer, CustomOpCodes.UpdatePlayerData, message);

// Handle using events
Mst.Events.AddListener(GameEventKeys.itemCrafted, (msg) => {
    var itemData = msg.As<CraftedItem>();
    // Handling
});
```

## Debugging and monitoring

```csharp
// Log all incoming OpCodes
Connection.OnMessageReceived += (msg) => {
    Debug.Log($"Received OpCode: {msg.OpCode} ({GetOpCodeName(msg.OpCode)})");
};

// Method to get the OpCode name
private string GetOpCodeName(ushort opCode)
{
    var fields = typeof(MstOpCodes).GetFields(BindingFlags.Public | BindingFlags.Static);
    foreach (var field in fields)
    {
        if ((ushort)field.GetValue(null) == opCode)
            return field.Name;
    }
    return "Unknown";
}
```
