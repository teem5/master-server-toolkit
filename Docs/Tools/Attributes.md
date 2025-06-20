# Master Server Toolkit - Attributes

## Description
Tools for working with attributes and annotations in the Unity editor that improve the inspector's user interface.

## HelpBox

Class for creating informational boxes in the Unity inspector with support for different message types.

### Main properties:

```csharp
public string Text { get; set; }     // Message text
public float Height { get; set; }    // Box height
public HelpBoxType Type { get; set; } // Box type (Info, Warning, Error)
```

### Constructors:

```csharp
// Create with explicit height
public HelpBox(string text, float height, HelpBoxType type = HelpBoxType.Info)

// Create with default height
public HelpBox(string text, HelpBoxType type = HelpBoxType.Info)

// Create an empty box
public HelpBox()
```

### Usage example:

```csharp
// Inside a ScriptableObject or MonoBehaviour
[SerializeField]
private HelpBox infoBox = new HelpBox("This is an info box", HelpBoxType.Info);

[SerializeField]
private HelpBox warningBox = new HelpBox("Warning! This is a warning", 60, HelpBoxType.Warning);

[SerializeField]
private HelpBox errorBox = new HelpBox("Error! This is an error message", HelpBoxType.Error);
```

### Integration in the editor:

```csharp
// In a custom editor
[CustomPropertyDrawer(typeof(HelpBox))]
public class HelpBoxDrawer : PropertyDrawer
{
    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label)
    {
        var textProp = property.FindPropertyRelative("Text");
        var typeProp = property.FindPropertyRelative("Type");
        var heightProp = property.FindPropertyRelative("Height");

        // Draw the help box depending on its type
        EditorGUI.HelpBox(
            new Rect(position.x, position.y, position.width, heightProp.floatValue),
            textProp.stringValue,
            (MessageType)typeProp.enumValueIndex
        );
    }

    public override float GetPropertyHeight(SerializedProperty property, GUIContent label)
    {
        return property.FindPropertyRelative("Height").floatValue;
    }
}
```

### Box types:

```csharp
public enum HelpBoxType
{
    Info,     // Informational message
    Warning,  // Warning message
    Error     // Error message
}
```

## Practical use

### Documenting settings:

```csharp
[Serializable]
public class ConnectionSettings
{
    [SerializeField]
    private HelpBox helpBox = new HelpBox(
        "Specify the server IP and port. Leave IP empty to use localhost.",
        HelpBoxType.Info
    );

    [SerializeField]
    private string serverIp;

    [SerializeField]
    private int serverPort = 5000;
}
```

### Component warnings:

```csharp
[RequireComponent(typeof(NetworkIdentity))]
public class PlayerController : MonoBehaviour
{
    [SerializeField]
    private HelpBox helpBox = new HelpBox(
        "This component requires NetworkIdentity to work correctly with the server",
        HelpBoxType.Warning
    );

    // Component implementation
}
```

### Error messages:

```csharp
[Serializable]
public class SecuritySettings
{
    [SerializeField]
    private HelpBox securityNote = new HelpBox(
        "WARNING! Never store sensitive data in source code!",
        80,
        HelpBoxType.Error
    );

    [SerializeField]
    private string certificatePath;

    [SerializeField]
    private string privateKey;
}
```
