# Master Server Toolkit - Quests

## 설명
작업 체인, 임시 제한 및 상을 설정할 가능성이있는 게임 퀘스트를 작성, 추적 및 수행하기위한 모듈.

## 주요 구조

### QuestData (ScriptableObject)
```csharp
[CreateAssetMenu(menuName = "Master Server Toolkit/Quests/New Quest")]
public class QuestData : ScriptableObject
{
    // Основная информация
    public string Key => key;                 // Уникальный ключ квеста
    public string Title => title;             // Название
    public string Description => description; // Описание
    public int RequiredProgress => requiredProgress; // Кол-во для завершения
    public Sprite Icon => icon;               // Иконка квеста
    
    // Сообщения для разных статусов
    public string StartMessage => startMessage;       // Сообщение при взятии квеста
    public string ActiveMessage => activeMessage;     // Сообщение во время выполнения
    public string CompletedMessage => completedMessage; // Сообщение при завершении
    public string CancelMessage => cancelMessage;      // Сообщение при отмене
    public string ExpireMessage => expireMessage;      // Сообщение при истечении срока
    
    // Настройки
    public bool IsOneTime => isOneTime;               // Одноразовый квест
    public int TimeToComplete => timeToComplete;      // Время на выполнение (мин)
    
    // Связи с другими квестами
    public QuestData ParentQuest => parentQuest;       // Родительский квест
    public QuestData[] ChildrenQuests => childrenQuests; // Дочерние квесты
}
```

### 퀘스트 상태
```csharp
public enum QuestStatus 
{ 
    Inactive,   // Квест неактивен
    Active,     // Квест активен
    Completed,  // Квест завершен
    Canceled,   // Квест отменен
    Expired     // Время квеста истекло
}
```

### iquestinfo 인터페이스
```csharp
public interface IQuestInfo
{
    string Id { get; set; }                 // Уникальный ID
    string Key { get; set; }                // Ключ квеста
    string UserId { get; set; }             // ID пользователя
    int Progress { get; set; }              // Текущий прогресс
    int Required { get; set; }              // Требуемый прогресс
    DateTime StartTime { get; set; }        // Время начала
    DateTime ExpireTime { get; set; }       // Время истечения
    DateTime CompleteTime { get; set; }     // Время завершения
    QuestStatus Status { get; set; }        // Статус
    string ParentQuestKey { get; set; }     // Ключ родительского квеста
    string ChildrenQuestsKeys { get; set; } // Ключи дочерних квестов
    bool TryToComplete(int progress);       // Метод для завершения квеста
}
```

## QuestsModule (서버)

```csharp
// Зависимости
AddDependency<AuthModule>();
AddDependency<ProfilesModule>();

// Настройки
[Header("Permission"), SerializeField]
protected bool clientCanUpdateProgress = false; // Может ли клиент обновлять прогресс

[Header("Settings"), SerializeField]
protected QuestsDatabase[] questsDatabases; // Базы данных квестов
```

### 기본 서버 작업
1. 퀘스트 목록을 얻습니다
2. 퀘스트의 시작
3. 퀘스트의 진행 상황 업데이트
4. 퀘스트의 폐지
5. 퀘스트 기간의 만료를 확인하십시오

### 프로필과 통합
```csharp
private void ProfilesModule_OnProfileLoaded(ObservableServerProfile profile)
{
    if (profile.TryGet(ProfilePropertyOpCodes.quests, out ObservableQuests property))
    {
        // Инициализация квестов пользователя
    }
}
```

## quesmoduleclient (클라이언트)

```csharp
// Получение списка доступных квестов
questsClient.GetQuests((quests) => {
    foreach (var quest in quests)
    {
        Debug.Log($"Quest: {quest.Title}, Status: {quest.Status}");
    }
});

// Начать квест
questsClient.StartQuest("quest_key", (isStarted, quest) => {
    if (isStarted)
        Debug.Log($"Started quest: {quest.Title}");
});

// Обновить прогресс квеста
questsClient.UpdateQuestProgress("quest_key", 5, (isUpdated) => {
    if (isUpdated)
        Debug.Log("Quest progress updated");
});

// Отменить квест
questsClient.CancelQuest("quest_key", (isCanceled) => {
    if (isCanceled)
        Debug.Log("Quest canceled");
});
```

## 퀘스트 생성의 예

### 퀘스트 데이터베이스를 만듭니다
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

## 퀘스트 생성
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
    TimeToComplete = 60 // 60 минут на выполнение
};
```

### 퀘스트 체인
```csharp
// Создание зависимостей между квестами
var mainQuest = ScriptableObject.CreateInstance<QuestData>();
mainQuest.name = "MainQuest";

var subQuest1 = ScriptableObject.CreateInstance<QuestData>();
subQuest1.name = "SubQuest1";
// Установка mainQuest как родительского для subQuest1

var subQuest2 = ScriptableObject.CreateInstance<QuestData>();
subQuest2.name = "SubQuest2";
// Установка mainQuest как родительского для subQuest2

// В родительском квесте указываем дочерние
// mainQuest.ChildrenQuests = new QuestData[] { subQuest1, subQuest2 };
```

## 모범 사례

1. ** 가장 좋은 식별을 위해 의미있는 퀘스트 키 생성 **
2. ** Group Quests ** 주제별 데이터베이스
3. ** 퀘스트 실행의 현실적인 마감일을 결정하십시오.
4. ** 퀘스트 체인 사용 ** 플롯 라인을 만듭니다.
5. ** 퀘스트를 처리 할 때 실패 허용 오차를 제공합니다. **
6. ** 명확한 메시지 제공 ** 각 퀘스트 상태에 대해.
