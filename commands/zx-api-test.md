Test an HTTP API endpoint and display the response.

## Usage
/api-test <url> [options]

- url: Full URL to request (e.g. https://api-topicinfo.nntitestserver.com/topinfo/addAnnouncement)
- options: method, body JSON, headers

## How to execute

Use PowerShell `Invoke-WebRequest` or `Invoke-RestMethod`:

### GET request
```powershell
powershell.exe -Command "
try {
  \$r = Invoke-RestMethod -Uri '<url>' -Method GET -TimeoutSec 15
  \$r | ConvertTo-Json -Depth 5
} catch {
  Write-Host \"Status: \$(\$_.Exception.Response.StatusCode.value__)\"
  Write-Host \$_.ErrorDetails.Message
}
"
```

### POST request with JSON body
```powershell
powershell.exe -Command "
\$body = '<json_body>' | ConvertFrom-Json | ConvertTo-Json
try {
  \$r = Invoke-RestMethod -Uri '<url>' -Method POST -Body \$body -ContentType 'application/json' -TimeoutSec 15
  \$r | ConvertTo-Json -Depth 5
} catch {
  Write-Host \"Status: \$(\$_.Exception.Response.StatusCode.value__)\"
  Write-Host \$_.ErrorDetails.Message
}
"
```

## Workflow

1. Parse the user's request to identify: URL, method (default GET), request body (if any), headers (if any)
2. Execute the request
3. Display: HTTP status, response body (formatted JSON if applicable), response time

## Common test patterns

```
# Health check
/api-test https://api.example.com/healthCheck

# POST with body
/api-test https://api.example.com/addAnnouncement POST {"flag":0,"productCode":"jxf","type":"MAINTAIN","title":"测试","content":"<p>test</p>","operator":"boots"}

# With custom header
/api-test https://api.example.com/endpoint GET headers={"X-Token":"abc123"}
```

## Notes
- If the URL uses HTTPS with self-signed cert, add `-SkipCertificateCheck` flag
- For large responses, show first 2000 chars and summarize the rest
- Always display HTTP status code prominently
- If response is `{}` (empty object), flag it as likely an error and suggest checking logs with /ssh-log or /log-search
