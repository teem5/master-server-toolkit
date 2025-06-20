# Master Server Toolkit - Ping

## 설명
핑 모듈 클라이언트와 서버 간의 연결을 확인하려면 서버의 접근성을 지연하고 테스트합니다.

## PingModule

핑 모듈의 주요 클래스.

### 설정 :
```csharp
[SerializeField, TextArea(3, 5)]
private string pongMessage = "Hello, Pong!";
```

### 속성:
```csharp
public string PongMessage { get; set; }
```

## 서버에서 사용하십시오

## H 초기화 :
```csharp
// Модуль автоматически регистрирует обработчик сообщений Ping
public override void Initialize(IServer server)
{
    server.RegisterMessageHandler(MstOpCodes.Ping, OnPingRequestListener);
}

// Обработчик пинг-запросов
private Task OnPingRequestListener(IIncomingMessage message)
{
    message.Respond(pongMessage, ResponseStatus.Success);
    return Task.CompletedTask;
}
```

### 설정 Pong :
```csharp
// Получение модуля
var pingModule = Mst.Server.Modules.GetModule<PingModule>();

// Установка сообщения
pingModule.PongMessage = "Game server is running!";
```

## 클라이언트에서 사용하십시오

### 핑 보내기 :
```csharp
// Отправка пинг-запроса
Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        // Получаем сообщение от сервера
        string pongMessage = response.AsString();
        
        // Вычисляем время отклика (RTT - Round Trip Time)
        float rtt = response.TimeElapsedSinceRequest;
        
        Debug.Log($"Ping successful. RTT: {rtt}ms. Message: {pongMessage}");
    }
    else
    {
        Debug.LogError("Ping failed. Server might be unavailable.");
    }
});
```

###주기적인 핑의 실현 :
```csharp
public class PingTester : MonoBehaviour
{
    [SerializeField] private float pingInterval = 1f;
    [SerializeField] private int maxFailedPings = 3;
    
    private float nextPingTime = 0f;
    private int failedPingsCount = 0;
    
    private void Update()
    {
        if (Time.time >= nextPingTime && Mst.Client.Connection.IsConnected)
        {
            nextPingTime = Time.time + pingInterval;
            SendPing();
        }
    }
    
    private void SendPing()
    {
        Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
        {
            if (status == ResponseStatus.Success)
            {
                // Сброс счётчика неудачных попыток
                failedPingsCount = 0;
                
                // Обновление отображения пинга в UI
                float rtt = response.TimeElapsedSinceRequest;
                UpdatePingDisplay(rtt);
            }
            else
            {
                failedPingsCount++;
                
                if (failedPingsCount >= maxFailedPings)
                {
                    // Обработка потери соединения
                    HandleConnectionLost();
                }
            }
        });
    }
    
    private void UpdatePingDisplay(float rtt)
    {
        // Пример обновления UI
        // pingText.text = $"Ping: {Mathf.RoundToInt(rtt)}ms";
    }
    
    private void HandleConnectionLost()
    {
        Debug.LogWarning("Connection to server lost!");
        // Обработка потери соединения с сервером
    }
}
```

## 확장 핑

### 추가 정보 전송을위한 모듈 업데이트 :

```csharp
public class EnhancedPingModule : PingModule
{
    [SerializeField] private int serverCurrentLoad = 0;
    [SerializeField] private int maxServerLoad = 100;
    
    private Task OnPingRequestListener(IIncomingMessage message)
    {
        // Создаем расширенный ответ с дополнительной информацией
        var pingResponse = new PingResponseInfo
        {
            Message = PongMessage,
            ServerTime = System.DateTime.UtcNow.Ticks,
            ServerLoad = serverCurrentLoad,
            MaxServerLoad = maxServerLoad,
            OnlinePlayers = Mst.Server.ConnectionsCount
        };
        
        // Преобразуем в JSON
        string jsonResponse = JsonUtility.ToJson(pingResponse);
        
        // Отправляем ответ
        message.Respond(jsonResponse, ResponseStatus.Success);
        return Task.CompletedTask;
    }
    
    // Обновление информации о нагрузке
    public void UpdateServerLoad(int currentLoad)
    {
        serverCurrentLoad = currentLoad;
    }
    
    [System.Serializable]
    private class PingResponseInfo
    {
        public string Message;
        public long ServerTime;
        public int ServerLoad;
        public int MaxServerLoad;
        public int OnlinePlayers;
    }
}
```

## 확장 된 답변의 클라이언트 처리 :

```csharp
// Отправка запроса
Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        string jsonResponse = response.AsString();
        
        try
        {
            // Парсинг JSON-ответа
            PingResponseInfo pingInfo = JsonUtility.FromJson<PingResponseInfo>(jsonResponse);
            
            // Использование информации
            Debug.Log($"Server message: {pingInfo.Message}");
            Debug.Log($"Server time: {new System.DateTime(pingInfo.ServerTime)}");
            Debug.Log($"Server load: {pingInfo.ServerLoad}/{pingInfo.MaxServerLoad}");
            Debug.Log($"Online players: {pingInfo.OnlinePlayers}");
            
            // Расчёт разницы во времени между клиентом и сервером
            long clientTime = System.DateTime.UtcNow.Ticks;
            TimeSpan timeDifference = new System.DateTime(pingInfo.ServerTime) - new System.DateTime(clientTime);
            Debug.Log($"Time difference: {timeDifference.TotalMilliseconds}ms");
        }
        catch
        {
            // Если ответ в старом формате, обрабатываем как строку
            Debug.Log($"Server message: {jsonResponse}");
        }
    }
});
```

## 다른 시스템과의 통합

### 연결 모니터링 :
```csharp
public class ConnectionMonitor : MonoBehaviour
{
    [SerializeField] private float pingInterval = 5f;
    [SerializeField] private float maxPingTime = 500f; // ms
    
    private List<float> pingsHistory = new List<float>();
    private int historySize = 10;
    
    private void Start()
    {
        StartCoroutine(PingRoutine());
    }
    
    private IEnumerator PingRoutine()
    {
        while (true)
        {
            if (Mst.Client.Connection.IsConnected)
            {
                SendPing();
            }
            
            yield return new WaitForSeconds(pingInterval);
        }
    }
    
    private void SendPing()
    {
        Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
        {
            if (status == ResponseStatus.Success)
            {
                float rtt = response.TimeElapsedSinceRequest;
                
                // Добавляем в историю
                pingsHistory.Add(rtt);
                
                // Поддерживаем ограниченный размер истории
                if (pingsHistory.Count > historySize)
                {
                    pingsHistory.RemoveAt(0);
                }
                
                // Проверка на высокий пинг
                if (rtt > maxPingTime)
                {
                    OnHighPingDetected(rtt);
                }
                
                // Уведомление о среднем пинге
                float averagePing = pingsHistory.Average();
                OnPingUpdated(rtt, averagePing);
            }
            else
            {
                OnPingFailed();
            }
        });
    }
    
    // События для интеграции с игровыми системами
    private void OnPingUpdated(float currentPing, float averagePing)
    {
        // Обновление UI или состояния игры
    }
    
    private void OnHighPingDetected(float ping)
    {
        // Предупреждение игрока о плохом соединении
    }
    
    private void OnPingFailed()
    {
        // Обработка неудачной попытки пинга
    }
}
```

### 자동 재 연결 :
```csharp
public class AutoReconnector : MonoBehaviour
{
    [SerializeField] private float checkInterval = 5f;
    [SerializeField] private int maxFailedPings = 3;
    [SerializeField] private int maxReconnectAttempts = 5;
    
    private int failedPingsCount = 0;
    private int reconnectAttempts = 0;
    
    private void Start()
    {
        StartCoroutine(ConnectionCheckRoutine());
    }
    
    private IEnumerator ConnectionCheckRoutine()
    {
        while (true)
        {
            if (Mst.Client.Connection.IsConnected)
            {
                // Проверка соединения через пинг
                CheckConnection();
            }
            else if (reconnectAttempts < maxReconnectAttempts)
            {
                // Попытка переподключения
                AttemptReconnect();
            }
            
            yield return new WaitForSeconds(checkInterval);
        }
    }
    
    private void CheckConnection()
    {
        Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
        {
            if (status == ResponseStatus.Success)
            {
                // Соединение работает, сбрасываем счётчики
                failedPingsCount = 0;
                reconnectAttempts = 0;
            }
            else
            {
                failedPingsCount++;
                
                if (failedPingsCount >= maxFailedPings)
                {
                    Debug.LogWarning("Connection lost. Attempting to reconnect...");
                    AttemptReconnect();
                }
            }
        });
    }
    
    private void AttemptReconnect()
    {
        reconnectAttempts++;
        
        Debug.Log($"Reconnect attempt {reconnectAttempts}/{maxReconnectAttempts}");
        
        // Попытка переподключения
        Mst.Client.Connection.Connect(Mst.Client.Connection.ConnectionIp, Mst.Client.Connection.ConnectionPort, (isSuccessful, error) =>
        {
            if (isSuccessful)
            {
                Debug.Log("Successfully reconnected");
                failedPingsCount = 0;
            }
            else
            {
                Debug.LogError($"Failed to reconnect: {error}");
            }
        });
    }
}
```

## 모범 사례

1. ** 연결 상태를 모니터링하려면 핑을 사용하십시오 ** -주기 점검은 일찍 문제를 감지하는 데 도움이됩니다.
2. ** 자동 재 연결 향상 ** 연결 손실을 감지 할 때
3. ** 연결의 안정성을 분석하려면 핑의 역사를 유지하십시오.
4. ** 핑을 UI로 표시하여 플레이어에게 연결의 품질에 대해 알리십시오.
5. ** 서버에 대한 추가 정보를 전송하려면 기본 기능을 확장하십시오.
6. ** 합리적인 핑 간격 설치 ** - 너무 빈번한 요청은 추가 부하를 생성 할 수 있습니다.
7. ** 게임 플레이를 연결의 현재 상태로 조정하십시오. 예를 들어, High Ping에서 전송 된 참조 수를 줄입니다.
