# Master Server Toolkit - Achievements

## Description
An achievements module for creating, tracking, and unlocking player achievements. Integrates with user profiles to save progress.

## AchievementsModule

Main class of the achievements module.

### Setup:
```csharp
[Header("Permission")]
[SerializeField] protected bool clientCanUpdateProgress = false;

[Header("Settings")]
[SerializeField] protected AchievementsDatabase achievementsDatabase;
```

### Dependencies:
The module requires:
- AuthModule - for user authentication
- ProfilesModule - for storing achievement progress

## Creating the achievements database

### AchievementsDatabase:
```csharp
// Create a ScriptableObject with achievements
[CreateAssetMenu(menuName = "Master Server Toolkit/Achievements Database")]
public class AchievementsDatabase : ScriptableObject
{
    [SerializeField] private List<AchievementData> achievements = new List<AchievementData>();

    public IReadOnlyCollection<AchievementData> Achievements => achievements;
}
```

### Defining achievements:
```csharp
// Created in Unity Editor
var database = ScriptableObject.CreateInstance<AchievementsDatabase>();

// Add achievements
database.Add(new AchievementData
{
    Key = "first_victory",
    Title = "First Victory",
    Description = "Win your first match",
    Type = AchievementType.Normal,
    MaxProgress = 1,
    Unlockable = true
});

database.Add(new AchievementData
{
    Key = "win_streak",
    Title = "Win Streak",
    Description = "Win 10 matches in a row",
    Type = AchievementType.Incremental,
    MaxProgress = 10,
    Unlockable = true
});

// Save the achievement database
AssetDatabase.CreateAsset(database, "Assets/Resources/AchievementsDatabase.asset");
AssetDatabase.SaveAssets();
```

## Achievement types

### AchievementType:
```csharp
public enum AchievementType
{
    Normal,         // Regular achievement (0/1)
    Incremental,    // Incremental achievement (0/N)
    Infinite        // Endless achievement (tracking only)
}
```

### Achievement data structure:
```csharp
[Serializable]
public class AchievementData
{
    public string Key;
    public string Title;
    public string Description;
    public AchievementType Type;
    public int MaxProgress;
    public bool Unlockable;
    public Sprite Icon;
    public AchievementExtraData[] ResultCommands;

    [Serializable]
    public class AchievementExtraData
    {
        public string CommandKey;
        public string[] CommandValues;
    }
}
```

## Updating achievement progress

### Server side:
```csharp
// Get the module
var achievementsModule = Mst.Server.Modules.GetModule<AchievementsModule>();

// Update achievement progress
void UpdateAchievement(string userId, string achievementKey, int progress)
{
    var packet = new UpdateAchievementProgressPacket
    {
        userId = userId,
        key = achievementKey,
        progress = progress
    };

    // Send update packet
    Mst.Server.SendMessage(MstOpCodes.ServerUpdateAchievementProgress, packet);
}

// Example usage
UpdateAchievement(user.UserId, "first_victory", 1);
```

### Client side (if allowed):
```csharp
// Client request to update progress
void UpdateAchievement(string achievementKey, int progress)
{
    var packet = new UpdateAchievementProgressPacket
    {
        key = achievementKey,
        progress = progress
    };

    Mst.Client.Connection.SendMessage(MstOpCodes.ClientUpdateAchievementProgress, packet, (status, response) =>
    {
        if (status == ResponseStatus.Success)
        {
            Debug.Log($"Achievement progress updated for {achievementKey}");
        }
    });
}
```

## Profile integration

Achievements are automatically synchronized with user profiles through the profile property:
```csharp
// Register the achievements property in ProfilesModule
profilesModule.RegisterProperty(ProfilePropertyOpCodes.achievements, null, () =>
{
    return new ObservableAchievements();
});

// Retrieve achievements from the profile
void GetAchievements(ObservableServerProfile profile)
{
    if (profile.TryGet(ProfilePropertyOpCodes.achievements, out ObservableAchievements achievements))
    {
        // List of all player achievements
        var allAchievements = achievements.GetAll();

        // Check if the achievement is unlocked
        bool isUnlocked = achievements.IsUnlocked("first_victory");

        // Get achievement progress
        int progress = achievements.GetProgress("win_streak");
    }
}
```

## Tracking unlock events

### Subscribing on the client:
```csharp
// In the client class
private void Start()
{
    Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.ClientAchievementUnlocked, OnAchievementUnlocked);
}

private void OnAchievementUnlocked(IIncomingMessage message)
{
    string achievementKey = message.AsString();
    Debug.Log($"Achievement unlocked: {achievementKey}");

    // Show the achievement unlock UI
    ShowAchievementUnlockedUI(achievementKey);
}
```

## Configuring achievement rewards

The module allows you to set commands that are executed when an achievement is unlocked:
```csharp
// Setting ResultCommands in an achievement
var achievement = new AchievementData
{
    Key = "100_matches_played",
    Title = "Veteran",
    Description = "Play 100 matches",
    Type = AchievementType.Incremental,
    MaxProgress = 100,
    Unlockable = true,

    // Commands to execute when unlocked
    ResultCommands = new[]
    {
        new AchievementExtraData
        {
            CommandKey = "add_currency",
            CommandValues = new[] { "gold", "100" }
        },
        new AchievementExtraData
        {
            CommandKey = "unlock_avatar",
            CommandValues = new[] { "veteran_avatar" }
        }
    }
};
```

### Handling commands:
```csharp
// Extending the module to handle commands
public class MyAchievementsModule : AchievementsModule
{
    protected override void OnAchievementResultCommand(IUserPeerExtension user, string key, AchievementExtraData[] resultCommands)
    {
        foreach (var command in resultCommands)
        {
            switch (command.CommandKey)
            {
                case "add_currency":
                    AddCurrency(user, command.CommandValues[0], int.Parse(command.CommandValues[1]));
                    break;

                case "unlock_avatar":
                    UnlockAvatar(user, command.CommandValues[0]);
                    break;

                // Other commands
                default:
                    logger.Error($"Unknown achievement command: {command.CommandKey}");
                    break;
            }
        }
    }

    private void AddCurrency(IUserPeerExtension user, string currencyType, int amount)
    {
        // Add currency to the player
    }

    private void UnlockAvatar(IUserPeerExtension user, string avatarId)
    {
        // Unlock the avatar for the player
    }
}
```

## Client wrapper

### AchievementsModuleClient:
```csharp
// Usage example
var client = Mst.Client.Modules.GetModule<AchievementsModuleClient>();

// Get the list of achievements
var achievements = client.GetAchievements();

// Display UI
void ShowAchievementsUI()
{
    foreach (var achievement in achievements)
    {
        // Create a UI element for the achievement
        var item = Instantiate(achievementItemPrefab, container);

        // Fill with data
        item.SetTitle(achievement.Title);
        item.SetDescription(achievement.Description);
        item.SetIcon(achievement.Icon);
        item.SetProgress(achievement.CurrentProgress, achievement.MaxProgress);
        item.SetUnlocked(achievement.IsUnlocked);
    }
}
```

## Best practices

1. **Use unique keys** for every achievement
2. **Separate single and incremental** achievements
3. **Validate on the server** to prevent cheating
4. **Execute reward logic on the server**
5. **Cache achievement data** on the client for quick access
6. **Unlock related achievements** automatically (for example, "Kill 10 monsters" unlocks "Kill 5 monsters")
7. **Gather analytics** on achievements to evaluate gameplay
