# Master Server Toolkit - Chat

## 설명
채널, 비공개 메시지 및 음란 한 단어 검사를 지원하는 채팅 시스템을 작성하기위한 모듈.

## ChatModule

채팅 제어를위한 메인 클래스.

### 설정 :
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

## 채널에서 작업하십시오

### 생성/채널 :
```csharp
// Получить или создать канал
var channel = chatModule.GetOrCreateChannel("general");

// Проверка на запрещенные слова
var channel = chatModule.GetOrCreateChannel("badword", true); // игнорировать проверку
```

### 채널 연결 :
```csharp
// Отправка запроса на сервер
Mst.Client.Connection.SendMessage(MstOpCodes.JoinChannel, "general");

// Получение текущих каналов
Mst.Client.Connection.SendMessage(MstOpCodes.GetCurrentChannels);
```

### 채널 관리 :
```csharp
// Покинуть канал
Mst.Client.Connection.SendMessage(MstOpCodes.LeaveChannel, "general");

// Установить канал по умолчанию
Mst.Client.Connection.SendMessage(MstOpCodes.SetDefaultChannel, "global");

// Получить список пользователей в канале
Mst.Client.Connection.SendMessage(MstOpCodes.GetUsersInChannel, "general");
```

## 게시물 보내기

### 게시물 유형 :
```csharp
public enum ChatMessageType : byte
{
    ChannelMessage, // Сообщение в канал
    PrivateMessage  // Приватное сообщение
}
```

### 게시물 보내기 :
```csharp
// Сообщение в канал
var channelMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.ChannelMessage,
    Receiver = "general",  // имя канала
    Message = "Hello everyone!"
};
Mst.Client.Connection.SendMessage(MstOpCodes.ChatMessage, channelMsg);

// Сообщение в личный канал (без указания получателя)
var localMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.ChannelMessage,
    Receiver = null, // отправится в DefaultChannel
    Message = "Hello local channel!"
};

// Приватное сообщение
var privateMsg = new ChatMessagePacket
{
    MessageType = ChatMessageType.PrivateMessage,
    Receiver = "username",  // имя пользователя
    Message = "Hello privately!"
};
```

## 사용자 관리

### 사용자 이름 설정 :
```csharp
// Если allowUsernamePicking = true
Mst.Client.Connection.SendMessage(MstOpCodes.PickUsername, "myUsername");
```

### ChatuserPeerestension과 함께 작업 :
```csharp
// Получить расширение пользователя
var chatUser = peer.GetExtension<ChatUserPeerExtension>();

// Изменить имя пользователя
chatModule.ChangeUsername(peer, "newUsername", true); // сохранить каналы

// Доступ к каналам пользователя
var channels = chatUser.CurrentChannels;
var defaultChannel = chatUser.DefaultChannel;
```

## 다른 모듈과의 통합

### AuthModule과 통합 :
```csharp
// При useAuthModule = true
authModule.OnUserLoggedInEvent += OnUserLoggedInEventHandler;
authModule.OnUserLoggedOutEvent += OnUserLoggedOutEventHandler;

// Автоматическое создание ChatUser при входе
```

### 인구 조사와 통합 :
```csharp
// При useCensorModule = true
// Автоматическая проверка сообщений на нецензурные слова
// Замена запрещенного сообщения на предупреждение
```

## castomization

### 게시물이 내려다 보입니다.
```csharp
protected override bool TryHandleChatMessage(ChatMessagePacket chatMessage, ChatUserPeerExtension sender, IIncomingMessage message)
{
    // Кастомная логика обработки
    if (chatMessage.Message.StartsWith("/"))
    {
        HandleCommand(chatMessage);
        return true;
    }
    
    return base.TryHandleChatMessage(chatMessage, sender, message);
}
```

## 사용자 정의 사용자 생성 :
```csharp
protected override ChatUserPeerExtension CreateChatUser(IPeer peer, string username)
{
    // Создание расширенного пользователя
    return new MyChatUserExtension(peer, username);
}
```

## 클라이언트 코드.

### 이벤트 구독 :
```csharp
// Подключение к чат-модулю
Mst.Client.Chat.Connection = connection;

// События
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

### 고객에게서 보내기 :
```csharp
// В канал
Mst.Client.Chat.SendToChannel("general", "Hello world!");

// Приватное сообщение
Mst.Client.Chat.SendToUser("username", "Secret message");

// В локальный канал
Mst.Client.Chat.SendToLocalChannel("Hello local!");
```

## 사용의 예

## 게임 채팅 생성 :
```csharp
public class GameChatUI : MonoBehaviour
{
    [Header("UI")]
    public InputField messageInput;
    public Text chatLog;
    public Dropdown channelDropdown;
    
    void Start()
    {
        // Присоединиться к общему каналу
        Mst.Client.Chat.JoinChannel("general");
        Mst.Client.Chat.SetDefaultChannel("general");
        
        // Событие получения сообщения
        Mst.Client.Chat.OnMessageReceivedEvent += OnMessageReceived;
        
        // События каналов
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

### 채팅 팀 시스템 :
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
                    // Отправить приватное сообщение
                }
                break;
                
            case "join":
                if (args.Length > 0)
                {
                    // Присоединиться к каналу
                }
                break;
        }
        
        message.Respond(ResponseStatus.Success);
        return true;
    }
    
    return base.TryHandleChatMessage(chatMessage, sender, message);
}
```

## 모범 사례

1. ** 자동 사용자 구성에 항상 AuthModule **를 사용하십시오.
2. ** 음란 한 단어를 필터링하기 위해 Censormodule을 구성 **
3. ** 학대를 방지하기 위해 채널의 길이를 제한하십시오 **
4. ** 민감한 정보는 개인 메시지를 사용하십시오 **
5. ** 다양한 목적을 위해 다양한 채널 만들기 ** (일반, 무역, 길드)
6. ** 빈 채널을 청소하십시오 ** 리소스를 최적화합니다
7. ** 중재 및 분석을 위해 모든 메시지 **
