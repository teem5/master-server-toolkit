# Master Server Toolkit - Chat

## Description
Module for creating a chat system with channel support, private messages, and profanity checking.

## ChatModule

Main class for chat management.

### Setup:
```csharp
[Header("General Settings")]
[SerializeField] protected bool useAuthModule = true;
[SerializeField] protected bool useCensorModule = true;
[SerializeField] protected bool allowUsernamePicking = true;

[SerializeField] protected bool setFirstChannelAsLocal = true;
[SerializeField] protected bool setLastChannelAsLocal = true;

[SerializeField] protected int minChannelNameLength = 5;
[SerializeField] protected int maxChannelNameLength = 25;
```

## Working with channels

### Creating or getting a channel:
```csharp
// Get or create a channel
var channel = chatModule.GetOrCreateChannel("general");

// Bypass censorship check
var channel = chatModule.GetOrCreateChannel("badword", true);
```

### Joining a channel:
```csharp
// Send a join request to the server
Mst.Client.Connection.SendMessage(MstOpCodes.JoinChannel, "general");

// Get current channels
Mst.Client.Connection.SendMessage(MstOpCodes.GetCurrentChannels);
```

### Channel management:
```csharp
// Leave a channel
Mst.Client.Connection.SendMessage(MstOpCodes.LeaveChannel, "general");

// Set default channel
Mst.Client.Connection.SendMessage(MstOpCodes.SetDefaultChannel, "global");

// Get list of users in a channel
Mst.Client.Connection.SendMessage(MstOpCodes.GetUsersInChannel, "general");
```

## Sending messages

### Message types:
```csharp
public enum ChatMessageType : byte
{
    ChannelMessage, // Channel message
    PrivateMessage  // Private message
}
```

### Sending messages:
```csharp
// Message to a channel
var channelMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.ChannelMessage,
    Receiver = "general",  // channel name
    Message = "Hello everyone!"
};
Mst.Client.Connection.SendMessage(MstOpCodes.ChatMessage, channelMsg);

// Message to the default local channel
var localMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.ChannelMessage,
    Receiver = null, // sent to DefaultChannel
    Message = "Hello local channel!"
};

// Private message
var privateMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.PrivateMessage,
    Receiver = "username",  // user name
    Message = "Hello privately!"
};
```

## User management

### Setting the username:
```csharp
// If allowUsernamePicking = true
Mst.Client.Connection.SendMessage(MstOpCodes.PickUsername, "myUsername");
```

### Working with ChatUserPeerExtension:
```csharp
// Get the user extension
var chatUser = peer.GetExtension<ChatUserPeerExtension>();

// Change username
chatModule.ChangeUsername(peer, "newUsername", true); // keep channels

// Access user channels
var channels = chatUser.CurrentChannels;
var defaultChannel = chatUser.DefaultChannel;
```

## Integration with other modules

### With AuthModule:
```csharp
// When useAuthModule = true
authModule.OnUserLoggedInEvent += OnUserLoggedInEventHandler;
authModule.OnUserLoggedOutEvent += OnUserLoggedOutEventHandler;

// ChatUser is created automatically on login
```

### With CensorModule:
```csharp
// When useCensorModule = true
// Messages are automatically checked for profanity
// Forbidden messages are replaced with a warning
```

## Customization

### Overriding message handling:
```csharp
protected override bool TryHandleChatMessage(ChatMessagePacket chatMessage, ChatUserPeerExtension sender, IIncomingMessage message)
{
    // Custom processing
    if (chatMessage.Message.StartsWith("/"))
    {
        HandleCommand(chatMessage);
        return true;
    }

    return base.TryHandleChatMessage(chatMessage, sender, message);
}
```

### Creating custom users:
```csharp
protected override ChatUserPeerExtension CreateChatUser(IPeer peer, string username)
{
    // Create an extended user
    return new MyChatUserExtension(peer, username);
}
```

## Client code (MstChatClient)

### Subscribing to events:
```csharp
// Connect to the chat module
Mst.Client.Chat.Connection = connection;

// Events
Mst.Client.Chat.OnMessageReceivedEvent += (message) => {
    Debug.Log($"[{message.Sender}]: {message.Message}");
};

Mst.Client.Chat.OnLeftChannelEvent += (channel) => {
    Debug.Log($"Left channel: {channel}");
};

Mst.Client.Chat.OnJoinedChannelEvent += (channel) => {
    Debug.Log($"Joined channel: {channel}");
};
```

### Sending messages from the client:
```csharp
// To a channel
Mst.Client.Chat.SendToChannel("general", "Hello world!");

// Private message
Mst.Client.Chat.SendToUser("username", "Secret message");

// To the local channel
Mst.Client.Chat.SendToLocalChannel("Hello local!");
```

## Usage examples

### Creating a game chat:
```csharp
public class GameChatUI : MonoBehaviour
{
    [Header("UI")]
    public InputField messageInput;
    public Text chatLog;
    public Dropdown channelDropdown;

    void Start()
    {
        // Join the general channel
        Mst.Client.Chat.JoinChannel("general");
        Mst.Client.Chat.SetDefaultChannel("general");

        // Message received event
        Mst.Client.Chat.OnMessageReceivedEvent += OnMessageReceived;

        // Channel events
        Mst.Client.Chat.OnChannelUsersChanged += OnChannelUsersChanged;
    }

    private void OnMessageReceived(ChatMessagePacket message)
    {
        string formattedMsg = $"{message.Sender}: {message.Message}\n";
        chatLog.text += formattedMsg;
    }

    public void SendMessage()
    {
        if (!string.IsNullOrEmpty(messageInput.text))
        {
            Mst.Client.Chat.SendToDefaultChannel(messageInput.text);
            messageInput.text = "";
        }
    }
}
```

### Chat command system:
```csharp
protected override bool TryHandleChatMessage(ChatMessagePacket chatMessage, ChatUserPeerExtension sender, IIncomingMessage message)
{
    if (chatMessage.Message.StartsWith("/"))
    {
        var command = chatMessage.Message.Split(' ')[0].Substring(1);
        var args = chatMessage.Message.Split(' ').Skip(1).ToArray();

        switch (command)
        {
            case "whisper":
                if (args.Length >= 2)
                {
                    var targetUser = args[0];
                    var privateMsg = string.Join(" ", args.Skip(1));
                    // Send a private message
                }
                break;

            case "join":
                if (args.Length > 0)
                {
                    // Join a channel
                }
                break;
        }

        message.Respond(ResponseStatus.Success);
        return true;
    }

    return base.TryHandleChatMessage(chatMessage, sender, message);
}
```

## Best practices

1. **Always use AuthModule** for automatic user setup
2. **Configure CensorModule** to filter profanity
3. **Limit channel name length** to prevent abuse
4. **Use private messages** for sensitive information
5. **Create separate channels** for different purposes (general, trade, guild)
6. **Clean up empty channels** to save resources
7. **Log all messages** for moderation and analytics
