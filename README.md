# SPRCSystem

## Overview

SPRCSystem is a database-backed application. This repository currently holds
project configuration and documentation.

## Getting started

### Database

The application connects to a SQL Server instance. Connection details are kept
in `.claude/docs/database_connect.md`, which is intentionally excluded from
version control (see `.gitignore`). Ask a team member for a copy, or create your
own using the following template:

```
Server=<host>\<instance>;Database=<database_name>;User Id=<user>;Password=<password>;TrustServerCertificate=True;
```

## Repository layout

| Path                  | Purpose                                          |
| --------------------- | ----------------------------------------------- |
| `CLAUDE.md`                       | Guidance for Claude Code when working in this repo |
| `.claude/docs/database_connect.md` | Local SQL Server credentials (git-ignored)        |
| `.gitignore`                      | Files excluded from version control               |
