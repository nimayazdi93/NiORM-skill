# niorm-skill

Agent skill for [NiORM](https://github.com/nimayazdi93/NiORM) — a lightweight .NET ORM for SQL Server and MongoDB.

Teaches AI agents how to map entities, build `DataCore` services, run CRUD/query operations, handle security, logging, and errors with NiORM.

## Install

```bash
npx skills add nimayazdi93/NiORM-skill -g -y
```

Install for the current project only (omit `-g`):

```bash
npx skills add nimayazdi93/NiORM-skill -y
```

List skills in this repo:

```bash
npx skills add nimayazdi93/NiORM-skill --list
```

## What's included

| File | Purpose |
|------|---------|
| `SKILL.md` | Main agent instructions — quick start, CRUD, querying, security |
| `references/api-reference.md` | Full API, namespaces, LINQ limits |
| `references/examples.md` | Copy-paste patterns from NiORM.Test |

## NiORM package

This skill documents the [NiORM NuGet package](https://www.nuget.org/packages/NiORM):

```bash
dotnet add package NiORM
```

## License

MIT — same as [NiORM](https://github.com/nimayazdi93/NiORM).

## skills.sh

**Live:** [skills.sh/nimayazdi93/NiORM-skill](https://skills.sh/nimayazdi93/NiORM-skill)

The skill appears on skills.sh automatically after installs via the CLI. If search (`npx skills find niorm`) does not return it yet, file an indexing request at [vercel-labs/skills](https://github.com/vercel-labs/skills/issues/new) using [`.github/INDEX_REQUEST.md`](.github/INDEX_REQUEST.md) as the issue body.
