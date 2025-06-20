# Master Server Toolkit - Notification

## Description
Notification module for sending messages from the server to clients. It can notify individual users, groups within rooms, or all connected users.

## NotificationModule

Main server class of the notification module.

### Setup:
```csharp
[Header("General Settings")]
[SerializeField, Tooltip("If true, notification module will subscribe to auth module, and automatically setup recipients when they log in")]
protected bool useAuthModule = true;
[SerializeField, Tooltip("If true, notification module will subscribe to rooms module to be able to send notifications to room players")]
protected bool useRoomsModule = true;
[SerializeField, Tooltip("Permission level to be able to send notifications")]
protected int notifyPermissionLevel = 1;
[SerializeField]
private int maxPromisedMessages = 10;
```

### Dependencies:
- AuthModule (optional) - automatically adds users to the recipient list when they log in
- RoomsModule (optional) - used to send notifications to players in rooms

## Core Methods

### Sending notifications:
```csharp
// Get the module
var notificationModule = Mst.Server.Modules.GetModule<NotificationModule>();

// Send to all users
notificationModule.NoticeToAll("The server will restart in 5 minutes");

// Send to all and save as a promised message (new users will get it when they log in)
notificationModule.NoticeToAll("Welcome to our world!", true);

// Send to a specific user (by peer ID)
notificationModule.NoticeToRecipient(peerId, "You earned a new achievement");

// Send to a group of users
List<int> peerIds = new List<int> { 123, 456, 789 };
notificationModule.NoticeToRecipients(peerIds, "A new group quest is available");

// Send to all users in the room
notificationModule.NoticeToRoom(roomId, new List<int>(), "The room will close in 2 minutes");

// Send to everyone in the room except specified users
List<int> ignorePeerIds = new List<int> { 123 };
notificationModule.NoticeToRoom(roomId, ignorePeerIds, "A player joined the room");
```

### Managing recipients:
```csharp
// Check if a recipient exists
bool hasUser = notificationModule.HasRecipient(userId);

// Get the recipient
NotificationRecipient recipient = notificationModule.GetRecipient(userId);

// Try to get the recipient
if (notificationModule.TryGetRecipient(userId, out NotificationRecipient recipient))
{
    // Send a notification
    recipient.Notify("Personal message");
}

// Add a recipient manually
NotificationRecipient newRecipient = notificationModule.AddRecipient(userExtension);

// Remove a recipient
notificationModule.RemoveRecipient(userId);
```

## Клиентская часть - MstNotificationClient

```csharp
// Get the client
var notificationClient = Mst.Client.Notifications;

// Subscribe to notifications
notificationClient.Subscribe((isSuccess, error) =>
{
    if (isSuccess)
    {
        Debug.Log("Successfully subscribed to notifications");
    }
    else
    {
        Debug.LogError($"Failed to subscribe to notifications: {error}");
    }
});

// Subscribe to the notification received event
notificationClient.OnNotificationReceivedEvent += OnNotificationReceived;

// Notification handler
private void OnNotificationReceived(string message)
{
    // Show notification to the user
    uiNotificationManager.ShowNotification(message);
}

// Unsubscribe from notifications
notificationClient.Unsubscribe((isSuccess, error) =>
{
    if (isSuccess)
    {
        Debug.Log("Successfully unsubscribed from notifications");
    }
    else
    {
        Debug.LogError($"Error unsubscribing from notifications: {error}");
    }
});

// Unsubscribe from the event
notificationClient.OnNotificationReceivedEvent -= OnNotificationReceived;
```

## Пакеты и структуры

### NotificationPacket:
```csharp
public class NotificationPacket : SerializablePacket
{
    public int RoomId { get; set; } = -1;               // ID комнаты (если отправляется в комнату)
    public string Message { get; set; } = string.Empty; // Текст уведомления
    public List<int> Recipients { get; set; } = new List<int>();        // Список получателей
    public List<int> IgnoreRecipients { get; set; } = new List<int>();  // Исключения
}
```

### NotificationRecipient:
```csharp
public class NotificationRecipient
{
    public string UserId { get; set; }
    public IPeer Peer { get; set; }
    
    // Отправка уведомления конкретному получателю
    public void Notify(string message)
    {
        Peer.SendMessage(MstOpCodes.Notification, message);
    }
}
```

## Серверная реализация - Пользовательский модуль уведомлений

Пример создания расширенного модуля уведомлений:

```csharp
public class GameNotificationModule : NotificationModule
{
    // Форматированные уведомления
    public void SendSystemNotification(string message)
    {
        string formattedMessage = $"[СИСТЕМА]: {message}";
        NoticeToAll(formattedMessage);
    }
    
    public void SendAdminNotification(string message, string adminName)
    {
        string formattedMessage = $"[АДМИН - {adminName}]: {message}";
        NoticeToAll(formattedMessage, true); // Сохраняем как обещанное сообщение
    }
    
    public void SendAchievementNotification(int peerId, string achievementTitle)
    {
        string formattedMessage = $"[ДОСТИЖЕНИЕ]: Вы получили '{achievementTitle}'!";
        NoticeToRecipient(peerId, formattedMessage);
    }
    
    // Уведомления с json-данными
    public void SendJSONNotification(int peerId, string type, object data)
    {
        var notification = new JSONNotification
        {
            Type = type,
            Data = JsonUtility.ToJson(data)
        };
        
        string jsonMessage = JsonUtility.ToJson(notification);
        NoticeToRecipient(peerId, jsonMessage);
    }
    
    // Класс для json-уведомлений
    [Serializable]
    private class JSONNotification
    {
        public string Type;
        public string Data;
    }
}
```

## Интеграция с UI

Пример обработки уведомлений в пользовательском интерфейсе:

```csharp
public class NotificationUIManager : MonoBehaviour
{
    [SerializeField] private GameObject notificationPrefab;
    [SerializeField] private Transform notificationsContainer;
    [SerializeField] private float displayTime = 5f;
    
    private void Start()
    {
        // Получить клиент уведомлений
        var notificationClient = Mst.Client.Notifications;
        
        // Подписаться на события
        notificationClient.OnNotificationReceivedEvent += ShowNotification;
        
        // Подписаться на получение уведомлений от сервера
        notificationClient.Subscribe((isSuccess, error) =>
        {
            if (!isSuccess)
            {
                Debug.LogError($"Failed to subscribe to notifications: {error}");
            }
        });
    }
    
    // Обработка простого текстового уведомления
    public void ShowNotification(string message)
    {
        // Проверка на JSON
        if (message.StartsWith("{") && message.EndsWith("}"))
        {
            try
            {
                // Пытаемся распарсить как JSON
                JsonNotification notification = JsonUtility.FromJson<JsonNotification>(message);
                
                // Отобразить в зависимости от типа
                switch (notification.Type)
                {
                    case "achievement":
                        ShowAchievementNotification(notification.Data);
                        break;
                    case "system":
                        ShowSystemNotification(notification.Data);
                        break;
                    default:
                        CreateTextNotification(message);
                        break;
                }
            }
            catch
            {
                // Если не JSON, отображаем как обычный текст
                CreateTextNotification(message);
            }
        }
        else
        {
            // Обычное текстовое сообщение
            CreateTextNotification(message);
        }
    }
    
    // Создание уведомления в UI
    private void CreateTextNotification(string text)
    {
        GameObject notification = Instantiate(notificationPrefab, notificationsContainer);
        notification.GetComponentInChildren<TextMeshProUGUI>().text = text;
        
        // Автоматически уничтожить через время
        Destroy(notification, displayTime);
    }
    
    // Кастомные обработчики для специальных уведомлений
    private void ShowAchievementNotification(string data)
    {
        // Кастомная логика для отображения уведомления о достижении
    }
    
    private void ShowSystemNotification(string data)
    {
        // Кастомная логика для отображения системного уведомления
    }
    
    [Serializable]
    private class JsonNotification
    {
        public string Type;
        public string Data;
    }
    
    private void OnDestroy()
    {
        // Отписаться
        var notificationClient = Mst.Client.Notifications;
        if (notificationClient != null)
        {
            notificationClient.OnNotificationReceivedEvent -= ShowNotification;
        }
    }
}
```

## Пример использования из комнаты

```csharp
public class RoomManager : MonoBehaviour, IRoomManager
{
    // Отправить уведомление всем игрокам в комнате, когда один из игроков готов
    public void OnPlayerReadyStatusChanged(int peerId, bool isReady)
    {
        var player = Mst.Server.Rooms.GetPlayer(currentRoomId, peerId);
        
        if (player != null && isReady)
        {
            var username = player.GetExtension<IUserPeerExtension>()?.Username ?? "Unknown";
            
            // Создать пакет уведомления
            var packet = new NotificationPacket
            {
                RoomId = currentRoomId,
                Message = $"Игрок {username} готов к игре!",
                IgnoreRecipients = new List<int> { peerId } // Не отправлять самому игроку
            };
            
            // Отправить уведомление через сервер
            Mst.Server.Connection.SendMessage(MstOpCodes.Notification, packet);
        }
    }
}
```

## Обещанные сообщения

Особенность модуля уведомлений - возможность сохранять "обещанные сообщения", которые будут доставлены новым пользователям при входе в систему. Это полезно для системных объявлений, новостей, которые должны получить все игроки.

```csharp
// Отправить всем и сохранить как обещанное сообщение
notificationModule.NoticeToAll("Новое обновление игры! Версия 1.2.0 доступна!", true);
```

Configuring how many promised messages are stored:
```csharp
[SerializeField] private int maxPromisedMessages = 10;
```

## Best Practices

1. **Separate notification types** – use different formatting or prefixes for various kinds of messages
2. **Use JSON for complex notifications** when structured data is required
3. **Manage permission levels** – configure `notifyPermissionLevel` to restrict who can send notifications
4. **Integrate with other modules** – trigger notifications from other modules' events
5. **Create notification filters** on the client so users can choose what they want to see
6. **Do not overuse promised messages** – reserve them for truly important system announcements
7. **Provide localization** for international projects
