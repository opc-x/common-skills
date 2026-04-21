Search application logs via Kibana/Elasticsearch.

Usage: /log-search <preset> <value>
- preset: trace | error | gateway
- value: trace ID, error keyword, or API path

Execute using the Kibana preset script:

```
powershell.exe -ExecutionPolicy Bypass -File "C:/Users/DEVTrump/plugins/easydevops-kibana/scripts/invoke-kibana-preset.ps1" -Preset <preset> -Value "<value>"
```

Presets:
- trace: Find logs by trace/request ID (last 2h, api-tomcat-*)
- error: Hunt exceptions by keyword (last 30m, api-tomcat-*)
- gateway: Correlate by URI/IP/header (last 30m, gateway_requests)

Optional overrides: -Index "pattern-*" -TimeGte "now-1h" -Size 50

For custom DSL queries, use invoke-kibana-dsl.ps1 directly.
Refer to CLAUDE.md for full usage details.
