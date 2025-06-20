# Master Server Toolkit - Authentication

## Description
Authentication module for registration, login, password recovery, and user management.

## AuthModule

Main class of the authentication module.

### Setup:
```csharp
[Header("Settings")]
[SerializeField] private int usernameMinChars = 4;
[SerializeField] private int usernameMaxChars = 12;
[SerializeField] private int userPasswordMinChars = 8;
[SerializeField] private bool emailConfirmRequired = true;

[Header("Guest Settings")]
[SerializeField] private bool allowGuestLogin = true;
[SerializeField] private string guestPrefix = "user_";

[Header("Security")]
[SerializeField] private string tokenSecret = "your-secret-key";
[SerializeField] private int tokenExpiresInDays = 7;
```

### Authentication types:

#### 1. By username and password:
```csharp
// Client sends
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_NAME, "playerName");
credentials.Set(MstDictKeys.USER_PASSWORD, "password123");
credentials.Set(MstDictKeys.USER_DEVICE_ID, "device123");
credentials.Set(MstDictKeys.USER_DEVICE_NAME, "iPhone 12");

// Encrypt and send
var encryptedData = Mst.Security.EncryptAES(credentials.ToBytes(), aesKey);
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, encryptedData);
```

#### 2. By email:
```csharp
// Send password to email
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_EMAIL, "player@game.com");
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

#### 3. By token:
```csharp
// Automatic login
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_AUTH_TOKEN, savedToken);
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

#### 4. Guest login:
```csharp
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_IS_GUEST, true);
credentials.Set(MstDictKeys.USER_DEVICE_ID, "device123");
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

## Registration
```csharp
// Create an account
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_NAME, "newPlayer");
credentials.Set(MstDictKeys.USER_PASSWORD, "securePassword");
credentials.Set(MstDictKeys.USER_EMAIL, "new@player.com");

// Encryption
var encrypted = Mst.Security.EncryptAES(credentials.ToBytes(), aesKey);
Mst.Client.Connection.SendMessage(MstOpCodes.SignUp, encrypted);
```

## User management

### Working with logged in users:
```csharp
// Get a user
var user = authModule.GetLoggedInUserByUsername("playerName");
var user = authModule.GetLoggedInUserById("userId");
var user = authModule.GetLoggedInUserByEmail("email@game.com");

// Check if online
bool isOnline = authModule.IsUserLoggedInByUsername("playerName");

// Get all online
var allOnline = authModule.LoggedInUsers;

// Sign out
authModule.SignOut("playerName");
authModule.SignOut(userExtension);
```

### Events:
```csharp
// Subscribe to events
authModule.OnUserLoggedInEvent += (user) => {
    Debug.Log($"User {user.Username} logged in");
};

authModule.OnUserLoggedOutEvent += (user) => {
    Debug.Log($"User {user.Username} logged out");
};

authModule.OnUserRegisteredEvent += (peer, account) => {
    Debug.Log($"New user registered: {account.Username}");
};
```

## Password recovery

### Requesting a code:
```csharp
// Client requests a code
Mst.Connection.SendMessage(MstOpCodes.GetPasswordResetCode, "email@game.com");

// Server sends the code by email
```

### Changing the password:
```csharp
var data = new MstProperties();
data.Set(MstDictKeys.RESET_PASSWORD_EMAIL, "email@game.com");
data.Set(MstDictKeys.RESET_PASSWORD_CODE, "123456");
data.Set(MstDictKeys.RESET_PASSWORD, "newPassword");

Mst.Connection.SendMessage(MstOpCodes.ChangePassword, data);
```

## Email confirmation
```csharp
// Request confirmation code
Mst.Connection.SendMessage(MstOpCodes.GetEmailConfirmationCode);

// Confirm email
Mst.Connection.SendMessage(MstOpCodes.ConfirmEmail, "confirmationCode");
```

## Additional properties
```csharp
// Bind extra data
var extraProps = new MstProperties();
extraProps.Set("playerLevel", 10);
extraProps.Set("gameClass", "warrior");

Mst.Connection.SendMessage(MstOpCodes.BindExtraProperties, extraProps);
```

## Database setup
```csharp
// Implementation of IAccountsDatabaseAccessor
public class MyDatabaseAccessor : IAccountsDatabaseAccessor
{
    public async Task<IAccountInfoData> GetAccountByUsernameAsync(string username)
    {
        // Get the account from DB
    }

    public async Task InsertAccountAsync(IAccountInfoData account)
    {
        // Save a new account
    }

    public async Task UpdateAccountAsync(IAccountInfoData account)
    {
        // Update the account
    }
}

// Create the factory
public class DatabaseFactory : DatabaseAccessorFactory
{
    public override void CreateAccessors()
    {
        var accessor = new MyDatabaseAccessor();
        Mst.Server.DbAccessors.AddAccessor(accessor);
    }
}
```

## Permission checks
```csharp
// Get peer info
Mst.Connection.SendMessage(MstOpCodes.GetAccountInfoByPeer, peerId);
Mst.Connection.SendMessage(MstOpCodes.GetAccountInfoByUsername, "playerName");

// Require minimum permission level
authModule.getPeerDataPermissionsLevel = 10;
```

## Validation customization
```csharp
// Override in a derived AuthModule
protected override bool IsUsernameValid(string username)
{
    if (!base.IsUsernameValid(username))
        return false;

    // Additional checks
    if (username.Contains("admin"))
        return false;

    return true;
}

protected override bool IsPasswordValid(string password)
{
    if (!base.IsPasswordValid(password))
        return false;

    // Complexity check
    bool hasUpper = password.Any(char.IsUpper);
    bool hasNumber = password.Any(char.IsDigit);

    return hasUpper && hasNumber;
}
```

## Email integration
```csharp
// Configure mailer component
[SerializeField] protected Mailer mailer;

// SMTP settings (in SmtpMailer)
smtpHost = "smtp.gmail.com";
smtpPort = 587;
smtpUsername = "yourgame@gmail.com";
smtpPassword = "app-password";
mailFrom = "noreply@yourgame.com";
```

## Command line arguments
```bash
# Token settings
./Server.exe -tokenSecret mysecret -tokenExpires 30

# Issuer/audience settings
./Server.exe -tokenIssuer http://mygame.com -tokenAudience http://mygame.com/users
```

## Best practices

1. **Always encrypt passwords before sending**
2. **Use tokens for auto login**
3. **Store sensitive data in a database**
4. **Configure email for password recovery**
5. **Check names for profanity with CensorModule**
6. **Limit login attempts**
7. **Use HTTPS in production**
