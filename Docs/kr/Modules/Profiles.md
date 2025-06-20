# Master Server Toolkit - Profiles

## 설명
관리 데이터, 변경 관찰 및 고객과 서버 간의 동기화에 대한 프로파일 모듈.

## ProfilesModule

사용자 프로필 관리를위한 메인 클래스.

### 설정 :
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

## 프로필의 속성

## 속성 시스템 생성 :
```csharp
// Создание популятора
public class PlayerStatsPopulator : IObservablePropertyPopulator
{
    public IProperty Populate()
    {
        var properties = new ObservableBase();
        
        // Базовые статы
        properties.Set("playerLevel", new ObservableInt(1));
        properties.Set("experience", new ObservableInt(0));
        properties.Set("coins", new ObservableInt(0));
        
        // Словарь для инвентаря
        var inventory = new ObservableDictStringInt();
        inventory.Add("sword", 1);
        inventory.Add("potion", 5);
        properties.Set("inventory", inventory);
        
        return properties;
    }
}
```

## 프로필과 함께 작업하십시오

### 프로필에 대한 액세스 (클라이언트) :
```csharp
// Запрос профиля
Mst.Client.Connection.SendMessage(MstOpCodes.ClientFillInProfileValues);

// Подписка на обновления
Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.UpdateClientProfile, OnProfileUpdated);

// Обработка ответа
private void OnProfileUpdated(IIncomingMessage message)
{
    var profile = new ObservableProfile();
    profile.FromBytes(message.AsBytes());
    
    // Доступ к свойствам
    int level = profile.GetProperty<ObservableInt>("playerLevel").Value;
    int coins = profile.GetProperty<ObservableInt>("coins").Value;
}
```

### 프로필에 대한 액세스 (서버) :
```csharp
// Получение профиля по Id
var profile = profilesModule.GetProfileByUserId(userId);

// Получение профиля по Peer
var profile = profilesModule.GetProfileByPeer(peer);

// Изменение профиля
profile.GetProperty<ObservableInt>("playerLevel").Add(1);
profile.GetProperty<ObservableInt>("coins").Set(100);
```

## 프로필 이벤트

### 변경 사항 구독 :
```csharp
// На сервере
profilesModule.OnProfileCreated += OnProfileCreated;
profilesModule.OnProfileLoaded += OnProfileLoaded;

// В профиле
profile.OnModifiedInServerEvent += OnProfileChanged;

// Конкретное свойство
profile.GetProperty<ObservableInt>("playerLevel").OnDirtyEvent += OnLevelChanged;
```

## 데이터베이스와 동기화

## Selume iprofilesdatabaseaccessor :
```csharp
public class ProfilesDatabaseAccessor : IProfilesDatabaseAccessor
{
    public async Task RestoreProfileAsync(ObservableServerProfile profile)
    {
        // Загрузка из БД
        var data = await LoadProfileDataFromDB(profile.UserId);
        if (data != null)
        {
            profile.FromBytes(data);
        }
    }
    
    public async Task UpdateProfilesAsync(List<ObservableServerProfile> profiles)
    {
        // Batch сохранение в БД
        foreach (var profile in profiles)
        {
            await SaveProfileToDB(profile.UserId, profile.ToBytes());
        }
    }
}
```

## 관찰 된 속성의 유형

## 기본 유형 :
```csharp
// Числовые
ObservableInt level = new ObservableInt(10);
ObservableFloat health = new ObservableFloat(100.0f);

// Строки
ObservableString name = new ObservableString("Player");

// Списки
ObservableListInt scores = new ObservableListInt();
scores.Add(100);

// Словари
ObservableDictStringInt items = new ObservableDictStringInt();
items.Add("sword", 1);
```

## 서버 업데이트

### 게임 서버에서 업데이트 보내기 :
```csharp
// Создание пакета обновлений
var updates = new ProfileUpdatePacket();
updates.UserId = userId;
updates.Properties = new MstProperties();
updates.Properties.Set("playerLevel", 15);
updates.Properties.Set("experience", 1500);

// Отправка на master server
Mst.Server.Connection.SendMessage(MstOpCodes.ServerUpdateProfileValues, updates);
```

## 성능 및 최적화

### 분해 설정 :
```csharp
// Задержка сохранения в БД (секунды)
saveProfileDebounceTime = 1;

// Задержка отправки обновлений клиенту
clientUpdateDebounceTime = 0.5f;

// Время до выгрузки профиля после выхода
unloadProfileAfter = 20;
```

### 제한:
```csharp
// Максимальный размер обновления
maxUpdateSize = 1048576;

// Тайм-аут загрузки профиля
profileLoadTimeoutSeconds = 10;
```

## 사용의 예

### 게임 통계 :
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
        Health.Set(100.0f); // Восстановление здоровья при уровне
        Experience.Set(0);
    }
}
```

## 클라이언트 프로필 :
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
        // Запрос профиля
        Mst.Client.Connection.SendMessage(MstOpCodes.ClientFillInProfileValues);
        
        // Регистрация обработчика обновлений
        Mst.Client.Connection.RegisterMessageHandler(MstOpCodes.UpdateClientProfile, OnProfileUpdate);
    }
    
    private void OnProfileUpdate(IIncomingMessage message)
    {
        if (profile == null)
            profile = new ObservableProfile();
            
        profile.ApplyUpdates(message.AsBytes());
        
        // Обновление UI
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

## 모범 사례

1. ** 인구 **를 사용하여 프로파일을 초기화합니다
2. ** 그룹 업데이트 ** 하중을 줄입니다
3. ** 참조 디바운드 ** 성능을 최적화합니다
4. ** 공격을 방지하려면 업데이트 크기를 확인하십시오 **
5. ** 안전을 위해 일반적인 속성 **를 사용하십시오
6. ** 이벤트 구독 ** 반응성 프로그래밍을 위해
7. ** 사용하지 않는 프로파일을 청소하십시오 ** 메모리를 해제하십시오
8. ** 중요한 데이터의 경우 백업을 향상시킵니다
