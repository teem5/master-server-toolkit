# Master Server Toolkit - Validation System

## Overview

The validation system in Master Server Toolkit provides a set of tools for checking user input in forms. It supports required field checks, validation using regular expressions, comparing field values and much more.

## Key Components

### IValidatableComponent

Base interface for all validation components:

```csharp
public interface IValidatableComponent
{
    bool IsValid();
}
```

### ValidatableBaseComponent

Abstract base class for validation components:

```csharp
public abstract class ValidatableBaseComponent : MonoBehaviour, IValidatableComponent
{
    [Header("Base Settings")]
    [SerializeField] protected bool isRequired = false;
    [SerializeField, TextArea(2, 10)] protected string requiredErrorMessage;
    [SerializeField] protected Color invalidColor = Color.red;
    [SerializeField] protected Color normalColor = Color.white;
    
    public abstract bool IsValid();
    
    protected void SetInvalidColor();
    protected void SetNormalColor();
}
```

### ValidatableInputFieldComponent

Validation component for text input fields:

```csharp
public class ValidatableInputFieldComponent : ValidatableBaseComponent
{
    [Header("Text Field Components")]
    [SerializeField] private TMP_InputField currentInputField;
    [SerializeField] private TMP_InputField compareToInputField;
    [SerializeField, TextArea(2, 10)] protected string compareErrorMessage;
    
    [Header("Text Field RegExp Validation")]
    [SerializeField, TextArea(2, 10)] protected string regExpPattern;
    [SerializeField, TextArea(2, 10)] protected string regExpErrorMessage;
    
    // Checks whether the field is required, matches the regular expression and
    // whether the value equals compareToInputField (if specified)
    public override bool IsValid();
}
```

### ValidatableDropdownComponent

Validation component for dropdown lists:

```csharp
public class ValidatableDropdownComponent : ValidatableBaseComponent
{
    [Header("Dropdown Components")]
    [SerializeField] private TMP_Dropdown currentDropdown;
    
    [Header("Dropdown Settings")]
    [SerializeField] protected List<int> invalidValues = new List<int>();
    [SerializeField, TextArea(2, 10)] protected string invalidValueErrorMessage;
    
    // Checks that the selected value is not in the invalidValues list
    public override bool IsValid();
}
```

### ValidationFormComponent

Component for validating an entire form:

```csharp
public class ValidationFormComponent : MonoBehaviour, IUIViewComponent
{
    [Header("Settings")]
    [SerializeField] protected bool validateOnStart = false;
    [SerializeField] protected bool validateOnEnable = false;
    [SerializeField] protected bool validateBeforeSubmit = true;
    
    [Header("Components")]
    [SerializeField] protected Button submitButton;
    
    // Events
    public UnityEvent OnFormValidEvent;
    public UnityEvent OnFormInvalidEvent;
    public UnityEvent OnSubmitEvent;
    
    // Public methods
    public bool Validate();
    public void Submit();
}
```

## Regular Expressions for Validation

### Email

```
^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
```

### Password (minimum 8 characters, at least one letter and one number)

```
^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d]{8,}$
```

### Username (letters and digits only, 3-16 characters)

```
^[a-zA-Z0-9]{3,16}$
```

### URL

```
^(https?:\/\/)?([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$
```

### Phone number (international format)

```
^\+?[1-9]\d{1,14}$
```

## Usage Examples

### Registration Form

```csharp
public class RegistrationForm : UIView
{
    [SerializeField] private ValidationFormComponent validationForm;
    
    // Input fields with validation components
    [SerializeField] private TMP_InputField usernameField;
    [SerializeField] private ValidatableInputFieldComponent usernameValidator;
    
    [SerializeField] private TMP_InputField emailField;
    [SerializeField] private ValidatableInputFieldComponent emailValidator;
    
    [SerializeField] private TMP_InputField passwordField;
    [SerializeField] private ValidatableInputFieldComponent passwordValidator;
    
    [SerializeField] private TMP_InputField confirmPasswordField;
    [SerializeField] private ValidatableInputFieldComponent confirmPasswordValidator;
    
    protected override void Awake()
    {
        base.Awake();
        
        // Configure validators
        usernameValidator.isRequired = true;
        usernameValidator.requiredErrorMessage = "Username is required";
        usernameValidator.regExpPattern = "^[a-zA-Z0-9]{3,16}$";
        usernameValidator.regExpErrorMessage = "Name must be 3-16 characters long (letters and digits only)";
        
        emailValidator.isRequired = true;
        emailValidator.requiredErrorMessage = "Email is required";
        emailValidator.regExpPattern = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$";
        emailValidator.regExpErrorMessage = "Invalid email format";
        
        passwordValidator.isRequired = true;
        passwordValidator.requiredErrorMessage = "Password is required";
        passwordValidator.regExpPattern = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{8,}$";
        passwordValidator.regExpErrorMessage = "Password must have at least 8 characters, one letter and one number";
        
        confirmPasswordValidator.isRequired = true;
        confirmPasswordValidator.requiredErrorMessage = "Password confirmation is required";
        confirmPasswordValidator.compareToInputField = passwordField;
        confirmPasswordValidator.compareErrorMessage = "Passwords do not match";
        
        // Subscribe to form events
        validationForm.OnFormValidEvent.AddListener(OnFormValid);
        validationForm.OnFormInvalidEvent.AddListener(OnFormInvalid);
        validationForm.OnSubmitEvent.AddListener(OnSubmit);
    }
    
    private void OnFormValid()
    {
        // Form passed validation
        Debug.Log("Form is valid");
    }
    
    private void OnFormInvalid()
    {
        // Form failed validation
        Debug.Log("Form is invalid");
    }
    
    private void OnSubmit()
    {
        // Submit form data
        string username = usernameField.text;
        string email = emailField.text;
        string password = passwordField.text;
        
        // Register the user
        RegisterUser(username, email, password);
    }
    
    private void RegisterUser(string username, string email, string password)
    {
        // User registration logic
        // ...
    }
}
```

### Login Form Validation

```csharp
public class LoginForm : UIView
{
    [SerializeField] private ValidationFormComponent validationForm;
    [SerializeField] private TMP_InputField usernameField;
    [SerializeField] private ValidatableInputFieldComponent usernameValidator;
    [SerializeField] private TMP_InputField passwordField;
    [SerializeField] private ValidatableInputFieldComponent passwordValidator;
    [SerializeField] private Button loginButton;
    
    private void Start()
    {
        // Configure validation
        usernameValidator.isRequired = true;
        passwordValidator.isRequired = true;
        
        // Trigger validation when the login button is pressed
        loginButton.onClick.AddListener(() => {
            if (validationForm.Validate())
            {
                // If validation succeeds, sign in
                Mst.Auth.SignInWithCredentials(usernameField.text, passwordField.text, (isSuccess, error) => {
                    if (isSuccess)
                    {
                        Hide();
                        ViewsManager.Show("MainMenuView");
                    }
                    else
                    {
                        Debug.LogError($"Login error: {error}");
                    }
                });
            }
        });
    }
}
```

### Custom Validator

```csharp
// Example of creating a custom validator to check age
public class AgeValidatorComponent : ValidatableBaseComponent
{
    [SerializeField] private TMP_InputField ageField;
    [SerializeField] private int minAge = 18;
    [SerializeField] private int maxAge = 100;
    [SerializeField] private string invalidAgeMessage = "Age must be between {0} and {1}";
    
    public override bool IsValid()
    {
        if (!ageField.interactable)
            return true;
            
        if (isRequired && string.IsNullOrEmpty(ageField.text))
        {
            Debug.LogError(requiredErrorMessage);
            SetInvalidColor();
            return false;
        }
        
        if (!int.TryParse(ageField.text, out int age))
        {
            Debug.LogError("Age must be a number");
            SetInvalidColor();
            return false;
        }
        
        if (age < minAge || age > maxAge)
        {
            Debug.LogError(string.Format(invalidAgeMessage, minAge, maxAge));
            SetInvalidColor();
            return false;
        }
        
        SetNormalColor();
        return true;
    }
}
```

## Best Practices

1. **Error messages**: Provide clear and understandable error messages that explain how to fix the issue
2. **Visual feedback**: Combine text messages with visual indicators (color, icons)
3. **Real-time validation**: Consider validating fields as their values change
4. **Prevent submission**: Block the submit button if the form is invalid
5. **Custom validators**: Create specialized validators for specific cases
6. **Error grouping**: Collect and display all errors at once instead of one at a time

The validation system in Master Server Toolkit allows you to flexibly configure form checks, providing a good user experience and preventing incorrect data from being sent to the server.
