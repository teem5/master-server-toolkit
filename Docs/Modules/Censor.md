# Master Server Toolkit - Censor

## Description
A censorship module for filtering unwanted content such as profanity or insults. Provides safe communication between players.

## CensorModule

Main class of the censorship module.

### Setup:
```csharp
[Header("Settings")]
[SerializeField] private TextAsset[] wordsLists;
[SerializeField, TextArea(5, 10)] private string matchPattern = @"\b{0}\b";
```

### Initialization:
```csharp
// Add the module to the scene
var censorModule = gameObject.AddComponent<CensorModule>();

// Configure forbidden word lists
public TextAsset[] wordsLists;
wordsLists = new TextAsset[] { forbiddenWordsAsset };
```

### Dictionary file format:
The dictionary is a text file with forbidden words separated by commas:
```
bad,words,list,here,separated,by,commas
```

## Usage in code

### Checking text:
```csharp
// Get the module instance
var censorModule = Mst.Server.Modules.GetModule<CensorModule>();

// Check text for forbidden words
bool hasBadWord = censorModule.HasCensoredWord("Text to check");

if (hasBadWord)
{
    // The text contains forbidden words
    Debug.Log("Text contains censored words");
}
else
{
    // The text is clean
    Debug.Log("Text is clean");
}
```

### Chat integration:
```csharp
// In the chat message handler
void HandleChatMessage(string message, IPeer sender)
{
    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();

    if (censorModule.HasCensoredWord(message))
    {
        // Reject the message
        sender.SendMessage(MstOpCodes.MessageRejected, "Message contains forbidden words");
        return;
    }

    // Send the message to all users
    BroadcastMessage(message);
}
```

### Checking player names:
```csharp
// During registration in AuthModule
protected override bool IsUsernameValid(string username)
{
    if (!base.IsUsernameValid(username))
        return false;

    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();

    // Check the name for forbidden words
    if (censorModule.HasCensoredWord(username))
        return false;

    return true;
}
```

## Match pattern configuration

The `matchPattern` parameter determines how forbidden words are checked:
```csharp
// Default: whole words using word boundaries
matchPattern = @"\b{0}\b";

// Stricter check: including partial matches
matchPattern = @"{0}";

// With separators: checks only words separated by spaces
matchPattern = @"(\s|^){0}(\s|$)";
```

## Extending the module

You can extend the base functionality by creating a subclass of CensorModule:
```csharp
public class EnhancedCensorModule : CensorModule
{
    [SerializeField] private bool useRegexPatterns = false;
    [SerializeField] private TextAsset regexPatterns;

    private List<Regex> patterns = new List<Regex>();

    protected override void ParseTextFiles()
    {
        base.ParseTextFiles();

        // Load additional regex patterns
        if (useRegexPatterns && regexPatterns != null)
        {
            var patternLines = regexPatterns.text.Split('\n');
            foreach (var pattern in patternLines)
            {
                if (!string.IsNullOrEmpty(pattern))
                {
                    patterns.Add(new Regex(pattern, RegexOptions.IgnoreCase | RegexOptions.Compiled));
                }
            }
        }
    }

    public override bool HasCensoredWord(string text)
    {
        // Check with the base method
        if (base.HasCensoredWord(text))
            return true;

        // Check using regex patterns
        foreach (var regex in patterns)
        {
            if (regex.IsMatch(text))
                return true;
        }

        return false;
    }

    // Additional method to get masked text
    public string GetCensoredText(string text, char maskChar = '*')
    {
        string result = text;

        // Replace forbidden words with a mask
        foreach (var word in censoredWords)
        {
            string pattern = string.Format(matchPattern, Regex.Escape(word));
            string replacement = new string(maskChar, word.Length);
            result = Regex.Replace(result, pattern, replacement, RegexOptions.IgnoreCase);
        }

        return result;
    }
}
```

## Best practices

1. **Update forbidden word lists regularly**
2. **Use word boundaries** (`\b{0}\b`) to avoid false positives
3. **Integrate with the chat system** for automatic message filtering
4. **Transform text before checking** to bypass simple tricks (e.g. `b@d w0rd`)
5. **Include multi-language support** for international projects
6. **Consider context** when filtering—some words may be acceptable in one case but not in another
7. **Use localized dictionaries** for different regions

## Integration examples

### Automatic warning system:
```csharp
public class ChatFilterSystem : MonoBehaviour
{
    [SerializeField] private int maxViolations = 3;
    private Dictionary<string, int> violations = new Dictionary<string, int>();

    public void CheckMessage(string username, string message)
    {
        var censorModule = Mst.Server.Modules.GetModule<CensorModule>();

        if (censorModule.HasCensoredWord(message))
        {
            if (!violations.ContainsKey(username))
                violations[username] = 0;

            violations[username]++;

            if (violations[username] >= maxViolations)
            {
                // Temporary ban the user
                BanUser(username, TimeSpan.FromMinutes(10));
            }
        }
    }
}
```

### Integration with user-generated content:
```csharp
// Validate user-provided names
public bool ValidateUserContent(string text)
{
    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();
    return !censorModule.HasCensoredWord(text);
}

// Use when creating items, clans, etc.
public bool CreateClan(string clanName, string clanTag)
{
    if (!ValidateUserContent(clanName) || !ValidateUserContent(clanTag))
    {
        return false;
    }

    // Create the clan
    // ...

    return true;
}
```
