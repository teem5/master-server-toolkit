# Master Server Toolkit - Ping

## Description
Ping module for checking the connection between client and server, measuring latency and testing server availability.

## PingModule

Core class of the Ping module.

### Setup:
```csharp
[SerializeField, TextArea(3, 5)]
private string pongMessage = "Hello, Pong!";
```

### Properties:
```csharp
public string PongMessage { get; set; }
```

## Server usage

### Initialization:
```csharp
// The module automatically registers a Ping message handler
public override void Initialize(IServer server)
{
    server.RegisterMessageHandler(MstOpCodes.Ping, OnPingRequestListener);
}

// Ping request handler
private Task OnPingRequestListener(IIncomingMessage message)
{
    message.Respond(pongMessage, ResponseStatus.Success);
    return Task.CompletedTask;
}
```

### Configuring the Pong message:
```csharp
// Get the module
var pingModule = Mst.Server.Modules.GetModule<PingModule>();

// Set the message
pingModule.PongMessage = "Game server is running!";
```

## Client Usage

### Sending a Ping:
```csharp
// Send a ping request
Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        // Receive message from the server
        string pongMessage = response.AsString();
        
        // Calculate the round trip time (RTT)
        float rtt = response.TimeElapsedSinceRequest;
        
        Debug.Log($"Ping successful. RTT: {rtt}ms. Message: {pongMessage}");
    }
    else
    {
        Debug.LogError("Ping failed. Server might be unavailable.");
    }
});
```

### Periodic ping implementation:
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
                // Reset the failed attempts counter
                failedPingsCount = 0;
                
                // Update ping display in the UI
                float rtt = response.TimeElapsedSinceRequest;
                UpdatePingDisplay(rtt);
            }
            else
            {
                failedPingsCount++;
                
                if (failedPingsCount >= maxFailedPings)
                {
                    // Handle connection loss
                    HandleConnectionLost();
                }
            }
        });
    }
    
    private void UpdatePingDisplay(float rtt)
    {
        // Example of updating the UI
        // pingText.text = $"Ping: {Mathf.RoundToInt(rtt)}ms";
    }
    
    private void HandleConnectionLost()
    {
        Debug.LogWarning("Connection to server lost!");
        // Handle server connection loss
    }
}
```

## Advanced Ping implementation

### Updating the module to send additional information:

```csharp
public class EnhancedPingModule : PingModule
{
    [SerializeField] private int serverCurrentLoad = 0;
    [SerializeField] private int maxServerLoad = 100;
    
    private Task OnPingRequestListener(IIncomingMessage message)
    {
        // Create an extended response with additional information
        var pingResponse = new PingResponseInfo
        {
            Message = PongMessage,
            ServerTime = System.DateTime.UtcNow.Ticks,
            ServerLoad = serverCurrentLoad,
            MaxServerLoad = maxServerLoad,
            OnlinePlayers = Mst.Server.ConnectionsCount
        };
        
        // Convert to JSON
        string jsonResponse = JsonUtility.ToJson(pingResponse);
        
        // Send the response
        message.Respond(jsonResponse, ResponseStatus.Success);
        return Task.CompletedTask;
    }
    
    // Update server load information
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

### Client handling of the extended response:

```csharp
// Send a request
Mst.Client.Connection.SendMessage(MstOpCodes.Ping, (status, response) =>
{
    if (status == ResponseStatus.Success)
    {
        string jsonResponse = response.AsString();
        
        try
        {
            // Parse the JSON response
            PingResponseInfo pingInfo = JsonUtility.FromJson<PingResponseInfo>(jsonResponse);
            
            // Use the information
            Debug.Log($"Server message: {pingInfo.Message}");
            Debug.Log($"Server time: {new System.DateTime(pingInfo.ServerTime)}");
            Debug.Log($"Server load: {pingInfo.ServerLoad}/{pingInfo.MaxServerLoad}");
            Debug.Log($"Online players: {pingInfo.OnlinePlayers}");
            
            // Calculate time difference between client and server
            long clientTime = System.DateTime.UtcNow.Ticks;
            TimeSpan timeDifference = new System.DateTime(pingInfo.ServerTime) - new System.DateTime(clientTime);
            Debug.Log($"Time difference: {timeDifference.TotalMilliseconds}ms");
        }
        catch
        {
            // If the response is in the old format, handle it as a string
            Debug.Log($"Server message: {jsonResponse}");
        }
    }
});
```

## Integration with other systems

### Connection monitoring:
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
                
                // Add to history
                pingsHistory.Add(rtt);
                
                // Keep history size limited
                if (pingsHistory.Count > historySize)
                {
                    pingsHistory.RemoveAt(0);
                }
                
                // Check for high ping
                if (rtt > maxPingTime)
                {
                    OnHighPingDetected(rtt);
                }
                
                // Notify about average ping
                float averagePing = pingsHistory.Average();
                OnPingUpdated(rtt, averagePing);
            }
            else
            {
                OnPingFailed();
            }
        });
    }
    
    // Events for integration with game systems
    private void OnPingUpdated(float currentPing, float averagePing)
    {
        // Update the UI or game state
    }
    
    private void OnHighPingDetected(float ping)
    {
        // Warn the player about poor connection
    }
    
    private void OnPingFailed()
    {
        // Handle a failed ping attempt
    }
}
```

### Automatic reconnection:
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
                // Check the connection with a ping
                CheckConnection();
            }
            else if (reconnectAttempts < maxReconnectAttempts)
            {
                // Attempt to reconnect
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
                // Connection is alive, reset counters
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
        
        // Attempt to reconnect
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

## Best Practices

1. **Use Ping to monitor connection status** – periodic checks help detect issues early
2. **Implement automatic reconnection** when a connection loss is detected
3. **Keep a ping history** to analyze connection stability
4. **Display ping in the UI** to inform players about connection quality
5. **Extend the base functionality** to send additional server information
6. **Set reasonable ping intervals** – requests that are too frequent may create extra load
7. **Adapt gameplay** to the connection state, e.g. reduce update frequency on high ping
