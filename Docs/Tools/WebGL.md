# Master Server Toolkit - WebGL

## Description
Module for improving WebGL platform support, containing helper components and utilities for dealing with Unity WebGL web-specific features and limitations.

## WebGlTextMeshProInput

Component that improves TextMeshPro input fields in WebGL builds. It solves issues with virtual keyboards on mobile devices and web-specific input behavior.

### Main features

```csharp
[RequireComponent(typeof(TMP_InputField))]
public class WebGlTextMeshProInput : MonoBehaviour, IPointerClickHandler
{
    [Header("Settings"), SerializeField]
    private string title = "Input Field"; // Title of the prompt dialog

    // Handle input field click
    public void OnPointerClick(PointerEventData eventData)
    {
        // Call native JavaScript prompt
    }

    // Handle input confirmation
    public void OnPromptOk(string message)
    {
        GetComponent<TMP_InputField>().text = message;
    }

    // Handle input cancellation
    public void OnPromptCancel()
    {
        GetComponent<TMP_InputField>().text = "";
    }
}
```

### Usage

```csharp
// Add to an existing input field
TMP_InputField inputField = GetComponent<TMP_InputField>();
inputField.gameObject.AddComponent<WebGlTextMeshProInput>();

// Configure in the Unity editor
// 1. Add the WebGlTextMeshProInput component to the object with TMP_InputField
// 2. Set the title for the prompt dialog
```

### JavaScript integration

WebGlTextMeshProInput uses a jslib plugin to invoke native JavaScript code:

```javascript
// MstWebGL.jslib
mergeInto(LibraryManager.library, {
  MstPrompt: function(name, title, defaultValue) {
    var result = window.prompt(UTF8ToString(title), UTF8ToString(defaultValue));
    
    if (result !== null) {
      var gameObject = UTF8ToString(name);
      SendMessage(gameObject, "OnPromptOk", result);
    } else {
      var gameObject = UTF8ToString(name);
      SendMessage(gameObject, "OnPromptCancel");
    }
  }
});
```

## Mobile support

This component is especially useful when using WebGL on mobile devices:

1. **Solves the issue with virtual keyboards** on iOS and Android
2. **Provides correct input** on devices with different screen formats
3. **Supports multiline input fields** via the native interface

### Example for multiline input

```csharp
public class WebGlMultilineInput : MonoBehaviour
{
    [SerializeField] private TMP_InputField inputField;
    [SerializeField] private Button editButton;
    [SerializeField] private string dialogTitle = "Enter text";
    
    private void Start()
    {
        // Add a handler for the edit button
        editButton.onClick.AddListener(OnEditButtonClick);
    }
    
    private void OnEditButtonClick()
    {
        // Show the prompt dialog
        WebGLInput.ShowPrompt(dialogTitle, inputField.text, OnPromptComplete);
    }
    
    private void OnPromptComplete(string result, bool isCancelled)
    {
        if (!isCancelled)
        {
            inputField.text = result;
            // Additional handling of the entered text
        }
    }
}
```

## Integration with other web features

### Clipboard interaction

```csharp
public class WebGlClipboard : MonoBehaviour
{
    // Copy text to the clipboard
    public void CopyToClipboard(string text)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        WebGLCopyToClipboard(text);
#else
        GUIUtility.systemCopyBuffer = text;
#endif
    }
    
    // Paste text from the clipboard
    public void PasteFromClipboard(TMP_InputField inputField)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        WebGLRequestClipboardContent(gameObject.name);
#else
        inputField.text = GUIUtility.systemCopyBuffer;
#endif
    }
    
    // Handler for receiving clipboard content
    public void OnClipboardContent(string content)
    {
        // Use the received content
        FindObjectOfType<TMP_InputField>().text = content;
    }
    
#if UNITY_WEBGL && !UNITY_EDITOR
    [DllImport("__Internal")]
    private static extern void WebGLCopyToClipboard(string text);
    
    [DllImport("__Internal")]
    private static extern void WebGLRequestClipboardContent(string gameObjectName);
#endif
}
```

### Adapting to screen orientation

```csharp
public class WebGlOrientationHandler : MonoBehaviour
{
    [SerializeField] private CanvasScaler canvasScaler;
    [SerializeField] private float portraitMatchWidthOrHeight = 0.5f;
    [SerializeField] private float landscapeMatchWidthOrHeight = 0;
    
    private void Start()
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        WebGLAddOrientationChangeListener(gameObject.name);
        UpdateCanvasScaling();
#endif
    }
    
    // Called from JavaScript when orientation changes
    public void OnOrientationChanged()
    {
        UpdateCanvasScaling();
    }
    
    private void UpdateCanvasScaling()
    {
        bool isPortrait = Screen.height > Screen.width;
        canvasScaler.matchWidthOrHeight = isPortrait ? portraitMatchWidthOrHeight : landscapeMatchWidthOrHeight;
    }
    
#if UNITY_WEBGL && !UNITY_EDITOR
    [DllImport("__Internal")]
    private static extern void WebGLAddOrientationChangeListener(string gameObjectName);
#endif
}
```

## Localization handling

The WebGlTextMeshProInput component also integrates with the Master Server Toolkit localization system for proper dialog titles:

```csharp
// Using localization for the title
private string title = "Input Field";

public void OnPointerClick(PointerEventData eventData)
{
#if UNITY_WEBGL && !UNITY_EDITOR && !UNITY_STANDALONE
    var input = GetComponent<TMP_InputField>();
    MstPrompt(name, Mst.Localization[title], input.text);
#endif
}
```

## Practical examples

### WebGL registration form

```csharp
public class WebGlRegistrationForm : MonoBehaviour
{
    [SerializeField] private TMP_InputField usernameField;
    [SerializeField] private TMP_InputField emailField;
    [SerializeField] private TMP_InputField passwordField;
    [SerializeField] private Button submitButton;
    
    private void Start()
    {
        // Add components to improve WebGL input
        usernameField.gameObject.AddComponent<WebGlTextMeshProInput>().title = "Enter Username";
        emailField.gameObject.AddComponent<WebGlTextMeshProInput>().title = "Enter Email";
        passwordField.gameObject.AddComponent<WebGlTextMeshProInput>().title = "Enter Password";
        
        // Configure the submit button
        submitButton.onClick.AddListener(OnSubmitButtonClick);
    }
    
    private void OnSubmitButtonClick()
    {
        // Validate input and submit the form
        if (string.IsNullOrEmpty(usernameField.text) || 
            string.IsNullOrEmpty(emailField.text) || 
            string.IsNullOrEmpty(passwordField.text))
        {
            ShowError("All fields are required");
            return;
        }
        
        // Send data to the server
        SendRegistrationData(usernameField.text, emailField.text, passwordField.text);
    }
    
    private void SendRegistrationData(string username, string email, string password)
    {
        // Data sending logic
    }
    
    private void ShowError(string message)
    {
        // Display an error
    }
}
```

### Saving and loading data

```csharp
public class WebGlStorageHandler : MonoBehaviour
{
    // Save data to LocalStorage
    public void SaveData(string key, string value)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        WebGLSaveToLocalStorage(key, value);
#else
        PlayerPrefs.SetString(key, value);
        PlayerPrefs.Save();
#endif
    }
    
    // Load data from LocalStorage
    public string LoadData(string key, string defaultValue = "")
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        return WebGLLoadFromLocalStorage(key, defaultValue);
#else
        return PlayerPrefs.GetString(key, defaultValue);
#endif
    }
    
    // Delete data
    public void DeleteData(string key)
    {
#if UNITY_WEBGL && !UNITY_EDITOR
        WebGLRemoveFromLocalStorage(key);
#else
        PlayerPrefs.DeleteKey(key);
        PlayerPrefs.Save();
#endif
    }
    
#if UNITY_WEBGL && !UNITY_EDITOR
    [DllImport("__Internal")]
    private static extern void WebGLSaveToLocalStorage(string key, string value);
    
    [DllImport("__Internal")]
    private static extern string WebGLLoadFromLocalStorage(string key, string defaultValue);
    
    [DllImport("__Internal")]
    private static extern void WebGLRemoveFromLocalStorage(string key);
#endif
}
```

## Best practices

1. **Use conditional compilation** for platform-specific code
```csharp
#if UNITY_WEBGL && !UNITY_EDITOR
    // WebGL specific code
#else
    // Code for other platforms
#endif
```

2. **Test on real mobile devices** to verify virtual keyboards

3. **Consider WebGL limitations**:
   - Lack of multithreading
   - Browser security restrictions
   - Input problems on mobile devices

4. **Provide alternative input methods** for complex forms

5. **Integrate with browser JavaScript APIs** to extend functionality:
   - LocalStorage for storing data
   - Clipboard API for working with the clipboard
   - Screen API for working with screen orientation

6. **Handle browser window focus loss** for proper application functioning
