# Master Server Toolkit - Terminal

## Description
A built-in console for Unity that allows you to execute commands, view logs, and interact with the game at runtime. The terminal hooks into Unity logs and provides an interface for registering and running custom commands.

## Main Components

### Terminal
The main class that controls the display and interaction with the console.

```csharp
// Get the instance
Terminal terminal = Terminal.Instance;

// Logging messages
Terminal.Log("Message for the console");
Terminal.Log(TerminalLogType.Warning, "Warning: {0}", "warning text");
Terminal.Log(TerminalLogType.Error, "Error: {0}", "error text");

// Managing state
terminal.SetState(TerminalState.OpenSmall);  // Open small window
terminal.SetState(TerminalState.OpenFull);   // Open full window
terminal.SetState(TerminalState.Close);      // Close the console
terminal.ToggleState(TerminalState.OpenSmall); // Toggle state
```

### CommandShell
Handles commands in the terminal.

```csharp
// Access to the command shell
CommandShell shell = Terminal.Shell;

// Execute a command
shell.RunCommand("help");

// Check for errors
if (Terminal.IssuedError)
{
    Debug.LogError(shell.IssuedErrorMessage);
}
```

### CommandHistory
Manages the command input history.

```csharp
// Access command history
CommandHistory history = Terminal.History;

// Get the previous command (press "up")
string prevCommand = history.Previous();

// Get the next command (press "down")
string nextCommand = history.Next();

// Add a command to history
history.Push("help");
```

### CommandAutocomplete
Provides command auto-completion.

```csharp
// Access autocompletion
CommandAutocomplete autocomplete = Terminal.Autocomplete;

// Register a new command for autocompletion
autocomplete.Register("my_command");

// Get completion options
string headText = "he";
string[] completions = autocomplete.Complete(ref headText);
// completions = ["lp", "llo"]
// headText = "he"
```

## Registering Commands

### Using attributes:
```csharp
// Register a command via attribute
public class MyCommands : MonoBehaviour
{
    [RegisterCommand("hello", "Prints a greeting")]
    static void HelloCommand(CommandArg[] args)
    {
        Terminal.Log("Hello, world!");
    }
    
    [RegisterCommand("add", "Adds two numbers")]
    static void AddCommand(CommandArg[] args)
    {
        if (args.Length < 2)
        {
            throw new CommandException("You must specify two numbers");
        }
        
        int a = args[0].Int;
        int b = args[1].Int;
        
        Terminal.Log("{0} + {1} = {2}", a, b, a + b);
    }
}
```

### Manual registration:
```csharp
// Register a command manually
Terminal.Shell.RegisterCommand("time", "Shows the current time", args => 
{
    Terminal.Log("Current time: {0}", DateTime.Now.ToString("HH:mm:ss"));
});

// Register a command with parameters
Terminal.Shell.RegisterCommand("greet", "Greets the user", args => 
{
    string name = args.Length > 0 ? args[0].String : "stranger";
    Terminal.Log("Hello, {0}!", name);
});
```

## Appearance Settings

```csharp
[Header("Window")]
[Range(0, 1)]
[SerializeField] float maxHeight = 0.7f;           // Maximum window height

[Range(100, 1000)]
[SerializeField] float toggleSpeed = 360;          // Open/close speed

[SerializeField] string toggleHotkey = "`";        // Hotkey to open
[SerializeField] string toggleFullHotkey = "#`";   // Hotkey for fullscreen mode

[Header("Input")]
[SerializeField] Font consoleFont;                 // Console font
[SerializeField] string inputCaret = ">";          // Input caret
[SerializeField] bool showGUIButtons;              // Show GUI buttons
[SerializeField] bool rightAlignButtons;           // Right align buttons

[Header("Theme")]
[SerializeField] Color backgroundColor = Color.black;  // Background color
[SerializeField] Color foregroundColor = Color.white;  // Foreground color
[SerializeField] Color shellColor = Color.white;       // Shell message color
[SerializeField] Color inputColor = Color.cyan;        // Input color
[SerializeField] Color warningColor = Color.yellow;    // Warning color
[SerializeField] Color errorColor = Color.red;         // Error color
```

## Argument Handling

CommandArg provides methods to convert string arguments into various types:

```csharp
Terminal.Shell.RegisterCommand("complex", "Demonstration of argument usage", args => 
{
    if (args.Length < 3)
    {
        throw new CommandException("Specify name, age and balance");
    }
    
    string name = args[0].String;  // Get string
    int age = args[1].Int;         // Get integer
    float balance = args[2].Float; // Get floating point number
    bool isPremium = args.Length > 3 ? args[3].Bool : false; // Get boolean value
    
    Terminal.Log("Name: {0}, Age: {1}, Balance: {2:F2}, Premium: {3}", 
        name, age, balance, isPremium ? "Yes" : "No");
});
```

## Usage Examples

### System commands:
```csharp
// help - List all commands
// clear - Clear the console
// version - Show version
// quit - Exit the application
```

### Creating game cheat codes:
```csharp
public class CheatCommands : MonoBehaviour
{
    [RegisterCommand("god", "Enables god mode")]
    static void GodModeCommand(CommandArg[] args)
    {
        bool enable = args.Length == 0 || args[0].Bool;
        PlayerManager.Instance.SetGodMode(enable);
        Terminal.Log("God mode: {0}", enable ? "enabled" : "disabled");
    }
    
    [RegisterCommand("gold", "Adds gold")]
    static void AddGoldCommand(CommandArg[] args)
    {
        int amount = args.Length > 0 ? args[0].Int : 100;
        PlayerManager.Instance.AddGold(amount);
        Terminal.Log("{0} gold added", amount);
    }
    
    [RegisterCommand("spawn", "Spawns an object")]
    static void SpawnCommand(CommandArg[] args)
    {
        if (args.Length == 0)
        {
            throw new CommandException("Specify the prefab name to spawn");
        }
        
        string prefabName = args[0].String;
        int count = args.Length > 1 ? args[1].Int : 1;
        
        GameManager.Instance.SpawnPrefab(prefabName, count);
        Terminal.Log("Spawned {0}x {1}", count, prefabName);
    }
}
```

### Integration with the logging system:
```csharp
public class CustomLogger : MonoBehaviour
{
    void Start()
    {
        // Redirects Unity logs to the terminal
        Application.logMessageReceived += HandleLog;
    }
    
    void HandleLog(string message, string stackTrace, LogType type)
    {
        TerminalLogType terminalLogType;
        
        switch (type)
        {
            case LogType.Warning:
                terminalLogType = TerminalLogType.Warning;
                break;
            case LogType.Error:
            case LogType.Exception:
                terminalLogType = TerminalLogType.Error;
                break;
            default:
                terminalLogType = TerminalLogType.Message;
                break;
        }
        
        Terminal.Log(terminalLogType, message);
    }
}
```

## Best Practices

1. **Use group prefixes** for related commands (e.g., `player_health`, `player_ammo`)
2. **Add informative descriptions** 
3. **Handle errors and edge cases** 
4. **Categorize messages** 
5. **Do not output confidential information** 
6. **Separate debug commands from player commands**
7. **Implement access levels** 
