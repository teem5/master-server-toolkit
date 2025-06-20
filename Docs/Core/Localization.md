# Master Server Toolkit - Localization

## Description
Localization system for multilingual game support. Supports loading translations from text files and changing the language at runtime.

## MstLocalization

Main class for working with localization.

### Main properties:
```csharp
// Current language
string Lang { get; set; }

// Get translation by key
string this[string key] { get; }

// Language change event
event Action<string> LanguageChangedEvent;
```

### Accessing localization:
```csharp
// Via the global instance
Mst.Localization.Lang = "ru";
string welcomeText = Mst.Localization["welcome_message"];

// Create your own instance
var localization = new MstLocalization();
```

## Localization file format

### File structure:
```
# Comments start with #
;key;en;ru;de

# UI messages
ui_welcome;Welcome!;Добро пожаловать!;Willkommen!
ui_loading;Loading...;Загрузка...;Wird geladen...
ui_error;Error occurred;Произошла ошибка;Fehler aufgetreten

# Game messages
game_start;Game Started;Игра началась;Spiel gestartet
game_over;Game Over;Игра окончена;Spiel beendet

# Buttons
btn_ok;OK;ОК;OK
btn_cancel;Cancel;Отмена;Abbrechen
btn_save;Save;Сохранить;Speichern
```

### File location:
```
Resources/
└── Localization/
    ├── localization.txt          # Main file
    └── custom_localization.txt   # Custom translations
```

## Using in code

### Basic usage:
```csharp
public class UIManager : MonoBehaviour
{
    [Header("UI Elements")]
    public Text titleText;
    public Text statusText;
    public Button okButton;
    
    void Start()
    {
        // Subscribe to language changes
        Mst.Localization.LanguageChangedEvent += OnLanguageChanged;
        
        // Set initial texts
        UpdateTexts();
    }
    
    void OnLanguageChanged(string newLang)
    {
        UpdateTexts();
    }
    
    void UpdateTexts()
    {
        titleText.text = Mst.Localization["game_title"];
        statusText.text = Mst.Localization["status_ready"];
        okButton.GetComponentInChildren<Text>().text = Mst.Localization["btn_ok"];
    }
}
```

### Creating a localization component:
```csharp
public class LocalizedText : MonoBehaviour
{
    [Header("Localization")]
    public string key;
    
    private Text textComponent;
    
    void Awake()
    {
        textComponent = GetComponent<Text>();
    }
    
    void Start()
    {
        Mst.Localization.LanguageChangedEvent += UpdateText;
        UpdateText(Mst.Localization.Lang);
    }
    
    void OnDestroy()
    {
        Mst.Localization.LanguageChangedEvent -= UpdateText;
    }
    
    void UpdateText(string lang)
    {
        if (textComponent && !string.IsNullOrEmpty(key))
        {
            textComponent.text = Mst.Localization[key];
        }
    }
}
```

### Changing language with saving:
```csharp
public class LanguageSelector : MonoBehaviour
{
    [Header("Available Languages")]
    public string[] availableLanguages = { "en", "ru", "de" };
    
    public Dropdown languageDropdown;
    
    void Start()
    {
        // Load saved language
        string savedLang = PlayerPrefs.GetString("SelectedLanguage", "en");
        Mst.Localization.Lang = savedLang;
        
        // Configure dropdown
        SetupDropdown();
    }
    
    void SetupDropdown()
    {
        languageDropdown.ClearOptions();
        
        var options = new List<Dropdown.OptionData>();
        int selectedIndex = 0;
        
        for (int i = 0; i < availableLanguages.Length; i++)
        {
            string lang = availableLanguages[i];
            options.Add(new Dropdown.OptionData(GetLanguageName(lang)));
            
            if (lang == Mst.Localization.Lang)
                selectedIndex = i;
        }
        
        languageDropdown.AddOptions(options);
        languageDropdown.value = selectedIndex;
        languageDropdown.onValueChanged.AddListener(OnLanguageSelected);
    }
    
    void OnLanguageSelected(int index)
    {
        string newLang = availableLanguages[index];
        
        // Change the language
        Mst.Localization.Lang = newLang;
        
        // Save selection
        PlayerPrefs.SetString("SelectedLanguage", newLang);
        PlayerPrefs.Save();
    }
    
    string GetLanguageName(string lang)
    {
        switch (lang)
        {
            case "en": return "English";
            case "ru": return "Russian";
            case "de": return "Deutsch";
            default: return lang.ToUpper();
        }
    }
}
```

## Динамическая локализация

### Localization with parameters:
```csharp
// In the localization file:
# player_score;Score: {0};Счет: {0};Punkte: {0}

// In code:
string scoreText = string.Format(
    Mst.Localization["player_score"], 
    currentScore
);

// Or using a helper:
public static string GetLocalizedFormat(string key, params object[] args)
{
    string template = Mst.Localization[key];
    return string.Format(template, args);
}

// Usage:
string message = GetLocalizedFormat("welcome_player", playerName);
```

### Enum localization:
```csharp
public enum GameMode
{
    Single,
    Multiplayer,
    Tournament
}

public static string GetLocalizedEnum<T>(T enumValue) where T : Enum
{
    string key = $"enum_{typeof(T).Name}_{enumValue}";
    return Mst.Localization[key];
}

// В файле локализации:
# enum_GameMode_Single;Single Player;Одиночная игра;Einzelspieler
# enum_GameMode_Multiplayer;Multiplayer;Многопользовательская игра;Mehrspieler
```

## Advanced features

### Custom file format:
```csharp
public class CustomLocalizationLoader
{
    public static void LoadJson(string jsonPath)
    {
        var jsonText = Resources.Load<TextAsset>(jsonPath).text;
        var locData = JsonUtility.FromJson<LocalizationData>(jsonText);
        
        foreach (var entry in locData.entries)
        {
            foreach (var translation in entry.translations)
            {
                Mst.Localization.RegisterKey(
                    translation.lang, 
                    entry.key, 
                    translation.value
                );
            }
        }
    }
}

[System.Serializable]
public class LocalizationData
{
    public LocalizationEntry[] entries;
}

[System.Serializable]
public class LocalizationEntry
{
    public string key;
    public Translation[] translations;
}

[System.Serializable]
public class Translation
{
    public string lang;
    public string value;
}
```

### Sprite localization:
```csharp
public class LocalizedSprite : MonoBehaviour
{
    [Header("Localized Sprites")]
    public Sprite englishSprite;
    public Sprite russianSprite;
    public Sprite germanSprite;
    
    private Image imageComponent;
    
    void Start()
    {
        imageComponent = GetComponent<Image>();
        Mst.Localization.LanguageChangedEvent += UpdateSprite;
        UpdateSprite(Mst.Localization.Lang);
    }
    
    void UpdateSprite(string lang)
    {
        switch (lang)
        {
            case "en": imageComponent.sprite = englishSprite; break;
            case "ru": imageComponent.sprite = russianSprite; break;
            case "de": imageComponent.sprite = germanSprite; break;
        }
    }
}
```

## Best practices

1. **Use namespaces for keys**:
```
ui_main_menu_title
ui_settings_volume
game_message_victory
error_network_timeout
```

2. **Store long texts separately**:
```
# tutorial_step1;Press [WASD] to move;Нажмите [WASD] для передвижения;...
```

3. **Validate translations on start**:
```csharp
void ValidateTranslations()
{
    string[] requiredKeys = { "ui_welcome", "btn_ok", "game_start" };
    string[] languages = { "en", "ru", "de" };
    
    foreach (var key in requiredKeys)
    {
        foreach (var lang in languages)
        {
            var oldLang = Mst.Localization.Lang;
            Mst.Localization.Lang = lang;
            
            if (Mst.Localization[key] == key)
            {
                Debug.LogWarning($"Missing translation for {key} in {lang}");
            }
            
            Mst.Localization.Lang = oldLang;
        }
    }
}
```

4. **Аргументы командной строки**:
```bash
# Set language on launch
./Game.exe -defaultLanguage ru
```

## Integration with the event system

```csharp
// Notify about language change
Mst.Localization.LanguageChangedEvent += (lang) => {
    Mst.Events.Invoke("languageChanged", lang);
};

// Notify about missing translations
public static void ReportMissingTranslation(string key, string lang)
{
    Mst.Events.Invoke("translationMissing", new { key, lang });
    Debug.LogWarning($"Missing translation: {key} for {lang}");
}
```
