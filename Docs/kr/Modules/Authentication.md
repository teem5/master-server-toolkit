# Master Server Toolkit - Authentication

## 설명
등록, 시스템 입력, 암호 복원 및 사용자 관리를위한 인증 모듈.

## AuthModule

인증 모듈의 주요 클래스.

### 설정 :
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

### 인증 유형 :

#### 1. 이름과 비밀번호로 :
```csharp
// Клиент отправляет
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_NAME, "playerName");
credentials.Set(MstDictKeys.USER_PASSWORD, "password123");
credentials.Set(MstDictKeys.USER_DEVICE_ID, "device123");
credentials.Set(MstDictKeys.USER_DEVICE_NAME, "iPhone 12");

// Шифрование и отправка
var encryptedData = Mst.Security.EncryptAES(credentials.ToBytes(), aesKey);
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, encryptedData);
```

### 2. 이메일로 :
```csharp
// Отправляет пароль на email
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_EMAIL, "player@game.com");
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

#### 3. 토큰의 경우 :
```csharp
// Автоматический вход
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_AUTH_TOKEN, savedToken);
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

### 4. 게스트 입력 :
```csharp
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_IS_GUEST, true);
credentials.Set(MstDictKeys.USER_DEVICE_ID, "device123");
Mst.Client.Connection.SendMessage(MstOpCodes.SignIn, credentials);
```

## 등록

```csharp
// Создание аккаунта
var credentials = new MstProperties();
credentials.Set(MstDictKeys.USER_NAME, "newPlayer");
credentials.Set(MstDictKeys.USER_PASSWORD, "securePassword");
credentials.Set(MstDictKeys.USER_EMAIL, "new@player.com");

// Шифрование
var encrypted = Mst.Security.EncryptAES(credentials.ToBytes(), aesKey);
Mst.Client.Connection.SendMessage(MstOpCodes.SignUp, encrypted);
```

## 사용자 관리

### poglot 사용자와 함께 작업 :
```csharp
// Получить пользователя
var user = authModule.GetLoggedInUserByUsername("playerName");
var user = authModule.GetLoggedInUserById("userId");
var user = authModule.GetLoggedInUserByEmail("email@game.com");

// Проверить залогинен ли
bool isOnline = authModule.IsUserLoggedInByUsername("playerName");

// Получить всех онлайн
var allOnline = authModule.LoggedInUsers;

// Разлогинить
authModule.SignOut("playerName");
authModule.SignOut(userExtension);
```

### 이벤트 :
```csharp
// Подписка на события
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

## 비밀번호 복구

### 코드 요청 :
```csharp
// Клиент запрашивает код
Mst.Connection.SendMessage(MstOpCodes.GetPasswordResetCode, "email@game.com");

// Сервер отправляет код на email
```

## 비밀번호 변경 :
```csharp
var data = new MstProperties();
data.Set(MstDictKeys.RESET_PASSWORD_EMAIL, "email@game.com");
data.Set(MstDictKeys.RESET_PASSWORD_CODE, "123456");
data.Set(MstDictKeys.RESET_PASSWORD, "newPassword");

Mst.Connection.SendMessage(MstOpCodes.ChangePassword, data);
```

## 확인 이메일

```csharp
// Запрос кода подтверждения
Mst.Connection.SendMessage(MstOpCodes.GetEmailConfirmationCode);

// Подтверждение email
Mst.Connection.SendMessage(MstOpCodes.ConfirmEmail, "confirmationCode");
```

## 추가 속성

```csharp
// Привязка дополнительных данных
var extraProps = new MstProperties();
extraProps.Set("playerLevel", 10);
extraProps.Set("gameClass", "warrior");

Mst.Connection.SendMessage(MstOpCodes.BindExtraProperties, extraProps);
```

## 데이터베이스 설정

```csharp
// Реализация IAccountsDatabaseAccessor
public class MyDatabaseAccessor : IAccountsDatabaseAccessor
{
    public async Task<IAccountInfoData> GetAccountByUsernameAsync(string username)
    {
        // Получение аккаунта из БД
    }
    
    public async Task InsertAccountAsync(IAccountInfoData account)
    {
        // Сохранение нового аккаунта
    }
    
    public async Task UpdateAccountAsync(IAccountInfoData account)
    {
        // Обновление аккаунта
    }
}

// Создание фабрики
public class DatabaseFactory : DatabaseAccessorFactory
{
    public override void CreateAccessors()
    {
        var accessor = new MyDatabaseAccessor();
        Mst.Server.DbAccessors.AddAccessor(accessor);
    }
}
```

## 액세스 권한 검증

```csharp
// Получение информации о пире
Mst.Connection.SendMessage(MstOpCodes.GetAccountInfoByPeer, peerId);
Mst.Connection.SendMessage(MstOpCodes.GetAccountInfoByUsername, "playerName");

// Требует минимальный уровень прав
authModule.getPeerDataPermissionsLevel = 10;
```

## 유효성 검사의 Castomization

```csharp
// Переопределение в наследнике AuthModule
protected override bool IsUsernameValid(string username)
{
    if (!base.IsUsernameValid(username))
        return false;
    
    // Дополнительные проверки
    if (username.Contains("admin"))
        return false;
    
    return true;
}

protected override bool IsPasswordValid(string password)
{
    if (!base.IsPasswordValid(password))
        return false;
    
    // Проверка на сложность
    bool hasUpper = password.Any(char.IsUpper);
    bool hasNumber = password.Any(char.IsDigit);
    
    return hasUpper && hasNumber;
}
```

## 이메일과 통합

```csharp
// Настройка mailer компонента
[SerializeField] protected Mailer mailer;

// Настройка Smtp (в SmtpMailer)
smtpHost = "smtp.gmail.com";
smtpPort = 587;
smtpUsername = "yourgame@gmail.com";
smtpPassword = "app-password";
mailFrom = "noreply@yourgame.com";
```

## 명령 줄 인수

```bash
# Токен настройки
./Server.exe -tokenSecret mysecret -tokenExpires 30

# Настройка issuer/audience
./Server.exe -tokenIssuer http://mygame.com -tokenAudience http://mygame.com/users
```

## 모범 사례

1. ** 보내기 전에 항상 암호를 암호화하십시오 **
2. ** 저자를 위해 토큰 사용 **
3. ** 데이터베이스에 기밀 데이터 저장 **
4. ** 비밀번호 복원을 위해 이메일을 설정 **
5. ** 인구 조사를 사용하여 검열 이름을 확인하십시오 **
6. ** 입력 시도 수 제한 **
7. ** 생산에 HTTPS 사용 **
