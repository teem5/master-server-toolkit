# Master Server Toolkit - Matchmaker

## 설명
기준별로 게임, 객실 및 로비를 찾는 모듈뿐만 아니라 서버의 사용 가능한 지역에 대한 정보.

## 주요 구성 요소

### matchmakermodule (서버)
```csharp
// Зависимости
AddOptionalDependency<LobbiesModule>();
AddOptionalDependency<RoomsModule>();
AddOptionalDependency<SpawnersModule>();

// Основные методы
public void AddProvider(IGamesProvider provider) // Добавить провайдер игр
```

### mstmatchmakerclient (클라이언트)
```csharp
// Поиск игр (без фильтров)
matchmaker.FindGames((games) => {
    Debug.Log($"Found {games.Count} games");
});

// Поиск игр с фильтрами
var filters = new MstProperties();
filters.Set("minPlayers", 2);
matchmaker.FindGames(filters, (games) => { });

// Получение регионов
matchmaker.GetRegions((regions) => {
    // regions[0].Name, regions[0].Ip, regions[0].PingTime
});
```

## 키 데이터 구조

### GameInfoPacket
```csharp
public class GameInfoPacket
{
    public int Id { get; set; }                    // ID игры
    public string Address { get; set; }            // Адрес подключения
    public GameInfoType Type { get; set; }         // Тип (Room, Lobby, Custom)
    public string Name { get; set; }               // Название
    public string Region { get; set; }             // Регион
    public bool IsPasswordProtected { get; set; }  // Требуется пароль
    public int MaxPlayers { get; set; }            // Максимум игроков
    public int OnlinePlayers { get; set; }         // Текущее число игроков
    public List<string> OnlinePlayersList { get; } // Список игроков
    public MstProperties Properties { get; set; }  // Доп. свойства
}
```

### RegionInfo
```csharp
public class RegionInfo
{
    public string Name { get; set; }  // Название региона
    public string Ip { get; set; }    // IP-адрес
    public int PingTime { get; set; } // Пинг (мс)
}
```

## CASET GAMI 제공 업체

### Igamesprovider 인터페이스
```csharp
public interface IGamesProvider
{
    IEnumerable<GameInfoPacket> GetPublicGames(IPeer peer, MstProperties filters);
}
```

### 최소 구현의 예
```csharp
public class CustomGamesProvider : MonoBehaviour, IGamesProvider
{
    private List<GameInfoPacket> games = new List<GameInfoPacket>();
    
    public IEnumerable<GameInfoPacket> GetPublicGames(IPeer peer, MstProperties filters)
    {
        // Фильтрация по региону
        if (filters.Has("region"))
        {
            return games.Where(g => g.Region == filters.AsString("region"));
        }
        
        return games;
    }
    
    // API для добавления/удаления игр
    public void AddGame(GameInfoPacket game) => games.Add(game);
    public void RemoveGame(int gameId) => games.RemoveAll(g => g.Id == gameId);
}

// Регистрация
matchmaker.AddProvider(gameObject.AddComponent<CustomGamesProvider>());
```

## 사용의 예

### 여러 필터가있는 게임을 검색하십시오
```csharp
var filters = new MstProperties();
filters.Set("region", "eu-west");     // Только европейский регион
filters.Set("minPlayers", 1);         // С минимум 1 игроком
filters.Set("gameMode", "deathmatch"); // Режим "deathmatch"

matchmaker.FindGames(filters, (games) =>
{
    // Найденные игры
    foreach (var game in games)
    {
        Debug.Log($"{game.Name} - {game.OnlinePlayers}/{game.MaxPlayers}");
        
        // Подключение к игре
        Mst.Client.Rooms.JoinRoom(game.Id, "", (successful, roomAccess) =>
        {
            if (successful)
                ConnectToGameServer(roomAccess);
        });
    }
});
```

### 최고의 핑이있는 지역의 선택
```csharp
matchmaker.GetRegions((regions) =>
{
    if (regions.Count > 0)
    {
        // Сортировка по пингу
        var bestRegion = regions.OrderBy(r => r.PingTime).First();
        PlayerPrefs.SetString("SelectedRegion", bestRegion.Name);
        
        Debug.Log($"Using region: {bestRegion.Name} ({bestRegion.PingTime}ms)");
    }
});
```

## 모범 사례

1. ** 필터링에 의미있는 속성을 사용하십시오 **
2. ** 최고의 필터 조직의 속성에 카테고리 추가 **
3. ** 클라이언트 코드에서 게임 부족의 경우 **
4. ** 분류 결과 ** 사용자 경험을 향상시킵니다
5. ** 지역 목록 ** 및 필요한 경우 검색 결과를 현금화하십시오.
6. ** 지역 목록 업데이트 ** 게임을 시작할 때
