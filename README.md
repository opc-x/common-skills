# infra-skills

Personal Claude Code slash-command skills (`/zx-*`).

## Structure

```
commands/          ← skill .md files
```

## Usage

Skills are symlinked from project `.claude/commands/` directories so Claude Code can discover them.

```bash
# To link into a project:
ln -s $(pwd)/commands/<skill>.md /path/to/project/.claude/commands/<skill>.md
```

## Skills

| Command | Purpose |
|---------|---------|
| `/zx-api-test` | Test HTTP API endpoints |
| `/zx-app-control` | Android App automation via ADB |
| `/zx-code-review` | Pre-deploy safety check (compile/bean/callchain) |
| `/zx-db-query` | Readonly Oracle query |
| `/zx-log-search` | Kibana/Elasticsearch log search |
| `/zx-outlook` | Read/send email via Outlook/OWA |
| `/zx-ssh-log` | SSH log fetching via JumpServer |
| `/zx-weekly-report` | Weekly report update |
