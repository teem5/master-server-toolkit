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

## Client side - MstNotificationClient

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

## Packets and structures

### NotificationPacket:
```csharp
public class NotificationPacket : SerializablePacket
{
    public int RoomId { get; set; } = -1;               // Room ID (if sent to a room)
    public string Message { get; set; } = string.Empty; // Notification text
    public List<int> Recipients { get; set; } = new List<int>();        // List of recipients
    public List<int> IgnoreRecipients { get; set; } = new List<int>();  // Exclusions
}
```

### NotificationRecipient:
```csharp
public class NotificationRecipient
{
    public string UserId { get; set; }
    public IPeer Peer { get; set; }
    
    // Send notification to a specific recipient
    public void Notify(string message)
    {
        Peer.SendMessage(MstOpCodes.Notification, message);
    }
}
```

## Server implementation - Custom notification module

Example of creating an extended notification module:

```csharp
public class GameNotificationModule : NotificationModule
{
    // Formatted notifications
    public void SendSystemNotification(string message)
    {
        string formattedMessage = $"[SYSTEM]: {message}";
        NoticeToAll(formattedMessage);
    }
    
    public void SendAdminNotification(string message, string adminName)
    {
        string formattedMessage = $"[ADMIN - {adminName}]: {message}";
        NoticeToAll(formattedMessage, true); // Save as a promised message
    }
    
    public void SendAchievementNotification(int peerId, string achievementTitle)
    {
        string formattedMessage = $"[ACHIEVEMENT]: You received '{achievementTitle}'!";
        NoticeToRecipient(peerId, formattedMessage);
    }
    
    // Notifications with JSON data
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
    
    // Class for JSON notifications
    [Serializable]
    private class JSONNotification
    {
        public string Type;
        public string Data;
    }
}
```

## UI integration

Example of handling notifications in the user interface:

```csharp
public class NotificationUIManager : MonoBehaviour
{
    [SerializeField] private GameObject notificationPrefab;
    [SerializeField] private Transform notificationsContainer;
    [SerializeField] private float displayTime = 5f;
    
    private void Start()
    {
        // Get the notification client
        var notificationClient = Mst.Client.Notifications;
        
        // Subscribe to events
        notificationClient.OnNotificationReceivedEvent += ShowNotification;
        
        // Subscribe to notifications from the server
        notificationClient.Subscribe((isSuccess, error) =>
        {
            if (!isSuccess)
            {
                Debug.LogError($"Failed to subscribe to notifications: {error}");
            }
        });
    }
    
    // Handle a simple text notification
    public void ShowNotification(string message)
    {
        // Check for JSON
        if (message.StartsWith("{") && message.EndsWith("}"))
        {
            try
            {
                // Try to parse as JSON
                JsonNotification notification = JsonUtility.FromJson<JsonNotification>(message);
                
                // Display depending on the type
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
                // If not JSON, display as plain text
                CreateTextNotification(message);
            }
        }
        else
        {
            // Plain text message
            CreateTextNotification(message);
        }
    }
    
    // Create a notification in the UI
    private void CreateTextNotification(string text)
    {
        GameObject notification = Instantiate(notificationPrefab, notificationsContainer);
        notification.GetComponentInChildren<TextMeshProUGUI>().text = text;
        
        // Automatically destroy after a delay
        Destroy(notification, displayTime);
    }
    
    // Custom handlers for special notifications
    private void ShowAchievementNotification(string data)
    {
        // Custom logic to display an achievement notification
    }
    
    private void ShowSystemNotification(string data)
    {
        // Custom logic to display a system notification
    }
    
    [Serializable]
    private class JsonNotification
    {
        public string Type;
        public string Data;
    }
    
    private void OnDestroy()
    {
        // Unsubscribe
        var notificationClient = Mst.Client.Notifications;
        if (notificationClient != null)
        {
            notificationClient.OnNotificationReceivedEvent -= ShowNotification;
        }
    }
}
```

## Example usage from a room

```csharp
public class RoomManager : MonoBehaviour, IRoomManager
{
    // Send a notification to all players in the room when one player is ready
    public void OnPlayerReadyStatusChanged(int peerId, bool isReady)
    {
        var player = Mst.Server.Rooms.GetPlayer(currentRoomId, peerId);
        
        if (player != null && isReady)
        {
            var username = player.GetExtension<IUserPeerExtension>()?.Username ?? "Unknown";
            
            // Create a notification packet
            var packet = new NotificationPacket
            {
                RoomId = currentRoomId,
                Message = $"Player {username} is ready to play!",
                IgnoreRecipients = new List<int> { peerId } // Do not send to the player themselves
            };
            
            // Send the notification via the server
            Mst.Server.Connection.SendMessage(MstOpCodes.Notification, packet);
        }
    }
}
```

## Promised messages

A special feature of the notification module is the ability to store "promised messages" that will be delivered to new users when they log in. This is useful for system announcements or news that every player should receive.

```csharp
// Send to everyone and save as a promised message
notificationModule.NoticeToAll("New game update! Version 1.2.0 available!", true);
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
