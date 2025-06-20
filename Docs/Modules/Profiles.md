# Master Server Toolkit - Profiles

## Description
Profiles module for managing user data, tracking changes and synchronizing between clients and servers.

## ProfilesModule

Main class for managing user profiles.

### Setup:
```csharp
[Header("General Settings")]
[SerializeField] protected int unloadProfileAfter = 20;
[SerializeField] protected int saveProfileDebounceTime = 1;
[SerializeField] protected int clientUpdateDebounceTime = 1;
[SerializeField] protected int editProfilePermissionLevel = 0;
[SerializeField] protected int maxUpdateSize = 1048576;

[Header("Timeout Settings")]
[SerializeField] protected int profileLoadTimeoutSeconds = 10;

[Header("Database")]
public DatabaseAccessorFactory databaseAccessorFactory;
[SerializeField] private ObservablePropertyPopulatorsDatabase populatorsDatabase;
```

## Свойства профиля

### Creating the property system:
```csharp
// Create a populator
public class PlayerStatsPopulator : IObservablePropertyPopulator
{
    public IProperty Populate()
    {
        var properties = new ObservableBase();
        
        // Basic stats
        properties.Set("playerLevel", new ObservableInt(1));
        properties.Set("experience", new ObservableInt(0));
        properties.Set("coins", new ObservableInt(0));
        
        // Inventory dictionary
        var inventory = new ObservableDictStringInt();
        inventory.Add("sword", 1);
        inventory.Add("potion", 5);
        properties.Set("inventory", inventory);
        
        return properties;
    }
}
```

## Working with profiles

### Accessing a profile (client):
```csharp
// Request profile
Mst.Client.Connection.SendMessage(MstOpCodes.ClientFillInProfileValues);

// Subscribe to updates
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.UpdateClientProfile, OnProfileUpdated);

// Handle the response
private void OnProfileUpdated(IIncomingMessage message)
{
    var profile = new ObservableProfile();
    profile.FromBytes(message.AsBytes());
    
    // Access properties
    int level = profile.GetProperty<ObservableInt>("playerLevel").Value;
    int coins = profile.GetProperty<ObservableInt>("coins").Value;
}
```

### Accessing a profile (server):
```csharp
// Get profile by ID
var profile = profilesModule.GetProfileByUserId(userId);

// Get profile by peer
var profile = profilesModule.GetProfileByPeer(peer);

// Modify profile
profile.GetProperty<ObservableInt>("playerLevel").Add(1);
profile.GetProperty<ObservableInt>("coins").Set(100);
```

## Profile events

### Subscribing to changes:
```csharp
// On the server
profilesModule.OnProfileCreated += OnProfileCreated;
profilesModule.OnProfileLoaded += OnProfileLoaded;

// In the profile
profile.OnModifiedInServerEvent += OnProfileChanged;

// Specific property
profile.GetProperty<ObservableInt>("playerLevel").OnDirtyEvent += OnLevelChanged;
```

## Database synchronization

### IProfilesDatabaseAccessor implementation:
```csharp
public class ProfilesDatabaseAccessor : IProfilesDatabaseAccessor
{
    public async Task RestoreProfileAsync(ObservableServerProfile profile)
    {
        // Load from DB
        var data = await LoadProfileDataFromDB(profile.UserId);
        if (data != null)
        {
            profile.FromBytes(data);
        }
    }
    
    public async Task UpdateProfilesAsync(List<ObservableServerProfile> profiles)
    {
        // Batch save to DB
        foreach (var profile in profiles)
        {
            await SaveProfileToDB(profile.UserId, profile.ToBytes());
        }
    }
}
```

## Типы наблюдаемых свойств

### Basic types:
```csharp
// Numbers
ObservableInt level = new ObservableInt(10);
ObservableFloat health = new ObservableFloat(100.0f);

// Strings
ObservableString name = new ObservableString("Player");

// Lists
ObservableListInt scores = new ObservableListInt();
scores.Add(100);

// Dictionaries
ObservableDictStringInt items = new ObservableDictStringInt();
items.Add("sword", 1);
```

## Серверные обновления

### Отправка обновлений с игрового сервера:
```csharp
// Создание пакета обновлений
var updates = new ProfileUpdatePacket();
updates.UserId = userId;
updates.Properties = new MstProperties();
updates.Properties.Set("playerLevel", 15);
updates.Properties.Set("experience", 1500);

// Send to the master server
Mst.Server.Connection.SendMessage(MstOpCodes.ServerUpdateProfileValues, updates);
```

## Performance and optimization

### Debounce settings:
```csharp
// Delay before saving to DB (seconds)
saveProfileDebounceTime = 1;

// Delay before sending updates to the client
clientUpdateDebounceTime = 0.5f;

// Time until a profile is unloaded after leaving
unloadProfileAfter = 20;
```

### Limitations:
```csharp
// Maximum update size
maxUpdateSize = 1048576;

// Profile load timeout
profileLoadTimeoutSeconds = 10;
```

## Примеры использования

### Gameplay statistics:
```csharp
public class PlayerStats
{
    public ObservableInt Level { get; private set; }
    public ObservableInt Experience { get; private set; }
    public ObservableFloat Health { get; private set; }
    public ObservableDictStringInt Inventory { get; private set; }
    
    public PlayerStats(ObservableServerProfile profile)
    {
        Level = profile.GetProperty<ObservableInt>("playerLevel");
        Experience = profile.GetProperty<ObservableInt>("experience");
        Health = profile.GetProperty<ObservableFloat>("health");
        Inventory = profile.GetProperty<ObservableDictStringInt>("inventory");
    }
    
    public void AddExperience(int amount)
    {
        Experience.Add(amount);
        
        if (Experience.Value >= GetExperienceForNextLevel())
        {
            LevelUp();
        }
    }
    
    private void LevelUp()
    {
        Level.Add(1);
        Health.Set(100.0f); // Restore health on level up
        Experience.Set(0);
    }
}
```

### Client profile:
```csharp
public class ProfileUI : MonoBehaviour
{
    [Header("UI References")]
    public Text levelText;
    public Text coinsText;
    public Text healthText;
    
    private ObservableProfile profile;
    
    void Start()
    {
        // Request profile
        Mst.Client.Connection.SendMessage(MstOpCodes.ClientFillInProfileValues);
        
        // Register update handler
        Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.UpdateClientProfile, OnProfileUpdate);
    }
    
    private void OnProfileUpdate(IIncomingMessage message)
    {
        if (profile == null)
            profile = new ObservableProfile();
            
        profile.ApplyUpdates(message.AsBytes());
        
        // Update UI
        UpdateUI();
    }
    
    private void UpdateUI()
    {
        levelText.text = $"Level: {profile.GetProperty<ObservableInt>("playerLevel").Value}";
        coinsText.text = $"Coins: {profile.GetProperty<ObservableInt>("coins").Value}";
        healthText.text = $"Health: {profile.GetProperty<ObservableFloat>("health").Value}";
    }
}
```

## Best Practices

1. **Use populators** to initialize profiles
2. **Group updates** to reduce load
3. **Configure debounce** to optimize performance
4. **Check update size** to prevent attacks
5. **Use typed properties** for safety
6. **Subscribe to events** for reactive programming
7. **Clean up unused profiles** to free memory
8. **Implement backups** for important data
