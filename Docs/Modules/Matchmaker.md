# Master Server Toolkit - Matchmaker

## Description
Module for searching games, rooms and lobbies by various criteria as well as retrieving information about available server regions.

## Main Components

### MatchmakerModule (Server)
```csharp
// Dependencies
AddOptionalDependency<LobbiesModule>();
AddOptionalDependency<RoomsModule>();
AddOptionalDependency<SpawnersModule>();

// Core methods
public void AddProvider(IGamesProvider provider) // Add game provider
```

### MstMatchmakerClient (Client)
```csharp
// Find games (without filters)
matchmaker.FindGames((games) => {
    Debug.Log($"Found {games.Count} games");
});

// Find games with filters
var filters = new MstProperties();
filters.Set("minPlayers", 2);
matchmaker.FindGames(filters, (games) => { });

// Get regions
matchmaker.GetRegions((regions) => {
    // regions[0].Name, regions[0].Ip, regions[0].PingTime
});
```

## Key Data Structures

### GameInfoPacket
```csharp
public class GameInfoPacket
{
    public int Id { get; set; }                    // Game ID
    public string Address { get; set; }            // Connection address
    public GameInfoType Type { get; set; }         // Type (Room, Lobby, Custom)
    public string Name { get; set; }               // Name
    public string Region { get; set; }             // Region
    public bool IsPasswordProtected { get; set; }  // Requires password
    public int MaxPlayers { get; set; }            // Max players
    public int OnlinePlayers { get; set; }         // Current players
    public List<string> OnlinePlayersList { get; } // Player list
    public MstProperties Properties { get; set; }  // Additional properties
}
```

### RegionInfo
```csharp
public class RegionInfo
{
    public string Name { get; set; }  // Region name
    public string Ip { get; set; }    // IP address
    public int PingTime { get; set; } // Ping (ms)
}
```

## Custom Game Providers

### IGamesProvider interface
```csharp
public interface IGamesProvider
{
    IEnumerable<GameInfoPacket> GetPublicGames(IPeer peer, MstProperties filters);
}
```

### Minimal implementation example
```csharp
public class CustomGamesProvider : MonoBehaviour, IGamesProvider
{
    private List<GameInfoPacket> games = new List<GameInfoPacket>();
    
    public IEnumerable<GameInfoPacket> GetPublicGames(IPeer peer, MstProperties filters)
    {
        // Filter by region
        if (filters.Has("region"))
        {
            return games.Where(g => g.Region == filters.AsString("region"));
        }
        
        return games;
    }
    
    // API for adding/removing games
    public void AddGame(GameInfoPacket game) => games.Add(game);
    public void RemoveGame(int gameId) => games.RemoveAll(g => g.Id == gameId);
}

// Registration
matchmaker.AddProvider(gameObject.AddComponent<CustomGamesProvider>());
```

## Usage Example

### Searching for games with multiple filters
```csharp
var filters = new MstProperties();
filters.Set("region", "eu-west");     // Only the European region
filters.Set("minPlayers", 1);         // At least 1 player
filters.Set("gameMode", "deathmatch"); // "deathmatch" mode

matchmaker.FindGames(filters, (games) =>
{
    // Found games
    foreach (var game in games)
    {
        Debug.Log($"{game.Name} - {game.OnlinePlayers}/{game.MaxPlayers}");
        
        // Connect to the game
        Mst.Client.Rooms.JoinRoom(game.Id, "", (successful, roomAccess) =>
        {
            if (successful)
                ConnectToGameServer(roomAccess);
        });
    }
});
```

### Selecting the region with the best ping
```csharp
matchmaker.GetRegions((regions) =>
{
    if (regions.Count > 0)
    {
        // Sort by ping
        var bestRegion = regions.OrderBy(r => r.PingTime).First();
        PlayerPrefs.SetString("SelectedRegion", bestRegion.Name);
        
        Debug.Log($"Using region: {bestRegion.Name} ({bestRegion.PingTime}ms)");
    }
});
```

## Best Practices

1. **Use meaningful property names** for filtering
2. **Add categories** to Properties for better organization
3. **Handle the absence of games** in client code
4. **Sort results** to improve the user experience
5. **Cache the region list** and search results if needed
6. **Refresh the region list** when the game starts
