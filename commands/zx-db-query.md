Run a readonly Oracle query for troubleshooting.

Usage: /db-query <alias> <sql>
- alias: pubdbdev (test) | pubdb (production) | pngp (production) | topicinfo-dev (test topicinfo) | topicinfo (production topicinfo)
- sql: SELECT query to run

Execute the query using the Oracle readonly triage script:

```
powershell.exe -ExecutionPolicy Bypass -File "C:/Users/DEVTrump/zx-projects/plugins/oracle-readonly-triage/scripts/Invoke-ReadonlyOracleQuery.ps1" -Alias <alias> -Sql "<sql>"
```

If no alias is specified, default to `pubdbdev` (test environment).
If production alias is used (pubdb/pngp), confirm with the user before executing.
Always use schema prefix: `pubdb.` for pubdb alias, `pngdev.` for pngp alias.
Always add row limits (WHERE rownum <= N) for safety.

Refer to CLAUDE.md for key tables and SQL templates.
