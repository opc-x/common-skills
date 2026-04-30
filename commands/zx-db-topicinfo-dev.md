Run a readonly Oracle query against the **topicinfo test database**.

- HOST: 119.13.81.81 PORT: 1521 SERVICE_NAME: pubdbdev
- Schema: `topic`
- Alias: `topicinfo-dev`

Execute:
```
/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -ExecutionPolicy Bypass -File "C:/Users/DEVTrump/zx-projects/plugins/oracle-readonly-triage/scripts/Invoke-ReadonlyOracleQuery.ps1" -Alias topicinfo-dev -Sql "<sql>"
```

Rules:
- SELECT only, always add `WHERE rownum <= N`
- Table prefix by product: U=UED, J=JXF, LH=乐玩, E=E68, RB=球盟
- Key tables: `TOPIC_<prefix>_INFO`, `TOPIC_<prefix>_STATUS`, `ANNOUNCEMENT_<prefix>`
- letterType: 0=维护通知, 1=优惠通知, 2=赛事通知, 3=维护公告, 4=优惠公告
