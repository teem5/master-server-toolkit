# Master Server Toolkit - System Views

## Overview

The Views system in Master Server Toolkit provides a powerful tool for organizing the user interface. It is based on the concept of individual views that can be shown, hidden and animated independently.

## Key Classes

### IUIView

Interface defining basic view operations:

```csharp
public interface IUIView
{
    string Id { get; }
    bool IsVisible { get; }
    RectTransform Rect { get; }
    bool IgnoreHideAll { get; set; }
    bool BlockInput { get; set; }
    bool UnlockCursor { get; set; }
    
    void Show(bool instantly = false);
    void Hide(bool instantly = false);
    void Toggle(bool instantly = false);
}
```

### UIView

Base view class implementing `IUIView`:

```csharp
[RequireComponent(typeof(Canvas), typeof(GraphicRaycaster), typeof(CanvasGroup))]
public class UIView : MonoBehaviour, IUIView
{
    [Header("Identity Settings")]
    [SerializeField] protected string id = "New View Id";
    [SerializeField] protected string title = "";

    [Header("Shared Settings")]
    [SerializeField] protected bool hideOnStart = true;
    [SerializeField] protected bool allwaysOnTop = false;
    [SerializeField] protected bool ignoreHideAll = false;
    [SerializeField] protected bool useRaycastBlock = true;
    [SerializeField] protected bool blockInput = false;
    [SerializeField] protected bool unlockCursor = false;
    
    // Events
    public UnityEvent OnShowEvent;
    public UnityEvent OnHideEvent;
    public UnityEvent OnShowFinishedEvent;
    public UnityEvent OnHideFinishedEvent;
    
    // Methods Show, Hide, Toggle
}
```

### UIViewPanel

`UIView` extension for creating panels:

```csharp
public class UIViewPanel : UIView
{
    // Additional functionality for panels
}
```

### UIViewSync

Component for synchronizing multiple views:

```csharp
public class UIViewSync : MonoBehaviour
{
    [SerializeField] protected string mainViewId;
    [SerializeField] protected List<string> syncViewIds = new List<string>();
    [SerializeField] protected bool hideOnStart = true;
    
    // Synchronizes the state of all attached views
}
```

### PopupView

Specialized view for dialog windows:

```csharp
public class PopupView : MonoBehaviour, IUIViewComponent
{
    public Button confirmButton;
    public Button declineButton;
    public TextMeshProUGUI titleText;
    public TextMeshProUGUI messageText;

    // Events for confirm/decline
    public UnityEvent OnConfirmEvent;
    public UnityEvent OnDeclineEvent;
}
```

## ViewsManager

Static class for managing all views:

```csharp
public static class ViewsManager
{
    // Check for input blocking and cursor unlocking
    public static bool AnyInputBlockViewVisible { get; }
    public static bool AnyCursorUnlockViewVisible { get; }

    // Registration and retrieval of views
    public static void Register(string viewId, IUIView view);
    public static void Unregister(string viewId);
    public static T GetView<T>(string viewId) where T : class, IUIView;

    // View management
    public static void Show(string viewId);
    public static void Hide(string viewId);
    public static void HideAllViews(bool instantly = false);
    public static void HideViewsByName(bool instantly = false, params string[] names);
}
```

## Components for Working with Views

### IUIViewComponent

Interface for components that work with views:

```csharp
public interface IUIViewComponent
{
    void OnOwnerShow(IUIView owner);
    void OnOwnerHide(IUIView owner);
}
```

### IUIViewInputHandler

Interface for input handlers:

```csharp
public interface IUIViewInputHandler : IUIViewComponent
{
    // Specific input handling logic
}
```

### UIViewKeyInputHandler

Keyboard input handler:

```csharp
public class UIViewKeyInputHandler : MonoBehaviour, IUIViewInputHandler
{
    [SerializeField] protected KeyCode toggleKey = KeyCode.Escape;
    [SerializeField] protected string toggleViewId = "";
    
    // Toggles the view when the specified key is pressed
}
```

### IUIViewTweener

Interface for view animators:

```csharp
public interface IUIViewTweener : MonoBehaviour
{
    IUIView UIView { get; set; }
    void PlayShow();
    void PlayHide();
    void OnFinished(UnityAction callback);
}
```

## UI Sound Components

Components for playing UI sounds:

```csharp
public class UIViewSound : MonoBehaviour, IUIViewComponent
{
    [SerializeField] protected AudioClip showClip;
    [SerializeField] protected AudioClip hideClip;
    
    // Plays sounds when the view appears/disappears
}

public class UIButtonSound : MonoBehaviour
{
    [SerializeField] protected AudioClip clickClip;
    [SerializeField] protected AudioClip hoverClip;
    
    // Adds sounds to buttons
}

public class UIToggleSound : MonoBehaviour 
{
    [SerializeField] protected AudioClip onClip;
    [SerializeField] protected AudioClip offClip;
    
    // Adds sounds to toggles
}
```

## Usage Examples

### Basic View Creation

```csharp
public class GameMenuView : UIView 
{
    [SerializeField] private Button playButton;
    [SerializeField] private Button optionsButton;
    [SerializeField] private Button quitButton;
    
    protected override void Awake()
    {
        base.Awake();
        
        // Subscribe to button events
        playButton.onClick.AddListener(OnPlayClick);
        optionsButton.onClick.AddListener(OnOptionsClick);
        quitButton.onClick.AddListener(OnQuitClick);
    }
    
    private void OnPlayClick()
    {
        // Hide the current view
        Hide();
        // Show the game selection view
        ViewsManager.Show("GameSelectionView");
    }
    
    private void OnOptionsClick()
    {
        ViewsManager.Show("OptionsView");
    }
    
    private void OnQuitClick()
    {
        Application.Quit();
    }
}
```

### Creating a Confirmation Dialog

```csharp
// Get the popup view
var popup = ViewsManager.GetView<PopupView>("ConfirmPopup");

// Set text
popup.SetTitle("Confirmation");
popup.SetMessage("Are you sure you want to quit?");

// Subscribe to events
popup.OnConfirmEvent.AddListener(() => {
    // Action on confirm
    Application.Quit();
});

popup.OnDeclineEvent.AddListener(() => {
    // Action on cancel
    popup.Hide();
});

// Show the popup
popup.Show();
```

### Working with Animated Views

```csharp
// Create an animated view
var view = gameObject.AddComponent<UIView>();
var tweener = gameObject.AddComponent<FadeTweener>(); // Implements IUIViewTweener

// Show with animation
view.Show(); // Animated
view.Show(true); // Instant

// Subscribe to animation finished event
view.OnShowFinishedEvent.AddListener(() => {
    Debug.Log("Show animation finished");
});
```
