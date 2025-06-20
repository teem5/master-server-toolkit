# Master Server Toolkit - Quests

## Description
Module for creating, tracking and completing game quests with support for quest chains, time limits and rewards.

## Main Structures

### QuestData (ScriptableObject)
```csharp
[CreateAssetMenu(menuName = "Master Server Toolkit/Quests/New Quest")]
public class QuestData : ScriptableObject
{
    // Basic information
    public string Key => key;                 // Unique quest key
    public string Title => title;             // Title
    public string Description => description; // Description
    public int RequiredProgress => requiredProgress; // Amount to complete
    public Sprite Icon => icon;               // Quest icon
    
    // Messages for different states
    public string StartMessage => startMessage;       // When the quest is taken
    public string ActiveMessage => activeMessage;     // While active
    public string CompletedMessage => completedMessage; // Upon completion
    public string CancelMessage => cancelMessage;      // On cancel
    public string ExpireMessage => expireMessage;      // When expired
    
    // Settings
    public bool IsOneTime => isOneTime;               // One-time quest
    public int TimeToComplete => timeToComplete;      // Time to complete (min)
    
    // Connections with other quests
    public QuestData ParentQuest => parentQuest;       // Parent quest
    public QuestData[] ChildrenQuests => childrenQuests; // Child quests
}
```

### Quest statuses
```csharp
public enum QuestStatus 
{ 
    Inactive,   // Quest is inactive
    Active,     // Quest is active
    Completed,  // Quest completed
    Canceled,   // Quest canceled
    Expired     // Quest time expired
}
```

### IQuestInfo interface
```csharp
public interface IQuestInfo
{
    string Id { get; set; }                 // Unique ID
    string Key { get; set; }                // Quest key
    string UserId { get; set; }             // User ID
    int Progress { get; set; }              // Current progress
    int Required { get; set; }              // Required progress
    DateTime StartTime { get; set; }        // Start time
    DateTime ExpireTime { get; set; }       // Expiration time
    DateTime CompleteTime { get; set; }     // Completion time
    QuestStatus Status { get; set; }        // Status
    string ParentQuestKey { get; set; }     // Parent quest key
    string ChildrenQuestsKeys { get; set; } // Child quest keys
    bool TryToComplete(int progress);       // Method to complete the quest
}
```

## QuestsModule (Сервер)

```csharp
// Зависимости
AddDependency<AuthModule>();
AddDependency<ProfilesModule>();

// Настройки
[Header("Permission"), SerializeField]
protected bool clientCanUpdateProgress = false; // Может ли клиент обновлять прогресс

[Header("Settings"), SerializeField]
protected QuestsDatabase[] questsDatabases; // Quest databases
```

### Main server operations
1. Retrieve the list of quests
2. Start a quest
3. Update quest progress
4. Cancel a quest
5. Check quest expiration

### Profile integration
```csharp
private void ProfilesModule_OnProfileLoaded(ObservableServerProfile profile)
{
    if (profile.TryGet(ProfilePropertyOpCodes.quests, out ObservableQuests property))
    {
        // Initialize player quests
    }
}
```

## QuestsModuleClient (Client)

```csharp
// Get available quests
questsClient.GetQuests((quests) => {
    foreach (var quest in quests)
    {
        Debug.Log($"Quest: {quest.Title}, Status: {quest.Status}");
    }
});

// Start a quest
questsClient.StartQuest("quest_key", (isStarted, quest) => {
    if (isStarted)
        Debug.Log($"Started quest: {quest.Title}");
});

// Update quest progress
questsClient.UpdateQuestProgress("quest_key", 5, (isUpdated) => {
    if (isUpdated)
        Debug.Log("Quest progress updated");
});

// Cancel a quest
questsClient.CancelQuest("quest_key", (isCanceled) => {
    if (isCanceled)
        Debug.Log("Quest canceled");
});
```

## Пример создания квестов

### Создание базы данных квестов
```csharp
// В редакторе Unity
[CreateAssetMenu(menuName = "Master Server Toolkit/Quests/QuestsDatabase")]
public class QuestsDatabase : ScriptableObject
{
    [SerializeField]
    private List<QuestData> quests = new List<QuestData>();
    
    public IReadOnlyCollection<QuestData> Quests => quests;
}

// Создание базы данных
var database = ScriptableObject.CreateInstance<QuestsDatabase>();
```

### Создание квеста
```csharp
// В редакторе Unity
var quest = ScriptableObject.CreateInstance<QuestData>();
quest.name = "CollectResources";
// Настройка квеста через инспектор

// Программно
var questData = new QuestData
{
    Key = "collect_wood",
    Title = "Collect Wood",
    Description = "Collect 10 pieces of wood",
    RequiredProgress = 10,
    TimeToComplete = 60 // 60 minutes to complete
};
```

### Цепочки квестов
```csharp
// Creating dependencies between quests
var mainQuest = ScriptableObject.CreateInstance<QuestData>();
mainQuest.name = "MainQuest";

var subQuest1 = ScriptableObject.CreateInstance<QuestData>();
subQuest1.name = "SubQuest1";
// Set mainQuest as the parent for subQuest1

var subQuest2 = ScriptableObject.CreateInstance<QuestData>();
subQuest2.name = "SubQuest2";
// Set mainQuest as the parent for subQuest2

// Specify child quests in the parent
// mainQuest.ChildrenQuests = new QuestData[] { subQuest1, subQuest2 };
```

## Best Practices

1. **Create meaningful quest keys** for better identification
2. **Group quests** into thematic databases
3. **Set realistic deadlines** for completing quests
4. **Use quest chains** to build storylines
5. **Ensure fault tolerance** when processing quests
6. **Provide clear messages** for each quest status
