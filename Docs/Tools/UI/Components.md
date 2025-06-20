# Master Server Toolkit - UI Components

## Overview

The UI components in Master Server Toolkit provide ready-made solutions for displaying data, configurable properties, progress bars and much more. They are created for easy integration with the views system and simplify building complex user interfaces.

## Key Components

### UIProperty

A universal component for displaying named properties with the ability to set an icon, value and progress bar:

```csharp
public class UIProperty : MonoBehaviour
{
    // Value display formats (F0 - F5)
    public enum UIPropertyValueFormat { F0, F1, F2, F3, F4, F5 }
    
    // Components
    [SerializeField] protected Image iconImage;
    [SerializeField] protected TextMeshProUGUI lableText;
    [SerializeField] protected TextMeshProUGUI valueText;
    [SerializeField] protected Image progressBar;
    [SerializeField] protected Color minColor = Color.red;
    [SerializeField] protected Color maxColor = Color.green;
    
    // Settings
    [SerializeField] protected string id = "propertyId";
    [SerializeField] protected float minValue = 0f;
    [SerializeField] protected float currentValue = 50f;
    [SerializeField] protected float maxValue = float.MaxValue;
    [SerializeField] protected float progressSpeed = 1f;
    [SerializeField] protected bool smoothValue = true;
    [SerializeField] protected string lable = "";
    [SerializeField] protected UIPropertyValueFormat formatValue = UIPropertyValueFormat.F1;
    [SerializeField] protected bool invertValue = false;
    
    // Methods for setting values
    public void SetMin(float value);
    public void SetMax(float value);
    public void SetValue(float value);
}
```

### UIProgressBar

A simplified version of UIProperty focused on displaying progress:

```csharp
public class UIProgressBar : UIProperty
{
    // Specialized version with specific logic for displaying progress
}
```

### UIProgressProperty

A component for binding a progress bar to a dynamically changing property:

```csharp
public class UIProgressProperty : MonoBehaviour
{
    [SerializeField] protected UIProperty property;
    [SerializeField] protected float updateInterval = 0.2f;
    [SerializeField] protected UnityEvent<float> onValueChangedEvent;
    
    // Bind to a value source and update automatically
}
```

### UILable

Component for working with text labels:

```csharp
public class UILable : MonoBehaviour
{
    [SerializeField] protected TextMeshProUGUI textComponent;
    [SerializeField] protected string text = "";
    
    public string Text { get; set; }
    public TextMeshProUGUI TextComponent { get; }
}
```

### UIMultiLable

Компонент для синхронизации нескольких текстовых меток:

```csharp
public class UIMultiLable : MonoBehaviour
{
    [SerializeField] protected string text = "";
    [SerializeField] protected List<TextMeshProUGUI> labels;
    
    // Synchronizes text for a group of labels
}
```

### DataTableLayoutGroup

Component for creating a table representation of data:

```csharp
public class DataTableLayoutGroup : MonoBehaviour
{
    [SerializeField] protected GameObject cellPrefab;
    [SerializeField] protected int rowsCount = 0;
    [SerializeField] protected int columnsCount = 0;
    [SerializeField] protected float spacing = 2f;
    [SerializeField] protected Vector2 cellSize = new Vector2(100f, 30f);
    
    // Methods for creating and filling the table
    public void SetValue(int row, int column, string value);
    public void Clear();
    public void Rebuild(int rows, int columns);
}
```

## Example Usage

### Displaying player statistics

```csharp
public class PlayerStatsView : UIView
{
    [SerializeField] private UIProperty healthProperty;
    [SerializeField] private UIProperty manaProperty;
    [SerializeField] private UIProperty experienceProperty;
    
    private Player player;
    
    public void Initialize(Player player)
    {
        this.player = player;
        
        // Set up properties
        healthProperty.SetMin(0);
        healthProperty.SetMax(player.maxHealth);
        healthProperty.SetValue(player.currentHealth);
        
        manaProperty.SetMin(0);
        manaProperty.SetMax(player.maxMana);
        manaProperty.SetValue(player.currentMana);
        
        experienceProperty.SetMin(0);
        experienceProperty.SetMax(player.experienceToNextLevel);
        experienceProperty.SetValue(player.currentExperience);
        
        // Subscribe to updates
        player.OnHealthChanged += (newValue) => healthProperty.SetValue(newValue);
        player.OnManaChanged += (newValue) => manaProperty.SetValue(newValue);
        player.OnExperienceChanged += (newValue) => experienceProperty.SetValue(newValue);
    }
}
```

### Creating a leaderboard

```csharp
public class LeaderboardView : UIView
{
    [SerializeField] private DataTableLayoutGroup dataTable;
    
    public void PopulateLeaderboard(List<PlayerScore> scores)
    {
        // Create a table with a size equal to the number of players and 3 columns
        dataTable.Rebuild(scores.Count, 3);
        
        // Fill the headers
        dataTable.SetValue(0, 0, "Ранг");
        dataTable.SetValue(0, 1, "Игрок");
        dataTable.SetValue(0, 2, "Счет");
        
        // Fill player data
        for (int i = 0; i < scores.Count; i++)
        {
            dataTable.SetValue(i + 1, 0, (i + 1).ToString());
            dataTable.SetValue(i + 1, 1, scores[i].PlayerName);
            dataTable.SetValue(i + 1, 2, scores[i].Score.ToString());
        }
    }
}
```

### Loading progress bar

```csharp
public class LoadingScreen : UIView
{
    [SerializeField] private UIProgressBar progressBar;
    [SerializeField] private UILable statusLabel;
    
    public void UpdateProgress(float progress, string status)
    {
        progressBar.SetValue(progress);
        statusLabel.Text = status;
    }
    
    // Usage:
    // loadingScreen.UpdateProgress(0.5f, "Loading assets...");
}
```

## Usage Recommendations

1. **Naming properties**: Assign unique IDs to properties for easy access
2. **Transition animation**: Use `smoothValue = true` for smooth value changes
3. **Formatting**: Choose an appropriate `formatValue` for readable numbers
4. **Inversion**: Use `invertValue` to invert progress bars (for example, for damage)
5. **Reactivity**: Subscribe to data change events to automatically update the UI

The UI components in Master Server Toolkit are intended to work both on their own and as part of more complex views. They provide basic functionality that can be extended and customized for the specific needs of your project.
