# Master Server Toolkit - Utilities

## Description
A set of general-purpose utilities and helper classes that simplify development. It includes design patterns, extensions of standard types, helper classes for Unity, and much more.

## Key Components

### Design Patterns

#### Singleton
```csharp
// Basic singleton for MonoBehaviour
public class PlayerManager : SingletonBehaviour<PlayerManager>
{
    // Accessible from anywhere in the project
    public static PlayerManager Instance => GetInstance();

    public void DoSomething()
    {
        // Implementation
    }
}

// Usage
PlayerManager.Instance.DoSomething();
```

#### Dynamic Singleton
```csharp
// Automatically created singleton
public class AudioManager : DynamicSingletonBehaviour<AudioManager>
{
    // Created on the scene if it doesn't exist
    public static AudioManager Instance => GetInstance();
}

// Usage
AudioManager.Instance.PlaySound("explosion");
```

#### Global Singleton
```csharp
// Singleton that persists across scenes
public class GameManager : GlobalDynamicSingletonBehaviour<GameManager>
{
    // Survives scene changes
    public static GameManager Instance => GetInstance();
}

// Usage
GameManager.Instance.StartNewGame();
```

#### Object Pool
```csharp
// Pool for efficient object management
public class BulletPool : MonoBehaviour
{
    [SerializeField] private Bullet bulletPrefab;
    [SerializeField] private int initialSize = 50;

    private GenericPool<Bullet> bulletPool;

    private void Awake()
    {
        bulletPool = new GenericPool<Bullet>(CreateBullet, initialSize);
    }

    private Bullet CreateBullet()
    {
        return Instantiate(bulletPrefab);
    }

    public Bullet GetBullet()
    {
        return bulletPool.Get();
    }

    public void ReturnBullet(Bullet bullet)
    {
        bulletPool.Return(bullet);
    }
}

// Usage
var bullet = bulletPool.GetBullet();
// After using it
bulletPool.ReturnBullet(bullet);
```

#### Object Registry
```csharp
// Registry for managing objects by key
public class ItemRegistry : BaseRegistry<string, ItemData>
{
    // Registration methods are implemented in the base class
}

// Usage
var registry = new ItemRegistry();
registry.Register("sword", new ItemData { /* data */ });
var item = registry.Get("sword");
```

### Extensions

#### String Extensions
```csharp
// Check if string is null or empty
if (username.IsNullOrEmpty())
{
    // Handle
}

// Hash a string
string passwordHash = password.GetMD5();

// Convert to Base64
string encoded = text.ToBase64();
string decoded = encoded.FromBase64();

// Get last segment of a path
string filename = filePath.GetLastSegment('\\');

// Split string safely
string[] parts = path.SplitSafe('/');
```

#### Byte Array Extensions
```csharp
// Convert to string
byte[] data = GetData();
string text = data.ToString(StringFormat.Utf8);

// Convert to Base64
string base64 = data.ToBase64();

// Create a subarray
byte[] header = data.SubArray(0, 10);

// Combine arrays
byte[] fullPacket = headerBytes.CombineWith(bodyBytes);

// Check for equality
if (hash1.BytesEqual(hash2))
{
    // Handle
}
```

#### Transform Extensions
```csharp
// Reset local transforms
transform.ResetLocal();

// Scale all child objects
transform.ScaleAllChildren(0.5f);

// Destroy all child objects
transform.DestroyChildren();

// Recursive search
var target = transform.FindRecursively("Player/Weapon/Barrel");

// Set local position on individual axes
transform.SetLocalX(10f);
transform.SetLocalY(5f);
transform.SetLocalZ(0f);
```

### Helper Classes

#### ScenesLoader
```csharp
// Asynchronously load a scene with progress
ScenesLoader.LoadSceneAsync("Level1", (progress) => {
    // Update loading indicator
    loadingBar.value = progress;
}, () => {
    // Called when loading is complete
    Debug.Log("Load complete");
});

// Load a scene with fade
ScenesLoader.LoadSceneWithFade("MainMenu", Color.black, 1.5f);

// Reload the current scene
ScenesLoader.ReloadCurrentScene();
```

#### ScreenshotMaker
```csharp
// Capture a screenshot of the screen
ScreenshotMaker.TakeScreenshot((texture) => {
    // Use the captured texture
    screenshotImage.texture = texture;
});

// Save screenshot to file
ScreenshotMaker.SaveScreenshot("Screenshots/screenshot.png");

// Capture screenshot from a specific camera
ScreenshotMaker.TakeScreenshot(myCamera, 1920, 1080, (texture) => {
    // Handle
});
```

#### SimpleNameGenerator
```csharp
// Generate random names
string randomName = SimpleNameGenerator.Generate(length: 6);

// Generate a name with a prefix
string playerName = SimpleNameGenerator.Generate("Player_", 4);

// Generate a name from syllables
string nameFromSyllables = SimpleNameGenerator.GenerateFromSyllables(3);

// Create a unique identifier
string uniqueId = SimpleNameGenerator.GenerateUniqueName();
```

#### MstWebBrowser
```csharp
// Open a URL in the external browser
MstWebBrowser.OpenURL("https://example.com");

// Open a URL with capability check
if (MstWebBrowser.CanOpenURL)
{
    MstWebBrowser.OpenURL("https://example.com");
}

// Open a local HTML file
MstWebBrowser.OpenLocalFile("Documentation.html");
```

#### NetWebRequests
```csharp
// Send a GET request
NetWebRequests.Get("https://api.example.com/data", (success, response) => {
    if (success)
    {
        // Handle response
        Debug.Log(response);
    }
});

// Send a POST request
var data = new Dictionary<string, string>
{
    { "username", "player1" },
    { "score", "100" }
};

NetWebRequests.Post("https://api.example.com/scores", data, (success, response) => {
    if (success)
    {
        Debug.Log("Score submitted");
    }
});

// Load a texture
NetWebRequests.GetTexture("https://example.com/image.jpg", (success, texture) => {
    if (success)
    {
        // Use the texture
        profileImage.texture = texture;
    }
});
```

### Serializable Structures

#### SerializedKeyValuePair
```csharp
// Serializable key-value pair for Unity Inspector
[Serializable]
public class StringIntPair : SerializedKeyValuePair<string, int> { }

public class InventoryManager : MonoBehaviour
{
    [SerializeField]
    private List<StringIntPair> startingItems = new List<StringIntPair>();

    private void Start()
    {
        foreach (var item in startingItems)
        {
            AddItem(item.Key, item.Value);
        }
    }

    private void AddItem(string itemId, int count)
    {
        // Implementation
    }
}
```

## Usage Examples

### Creating a Game Manager
```csharp
public class GameManager : GlobalDynamicSingletonBehaviour<GameManager>
{
    // Game state
    public GameState CurrentState { get; private set; }

    // Events
    public event Action<GameState> OnGameStateChanged;

    // Change game state
    public void ChangeState(GameState newState)
    {
        CurrentState = newState;
        OnGameStateChanged?.Invoke(newState);
    }

    // Load a new level
    public void LoadLevel(int levelIndex)
    {
        ChangeState(GameState.Loading);

        ScenesLoader.LoadSceneAsync($"Level_{levelIndex}", (progress) => {
            // Update progress
        }, () => {
            ChangeState(GameState.Playing);
        });
    }
}

// Usage
GameManager.Instance.LoadLevel(1);
```

### Implementing a Pooling System
```csharp
public class EffectsPool : SingletonBehaviour<EffectsPool>
{
    [Serializable]
    public class EffectPoolData
    {
        public string effectId;
        public GameObject prefab;
        public int initialSize;
    }

    [SerializeField] private List<EffectPoolData> effectsData;

    private Dictionary<string, GenericPool<GameObject>> effectPools = new Dictionary<string, GenericPool<GameObject>>();

    protected override void Awake()
    {
        base.Awake();

        // Initialize pools
        foreach (var data in effectsData)
        {
            var pool = new GenericPool<GameObject>(() => Instantiate(data.prefab), data.initialSize);
            effectPools.Add(data.effectId, pool);
        }
    }

    public GameObject SpawnEffect(string effectId, Vector3 position, Quaternion rotation)
    {
        if (effectPools.TryGetValue(effectId, out var pool))
        {
            var effect = pool.Get();
            effect.transform.position = position;
            effect.transform.rotation = rotation;
            effect.SetActive(true);

            return effect;
        }

        Debug.LogWarning($"Effect {effectId} not found in pools");
        return null;
    }

    public void ReturnEffect(string effectId, GameObject effect)
    {
        if (effectPools.TryGetValue(effectId, out var pool))
        {
            effect.SetActive(false);
            pool.Return(effect);
        }
    }
}

// Usage
void PlayExplosion(Vector3 position)
{
    var effect = EffectsPool.Instance.SpawnEffect("explosion", position, Quaternion.identity);

    // Automatically return to pool after 2 seconds
    StartCoroutine(ReturnAfterDelay("explosion", effect, 2f));
}

IEnumerator ReturnAfterDelay(string effectId, GameObject effect, float delay)
{
    yield return new WaitForSeconds(delay);
    EffectsPool.Instance.ReturnEffect(effectId, effect);
}
```

### Creating a Settings Manager
```csharp
public class SettingsManager : SingletonBehaviour<SettingsManager>
{
    // Settings
    public float MasterVolume { get; private set; } = 1f;
    public float MusicVolume { get; private set; } = 0.8f;
    public float SfxVolume { get; private set; } = 1f;
    public int QualityLevel { get; private set; } = 2;
    public bool Fullscreen { get; private set; } = true;

    // Events
    public event Action OnSettingsChanged;

    private void Start()
    {
        LoadSettings();
    }

    public void SetMasterVolume(float volume)
    {
        MasterVolume = Mathf.Clamp01(volume);
        OnSettingsChanged?.Invoke();
        SaveSettings();
    }

    // Similar methods for other settings

    private void SaveSettings()
    {
        PlayerPrefs.SetFloat("MasterVolume", MasterVolume);
        PlayerPrefs.SetFloat("MusicVolume", MusicVolume);
        PlayerPrefs.SetFloat("SfxVolume", SfxVolume);
        PlayerPrefs.SetInt("QualityLevel", QualityLevel);
        PlayerPrefs.SetInt("Fullscreen", Fullscreen ? 1 : 0);
        PlayerPrefs.Save();
    }

    private void LoadSettings()
    {
        MasterVolume = PlayerPrefs.GetFloat("MasterVolume", 1f);
        MusicVolume = PlayerPrefs.GetFloat("MusicVolume", 0.8f);
        SfxVolume = PlayerPrefs.GetFloat("SfxVolume", 1f);
        QualityLevel = PlayerPrefs.GetInt("QualityLevel", 2);
        Fullscreen = PlayerPrefs.GetInt("Fullscreen", 1) == 1;

        OnSettingsChanged?.Invoke();
    }
}

// Usage
SettingsManager.Instance.SetMasterVolume(0.5f);
```

## Best Practices

1. **Use singletons carefully** – they simplify code but may cause dependency issues
2. **Prefer pooling for frequently created/destroyed objects** – this greatly improves performance
3. **Use method extensions to improve code readability**
4. **Remember the experimental status** – some utilities may not work on all platforms
5. **Avoid unnecessary complexity** – utilities should simplify your code, not complicate it
6. **Favor strong typing** to prevent runtime errors
7. **Document your own extensions** to these utilities for future maintenance
