# Master Server Toolkit - Spawner

## 설명
로드 밸런싱 및 큐를 지원하는 다양한 지역에서 게임 서버를 시작하는 프로세스를 관리하기위한 모듈.

## 주요 구성 요소

### SpawnersModule
```csharp
// Настройки
[SerializeField] protected int createSpawnerPermissionLevel = 0; // Минимальный уровень прав для регистрации спаунера
[SerializeField] protected float queueUpdateFrequency = 0.1f;    // Частота обновления очередей
[SerializeField] protected bool enableClientSpawnRequests = true; // Разрешить запросы на спаун от клиентов

// События
public event Action<RegisteredSpawner> OnSpawnerRegisteredEvent;  // При регистрации спаунера
public event Action<RegisteredSpawner> OnSpawnerDestroyedEvent;   // При удалении спаунера
public event SpawnedProcessRegistrationHandler OnSpawnedProcessRegisteredEvent; // При регистрации процесса
```

### RegisteredSpawner
```csharp
// Основные свойства
public int SpawnerId { get; }                // Уникальный ID спаунера
public IPeer Peer { get; }                   // Подключение к спаунеру
public SpawnerOptions Options { get; }       // Настройки спаунера
public int ProcessesRunning { get; }         // Количество запущенных процессов
public int QueuedTasks { get; }              // Количество задач в очереди

// Методы
public bool CanSpawnAnotherProcess();        // Может ли запустить еще один процесс
public int CalculateFreeSlotsCount();        // Расчет количества свободных слотов
```

### SpawnerOptions
```csharp
// Настройки спаунера
public string MachineIp { get; set; }           // IP машины спаунера 
public int MaxProcesses { get; set; } = 5;      // Максимальное количество процессов
public string Region { get; set; } = "Global";  // Регион спаунера
public Dictionary<string, string> CustomOptions { get; set; } // Дополнительные настройки
```

### SpawnTask
```csharp
// Свойства
public int Id { get; }                      // ID задачи
public RegisteredSpawner Spawner { get; }   // Спаунер, выполняющий задачу
public MstProperties Options { get; }       // Настройки запуска
public SpawnStatus Status { get; }          // Текущий статус
public string UniqueCode { get; }           // Уникальный код для безопасности
public IPeer Requester { get; set; }        // Запросивший клиент
public IPeer RegisteredPeer { get; private set; } // Зарегистрированный процесс

// События
public event Action<SpawnStatus> OnStatusChangedEvent; // При изменении статуса
```

## spawnstatus 프로세스 상태
```csharp
public enum SpawnStatus
{
    None,               // Начальный статус
    Queued,             // В очереди
    ProcessStarted,     // Процесс запущен
    ProcessRegistered,  // Процесс зарегистрирован
    Finalized,          // Финализирован
    Aborted,            // Прерван
    Killed              // Убит
}
```

## 작업 과정

### 버스터 등록
```csharp
// Клиент
var options = new SpawnerOptions
{
    MachineIp = "192.168.1.10",
    MaxProcesses = 10,
    Region = "eu-west",
    CustomOptions = new Dictionary<string, string> {
        { "mapsList", "map1,map2,map3" }
    }
};

Mst.Client.Spawners.RegisterSpawner(options, (successful, spawnerId) =>
{
    if (successful)
        Debug.Log($"Spawner registered with ID: {spawnerId}");
});

// Сервер
RegisteredSpawner spawner = CreateSpawner(peer, options);
```

### 게임 서버 출시 요청
```csharp
// Клиент
var spawnOptions = new MstProperties();
spawnOptions.Set(Mst.Args.Names.RoomName, "MyGame");
spawnOptions.Set(Mst.Args.Names.RoomMaxPlayers, 10);
spawnOptions.Set(Mst.Args.Names.RoomRegion, "eu-west");

Mst.Client.Spawners.RequestSpawn(spawnOptions, (successful, spawnId) =>
{
    if (successful)
    {
        Debug.Log($"Spawn request created with ID: {spawnId}");
        
        // Подписка на изменения статуса
        Mst.Client.Spawners.OnStatusChangedEvent += OnSpawnStatusChanged;
    }
});

// Сервер
SpawnTask task = Spawn(options, region);
```

### 출시를위한 스파이너 선택
```csharp
// Фильтрация спаунеров по региону
var spawners = GetSpawnersInRegion(region);

// Сортировка по доступным слотам
var availableSpawners = spawners
    .OrderByDescending(s => s.CalculateFreeSlotsCount())
    .Where(s => s.CanSpawnAnotherProcess())
    .ToList();

// Выбор спаунера
if (availableSpawners.Count > 0)
{
    return availableSpawners[0];
}
```

### 스폰 프로세스 완료
```csharp
// На стороне процесса
var finalizationData = new MstProperties();
finalizationData.Set("roomId", 12345);
finalizationData.Set("connectionAddress", "192.168.1.10:12345");

// Отправка данных о завершении
Mst.Client.Spawners.FinalizeSpawn(spawnTaskId, finalizationData);

// На стороне клиента
Mst.Client.Spawners.GetFinalizationData(spawnId, (successful, data) =>
{
    if (successful)
    {
        string connectionAddress = data.AsString("connectionAddress");
        int roomId = data.AsInt("roomId");
        
        // Подключение к комнате
        ConnectToRoom(connectionAddress, roomId);
    }
});
```

## 지역 사무소

```csharp
// Получение всех регионов
List<RegionInfo> regions = spawnersModule.GetRegions();

// Получение спаунеров в конкретном регионе
List<RegisteredSpawner> regionalSpawners = spawnersModule.GetSpawnersInRegion("eu-west");

// Создание спаунера в регионе
var options = new SpawnerOptions { Region = "eu-west" };
var spawner = spawnersModule.CreateSpawner(peer, options);
```

## 밸런싱 하중

```csharp
// Пример балансировки по наименее загруженным серверам
public RegisteredSpawner GetLeastBusySpawner(string region)
{
    var spawners = GetSpawnersInRegion(region);
    
    // Если нет спаунеров в регионе, используем все доступные
    if (spawners.Count == 0)
        spawners = GetSpawners();
    
    // Сортировка по загруженности
    return spawners
        .OrderByDescending(s => s.CalculateFreeSlotsCount())
        .FirstOrDefault(s => s.CanSpawnAnotherProcess());
}

// Расширенная логика выбора спаунера
public RegisteredSpawner GetOptimalSpawner(MstProperties options)
{
    string region = options.AsString(Mst.Args.Names.RoomRegion, "");
    string gameMode = options.AsString("gameMode", "");
    
    // Фильтрация по региону и игровому режиму
    var filtered = spawnersList.Values
        .Where(s => (string.IsNullOrEmpty(region) || s.Options.Region == region) &&
                  (string.IsNullOrEmpty(gameMode) || 
                   s.Options.CustomOptions.ContainsKey("gameModes") && 
                   s.Options.CustomOptions["gameModes"].Contains(gameMode)))
        .ToList();
    
    return filtered
        .OrderByDescending(s => s.CalculateFreeSlotsCount())
        .FirstOrDefault(s => s.CanSpawnAnotherProcess());
}
```

## 실용적인 예

### 시스템 설정 :
```csharp
// 1. Регистрация спаунеров
RegisterSpawner("EU", "10.0.0.1", 10);
RegisterSpawner("US", "10.0.1.1", 15);
RegisterSpawner("ASIA", "10.0.2.1", 8);

// 2. Клиентский запрос на создание игрового сервера
var options = new MstProperties();
options.Set(Mst.Args.Names.RoomName, "CustomGame");
options.Set(Mst.Args.Names.RoomMaxPlayers, 16);
options.Set(Mst.Args.Names.RoomRegion, "EU");
options.Set("gameMode", "deathmatch");

// 3. Обработка запроса, выбор спаунера и запуск процесса
SpawnTask task = spawnersModule.Spawn(options, "EU");

// 4. Клиент ожидает завершения процесса
// 5. Созданный процесс регистрируется и отправляет данные для подключения
```

## 모범 사례

1. ** 지역별 그룹 스파이너 ** 최적의 지연을위한
2. ** 서버의 전력을 고려하여 각 스파이너마다 프로세스 한계 설정 **
3. ** 유연한 튜닝 스패너에는 사용자 정의 옵션 사용 **
4. ** 고장 공차 향상 ** - 스파이너를 사용할 수없는 경우 작업을 다른 지역으로 리디렉션하십시오.
5. ** 좁은 장소를 감지하기 위해 스파이너의 작업량 **
6. ** 작업에 대한 타임 아웃 추가 ** 멈추지 않도록 줄을 서십시오.
