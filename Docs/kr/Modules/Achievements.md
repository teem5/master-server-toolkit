# Master Server Toolkit - Achievements

## 설명
플레이어의 업적을 생성, 추적 및 잠금 해제하기위한 성취 모듈.진행 상황을 유지하기 위해 사용자 프로파일과 통합됩니다.

## AchievementsModule

성취 모듈의 주요 클래스.

### 설정 :
```csharp
[Header("Permission")]
[SerializeField] protected bool clientCanUpdateProgress = false;

[Header("Settings")]
[SerializeField] protected AchievementsDatabase achievementsDatabase;
```

### 종속성 :
모듈에는 다음이 필요합니다.
- 인증 - 사용자 인증을 위해
- 프로파일 모듈 - 업적의 진행을 보존합니다

## 성과 데이터베이스를 만듭니다

### AchievementsDatabase:
```csharp
// Создание ScriptableObject с достижениями
[CreateAssetMenu(menuName = "Master Server Toolkit/Achievements Database")]
public class AchievementsDatabase : ScriptableObject
{
    [SerializeField] private List<AchievementData> achievements = new List<AchievementData>();
    
    public IReadOnlyCollection<AchievementData> Achievements => achievements;
}
```

### 업적 정의 :
```csharp
// Создание в Unity Editor
var database = ScriptableObject.CreateInstance<AchievementsDatabase>();

// Добавление достижений
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

// Сохранение базы достижений
AssetDatabase.CreateAsset(database, "Assets/Resources/AchievementsDatabase.asset");
AssetDatabase.SaveAssets();
```

## 업적 유형

### AchievementType:
```csharp
public enum AchievementType
{
    Normal,         // Обычное достижение (0/1)
    Incremental,    // Постепенное достижение (0/N)
    Infinite        // Бесконечное достижение (трекинг без разблокировки)
}
```

### 성취 데이터 구조 :
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

## 업적 진행 상황 업데이트

서버에서 ### :
```csharp
// Получить модуль
var achievementsModule = Mst.Server.Modules.GetModule<AchievementsModule>();

// Обновить прогресс достижения
void UpdateAchievement(string userId, string achievementKey, int progress)
{
    var packet = new UpdateAchievementProgressPacket
    {
        userId = userId, 
        key = achievementKey,
        progress = progress
    };
    
    // Отправить пакет обновления
    Mst.Server.SendMessage(MstOpCodes.ServerUpdateAchievementProgress, packet);
}

// Пример использования
UpdateAchievement(user.UserId, "first_victory", 1);
```

클라이언트 측의 ### (허용 된 경우) :
```csharp
// Клиентский запрос на обновление прогресса
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

## 프로파일과 통합

성과는 프로파일 속성을 통해 사용자 프로파일과 자동으로 동기화됩니다.

```csharp
// Регистрация свойства достижений в ProfilesModule
profilesModule.RegisterProperty(ProfilePropertyOpCodes.achievements, null, () =>
{
    return new ObservableAchievements();
});

// Получение достижений из профиля
void GetAchievements(ObservableServerProfile profile)
{
    if (profile.TryGet(ProfilePropertyOpCodes.achievements, out ObservableAchievements achievements))
    {
        // Список всех достижений пользователя
        var allAchievements = achievements.GetAll();
        
        // Проверка разблокировано ли достижение
        bool isUnlocked = achievements.IsUnlocked("first_victory");
        
        // Получить прогресс достижения
        int progress = achievements.GetProgress("win_streak");
    }
}
```

## 추적 이벤트 잠금 해제

고객의 이벤트에 대한 ### 구독 :
```csharp
// В клиентском классе
private void Start()
{
    Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.ClientAchievementUnlocked, OnAchievementUnlocked);
}

private void OnAchievementUnlocked(IIncomingMessage message)
{
    string achievementKey = message.AsString();
    Debug.Log($"Achievement unlocked: {achievementKey}");
    
    // Показать интерфейс разблокировки достижения
    ShowAchievementUnlockedUI(achievementKey);
}
```

## 성과에 대한 상 설정

모듈을 사용하면 잠금 해제시 실행될 명령을 구성 할 수 있습니다.

```csharp
// Настройка ResultCommands в достижении
var achievement = new AchievementData
{
    Key = "100_matches_played",
    Title = "Veteran",
    Description = "Play 100 matches",
    Type = AchievementType.Incremental,
    MaxProgress = 100,
    Unlockable = true,
    
    // Команды для выполнения при разблокировке
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

### 팀 처리 :
```csharp
// Расширение модуля для обработки команд
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
                
                // Другие команды
                default:
                    logger.Error($"Unknown achievement command: {command.CommandKey}");
                    break;
            }
        }
    }
    
    private void AddCurrency(IUserPeerExtension user, string currencyType, int amount)
    {
        // Добавление валюты игроку
    }
    
    private void UnlockAvatar(IUserPeerExtension user, string avatarId)
    {
        // Разблокировка аватара игроку
    }
}
```

## 클라이언트 랩

### AchievementsModuleClient:
```csharp
// Пример использования
var client = Mst.Client.Modules.GetModule<AchievementsModuleClient>();

// Получение списка достижений
var achievements = client.GetAchievements();

// Отображение интерфейса
void ShowAchievementsUI()
{
    foreach (var achievement in achievements)
    {
        // Создать элемент интерфейса для достижения
        var item = Instantiate(achievementItemPrefab, container);
        
        // Заполнить данными
        item.SetTitle(achievement.Title);
        item.SetDescription(achievement.Description);
        item.SetIcon(achievement.Icon);
        item.SetProgress(achievement.CurrentProgress, achievement.MaxProgress);
        item.SetUnlocked(achievement.IsUnlocked);
    }
}
```

## 모범 사례

1. ** 각 업적에 대해 고유 키 **를 사용하십시오
2. ** 별도의 단일 및 부수적 ** 업적
3. ** 속임수를 방지하기 위해 서버 측에서 확인을 소개합니다.
4. ** 서버에서 상을 수상한 논리를 수행 **
5. ** 클라이언트에 대한 성과를 캐싱 ** 빠른 액세스
6. ** 비슷한 업적 잠금 해제 ** 자동으로 (예를 들어, "10 명의 몬스터 킬"을 자동으로 잠금 해제 "5 몬스터 킬")
7. ** 게임 플레이를 평가하기 위해 성취하여 분석 수집 **
