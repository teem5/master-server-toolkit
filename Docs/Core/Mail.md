# Master Server Toolkit - Mail

## Description
Email sending system via SMTP protocol with HTML template support.

## SmtpMailer

Main class for sending mail via SMTP.

### Setup:
```csharp
[Header("E-mail settings")]
public string smtpHost = "smtp.gmail.com";
public string smtpUsername = "your-email@gmail.com";
public string smtpPassword = "your-app-password";
public int smtpPort = 587;
public bool enableSsl = true;
public string mailFrom = "noreply@yourgame.com";
public string senderDisplayName = "Your Game Name";

[Header("E-mail template")]
[SerializeField] protected TextAsset emailBodyTemplate;
```

### Usage example:
```csharp
var mailer = GetComponent<SmtpMailer>();

// Send a simple email
await mailer.SendMailAsync("player@example.com", "Welcome!", "Thank you!");

// With an HTML template
await mailer.SendMailAsync(email, "Code: " + code, emailBody);
```

## Mail templates

### HTML template:
```html
<h1>#{MESSAGE_SUBJECT}</h1>
<div>#{MESSAGE_BODY}</div>
<footer>© #{MESSAGE_YEAR} Game Name</footer>
```

### Replacement tokens:
- `#{MESSAGE_SUBJECT}` - Subject
- `#{MESSAGE_BODY}` - Body text
- `#{MESSAGE_YEAR}` - Current year

## SMTP provider setup

### Gmail:
```csharp
smtpHost = "smtp.gmail.com";
smtpPort = 587;
enableSsl = true;
// Use an App Password
```

### SendGrid:
```csharp
smtpHost = "smtp.sendgrid.net";
smtpPort = 587;
smtpUsername = "apikey";
smtpPassword = "your-sendgrid-api-key";
```

## Integration examples

### Email confirmation:
```csharp
public async Task<bool> SendConfirmationCode(string email, string code)
{
    string subject = "Confirm your email";
    string body = $"Your code: <strong>{code}</strong>";
    
    return await mailer.SendMailAsync(email, subject, body);
}
```

### Password reset:
```csharp
public async Task<bool> SendPasswordReset(string email, string resetLink)
{
    string subject = "Password Reset";
    string body = $"<a href='{resetLink}'>Reset Password</a>";
    
    return await mailer.SendMailAsync(email, subject, body);
}
```

## Configuration via command line arguments

```bash
# At launch
./Server.exe -smtpHost smtp.gmail.com -smtpUsername game@gmail.com -smtpPassword app-password
```

## Arguments are parsed automatically:
```csharp
smtpHost = Mst.Args.AsString(Mst.Args.Names.SmtpHost, smtpHost);
smtpUsername = Mst.Args.AsString(Mst.Args.Names.SmtpUsername, smtpUsername);
smtpPassword = Mst.Args.AsString(Mst.Args.Names.SmtpPassword, smtpPassword);
```

## Important notes
- Does not work in WebGL
- Errors are logged asynchronously
- Use an App Password for Gmail
- Check firewall settings
- HTML templates are loaded from Resources

## Best practices
1. Store SMTP passwords securely
2. Use asynchronous calls
3. Validate email addresses
4. Implement sending limits
5. Test with different providers
