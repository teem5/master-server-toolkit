# Master Server Toolkit - Censor

## 설명
외설적 인 어휘 또는 모욕과 같은 원치 않는 콘텐츠를 필터링하기위한 검열 모듈.플레이어 간의 안전한 의사 소통을 제공합니다.

## CensorModule

검열 모듈의 주요 클래스.

### 설정 :
```csharp
[Header("Settings")]
[SerializeField] private TextAsset[] wordsLists;
[SerializeField, TextArea(5, 10)] private string matchPattern = @"\b{0}\b";
```

## H 초기화 :
```csharp
// Добавление модуля на сцену
var censorModule = gameObject.AddComponent<CensorModule>();

// Настройка списков запрещенных слов
public TextAsset[] wordsLists;
wordsLists = new TextAsset[] { forbiddenWordsAsset };
```

### 사전 형식 :
사전은 쉼표로 분리 된 단어가 금지 된 텍스트 파일입니다.
```
bad,words,list,here,separated,by,commas
```

## 코드에서 사용하십시오

## 텍스트 클래싱 :
```csharp
// Получение экземпляра модуля
var censorModule = Mst.Server.Modules.GetModule<CensorModule>();

// Проверка текста на наличие запрещенных слов
bool hasBadWord = censorModule.HasCensoredWord("Text to check");

if (hasBadWord)
{
    // Текст содержит запрещенные слова
    Debug.Log("Text contains censored words");
}
else
{
    // Текст безопасен
    Debug.Log("Text is clean");
}
```

### 채팅 통합 :
```csharp
// В обработчике сообщений чата
void HandleChatMessage(string message, IPeer sender)
{
    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();
    
    if (censorModule.HasCensoredWord(message))
    {
        // Отклонить сообщение
        sender.SendMessage(MstOpCodes.MessageRejected, "Message contains forbidden words");
        return;
    }
    
    // Отправить сообщение всем пользователям
    BroadcastMessage(message);
}
```

## 사용자의 클래스 이름 :
```csharp
// При регистрации в AuthModule
protected override bool IsUsernameValid(string username)
{
    if (!base.IsUsernameValid(username))
        return false;
    
    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();
    
    // Проверка имени на запрещенные слова
    if (censorModule.HasCensoredWord(username))
        return false;
    
    return true;
}
```

## Patterna Patterna 설정

`matchpattern '매개 변수는 금지 된 단어를 확인하는 방법을 결정합니다.

```csharp
// По умолчанию: Целые слова (используя границы слов)
matchPattern = @"\b{0}\b";

// Более строгая проверка: включая частичные совпадения
matchPattern = @"{0}";

// С разделителями: проверяет только слова, разделенные пробелами
matchPattern = @"(\s|^){0}(\s|$)";
```

## 모듈의 확장

Censormodule에 상속인을 만들어 기본 기능을 확장 할 수 있습니다.

```csharp
public class EnhancedCensorModule : CensorModule
{
    [SerializeField] private bool useRegexPatterns = false;
    [SerializeField] private TextAsset regexPatterns;
    
    private List<Regex> patterns = new List<Regex>();
    
    protected override void ParseTextFiles()
    {
        base.ParseTextFiles();
        
        // Загрузка дополнительных регулярных выражений
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
        // Проверка базовым методом
        if (base.HasCensoredWord(text))
            return true;
        
        // Проверка с помощью регулярных выражений
        foreach (var regex in patterns)
        {
            if (regex.IsMatch(text))
                return true;
        }
        
        return false;
    }
    
    // Дополнительный метод для получения замаскированного текста
    public string GetCensoredText(string text, char maskChar = '*')
    {
        string result = text;
        
        // Замена запрещенных слов маской
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

## 모범 사례

1. ** 정기적으로 사전을 업데이트 ** 금지 된 단어
2. ** 허위 작품을 피하려면 단어의 경계를 사용하십시오 (`\ b {0} \ b`)
3. ** 자동 게시 메시지를 위해 채팅 시스템과 통합 **
4. ** 텍스트 전환을 추가 ** 간단한 속임수를 우회하기 전에 확인하기 전에 (예 : "B@D w0rd")
5. ** 국제 프로젝트 용 다국어 지원 ** 켜기 **
6. ** 컨텍스트를 고려하십시오 ** 필터링 중에 - 어떤 단어는 한 컨텍스트에서는 정상이지만 다른 단어에서는 바람직하지 않습니다.
7. ** 다른 지역의 경우 현지 사전 **를 사용하십시오

## 통합의 예

### 자동 경고 시스템 :
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
                // Временный бан пользователя
                BanUser(username, TimeSpan.FromMinutes(10));
            }
        }
    }
}
```

### 사용자 콘텐츠와 통합 :
```csharp
// Проверка пользовательских названий
public bool ValidateUserContent(string text)
{
    var censorModule = Mst.Server.Modules.GetModule<CensorModule>();
    return !censorModule.HasCensoredWord(text);
}

// Использование при создании предметов, кланов и т.д.
public bool CreateClan(string clanName, string clanTag)
{
    if (!ValidateUserContent(clanName) || !ValidateUserContent(clanTag))
    {
        return false;
    }
    
    // Создание клана
    // ...
    
    return true;
}
```
