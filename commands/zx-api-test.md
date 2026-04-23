Test an HTTP API endpoint and display the response.

## Usage
/api-test <url> [options]

- url: Full URL to request (e.g. https://api-topicinfo.nntitestserver.com/topinfo/addAnnouncement)
- options: method, body JSON, headers

## How to execute

Use `curl` (WSL2/Linux environment — do NOT use powershell.exe):

### GET request
```bash
curl -sk '<url>' -w "\nHTTP_STATUS:%{http_code}" --connect-timeout 15 | python3 -c "
import sys, json
raw = sys.stdin.read()
body, status = raw.rsplit('HTTP_STATUS:', 1)
print(f'HTTP {status.strip()}')
try: print(json.dumps(json.loads(body), indent=2, ensure_ascii=False))
except: print(body[:2000])
"
```

### POST request with JSON body
```bash
curl -sk -X POST '<url>' \
  -H 'Content-Type: application/json' \
  -d '<json_body>' \
  -w "\nHTTP_STATUS:%{http_code}" --connect-timeout 15 | python3 -c "
import sys, json
raw = sys.stdin.read()
body, status = raw.rsplit('HTTP_STATUS:', 1)
print(f'HTTP {status.strip()}')
try: print(json.dumps(json.loads(body), indent=2, ensure_ascii=False))
except: print(body[:2000])
"
```

### POST request with form body (application/x-www-form-urlencoded)
```bash
curl -sk -X POST '<url>' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data '<key=value&key2=value2>' \
  -w "\nHTTP_STATUS:%{http_code}" --connect-timeout 15 | python3 -c "
import sys, json
raw = sys.stdin.read()
body, status = raw.rsplit('HTTP_STATUS:', 1)
print(f'HTTP {status.strip()}')
try: print(json.dumps(json.loads(body), indent=2, ensure_ascii=False))
except: print(body[:2000])
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
- Always use `curl -sk` for HTTPS with self-signed certs (`-k` skips cert check, `-s` silent mode)
- For large responses, show first 2000 chars and summarize the rest
- Always display HTTP status code prominently
- If response is `{}` (empty object), flag it as likely an error and suggest checking logs with /ssh-log or /log-search
- **AES-encrypted form body**: if requestData is already URL-encoded (from `urllib.parse.quote`), use `--data` NOT `--data-urlencode`. Using `--data-urlencode` on a pre-encoded value causes double-encoding → backend SQL injection interceptor receives null → NPE
