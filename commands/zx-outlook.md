Read and send emails from trump@dc66.net via Outlook/OWA.

Usage: /outlook <action> [args]

Actions:
- list [N] [folder]     — List latest N emails (default 10) from inbox or folder
- read <id>             — Read full email by message ID
- search <keyword>      — Search emails by subject/body keyword
- send <to> <subject> <body>  — Send an email
- unread [N]            — List unread emails

## How it works
Tokens are captured from the running Edge debug session (port 9222) which has OWA loaded.
Token is cached at C:/Users/DEVTrump/.claude/owa_bearer_token.txt

## Step 1: Ensure a valid token

Run the token capture script if no token exists or it might be expired:

```powershell
powershell.exe -ExecutionPolicy Bypass -File "C:/Users/DEVTrump/.claude/get_owa_token.ps1"
```

Then load the token:

```powershell
$token = Get-Content "C:/Users/DEVTrump/.claude/owa_bearer_token.txt" -Raw
$headers = @{
    Authorization = "Bearer $token"
    "Content-Type" = "application/json"
    Accept = "application/json"
}
```

## Step 2: Execute the requested action

### List emails
```powershell
$r = Invoke-RestMethod "https://outlook.office.com/api/v2.0/me/messages?`$top=10&`$select=Subject,From,ReceivedDateTime,IsRead,Id&`$orderby=ReceivedDateTime desc" -Headers $headers
$r.value | ForEach-Object { "$($_.ReceivedDateTime) | $(if($_.IsRead){'R'}else{'U'}) | $($_.From.EmailAddress.Address) | $($_.Subject)" }
```

### List unread emails
```powershell
$r = Invoke-RestMethod "https://outlook.office.com/api/v2.0/me/messages?`$top=10&`$filter=IsRead eq false&`$select=Subject,From,ReceivedDateTime,Id&`$orderby=ReceivedDateTime desc" -Headers $headers
$r.value | ForEach-Object { "$($_.ReceivedDateTime) | $($_.From.EmailAddress.Address) | $($_.Subject)" }
```

### Read email by ID (get Id from list first)
```powershell
$r = Invoke-RestMethod "https://outlook.office.com/api/v2.0/me/messages/<MESSAGE_ID>" -Headers $headers
Write-Host "From: $($r.From.EmailAddress.Name) <$($r.From.EmailAddress.Address)>"
Write-Host "Subject: $($r.Subject)"
Write-Host "Date: $($r.ReceivedDateTime)"
Write-Host "---"
Write-Host $r.Body.Content
```

### Search emails
```powershell
$keyword = "your-search-term"
$r = Invoke-RestMethod "https://outlook.office.com/api/v2.0/me/messages?`$top=10&`$search=`"$keyword`"&`$select=Subject,From,ReceivedDateTime,IsRead,Id" -Headers $headers
$r.value | ForEach-Object { "$($_.ReceivedDateTime) | $($_.From.EmailAddress.Address) | $($_.Subject)" }
```

### Send email
```powershell
$body = @{
    Message = @{
        Subject = "Your Subject"
        Body = @{ ContentType = "Text"; Content = "Email body text here" }
        ToRecipients = @(@{ EmailAddress = @{ Address = "recipient@example.com" } })
    }
    SaveToSentItems = $true
} | ConvertTo-Json -Depth 10

Invoke-RestMethod "https://outlook.office.com/api/v2.0/me/sendmail" -Method POST -Headers $headers -Body $body
Write-Host "Email sent."
```

## Token refresh
If you get 401, re-run the token capture:
```powershell
powershell.exe -ExecutionPolicy Bypass -File "C:/Users/DEVTrump/.claude/get_owa_token.ps1"
```
