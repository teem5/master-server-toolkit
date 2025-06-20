# Master Server Toolkit - DebounceThrottle

## Description
A system for managing frequent function calls using the Debounce and Throttle patterns. It helps optimize performance by controlling how many operations are executed per unit of time.

## Main Components

### DebounceDispatcher
Delayed invocation—executes a function only once after a series of calls using the specified delay.

```csharp
// Create with an interval in milliseconds
var debouncer = new DebounceDispatcher(500); // 500 ms delay

// Use with an asynchronous function
await debouncer.DebounceAsync(async () => {
    await SaveDataAsync();
});

// Use with a synchronous function
debouncer.Debounce(() => {
    UpdateUi();
});
```

### ThrottleDispatcher
Call limiter—ensures a function is invoked no more often than the specified interval, discarding intermediate calls.

```csharp
// Create with an interval and options
var throttler = new ThrottleDispatcher(
    1000,                       // 1000 ms interval
    delayAfterExecution: false, // Start counting from the beginning of execution
    resetIntervalOnException: true // Reset interval if an exception occurs
);

// Use with an asynchronous function
await throttler.ThrottleAsync(async () => {
    await SendAnalyticsDataAsync();
});

// Use with a synchronous function
throttler.Throttle(() => {
    UpdateProgressBar();
});
```

## Generic Versions

### DebounceDispatcher<T>
A version with a typed return value.

```csharp
var typedDebouncer = new DebounceDispatcher<int>(500);

// Getting a result from the function
int result = await typedDebouncer.DebounceAsync(async () => {
    return await CalculateValueAsync();
});
```

### ThrottleDispatcher<T>
A version with a typed return value.

```csharp
var typedThrottler = new ThrottleDispatcher<List<string>>(1000);

// Getting a result from the function
List<string> items = await typedThrottler.ThrottleAsync(async () => {
    return await FetchItemsAsync();
});
```

## Practical Examples

### Real-Time Search

```csharp
public class SearchHandler : MonoBehaviour
{
    private DebounceDispatcher debouncer;

    private void Start()
    {
        debouncer = new DebounceDispatcher(300); // 300 ms delay
    }

    // Called when the text in the input field changes
    public void OnSearchTextChanged(string text)
    {
        debouncer.Debounce(() => {
            // Query the API only after the user stops typing
            StartCoroutine(PerformSearch(text));
        });
    }
}
```

### Limiting Server Requests

```csharp
public class ApiClient : MonoBehaviour
{
    private ThrottleDispatcher throttler;

    private void Start()
    {
        // No more than one request per second
        throttler = new ThrottleDispatcher(1000);
    }

    public async Task<PlayerData> GetPlayerDataAsync(string playerId)
    {
        return await throttler.ThrottleAsync(async () => {
            // Server request limited by frequency
            return await FetchPlayerDataFromServerAsync(playerId);
        });
    }
}
```

### Handling User Input

```csharp
public class InputHandler : MonoBehaviour
{
    private ThrottleDispatcher throttler;

    private void Start()
    {
        // Process no more than 10 times per second
        throttler = new ThrottleDispatcher(100);
    }

    private void Update()
    {
        if (Input.GetKey(KeyCode.Space))
        {
            throttler.Throttle(() => {
                // Action triggered by the space bar, limited by frequency
                FireWeapon();
            });
        }
    }
}
```

### Auto-Save with Delayed Start

```csharp
public class DocumentEditor : MonoBehaviour
{
    private DebounceDispatcher saveDebouncer;

    private void Start()
    {
        // Save two seconds after the last change
        saveDebouncer = new DebounceDispatcher(2000);
    }

    public void OnDocumentChanged()
    {
        // Visual indicator of unsaved changes
        ShowUnsavedChangesIndicator();

        // Delayed save
        saveDebouncer.Debounce(() => {
            SaveDocument();
            HideUnsavedChangesIndicator();
        });
    }
}
```

## Differences Between Debounce and Throttle

### Debounce (Delayed Invocation)
- Delays execution until the specified interval passes without any calls
- Ideal for events that occur rapidly but should be processed only after they finish
- Examples: search while typing, window resize, scrolling

### Throttle (Frequency Limiter)
- Ensures a function runs no more often than once per specified interval
- Ideal for limiting the rate of repeating actions
- Examples: sending analytics, updating the interface, API requests

## Options and Settings

### DebounceDispatcher
- `interval` – the time in milliseconds that must pass without calls before the function is executed

### ThrottleDispatcher
- `interval` – the minimum time in milliseconds between function calls
- `delayAfterExecution` – if true, the interval counts from when the call finishes
- `resetIntervalOnException` – if true, resets the interval when an exception occurs
