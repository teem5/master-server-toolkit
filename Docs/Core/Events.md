# Master Server Toolkit - Events

## Description
An event system for communication between components without tight coupling. Includes event channels and typed messages.

## MstEventsChannel

Channel used to send and receive events.

### Creating a channel:
```csharp
// Create a channel with exception handling
var defaultChannel = new MstEventsChannel("default", true);

// Create a simple channel
var gameChannel = new MstEventsChannel("game");

// Use the global channel
var globalChannel = Mst.Events;
```

### Main methods:
```csharp
// Invoke an event without data
channel.Invoke("playerDied");

// Invoke an event with data
channel.Invoke("scoreUpdated", 100);
channel.Invoke("playerJoined", new Player("John"));

// Subscribe to events
channel.AddListener("gameStarted", OnGameStarted);
channel.AddListener("levelCompleted", OnLevelCompleted, true);

// Unsubscribe from events
channel.RemoveListener("gameStarted", OnGameStarted);
channel.RemoveAllListeners("playerDied");
```

## EventMessage

Container for event data with typed access.

### Creation and usage:
```csharp
// Create an empty message
var emptyMsg = EventMessage.Empty;

// Create with data
var scoreMsg = new EventMessage(150);
var playerMsg = new EventMessage(new Player { Name = "Alex", Level = 10 });

// Retrieve data
int score = scoreMsg.As<int>();
float damage = damageMsg.AsFloat();
string text = textMsg.AsString();
bool isWinner = resultMsg.AsBool();

// Check if data exists
if (message.HasData())
{
    var data = message.As<MyData>();
}
```

## Usage examples

### 1. Game events:
```csharp
public class GameManager : MonoBehaviour
{
    private MstEventsChannel gameEvents;
    
    void Start()
    {
        gameEvents = new MstEventsChannel("game");
        
        // Subscribe to events
        gameEvents.AddListener("playerJoined", OnPlayerJoined);
        gameEvents.AddListener("scoreChanged", OnScoreChanged);
        gameEvents.AddListener("gameOver", OnGameOver);
    }
    
    private void OnPlayerJoined(EventMessage msg)
    {
        var player = msg.As<Player>();
        Debug.Log($"Player {player.Name} joined the game");
        
        // Notify other players
        gameEvents.Invoke("playerListUpdated", GetPlayerList());
    }
    
    private void OnScoreChanged(EventMessage msg)
    {
        int newScore = msg.AsInt();
        UpdateScoreUI(newScore);
        
        // Check for a record
        if (newScore > highScore)
        {
            gameEvents.Invoke("newRecord", newScore);
        }
    }
}
```

### 2. UI events:
```csharp
public class UIManager : MonoBehaviour
{
    void Start()
    {
        // Subscribe to global events
        Mst.Events.AddListener("connectionLost", OnConnectionLost);
        Mst.Events.AddListener("dataLoaded", OnDataLoaded);
        Mst.Events.AddListener("errorOccurred", OnError);
    }
    
    private void OnConnectionLost(EventMessage msg)
    {
        ShowReconnectDialog();
    }
    
    private void OnDataLoaded(EventMessage msg)
    {
        var data = msg.As<GameData>();
        UpdateUI(data);
        HideLoadingScreen();
    }
    
    private void OnError(EventMessage msg)
    {
        string errorText = msg.AsString();
        ShowErrorDialog(errorText);
    }
}
```

### 3. Inter-component communication:
```csharp
public class InventorySystem : MonoBehaviour
{
    private MstEventsChannel inventoryEvents;
    
    void Start()
    {
        inventoryEvents = new MstEventsChannel("inventory");
        
        // Subscribe to game events
        Mst.Events.AddListener("itemDropped", OnItemDropped);
        Mst.Events.AddListener("playerDied", OnPlayerDied);
    }
    
    public void AddItem(Item item)
    {
        if (CanAddItem(item))
        {
            // Add item
            items.Add(item);
            
            // Notify about inventory change
            inventoryEvents.Invoke("itemAdded", item);
            
            // Check quests
            if (IsQuestItem(item))
            {
                Mst.Events.Invoke("questItemObtained", item);
            }
        }
        else
        {
            inventoryEvents.Invoke("inventoryFull", item);
        }
    }
}
```

## Named channels

### Creating specialized channels:
```csharp
// Channel for combat system
var combatChannel = new MstEventsChannel("combat");
combatChannel.AddListener("enemySpotted", OnEnemySpotted);
combatChannel.AddListener("damageDealt", OnDamageDealt);

// Channel for social system
var socialChannel = new MstEventsChannel("social");
socialChannel.AddListener("friendRequestReceived", OnFriendRequest);
socialChannel.AddListener("messageReceived", OnChatMessage);

// Channel for economy
var economyChannel = new MstEventsChannel("economy");
economyChannel.AddListener("purchaseCompleted", OnPurchase);
economyChannel.AddListener("currencyChanged", OnCurrencyUpdate);
```

## Best practices

1. **Use meaningful event names**:
```csharp
// Good
"playerJoinedLobby"
"itemCraftingCompleted"
"achievementUnlocked"

// Bad
"event1"
"update"
"changed"
```

2. **Strongly type event data**:
```csharp
// Define structures for complex data
public struct ScoreChangedData
{
    public int oldScore;
    public int newScore;
    public string reason;
}

// Use them in events
channel.Invoke("scoreChanged", new ScoreChangedData 
{ 
    oldScore = 100, 
    newScore = 150, 
    reason = "enemyKilled" 
});
```

3. **Unsubscribe from events**:
```csharp
void OnDestroy()
{
    // Unsubscribe from all events
    Mst.Events.RemoveListener("playerJoined", OnPlayerJoined);
    channel.RemoveAllListeners();
}
```

4. **Use channels for logical grouping**:
Game events → "game"
UI events → "ui"
Network events → "network"
System events → "system"

5. **Handle exceptions**:
```csharp
// When creating a channel enable exception handling
var channel = new MstEventsChannel("game", true);
```

## Integration with other systems

```csharp
// Integration with analytics
Mst.Events.AddListener("levelCompleted", (msg) => {
    var data = msg.As<LevelCompletionData>();
    Analytics.TrackLevelCompletion(data);
});

// Integration with saves
Mst.Events.AddListener("gameStateChanged", (msg) => {
    SaveSystem.SaveGameState(msg.As<GameState>());
});

// Integration with networking
Mst.Events.AddListener("playerAction", (msg) => {
    NetworkManager.SendPlayerAction(msg.As<PlayerAction>());
});
```
